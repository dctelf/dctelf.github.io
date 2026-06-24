---
title: What are the different characteristics of client auth certificate signing for mTLS?
date: 2024-12-09 18:00:00 +/-0000
categories: [Authentication]
tags: [x509, ca, signing, authentication, mtls]     # TAG names should always be lowercase
---

A question came up recently: why not just use a public CA signed certificate when mTLS runs between two separate parties/organisations?

This post explores the question, though mTLS itself may not be the right choice here. - this [twitter thread by Colm MacCárthaigh](https://x.com/colmmacc/status/1057017343438540801) and this discussion from the [security, cryptography, whatever podcast [33m 20s in]](https://securitycryptographywhatever.com/2021/12/29/the-feeling-s-mutual-mtls-with-colm-maccarthaigh/) expand on many drawbacks in the use of mTLS as part of a client authentication and authorisation architecture.  mTLS is common enough in practice that the certificate choices are worth understanding.

## Questions to explore

To explore this further, I'll look at the following questions and demonstrate answers where applicable:


- what are the common approaches to mTLS  & which properties of client certificates are used to identify the client?
- are certificate extensions applicable to client authentication?
- what's to stop a client certificate being *passed off* as legitimate by a different client?
- what are the conceptual characteristic differences between the following client certificate signing approaches:
    - self-signed
    - private CA signed
    - public CA signed
- are there any public CAs that consider client certificates differently?

## Common approach to mTLS & certificate properties

It's worth being aware of the concepts of [clients authenticating services](/posts/service-certificates/) first - that post highlights the various RFCs and mechanisms in use for the more typical/widespread service authentication mechanism of WebPKI.

After hunting through various articles and RFCs, there is no de-facto standard for the client equivalent of the *presented identifier* as laid out for service identity validation in [RFC9525](https://datatracker.ietf.org/doc/html/rfc9525).  [RFC5280 section 4.2.1.6](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.6) identifies `subjectAltName` as the appropriate field for identity information, but does not define options that necessarily apply to client authentication, and states `Other options exist, including completely local definitions.`.

The documented approaches for the client auth aspect of mTLS are typically found in:
- load balancer/API gateway docs (due to these components often handling aspects of TLS to support header/content inspection and injection)
    - [GCP load balancer mTLS](https://cloud.google.com/load-balancing/docs/mtls)
    - [AWS ALB mTLS](https://aws.amazon.com/blogs/networking-and-content-delivery/introducing-mtls-for-application-load-balancer/)
    - [AWS API gateway mTLS](https://aws.amazon.com/blogs/compute/introducing-mutual-tls-authentication-for-amazon-api-gateway/)
    - [F5 LTM mTLS](https://www.f5.com/labs/learning-center/what-is-mtls)
- ...and the docs in relation to service mesh/Kubernetes service architectures, where machine-to-machine authentication and authorisation is frequently found.
    - [istio (+Envoy) mTLS](https://istio.io/latest/docs/concepts/security/#mutual-tls-authentication)
    - [Hashicorp Consul mTLS](https://developer.hashicorp.com/consul/docs/connect/connect-internals#mutual-transport-layer-security-mtls)

In the world of service-to-service (/machine-to-machine) authentication, the [SPIFFE project](https://spiffe.io/) proposes a fairly widely adopted standardised approach for the issuance of identity, introducing the concept of a [SPIFFE verifiable ID (SVID)](https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE.md#3-the-spiffe-verifiable-identity-document) formed from an [x.509 certificate](https://github.com/spiffe/spiffe/blob/main/standards/X509-SVID.md).  The SPIFFE standards also support definition of an SVID through JWTs.  From the x509 perspective the mTLS aspect entails clients containing a valid SPIFFE ID (e.g. spiffe://acme.com/billing/payments) within the subjectAltName field of the client certificate - the standard then defines how services validate these IDs (considering things like trust domains, use for authorisation etc.).

The links at the top of this post referencing Colm MacCárthaigh also highlight that, often, many implementations are less considered than the SPIFFE standards and just entail some string matching from some field of a certificate.  Effectively, the service....

- *...accepts that the string in the presented certificate is the identity of the client*
- *... trusts the contents (and integrity) of the certificate as it is signed by a CA that we trust, and we can verify the chain by the signing algorithms*
- *... trusts the signing CA to have verified in some form that the identity string in the CSR is legitimate before signing*

> The core theme here is that there is a standard ([RFC5280 section 4.2.1.6](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.6)) stating that the subjectAltName is the appropriate field of an x509 to store identity information, but the standard allows this to be defined in any form.<br /><br />For the purposes of client authentication there are various approaches in use, ultimately this appears to be a choice dependent on the nature of the implementation - many cloud services that sit in front of backend services (API gateways, load balancers etc.) verify the certificate chain against a trust store, but then pass through certificate fields (such as subjectAltName) to the backend service, effectively passing on elements of authentication and authorisation.<br /><br />This re-affirms the potential limitations/issues with mTLS - the protocol offers the property that the connection will be permitted when a trusted-CA signed cert is presented at negotiation time, and can ensure the integrity of the data held within that certificate, but beyond that the identification and authorisation mechanisms are not defined by RFCs and are open to local implementation decisions.
{: .prompt-info }

## Certificate extensions

Given that there are limited standards guiding choices implementers should/must make when deploying mTLS (i.e. there is no matching *client identity* equivalent of [Service identity - RFC9525](https://datatracker.ietf.org/doc/html/rfc9525)), the certificate requirements effectively fall back to the core standards for [TLS](https://datatracker.ietf.org/doc/html/rfc9525) & [X509](https://datatracker.ietf.org/doc/html/rfc5280).

This is illustrated clearly through examples such as the [GCP load balancer mTLS docs](https://cloud.google.com/load-balancing/docs/mtls#certificate-requirements) ...

![GCP load balancing mTLS certificate requirements](assets/img/2024-12-09-GCP-LB-doc-extract.png){: width="972" height="589"}
_GCP load balancing mTLS certificate requirements_

> The core theme here is that the certificate extensions are very similar to service identity - end-entity/leaf certificates must not be CA certificates, keyUsage fields must be set accordingly etc.<br /><br />**One notable difference (although I'll touch more on this below when addressing public CA signed certs) is the role of the `Client Authentication` value within the Extended Key Usage field.**
{: .prompt-info }

## Passing off a copied certificate

When considering the different signing approaches, the role of the private key in a key exchange is well covered in various articles and fairly intuitive, but how the association of a private key to the public key prevents a service (or client) copying a certificate and passing it off is less obvious.  

A later post will cover how this works within TLS (it's part of the [Certificate Verify](https://datatracker.ietf.org/doc/html/rfc8446#section-4.4.3) aspect of the TLS handshake), but in short: during the mTLS handshake, the client generates a signature with its private key over data already known to both parties, proving it holds the private key matching the presented certificate.


## Differing signing approaches

When considering the different approaches to certificate signing, there are a few important concepts:

1. Having the service only lodge something it trusts into its trust store
2. The CA (/signing party) mechanism of verifying identity before signing a certificate for a client
3. Ability of the CA to set/accept the expected attributes of a client certificate when signing
4. The number of clients that may need to authenticate to a service, the *trust domain* considerations, and the presence of other secure channels within that domain


### Self signed certificates

Where a self-signed client certificate is used;

1. That self signed certificate needs to be added to the trust store of the service.  The certificate does not need to be kept as a secret (see the point above on *passing off*), but the service needs some way to ensure the integrity and authenticity of the certificate before adding to the trust store.
2. Assuming each client generates its own self-signed certificate and no private keys are shared, identity verification is entirely the service's responsibility at the point it accepts the certificate into its trust store — no other party is involved.
3. As no other party is involved, the client is in full control of the certificate attributes set
4. with self signed certificates;
    - in a single client scenario the use of mTLS places focus on the point where the service accepts the certificate into the truststore.  This entails no need for secrecy of the certificate, but does necessitate some means to ensure integrity and authenticity of the accepted certificate
    - in a multi-client scenario this is likely to become unwieldy — this is one driver for PKI in the first place, which avoids the need for a "web" of trust, instead focussing trust on some trust anchor/authority


### Private CA signed certificates

Where a private CA signed certificate is used;

1. The CA certificate needs to be added to the trust store of the service, and as for all other scenarios, the integrity and authenticity of this service must be ensured before addition to the truststore of the service.
2. The service trusts that the CA has verified the claimed identity before signing — with private CAs this is controlled within the service's trust domain, and can be adapted to local needs (e.g. a separate auth process as part of the signing request cycle).
3. As a private CA, the CA is free to set attributes to meet the needs of the environment - this extends to local choices on the contents of the subjectAltName extension field on certificates (e.g. the SPIFFE examples above), and to ensure that the extendedKeyUsage fields are set appropriately.
4. with private CA signed certificates;
    - in a single client scenario this all starts to feel like a lot of complexity and abstraction relative to self signed certificates
    - in a multi-client scenario this starts to make more sense — a centralised authority trusted to grant access to many clients removes the burden of verification from the service itself

### Public CA signed certificates

Where a public CA signed certificate is used;

1. As above, the CA certificate needs to be added to the trust store of the service
2. Public CAs are based on domain validation as the core identity verification mechanism. From a client identity perspective, validating control over a domain is technically decoupled from the client identity (the service need not perform a DNS query to facilitate client access), making this process largely irrelevant for client auth — particularly in large organisations where the team handling a B2B integration may be far removed from the team managing public-facing internet services.
3. The certificate attributes and extensions typically required for WebPKI service identity are almost identical to those for client identity - the exception is the `Client Authentication` value within the Extended Key Usage field, however (more detail below), this appears to be set by many public CAs anyway when signing DV certs.
4. with public CA signed certificates;
    - in a single client scenario this all starts to feel like a lot of complexity and abstraction relative to self signed certificates
    - in a multi-client scenario this starts to make more sense — a centralised authority trusted to grant access to many clients removes the burden of verification from the service itself

> An important observation with public CA signed certificates vs. private CA signed certificates: public CAs will sign certificates to any requester that can meet the domain verification need, with not further identity challenge.  Therefore, trusting a public CA entails accepting certificates from anyone who validly requests one - this re-enforce the need to also authorise the claimed identity within the subjectAltName field.  At least with private CAs, issuance can only occur to requesters within the trust domain.
{: .prompt-info }


## Public CA treatment of certificates for client auth

After searching on the use of public CA certs for client auth, some observations from the time of writing (late 2024) emerged:

1. Many CAs issued certificates with both client and server authentication set within the extended key usage field
2. Checking a handful of prominent websites, almost all had both client and server auth set in their certs (from a variety of CAs) — only exception I found was google.com
3. There were discussions in [forums](https://community.letsencrypt.org/t/extendedkeyusage-tls-client-authentication-in-tls-server-certificates/59140) and on mailing lists about splitting root certificates for service and client auth over time

**Update (2026):** This split has since happened. Google Chrome's root program mandated separation of TLS client and server authentication into distinct PKIs, with the requirement taking effect June 15, 2026. Let's Encrypt published [Ending TLS Client Authentication Certificate Support in 2026](https://letsencrypt.org/2025/05/14/ending-tls-client-authentication) and stopped including the client auth EKU in its certificates by default from February 2026. This makes the conclusion below even clearer — public CA signed server certificates can no longer be used for client auth at all.

> The general conclusion is that public CAs and the wider CA programme world don't dictate the approach to client certificates. Public CAs may issue DV certificates with extensions that fit client auth use cases, but no standard requires certificates to be formed in any particular way. There is no good reason to use public CA signed certs for client auth.
{: .prompt-info }

## Summary

- There is no RFC equivalent of RFC9525 for client identity — RFC5280 permits identity information in `subjectAltName` but leaves the format to local implementation. In practice, approaches vary widely across load balancers, API gateways, and service meshes.
- Certificate extensions for client auth closely mirror those for service auth, with `CA:FALSE` and appropriate `keyUsage` fields required. The notable difference is the `Client Authentication` value in `extendedKeyUsage`.
- Passing off a copied certificate is prevented by the TLS handshake itself — the client must prove possession of the private key corresponding to the presented certificate via the Certificate Verify mechanism.
- Self-signed certificates place the full burden of identity verification on the service at the point of trust store acceptance. Private CA signed certificates delegate that verification to a trusted authority within a controlled domain. Public CA signed certificates rely on domain validation, which is largely irrelevant to client identity.
- Public CAs previously issued dual-use certificates covering both server and client auth, but this has ended — Google Chrome's root program mandated separation from June 2026, and CAs including Let's Encrypt have stopped issuing certificates with the client auth EKU. There is no good reason to use public CA signed certificates for client auth.


