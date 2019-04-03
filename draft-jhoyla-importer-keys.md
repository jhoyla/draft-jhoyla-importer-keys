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

Numerous protocol drafts have been proposed that inject key material into the TLS 1.3 handshake.
For this to not introduce vulnerabilities this needs to be done with agreement by both sides on the context around the key material.
In particular they need to agree on various parameters, such as the method used to produce the extra key material, the role played by each actor, and the key material itself.
This draft describes how to inject key material whilst ensuring agreement on all the relevant context.

--- middle

#Introduction
TLS 1.3 defines an exporter keys interface to allow other protocols to build on top of it.
The exporter keys interface provides channel bindings that other protocols can agree on, allowing them to build off the security of TLS.
TLS 1.3, however, does not provide an equivalent interface for importing keys, which makes it difficult to build off the security of other protocols to strengthen TLS.
The most similar interface is the out-of-band pre-shared key interface, however it is possible to use this interface insecurely.
This drafts describes a change to the use of the OOB PSK interface to make it easier to use secondary protocols to improve the security guarantees of TLS.

Various drafts have been proposed that extend the TLS handshake to provide various different extra guarantees.
The purpose of this draft is to provide a structured way of doing so, using the OOB PSK interface, with clear guarantees.

When using TLS's out-of-band pre-shared key (OOB PSK) interface keys a pair (PSK_ID, PSK) are included in the TLS handshake.
We can consider OOB keys as having been established through some protocol, whether this is a different handshake protocol, a phone call, or manual configuration.
Together this OOB protocol and the TLS handshake can be considered a compound authentication protocol.

##Compound Authentication
A compound protocol is simply a number of protocols run in sequence, with some of the outputs of each protocol being inputs into the next, that are intended to give different guarantees than running the protocols individually.
A compound authentication protocol is simply a compound protocol with multiple authentication layers, that intends to achieve some stronger authentication guarantee, specifically compound authentication.
Compound authentication is the property that, given a run of a compound authentication protocol, if any layer of the protocol's authentication was succesful then all layers are authentic.
For a compound authentication protocol to be secure, i.e. to provide compound authentication, both parties must agree on a channel binding for each layer, and further they must agree on the complete list of channel bindings.
This assumes the Bhargavan / Delignat-Lavaut hypothesis.
This property can be used to guarantee that the injected key material is authentic, and further, in the case that there exists a flaw in TLS, to ensure the authenticity of TLS session, despite the flaws.
