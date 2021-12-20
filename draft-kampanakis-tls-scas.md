---
title: "Suppressing CA Certificates in TLS"
abbrev: Suppress CAs
docname: draft-kampanakis-tls-scas 
category: std
ipr: trust200902
area: Transport
workgroup: TLS

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping]

author:
  -
    ins: M. Thomson
    name: Martin Thomson
    org: Mozilla
    email: mt@lowentropy.net

  -
    ins: P. Kampanakis
    name: Panos Kampanakis
    org: AWS
    email: kpanos@amazon.com

  -
    ins: C. Bytheway 
    name: Cameron Bytheway
    org: AWS
    email: bythewc@amazon.com
    
  -
    ins: B.E. Westerbaan
    name: Bas Westerbaan
    org: Cloudflare
    email: bas@cloudflare.com

normative:


informative:



--- abstract

A TLS client that has access to the complete set of published intermediate
certificates can inform servers of this fact so that the server can avoid
sending intermediates, reducing the size of the TLS handshake.


--- middle

# Introduction

In some uses of public key infrastructure (PKI) intermediate certificates are
used to sign end-entity certificates.  In the web PKI, clients require that
certificate authorities disclose all intermediate certificates that they
create.  Though the set of intermediate certificates is large, the size is
bounded, so it is possible to provide a complete set of certificates.

[EDNOTE: More motivation herer from EAP drafts, QUIC, and TLS papers about PQ ] 

For a client that has all intermediates, having the server send intermediates
in the TLS handshake increases the size of the handshake unnecessarily.  This
document creates a signal that a client can send that informs the server that
it has a complete set of intermediates.  A server that receives this signal can
limit the certificate chain it sends to just the end-entity certificate, saving
on handshake size.

This mechanism is intended to be complementary with certificate compression
{{?RFC8879=rfc8879}} in that it reduces the size
of the handshake.


# Terms and Definitions

{::boilerplate bcp14-tagged}


# Got Intermediates Flag

A client that believes that it has a current, complete set of intermediate
certificates to authenticate the server sends the tls_flags extension 
{{!TLS-FLAGS=I-D.ietf-tls-tlsflags}} with the 0xTBD1 flag set to 1 in 
its ClientHello message. 

A server that receives a value of 1 in the 0xTBD1 flag of a ClientHello
message SHOULD omit all certificates other than the end-entity certificate 
from its Certificate message that it sends in response. As per 
{{!TLS-FLAGS=I-D.ietf-tls-tlsflags}}, the server will also acknowledge 
support by sending the tls_flags extension in the Certificate message 
with the 0xTBD1 flag set to 1. Otherwise if it does not support CA 
certificate suppression, the server SHOULD ignore the 0xTBD1 flag. 

A server that believes that it has a current, complete set of intermediate
certificates to authenticate the client sends the tls_flags extension 
{{!TLS-FLAGS=I-D.ietf-tls-tlsflags}} with the 0xTBD2 flag set to 1 in 
its CertificateRequest message. 

A client that receives a value of 1 in the 0xTBD2 flag in a CertificateRequest 
message SHOULD omit all certificates other than the end-entity certificate 
from the Certificate message that it sends in response. As per 
{{!TLS-FLAGS=I-D.ietf-tls-tlsflags}}, the client will also acknowledge 
support by sending the tls_flags extension in the Certificate message 
with the 0xTBD2 flag set to 1. Otherwise if it does not support CA 
certificate suppression, the client SHOULD ignore the 0xTBD1 flag. 

The 0xTBD1 and 0xTBD2 flags can only be sent in a ClientHello, Certificate 
or CertificateRequest message. Endpoints that receive a value of 1 in 
any other handshake message MUST generate a fatal illegal_parameter alert.

[ EDNOTE: Discuss caching  beyond the scope of this document. ]

[ EDNOTE: Add Note about using the old draft to include the chain fingerprint in order for the peer to confirm the peer has the right cert chain, in order to avoid inadvertent issues ] 

# Security Considerations

This creates an unencrypted signal that might be used to identify which clients
believe that they have all intermediates.  This might allow clients to be more
effectively fingerprinted by peers and any elements on the network path.

Mitigations 


# IANA Considerations

This document registers the 0xTBD flag in the registry created by
{{!TLS-FLAGS=I-D.ietf-tls-tlsflags}}.


--- back

