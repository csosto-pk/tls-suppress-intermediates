---
title: "Suppressing CA Certificates in TLS 1.3"
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
        ins: B.E. Westerbaan
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
        ins: G. Tamvada
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

  NIST_PQ:
    author:
      -
        ins: NIST
        name: National Institute of Standards and Technology
    title: "Post-Quantum Cryptography"
    date: 2021
    target: https://csrc.nist.gov/projects/post-quantum-cryptography


--- abstract

A TLS client or server that has access to the complete set of published
intermediate certificates can inform its peer to avoid sending
certificate authority certificates, thus reducing the size of
the TLS handshake.


--- middle

# Introduction

The most data heavy part of a TLS handshake is authentication. It usually
consists of a signature, an end-entity certificate and Certificate Authority
(CA) certificates used to authenticate the end-entity to a trusted root CA.
These chains can sometime add to a few kB of data which could be problematic
for some usecases. {{?EAPTLSCERT=I-D.ietf-emu-eaptlscert}} and
{{?EAP-TLS13=I-D.ietf-emu-eap-tls13}} discuss the issues big certificate
chains in EAP authentication. Additionally, it is known that IEEE 802.15.4
{{IEEE802154}} mesh networks and Wi-SUN {{WISUN}} Field Area Networks
often notice significant delays due to EAP-TLS authentication in
constrained bandwidth mediums.

To alleviate the data exchanged in TLS
{{?RFC8879=rfc8879}} shrinks certificates by compressing them.
{{?CBOR-CERTS=I-D.ietf-cose-cbor-encoded-cert}} uses different
certificate encodings for constrained environments. On the other hand,
{{?CTLS=I-D.ietf-tls-ctls}} proposes the use of certificate dictionaries
to omit sending CA certificates in a Compact TLS handshake.

In a post-quantum context
{{?I-D.hoffman-c2pq}}{{NIST_PQ}}{{?I-D.ietf-tls-hybrid-design}}, the TLS
authentication data issue is exacerbated. {{CONEXT-PQTLS13SSH}}{{NDSS-PQTLS13}}
show that post-quantum certificate chains exceeding the initial TCP
congestion window (10MSS {{?RFC6928=rfc6928}}) will slow down the handshake due
to the extra round-trips they introduce. {{PQTLS}} shows that big certificate
chains (even smaller than the initial TCP congestion window) will
slow down the handshake in lossy environments. {{TLS-SUPPRESS}}
quantifies the post-quantum authentication data in QUIC and TLS
and shows that even the leanest post-quantum signature algorithms
will impact QUIC and TLS. {{CL-BLOG}} also shows that 9-10 kilobyte
certificate chains (even with 30MSS initial TCP congestion window)
will lead to double digit TLS handshake slowdowns. What's more, it
shows that some clients or middleboxes cannot handle chains larger
than 10kB.

Mechanisms like
{{?RFC8879=rfc8879}}{{?CBOR-CERTS=I-D.ietf-cose-cbor-encoded-cert}}
would not alleviate the issue with post-quantum certificates
as the bulk of the certificate size is in the post-quantum
public key or signature which is incompressible.

Thus, this document introduces a backwards-compatible mechanism
to shrink the certificate data exchanged in TLS 1.3. In some uses
of public key infrastructure (PKI), intermediate CA certificates
sign end-entity certificates.  In the web PKI, clients require
that certificate authorities disclose all intermediate certificates
that they create. Although the set of intermediate certificates
is large, the size is bounded. Additionally, in some usecases the
set of communicating peers is limited.

For a client or server that has the necessary intermediates,
receiving them during the TLS handshake, increases the data
transmission unnecessarily. This
document defines a signal that a client or server can send to
inform its peer that it already has the intermediate CA
certificates. A peer that receives this signal can
limit the certificate chain it sends to just the
end-entity certificate, saving on handshake size.

This mechanism is intended to be complementary with
certificate compression {{?RFC8879=rfc8879}} in that it
further reduces the size of the handshake especially
for post-quantum certificates.

It is worth noting that {{?RFC7924}} attempted to address the
issue by omitting all certificates in the handshake if the client
or server had cached the peer certificate. This standard has not
seen wide adoption and could allow for TLS session correlation.
Additionally, the short lifetime certificates used today and the
large size of peers in some usecases make the peer certificate
cache update and maintenance mechanism challenging --- not the
least because of privacy concerns.  The
mechanism proposed in this document is not susceptible to
these challenges.

# Terms and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED",
"MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 {{!RFC2119=rfc2119}} {{!RFC8174=rfc8174}} when,
and only when, they appear in all capitals, as shown here.


# Suppress CA Certificates Flag

The goal is when a client or server has the intermediate CAs
to build the certificate chain for the peer it is establishing
a TLS connection with, to signal to the peer to not send
theese certificates. TLS {{?RFC5246=rfc5246}}
{{?RFC8446=rfc8446}} allow for the root CA certificate to be
omitted from the handshake under the assumption that
the remote peer already possesses it in order to validate
its peers. Thus, a client or server in possession of
the CA certificates would only need the peer end-entity
certificate to validate its identity which would alleviate
the data flowing in TLS.

