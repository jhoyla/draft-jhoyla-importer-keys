---
docname: draft-jhoyla-importer-keys-latest
title: The Importer keys draft
date: {DATE}

workgroup: jhoyla
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
-
    ins: J. Hoyland
    name: Jonathan Hoyland
    org: Cloudflare
    email: jhoyland@cloudflare.com
    role: editor
-
    ins: C. Wood
    name: Christopher A. Wood
    org: Apple Inc.
    email: cawood@apple.com
    role: editor

normative:
    RFC5056:
    RFC8446:

informative:
    EA: I-D.ietf-tls-exported-authenticator
    CCB: DOI.10.14722/ndss.2015.23277
    Selfie:
        title: "Selfie: reflections on TLS 1.3 with PSK"
        author:
            -
                ins: N. Drucker
                name: Nir Drucker
            -
                ins: S. Gueron
                name: Shay Gueron
        date: 2019
        target: https://eprint.iacr.org/2019/347.pdf

--- abstract

Numerous protocol drafts have been proposed that modify the TLS 1.3 handshake by injecting key material into the key schedule.
For this to not introduce vulnerabilities this needs to be done with care.
There needs to be agreement by both sides on the context around the key material, and there needs to be assurance that the modifications don't reduce the security of the handshake.
This draft provides guidance on how to inject key material safely whilst ensuring agreement on all the relevant context.

--- middle

#Introduction

Numerous protocol drafts have been proposed that modify the security guarantees of TLS 1.3 in various ways.
However modifying the protocol, especially modifying the key schedule could invalidate the formal analyses that have been performed on it.
This makes it difficult to work out the security properties of the modified protocol.

TLS 1.3 currently provides a fixed interface for producing channel bindings, allowing other protocols to be run over the top of TLS, and to benefit from its security guarantees, producing a stronger combined protocol.
This type of modification is well understood, and can be shown to not weaken the guarantees of the underlying TLS session.
TLS however does not currently provide a simple way of running TLS over the top of other protocols and allowing TLS to benefit from their guarantees.
The closest mechanism to this is the out-of-band (OOB) pre-shared key (PSK) mechanism.
The OOB PSK mechanism takes a pair of (PSK_ID, PSK), and uses them to establish a TLS connection.
However this mechanism does not use certificates, relying on the OOB PSK to provide implicit authentication.
A recently published paper on TLS 1.3 points out a way the OOB PSK mechanism can be abused.
If more than two agents have access to the OOB PSK then the lack of explicit authentication in the TLS handshake can be used to confuse the agents about who is on the other end of the connection.

In this draft we propose a modification to the OOB PSK mechanism to mitigate this issue, and further to provide a uniform interface similar to the exporter keys interface for layering TLS on top of other protocols.
By relying on the literature on channel bindings, we offer a mechanism with explicit security goals.
Protocol drafts that use this mechanism will thus find it easier to articulate what the security goals they want to achieve are.

A further benefit of producing a uniform mechanism for binding TLS sessions to other protocols is that it becomes possible to reason formally about whether any binding can reduce the security of a TLS session.
We base this design off the work of Bhargavan et al. {{CCB}}

##Channel Bindings
A channel binding is a value produced at the end of a handshake that can be used to bind the channels established by different handshakes together.
This allows one to run a sequence of handshake protocols, delegating different security properties to different handshakes, and achieve a single security context.
For example, Exported Authenticators, {{EA}}, are used to add a secondary certificate to a TLS 1.3 connection.
An EA includes a channel binding from a TLS handshake generated through the TLS exporter keys interface.
This uniquely binds an EA to a particular TLS channel.
Cryptographically agreeing on the TLS channel and the EA creates a channel that is authenticated by both the original certificate and the secondary certificate.

TLS 1.3's exporter keys interface provides channel bindings that other protocols can agree on, allowing them to build off the security of TLS.
{{RFC5056}} defines a unique channel binding, which we refer to simply as a channel binding,  as a public value that uniquely identifies a run, such that if all parties involved in a protocol run agree on a channel binding they agree on all the parameters of that run.

By agreeing on the channel binding of one protocol handshake in another, it is possible to reason about the security of each handshake in terms of the other.
Thus by agreeing on the exporter key of a TLS session in a run of a different protocol (and proving possession of the relevant secrets) an actor can leverage the authentication guarantees of TLS in the latter protocol.

