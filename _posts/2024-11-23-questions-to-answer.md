---
title: Questions to answer
date: 2024-11-23 13:00:00 +/-0000
categories: [About]
tags: []     # TAG names should always be lowercase
---


This page is a list of questions that emerge as I work through some of the other posts on this site.  Over time, I'll aim to address all of these through examples across the various posts.

At some stage, I'll cross reference these as some sort of index to other posts:

- **CAs and certs**
  - [ ] What makes an intermediate certificate intermediate?
    - [ ] What are the observable differences between root, intermediate, and end-entity certificates?
    - [ ] The x509v3 extensions appear to restrict certificate usage, what happened before these extensions were added?
    - [ ] Can I sign any certificate with any other e.g. asking openssl to ignore these v3 extensions?
    - [ ] Is enforcement ultimately a responsibility of the validating party to ensure that a non-signing certificate has not been used in a chain?
    - [ ] How do different clients behave when attempting to validate a broken certificate chain?
  - [ ] How does certificate signing work (RSA)?
  - [ ] how does certificate signing work (ECDSA)?


- **mutually authenticated TLS**
  - [ ] What cryptographic mechanism is used by the client and server to prove that the client has the private key for the client certificate presented?
  - [ ] What is the practical difference between self-signed, private CA signed, and public CA signed certs when used for client auth?


- **Cipher suites**
  - [ ] Cipher suite name strings are a bit unintuitive (to me) as they don't appear to be structured - how do they hang together?

