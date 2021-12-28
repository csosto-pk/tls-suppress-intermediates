---
title: "Suppressing CA Certificates in TLS"
abbrev: Suppress CAs
docname: draft-kampanakis-tls-scas-latest
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

  IEEE802154:
    title: "IEEE Standard for Low-Rate Wireless Networks"
    seriesinfo:
      DOI: 10.1109/IEEESTD.2020.9144691
    date: 23 July 2020

  WISUN:
    title: WI-SUN Alliance
    target: https://wi-sun.org/

  CL-BLOG:
    author:
      -
        ins: B. Westerbaan
        name: Bas Westerbaan
    title: Sizing Up Post-Quantum Signatures
    date: 8 November 2021
    target: https://blog.cloudflare.com/sizing-up-post-quantum-signatures/

  CONEXT-PQTLS13SSH:
    author:
      -
        ins: D. Sikeridis
        name: Dimitrios Sikeridis
      -
        ins: P. Kampanakis
        name: Panos Kampanakis
      -
        ins: M. Devetsikiotis
        name: Michael Devetsikiotis
    title: "Assessing the Overhead of Post-Quantum Cryptography in TLS 1.3 and SSH"
    date: November 2020
    seriesinfo:
      DOI: 10.1145/3386367.3431305
      ISBN: 9781450379489

  NDSS-PQTLS13:
    author:
      -
        ins: D. Sikeridis
        name: Dimitrios Sikeridis
      -
        ins: P. Kampanakis
        name: Panos Kampanakis
      -
        ins: M. Devetsikiotis
        name: Michael Devetsikiotis
    title: "Post-Quantum Authentication in TLS 1.3: A Performance Study"
    date: February 2020
    seriesinfo:
      DOI: 10.14722/ndss.2020.24203

  PQTLS:
    author:
      -
        ins: C. Paquin
        name: Christian Paquin
      -
        ins: D. Stebila
        name: Douglas Stebila
      -
        ins: G. amvada
        name: Goutam Tamvada
    title: "Benchmarking Post-Quantum Cryptography in TLS"
    date: 2019
    target: https://ia.cr/2019/1447

  TLS-SUPPRESS:
    author:
      -
        ins: P. Kampanakis
        name: Panos Kampanakis
      -
        ins: M. Kallitsis
        name: Michael Kallitsis
    title: "Speeding up post-quantum TLS handshakes by suppressing intermediate CA certificates"
    date: 2021
    target: https://www.amazon.science/publications/speeding-up-post-quantum-tls-handshakes-by-suppressing-intermediate-ca-certificates

  ICA-PRELOAD:
    author:
      -
        ins: D. Keeler
        name: Dana Keeler
    title: "Preloading Intermediate CA Certificates into Firefox"
    date: November 13, 2020
    target: https://blog.mozilla.org/security/2020/11/13/preloading-intermediate-ca-certificates-into-firefox/


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

{{?CBOR-CERTS=I-D.ietf-cose-cbor-encoded-cert}} talks about the cert size
problem in constrained environments.

{{?EAPTLSCERT=I-D.ietf-emu-eaptlscert}} and {{?EAP-TLS13=I-D.ietf-emu-eap-tls13}}
talk about the problem of big TLS cert chains for EAP authentication.

IEEE 802.15.4 {{IEEE802154}} mesh networks and Wi-SUN {{WISUN}} Field Area Networks  often notice significant delays due to EAP-TLS in constrained bandwidth mediums.

{{?CTLS=I-D.ietf-tls-ctls}} proposes cert dictionaries to omit
Intermediate CA (ICA) certificates.

{{CONEXT-PQTLS13SSH}} {{NDSS-PQTLS13}} show that cert chains exceeding TCP initcwnd slow down the handshake (round-trips)

{{PQTLS}} shows that big certificate chains (even smaller than the TCP initcwnd) slow down the handshake in lossy conditions.

{{TLS-SUPPRESS}} discusses issue about QUIC and TLS and PQ certificates.

Cloudflare Blog {{CL-BLOG}} shows that >9-10KB cert chains (even with 30MSS initcwnd) leads to double digit slowdowns. It also shows that some clients or middleboxes cannot handle the post-quantum handshake sizes.

{{?RFC7924}}. No widespread adoption 5 years after standardization.
Could allow TLS session correlation. Short-lived server certs will lead to frequent cache maintenance. Too many destinations will lead to frequent cache maintenance.