TLS 1.3, however, does not provide an equivalent interface for importing keys, which makes it difficult to build off the security of other protocols to strengthen TLS.
The most similar interface is the out-of-band pre-shared key interface, however it is possible to use this interface insecurely.
This drafts describes how to make use of the OOB PSK interface to make it easier to use secondary protocols to improve the security guarantees of TLS.

Various drafts have been proposed that modify the TLS handshake to provide various different extra guarantees.
The purpose of this draft is to provide a structured way of increasing the security of the TLS handshake, using the OOB PSK interface, with clear security goals.

The key idea is to ensure that inputs to the OOB PSK interface constitute channel bindings.

##Compound Authentication
When a number of handshakes are bound together we can use channel bindings to produce a single channel with the combined authentication properties of each channel.
We call a sequence of handshakes bound in this way a compound authentication protocol, and refer to each individual handshake as a protocol layer.
Compound authentication is the property that, given a run of a compound authentication protocol, if any layer of the protocol successfully authenticates, then all layers are authentic.

For example when a secondary certificate is added to a TLS channel using an EA this can be described as a compound authentication protocol with two layers, the TLS layer, and the EA layer.
Compound authentication says that if the TLS layer was established correctly, then the EA was created authentically, and also that if the EA was created authentically then the TLS layer was established correctly.

To apply this logic to imported keys we look to the OOB PSK interface.
When using TLS's out-of-band pre-shared key (OOB PSK) interface a pair (PSK_ID, PSK) are included in the TLS handshake.
We can consider OOB keys as having been established through some protocol, whether this is a different handshake protocol, a phone call, or manual configuration.
Together this OOB protocol and the TLS handshake can be considered a compound authentication protocol.

Compound authentication can be used to guarantee that the injected key material is authentic, and further, in the case that there exists a flaw in TLS, to ensure the authenticity of TLS session, despite the flaws.

##Contributive Channel Bindings
For a compound authentication protocol to be secure, i.e. to provide compound authentication, both parties must agree on a channel binding for each layer, and further they must agree on the complete list of channel bindings.

The Bhargavan hypothesis states that if every layer of a compound protocol agrees on what it refers to as contributive channel bindings, then they achieve compound authentication.
For channel bindings to be contributive they need to commit to various pieces of information, commiting to a special "⊥" value if they aren't defined.
Namely:

1. c_i : The initiators (public) credential
2. c_r : The responders (public) credential
3. sid : A global session identifier
4. cb : A channel binding for the current instance\*
5. cb_in : A channel binding for the previous protocol layer (if any)\*\*
6. secrets : Any session specific secrets established in the protocol run

\* This simply means that a contributive channel binding must also meet the definition of a regular channel binding, i.e. it must commit to enough parameters of the protocol that it is computationally infeasible to find two protocol runs with different parameters that produce the same channel binding, for example a hash over the protocol transcript and all session secrets.

\*\* This requires a chain of protocols to have a strict ordering, so even if someone wishes to inject multiple different pieces of key material into the OOB PSK interface it remains non-ambiguous.

This gives us a strong starting point for describing a safe usage of the OOB PSK interface.

#OOB PSKs
The key requirement is for the PSK_ID to be a contributive channel binding.
We thus provide an interface with the requisite fields as follows.
For a given raw (PSK_ID, PSK) pair the constructed (PSK_ID, PSK) pair should be constructed as follows:

    PSK_ID = HKDF-Expand(raw_PSK, c_i, c_r, sid, cb, cb_in, secrets, raw_PSK_ID)

\[\[This is a bit weird, because if c_i and c_r are both set to ⊥ then the Selfie attack is actually the expected behaviour\]\]

\[\[cb needs to include a globally unique protocol label for every different draft. Some kind of IANA registry maybe?\]\]
We also modify the key to bind the key and the channel binding.

     PSK = HKDF-Expand(raw_PSK, PSK_ID)

\[\[Does this make sense/achieve anything?\]\]

#Examples
\[\[ToDo: Show how to apply the modified OOB PSK to all relevant drafts\]\]

#Other changes
\[\[ToDo: Consider optionally including the certificate in OOB PSK handshakes to achieve stronger auth guarantees in terms of compound auth\]\]

#Security Considerations

##Seperation of protocols from the TLS handshake
By using an HKDF on both the PSK and PSK_ID we isolate the TLS handshake from the prior protocols, reducing the possibility of the modifications weakening the TLS handshake.

##Compound Authentication
\[\[How does the lack of explicit auth in the OOB PSK handshake affect compound auth properties\]\]

#IANA Considerations
\[\[Protocol labels need to be globally unique. IANA registry?\]\]
