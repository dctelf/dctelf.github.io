---
title: What makes an intermediate certificate intermediate?
date: 2024-12-02 18:00:00 +/-0000
categories: [PKI]
tags: [x509, ca, signing]     # TAG names should always be lowercase
---

# Starting point

Having spent a little time creating dummy CAs, signing certificates and viewing lots of `openssl x509 -text` output, I came to the realisation that I wasn't entirely clear which x509 certificate attributes I should expect to see which denote the purpose of a certificate.

## Questions to explore
3
To further understanding, I'd like to explore the following questions and where applicable demonstrate the answers:

- What are the observable differences between root, intermediate, and *end-entity* certificates
- The x509v3 extensions appear to restrict certificate usage, what happened before these extensions were added?
- Fundamentally, can I sign any certificate with any other e.g. asking openssl to ignore these v3 extensions?  
- Is enforcement ultimately a responsibility of the validating party to ensure that a non-signing certificate has not been used in a chain?
- How do different clients behave when attempting to validate a broken certificate chain 


## Certificate differences

Let's use wikipedia.com and look at the certificate chain it presents with `openssl s_client` (with a bit of awk to extract the relevant lines for the presented certificate chain, up to 9 certificates, rarely see more than 3 or 4 in most scenarios):

```bash
$ echo "GET /" | \
openssl s_client -showcerts -connect www.wikipedia.com:443 -verify_return_error  2>&1 | \
awk '/^\s+[0-9]/,/\s+v:/ {print }'
```

<details>
<summary>
  
Click to expand
</summary>
<p>

```text
  Code here
```
</p>
</details>



This returns:

```text
  0 s:CN = wikipedia.com
   i:C = US, O = Let's Encrypt, CN = E5
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: ecdsa-with-SHA384
   v:NotBefore: Oct 31 23:18:47 2024 GMT; NotAfter: Jan 29 23:18:46 2025 GMT
 1 s:C = US, O = Let's Encrypt, CN = E5
   i:C = US, O = Internet Security Research Group, CN = ISRG Root X1
   a:PKEY: id-ecPublicKey, 384 (bit); sigalg: RSA-SHA256
   v:NotBefore: Mar 13 00:00:00 2024 GMT; NotAfter: Mar 12 23:59:59 2027 GMT
```

Core observation is that the issuer of the end-entity/service certificate is the subject of the next certificate up in the chain (and so on).  This subject -> issuer chain extends all the way to the root certificate in the trust store of the client.

From a root certificate perspective, the way `openssl s_client` displays this isn't intuitive (it only displays the certificate bundle presented by the service, but not the root), additionally I have a suspicion that openssl is always using my machines trust store and validating the full chain, so there is no easy way to see the difference between a chain that isn't validated against a root vs. one that is with openssl.

To show the similar fields from the matching root;

```bash
$ cat /etc/ssl/certs/ISRG_Root_X1.pem | openssl x509 -text -noout | egrep 'Issuer:|Subject:'
        Issuer: C = US, O = Internet Security Research Group, CN = ISRG Root X1
        Subject: C = US, O = Internet Security Research Group, CN = ISRG Root X1
```

Clearly there is a cryptographic mechanism of signing that makes the **trust** in all this hang together, but the hierarchy is reflected in the chain of subject and issuer - joining these 2 outputs together demonstrates one answer to the first question from above - What are the observable differences between *end-entity*, intermediate and root certificates:

```text
* end entity/subject
s:CN = wikipedia.com
i:C = US, O = Let's Encrypt, CN = E5

* intermediate
s:C = US, O = Let's Encrypt, CN = E5
i:C = US, O = Internet Security Research Group, CN = ISRG Root X1

* root
s:C = US, O = Internet Security Research Group, CN = ISRG Root X1
i:C = US, O = Internet Security Research Group, CN = ISRG Root X1

```