For a client that has all intermediates, having the server send intermediates
in the TLS handshake increases the size of the handshake unnecessarily.  This
document creates a signal that a client can send that informs the server that
it has a complete set of intermediates.  A server that receives this signal can
limit the certificate chain it sends to just the end-entity certificate, saving
on handshake size.

This mechanism is intended to be complementary with certificate compression
{{?RFC8879=rfc8879}} in that it reduces the size of the handshake especially
for post-quantum certificates.



# Terms and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED",
"MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 {{!RFC2119=rfc2119}} {{!RFC8174=rfc8174}} when,
and only when, they appear in all capitals, as shown here.


# Suppress CA Certificates Flag

The goals if when the peer has the CAs to build the certificate chain
it can signal to the peer to not send them and alleviate the data.

It is beyond the scope of this document to define caching. Cases where
all CAs can be cached like {{ICA-PRELOAD}}. Other usecase like will need caching and update  mechanism for cache hit and misses. Some are discussed in {{TLS-SUPPRESS}}.

## Client

A client that believes that it has a current, complete set of intermediate
certificates to authenticate the server sends the tls_flags extension
{{!TLS-FLAGS=I-D.ietf-tls-tlsflags}} with the 0xTBD1 flag set to 1 in
its ClientHello message.

A server that receives a value of 1 in the 0xTBD1 flag of a ClientHello
<<<<<<< HEAD
message SHOULD omit all certificates other than the end-entity certificate
from its Certificate message that it sends in response. As per
{{!TLS-FLAGS=I-D.ietf-tls-tlsflags}}, the server SHOULD also acknowledge
support by sending the tls_flags extension in the Certificate message
with the 0xTBD1 flag set to 1. Otherwise if it does not support CA
certificate suppression, the server SHOULD ignore the 0xTBD1 flag.

## Server (mutual TLS authentication)

In a mutual TLS authentication scenario, a server that believes that it
has a current, complete set of intermediate certificates to authenticate
the server sends the tls_flags extension {{!TLS-FLAGS=I-D.ietf-tls-tlsflags}}
with the 0xTBD2 flag set to 1 in its CertificateRequest message.

A client that receives a value of 1 in the 0xTBD2 flag in a CertificateRequest
message SHOULD omit all certificates other than the end-entity certificate
from the Certificate message that it sends in response. As per
{{!TLS-FLAGS=I-D.ietf-tls-tlsflags}}, the client SHOULD also acknowledge
support by sending the tls_flags extension in its Certificate message
with the 0xTBD2 flag set to 1. Otherwise if it does not support CA
certificate suppression, the client SHOULD ignore the 0xTBD1 flag.

The 0xTBD1 and 0xTBD2 flags can only be sent in a ClientHello, Certificate
or CertificateRequest message. Endpoints that receive a value of 1 in
=======
message SHOULD omit all certificates other than the end-entity certificate 
from its Certificate message that it sends in response. As per 
{{!TLS-FLAGS=I-D.ietf-tls-tlsflags}}, the server will also acknowledge 
support by sending the tls_flags extension in the Certificate message 
with the 0xTBD1 flag set to 1. Otherwise if it does not support CA 
certificate suppression, the server SHOULD ignore the 0xTBD1 flag. 

## Server (mutual TLS authentication)

In a mutual TLS authentication scenario, a server that believes that it 
has a current, complete set of intermediate certificates to authenticate 
the server sends the tls_flags extension {{!TLS-FLAGS=I-D.ietf-tls-tlsflags}} 
with the 0xTBD2 flag set to 1 in its CertificateRequest message. 

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

EDNOTE: Discuss caching  beyond the scope of this document.

EDNOTE: Add Note about optionally using the old draft to include the chain fingerprint in order for the peer to confirm the peer has the right cert chain, in order to avoid inadvertent issues

# Security Considerations

This creates an unencrypted signal that might be used to identify which clients
believe that they have all intermediates. It does not reveal information about the destination or
source. This might allow clients to be more
effectively fingerprinted by peers and any elements on the network path.
Mitigations of {{?ESNI=I-D.ietf-tls-esni}}.

Correlation of data transferred in each direction could allow for destination could allows for correlation. Flipping a coin to request suppression
or not.


# IANA Considerations

This document registers the 0xTBD1, 0xTBD2 flags in the registry created by
{{!TLS-FLAGS=I-D.ietf-tls-tlsflags}}.


--- back