It is beyond the scope of this document to define how CA certificates
are identified and stored. In some usecases {{ICA-PRELOAD}} the peer
may assume that all intermediates are available locally. In other
usecases where not all CA certificates can be stored, there may be
intermediate CA certificate caching and updating mechanisms.
Some options for such mechanisms are discussed in {{TLS-SUPPRESS}}.

EDNOTE: One additional option could be to use a TLS extension like
the one defined in {{?RFC7924}} to include the chain fingerprint so
the peer can confirm that he does not need to send the chain because
the peer asking for suppression has the correct chain to validate the
server. That could prevent inadvertent mistakes where the client thinks
it has the intermediates to validate the server, but what it has is wrong.
The shortcoming is that could be used as a cookie.
Alternatively we could HMAC the chain to make it indistinguisable.
Another option is for the server to provide a ticket so client returning
visits tell the server that the client has the ICAs and it does not
need to send them. These options require further evaluation only if
we think that they are worth the effort.


## Client

A client that believes that it has a current, complete set of intermediate
certificates to authenticate the server sends the tls_flags extension
{{!TLS-FLAGS=I-D.ietf-tls-tlsflags}} with the 0xTBD1 flag set to 1 in
its ClientHello message.

To prevent a failed TLS connection, a client MAY choose not to send the
flag if its list of ICAs hasn't been updated in TBD3 time or has any other
reason to believe it does not include the ICAs for its peer.

A server that receives a value of 1 in the 0xTBD1 flag of a ClientHello
message SHOULD omit all certificates other than the end-entity certificate
from its Certificate message that it sends in response. As per
{{!TLS-FLAGS=I-D.ietf-tls-tlsflags}}, the server shall also acknowledge
support by sending the tls_flags extension in the Certificate message
with the 0xTBD1 flag set to 1. Otherwise if it does not support CA
certificate suppression, the server SHOULD ignore the 0xTBD1 flag.

To prevent a failed TLS connection, a server could chose to not send its
intermediates regardless of the flag from the client, if it has a reason
to believe the issuing CAs do not exist in the client ICA list.

The 0xTBD1 flag can only be sent in a ClientHello message and the
Certificate response message from the server. Endpoints that receive
a 0xTBD1 flag with avalue of 1 in any other handshake message MUST
generate a fatal illegal_parameter alert.

## Server (mutual TLS authentication)

In a mutual TLS authentication scenario, a server that believes that it
has a current, complete set of intermediate certificates to authenticate
the client, sends the tls_flags extension {{!TLS-FLAGS=I-D.ietf-tls-tlsflags}}
with the 0xTBD2 flag set to 1 in its CertificateRequest message. 

To prevent a failed TLS connection, a server MAY choose not to send the
flag if its list of ICAs hasn't been updated in TBD3 time or has any other
reason to believe it does not include the ICAs for its peer.

A client that receives a value of 1 in the 0xTBD2 flag in a CertificateRequest 
message SHOULD omit all certificates other than the end-entity certificate 
from the Certificate message that it sends in response. As per 
{{!TLS-FLAGS=I-D.ietf-tls-tlsflags}}, the client shall also acknowledge 
support by sending the tls_flags extension in the Certificate message 
with the 0xTBD2 flag set to 1. Otherwise if it does not support CA 
certificate suppression, the client SHOULD ignore the 0xTBD2 flag. 

To prevent a failed TLS connection, a client could chose to not send its
intermediates regardless of the flag from the server, if it has a reason
to believe the issuing CAs do not exist in the server ICA list.

The 0xTBD2 flag can only be sent in a CertificateRequest message and the
Certificate response message from the client. Endpoints that receive
a 0xTBD2 flag with avalue of 1 in any other handshake message MUST
generate a fatal illegal_parameter alert.


# Security Considerations

This document creates an unencrypted signal in the ClientHello that
might be used to identify which clients believe that they have
intermediates to build the certificate chain for their peer.
Although it does not reveal any additional information about the
peers, it might allow clients to be more effectively fingerprinted
by peers or any passive observers in the network path. A
mitigation against this concern is to encrypt the ClientHello in
TLS 1.3 {{?ESNI=I-D.ietf-tls-esni}} which would hide the CA certificate
suppression signal.

Even when the 0xTBD1 and 0xTBD2 flags are encrypted in the handshake,
a passive observer could fingerprint the peers by analyzing the TLS
handshake data sizes flowing each direction. Widespread adoption of
the TLS suppression mechanism described in this document will deem
the use of the signal for fingerprinting impractical.
<!-- [EDNOTE: Commenting this out as the probabilistic TLS suppression for the same source-destination would reveal trying to hide TLS suppression. Maybe we can rethink it later] 
To alleviate this
concern the client or server could randomly chose to not request
suppression although it has the CA certificates to validate the peer.
That would prevent a passive attacker concluding if the CA certificate
suppression signal is supported by the client or server. The
probability distribution for chosing to request suppression or not
is a trade-off decision between the risk of fingerprinting and TLS
performance. --> 


# IANA Considerations

This document registers the 0xTBD1, 0xTBD2 flags in the registry created by
{{!TLS-FLAGS=I-D.ietf-tls-tlsflags}}.


--- back

