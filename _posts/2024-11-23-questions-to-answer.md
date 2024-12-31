---
title: Questions to answer
date: 2024-11-23 13:00:00 +/-0000
categories: [About]
tags: []     # TAG names should always be lowercase
---


This page is a list of questions that emerge as I work through some of the other posts on this site.  Over time, I'll aim to address all of these through examples across the various posts.

At some stage, I'll cross reference these as some sort of index to other posts:

- **CAs and certs**
  - [x] What makes an intermediate certificate intermediate?
    - [x] What are the observable differences between root, intermediate, and end-entity certificates?
    - [x] The x509v3 extensions appear to restrict certificate usage, what happened before these extensions were added?
    - [x] Can I sign any certificate with any other e.g. asking openssl to ignore these v3 extensions?
    - [x] Is enforcement ultimately a responsibility of the validating party to ensure that a non-signing certificate has not been used in a chain?
    - [x] How do different clients behave when attempting to validate a broken certificate chain?
  - [ ] How does certificate signing work (RSA)?
  - [ ] how does certificate signing work (ECDSA)?
  - [ ] What measures are in place to detect mis-issuance/unexpected signing of end-entity certificates both in the PKI architecture, and also by significant clients (browser vendors etc.)?
  - [ ] How does revocation work?

- **Service identity & authentication**
  - [ ] what properties of the service certificate are used to authenticate the service by the client?
  - [ ] which certificate extensions are applicable to service authentication?

- **mutually authenticated TLS**
  - [x] what are the common approaches to mTLS  & which properties of client certificates are used to identify the client?
  - [x] are certificate extensions applicable to client authentication?
  - [x] What's to stop a client certificate being *passed off* as legitimate by a different client?
  - [x] What are the conceptual characteristic differences between the following client certificate signing approaches:
      - [x] self-signed
      - [x] private CA signed
      - [x] public CA signed
  - [x] are there any public CAs that consider client certificates differently?

- **TLS protocol**
  - [ ] What stops certificates being copied and used by other services/clients?
    - [ ] How does a service (or client) prove ownership of the private part of their certificate?
    - [ ] How intrinsically linked is private key ownership verification and key exchanging?

- **Cipher suites**
  - [ ] Cipher suite name strings are a bit unintuitive (to me) as they don't appear to be structured - how do they hang together?

