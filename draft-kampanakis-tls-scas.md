---
title: "Suppressing CA Certificates in TLS 1.3"
abbrev: Suppress CAs
docname: draft-kampanakis-tls-scas-latest-03
category: exp
ipr: trust200902
area: Transport
workgroup: TLS

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping]

author:
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

  -
    ins: M. Thomson
    name: Martin Thomson
    org: Mozilla
    email: mt@lowentropy.net

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

  FILOSOTTILE:
    author:
      -
        ins: F. Valsorda
        name: Filippo Valsorda
    title: "filippo.io/intermediates"
    date: 2022
    target: https://github.com/FiloSottile/intermediates

  QUIC-CERTS:
    author:
      -
        ins: M. Nawrocki
        name: Marcin Nawrocki
      -
        ins: P. Tehrani
        name: Pouyan Fotouhi Tehrani
      -
        ins: R. Hiesgen
        name: Raphael Hiesgen
      -
        ins: J. Mucke
        name: Jonas Mucke
      -
        ins: T. Schmidt
        name: Thomas C. Schmidt        
      -
        ins: M. Wahlisch 
        name: Matthias Wahlisch

    title: "On the Interplay between TLS Certificates and QUIC Performance"
    date: November 2022
    seriesinfo:
      DOI: 10.1145/3555050.3569123

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
for some use cases. {{?EAPTLSCERT=I-D.ietf-emu-eaptlscert}} and
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
than 10kB. {{QUIC-CERTS}} also shows discusses how classical RSA
certificate chains often exceed the QUIC amplification, an issue
which will happen almost always with post-quantum certicates.

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
is large, the size is bounded. Additionally, in some use cases the
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
large size of peers in some use cases make the peer certificate
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
these certificates. TLS {{?RFC5246=rfc5246}}
{{?RFC8446=rfc8446}} allows for the root CA certificate to be
omitted from the handshake under the assumption that
the remote peer already possesses it in order to validate
its peers. Thus, a client or server in possession of
the CA certificates would only need the peer end-entity
certificate to validate its identity which would alleviate
the data flowing in TLS.


This draft assumes that the endpoint can keep as set of ICAs
in memory to use them while building certificate chains to
authenticate a peer. Most usually the set will be stored locally
in non-volatile memory. In constrained devices the intermediates
could be cached, kept and updated only in volatile memory
especially when the communicating peers' PKI domains are
limited. 

How CA certificates are identified and stored is dependent on
the use case. In some use cases (e.g. WebPKI {{ICA-PRELOAD}}) the
peer may assume that all intermediates are assembled, distributed
and updated regularly using an out-of-band mechanism. In other
use cases when the communicating peers' PKI domains are
limited and not all CA certificates can be stored (i.e.,
constrained devices), or distributed, intermediates could be
cached and updated dynamically using a caching mechanism. 
Such mechanisms are discussed in {{TLS-SUPPRESS}}.

Although this document uses mechanisms to minimize TLS
authentication failures due to stale or incomplete ICA lists,
an endpoint is expected to re-attempt a TLS connection if it
failed to authenticate a peer certificate after requesting ICA
suppression. \[EDNOTE: draft-ietf-tls-esni already requires
the client to retry a connection when ECH is "securely replaced
by the server" or "securely disabled by the server". \]

\[EDNOTE: To prevent failuers, one additional option
could be to use a TLS extension like the one defined
in {{?RFC7924}} to include the chain fingerprint so the
peer can confirm that he does not need to send the chain because
the peer asking for suppression has the correct chain to validate the
server. That could prevent inadvertent mistakes where the client thinks
it has the intermediates to validate the server, but what it has is wrong.
The shortcoming is that could be used as a cookie.
Alternatively we could HMAC the chain to make it indistinguisable.
Another option is for the server to provide a ticket so client returning
visits tell the server that the client has the ICAs and it does not
need to send them. These options require further evaluation only if
we think that the complexity is worth the benefit.\]

The 0xTBD1 flag used to signal CA suppression can only be sent
in a ClientHello or CertificateRequest message as defined below.
Endpoints that receive a 0xTBD1 flag with a value of 1 in any other
handshake message MUST generate a fatal illegal_parameter alert.


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
from its Certificate message that it sends in response. Otherwise if it
does not support CA certificate suppression, the server SHOULD ignore the
0xTBD1 flag.

To prevent a failed TLS connection, a server could choose to send its
intermediates regardless of the flag from the client, if it has a reason
to believe the issuing CAs do not exist in the client ICA list. For
example, if the server's certificate chain contains ICAs with
technical constraints which are not disclosed, the server SHOULD send
the chain back to the client regardless of the suppression flag in the
ClientHello. 

If the connection still fails because the client cannot build the
certificate chain to authenticate the server, the client MUST NOT
send the flag in a subsequent connection to the server.

## Server (mutual TLS authentication)

In a mutual TLS authentication scenario, a server that believes that it
has a current, complete set of intermediate certificates to authenticate
the client, sends the tls_flags extension {{!TLS-FLAGS=I-D.ietf-tls-tlsflags}}
with the 0xTBD1 flag set to 1 in its CertificateRequest message.

To prevent a failed TLS connection, a server MAY choose not to send the
flag if its list of ICAs hasn't been updated in TBD3 time or has any other
reason to believe it does not include the ICAs for its peer.

A client that receives a value of 1 in the 0xTBD1 flag in a CertificateRequest
message SHOULD omit all certificates other than the end-entity certificate
from the Certificate message that it sends in response. Otherwise if it
does not support CA certificate suppression, the client SHOULD ignore the
0xTBD flag.

To prevent a failed TLS connection, a client could choose to send its
intermediates regardless of the flag from the server, if it has a reason
to believe the issuing CAs do not exist in the server ICA list. For
example, if the client's certificate chain contains ICAs with technical
constraints which are not disclosed, the client SHOULD send the chain
back to the server regardless of the CA suppression flag in the
CertificateRequest. \[EDNOTE: MSRP 2.8 may require constrained
intermediates which would mean this could change for WebPKI.\]

If the connection still fails because the server cannot build the
certificate chain to authenticate the client, the server MUST NOT
send the flag in a subsequent connection from the client.
\[EDNOTE: There is a challenge with this in that the server needs to keep
track of failed client connections.\]


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

Even when the 0xTBD1 flag is encrypted in the handshake,
a passive observer could fingerprint the peers by analyzing the TLS
handshake data sizes flowing each direction. Widespread adoption of
the TLS CA suppression mechanism described in this document will deem
the use of the signal for fingerprinting impractical.
<!-- [EDNOTE: Commenting this out as the probabilistic TLS suppression for the same source-destination would reveal trying to hide TLS suppression. Maybe we can rethink it later] 
To alleviate this
concern the client or server could randomly choose to not request
suppression although it has the CA certificates to validate the peer.
That would prevent a passive attacker concluding if the CA certificate
suppression signal is supported by the client or server. The
probability distribution for chosing to request suppression or not
is a trade-off decision between the risk of fingerprinting and TLS
performance. --> 


# IANA Considerations

This document registers the 0xTBD1 in the registry created by
{{!TLS-FLAGS=I-D.ietf-tls-tlsflags}}. 

# Acknowledgements 

We would like to thank Ilari Liusvaara, Ryan Sleevi
Filippo Valsorda and for their valuable feedback contributions
to this document.

The authors would also like to thank Filippo Valsorda for his feedback
regarding ICA lists {{FILOSOTTILE}}.


--- back

