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
  - [x] How does certificate signing work (RSA)?
  - [ ] how does certificate signing work (ECDSA)?
  - [ ] What measures are in place to detect mis-issuance/unexpected signing of end-entity certificates both in the PKI architecture, and also by significant clients (browser vendors etc.)?
    - [ ] What are certificate transparency (CT) logs
  - [ ] How does revocation work?
    - [ ] What are the differences between CRLs and OCSP?
    - [ ] What's the prevalent approach these days - are CRLs making a return?
    - [ ] Why are CRLs and OCSP served over unencrypted HTTP?
    - [ ] How are CRL lists formed and validated?

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

- **General crypto**
  - [ ] Cipher suite name strings are a bit unintuitive (to me) as they don't appear to be structured - how do they hang together?


- **Asymmetric/public-key crypto core themes**
  - [ ] A simple dive into textbook RSA for both encryption and signing?
  - [ ] Why is textbook RSA problematic from a signing perspective?
  - [ ] Why is textbook RSA problematic from an encryption perspective?
  - [ ] What are the attacks on RSA?

  **Symmetric ciphers and modes of operation**
  - [ ] What is authenticated encryption with associated data (AEAD)?