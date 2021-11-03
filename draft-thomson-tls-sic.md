---
title: "Suppressing Intermediate Certificates in TLS"
abbrev: Suppress Intermediates
docname: draft-thomson-tls-sic-latest
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

For a client that has all intermediates, having the server send intermediates
in the TLS handshake increases the size of the handshake unnecessarily.  This
document creates a signal that a client can send that informs the server that
it has a complete set of intermediates.  A server that receives this signal can
limit the certificate chain it sends to just the end-entity certificate, saving
on handshake size.

This mechanism is intended to be complementary with certificate compression
{{?COMPRESS=I-D.ietf-tls-certificate-compression}} in that it reduces the size
of the handshake.


# Terms and Definitions

{::boilerplate bcp14-tagged}


# Got Intermediates Flag

A client that believes that it has a current, complete set of intermediate
certificates sends the tls_flags extension {{!TLS-FLAGS=I-D.nir-tls-tlsflags}}
with the 0xTBD flag set to 1.  A server can also set the flag in a
CertificateRequest extension.

A server that receives a value of 1 in the 0xTBD flag from a ClientHello
message SHOULD omit all certificates other than the end-entity certificate from
its Certificate message.  A client that receives a value of 1 in the 0xTBD flag
in a CertificateRequest message SHOULD omit all certificates other than the
end-entity certificate from the Certificate message that it sends in response.

The 0xTBD flag can only be send in a ClientHello or CertificateRequest message.
Endpoints that receive a value of 1 in any other handshake message MUST
generate a fatal illegal_parameter alert.


# Security Considerations

This creates an unencrypted signal that might be used to identify which clients
believe that they have all intermediates.  This might allow clients to be more
effectively fingerprinted by peers and any elements on the network path.


# IANA Considerations

This document registers the 0xTBD flag in the registry created by
{{!TLS-FLAGS=I-D.nir-tls-tlsflags}}.


--- back

