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

--- abstract

Numerous protocol drafts have been proposed that modify the TLS 1.3 handshake by injecting key material into the key schedule.
For this to not introduce vulnerabilities this needs to be done with care.
There needs to be agreement by both sides on the context around the key material, and there needs to be assurance that the modifications don't reduce the security of the handshake.
This draft describes how to inject key material safely whilst ensuring agreement on all the relevant context.

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
We base this design off the work of Bhargavan et al.

##Channel Bindings
The exporter keys interface provides channel bindings that other protocols can agree on, allowing them to build off the security of TLS.
We define a channel binding as a public value that uniquely identifies a run, such that if all parties involved in a protocol run agree on a channel binding they agree on all the parameters of that run.
By agreeing on the channel binding of one protocol in another, it is possible to reason about the security of each protocol in terms of the other.
Thus by agreeing on the exporter key of a TLS session in a run of a different protocol (and proving possession of the relevant secrets) an actor can leverage the authentication guarantees of TLS in the latter protocol.

TLS 1.3, however, does not provide an equivalent interface for importing keys, which makes it difficult to build off the security of other protocols to strengthen TLS.
The most similar interface is the out-of-band pre-shared key interface, however it is possible to use this interface insecurely.
This drafts describes a change to the use of the OOB PSK interface to make it easier to use secondary protocols to improve the security guarantees of TLS.

Various drafts have been proposed that extend the TLS handshake to provide various different extra guarantees.
The purpose of this draft is to provide a structured way of doing so, using the OOB PSK interface, with clear security goals.

When using TLS's out-of-band pre-shared key (OOB PSK) interface keys a pair (PSK_ID, PSK) are included in the TLS handshake.
We can consider OOB keys as having been established through some protocol, whether this is a different handshake protocol, a phone call, or manual configuration.
Together this OOB protocol and the TLS handshake can be considered a compound authentication protocol.

##Compound Authentication
A compound protocol is simply a number of protocols run in sequence, with some of the outputs of each protocol being inputs into the next, that are intended to give different guarantees than running the protocols individually.
A compound authentication protocol is simply a compound protocol with multiple authentication layers, that intends to achieve some stronger authentication guarantee, specifically compound authentication.
Compound authentication is the property that, given a run of a compound authentication protocol, if any layer of the protocol's authentication was succesful then all layers are authentic.
For a compound authentication protocol to be secure, i.e. to provide compound authentication, both parties must agree on a channel binding for each layer, and further they must agree on the complete list of channel bindings.
This assumes the Bhargavan hypothesis.
This property can be used to guarantee that the injected key material is authentic, and further, in the case that there exists a flaw in TLS, to ensure the authenticity of TLS session, despite the flaws.

##Contributive Channel Bindings
The Bhargavan hypothesis states that if every layer of a compound protocol agrees on what it calls contributive channel bindings, then they achieve compound authentication.
For channel bindings to be contributive they need to commit to various pieces of information.
Namely: \[\[insert list\]\]

This gives us a strong starting point for redesigning the OOB PSK interface.

#OOB PSKs
The heart of our modification is to require the PSK_ID to be a contributive channel binding.
We thus provide an interface with the requisite fields as follows.

\[\[Function\]\]

We also modify the key to bind the key and the channel binding.
\[\[Function\]\]

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
