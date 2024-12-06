---
title: What makes an intermediate certificate intermediate?
date: 2024-12-02 18:00:00 +/-0000
categories: [PKI]
tags: [x509, ca, signing]     # TAG names should always be lowercase
---

# Starting point

Having spent a little time creating dummy CAs, signing certificates and viewing lots of `openssl x509 -text` output, I came to the realisation that I wasn't entirely clear which x509 certificate attributes I should expect to see which denote the purpose of a certificate.

## Questions to explore

To further understanding, I'd like to explore the following questions and where applicable demonstrate the answers:

- What are the observable differences between root, intermediate, and *end-entity* certificates
- The x509v3 extensions appear to restrict certificate usage, what happened before these extensions were added?
- Can I sign a certificate with another that's not intended to be a root/intermediate e.g. asking openssl to ignore these v3 extensions?
- Is enforcement ultimately a responsibility of the validating party to ensure that a non-signing certificate has not been used in a chain?
- How do different clients behave when attempting to validate a broken certificate chain 


## Certificate differences

### Subject and issuer certificate fields

Let's use wikipedia.com and look at the certificate chain it presents with `openssl s_client` (with a bit of awk to extract the relevant lines for the presented certificate chain, up to 9 certificates, rarely see more than 3 or 4 in most scenarios):

```bash
$ echo "GET /" | \
openssl s_client -showcerts -connect www.wikipedia.com:443 -verify_return_error  2>&1 | \
awk '/^\s+[0-9]/,/\s+v:/ {print }'
```

<details markdown="1">
<summary>
<i>Expand for full openssl s_client output</i>

</summary>

```text
$ echo "GET /" | openssl s_client -showcerts -connect www.wikipedia.com:443 -verify_return_error
CONNECTED(00000003)
depth=2 C = US, O = Internet Security Research Group, CN = ISRG Root X1
verify return:1
depth=1 C = US, O = Let's Encrypt, CN = E5
verify return:1
depth=0 CN = wikipedia.com
verify return:1
---
Certificate chain
 0 s:CN = wikipedia.com
   i:C = US, O = Let's Encrypt, CN = E5
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: ecdsa-with-SHA384
   v:NotBefore: Oct 31 23:18:47 2024 GMT; NotAfter: Jan 29 23:18:46 2025 GMT
-----BEGIN CERTIFICATE-----
MIIFTTCCBNSgAwIBAgISA713TCJy7J9coRpo9VWF9QYXMAoGCCqGSM49BAMDMDIx
CzAJBgNVBAYTAlVTMRYwFAYDVQQKEw1MZXQncyBFbmNyeXB0MQswCQYDVQQDEwJF
NTAeFw0yNDEwMzEyMzE4NDdaFw0yNTAxMjkyMzE4NDZaMBgxFjAUBgNVBAMTDXdp
a2lwZWRpYS5jb20wWTATBgcqhkjOPQIBBggqhkjOPQMBBwNCAARytAnZIrrqdbAj
YnczbBJepbPE6W6PjAJHVlJZnh8e+UwRX3EKLCLLgp44NENFK4Jcd84xxPG+mN7c
ct/tCTR6o4ID4jCCA94wDgYDVR0PAQH/BAQDAgeAMB0GA1UdJQQWMBQGCCsGAQUF
BwMBBggrBgEFBQcDAjAMBgNVHRMBAf8EAjAAMB0GA1UdDgQWBBThg433eChTZtwP
wg2vOojHgvIkozAfBgNVHSMEGDAWgBSfK1/PPCFPnQS37SssxMZwi9LXDTBVBggr
BgEFBQcBAQRJMEcwIQYIKwYBBQUHMAGGFWh0dHA6Ly9lNS5vLmxlbmNyLm9yZzAi
BggrBgEFBQcwAoYWaHR0cDovL2U1LmkubGVuY3Iub3JnLzCCAekGA1UdEQSCAeAw
ggHcggsqLmVuLXdwLmNvbYILKi5lbi13cC5vcmeCDyoubWVkaWF3aWtpLmNvbYIQ
Ki52b3lhZ2V3aWtpLmNvbYIQKi52b3lhZ2V3aWtpLm9yZ4IQKi53aWlraXBlZGlh
LmNvbYIOKi53aWtpYm9vay5jb22CDyoud2lraWJvb2tzLmNvbYIPKi53aWtpZXBk
aWEuY29tgg8qLndpa2llcGRpYS5vcmeCECoud2lraWlwZWRpYS5vcmeCECoud2lr
aWp1bmlvci5jb22CECoud2lraWp1bmlvci5uZXSCECoud2lraWp1bmlvci5vcmeC
Dyoud2lraXBlZGlhLmNvbYIJZW4td3AuY29tggllbi13cC5vcmeCDW1lZGlhd2lr
aS5jb22CDnZveWFnZXdpa2kuY29tgg52b3lhZ2V3aWtpLm9yZ4IOd2lpa2lwZWRp
YS5jb22CDHdpa2lib29rLmNvbYINd2lraWJvb2tzLmNvbYINd2lraWVwZGlhLmNv
bYINd2lraWVwZGlhLm9yZ4IOd2lraWlwZWRpYS5vcmeCDndpa2lqdW5pb3IuY29t
gg53aWtpanVuaW9yLm5ldIIOd2lraWp1bmlvci5vcmeCDXdpa2lwZWRpYS5jb20w
EwYDVR0gBAwwCjAIBgZngQwBAgEwggEEBgorBgEEAdZ5AgQCBIH1BIHyAPAAdgDP
EVbu1S58r/OHW9lpLpvpGnFnSrAX7KwB0lt3zsw7CAAAAZLlFWOoAAAEAwBHMEUC
IQDOQfIBwT2aOdrc6rFQG83V5VIy/StPJsBkp/cwmoueIQIgGciNTDnZVQXASCoL
0ajX3BH4DtoXtHn608o3pWUkvpUAdgB9WR4S4XgqexxhZ3xe/fjQh1wUoE6VnrkD
L9kOjC55uAAAAZLlFWNYAAAEAwBHMEUCIQCm2m4aDT5gs4aDws4ImktVBg/8pt2E
uWsR6NKhwtr/6gIgZOncM/2inGNQmuA+GjXPKX7xgmfNgOL0PEYgPL3FaeEwCgYI
KoZIzj0EAwMDZwAwZAIwIBCXYaRy2WTUBfXVdvqxHkwDWOpiwkGfqzjFnSiI/PUj
XyIZ/mSqjLbRqdCKOqmYAjBvqfap85eomWIgj224K1vGbWDk/Ku3ITz968PslQaK
ptg9/G94hhVkiWqqT+n6uF8=
-----END CERTIFICATE-----
 1 s:C = US, O = Let's Encrypt, CN = E5
   i:C = US, O = Internet Security Research Group, CN = ISRG Root X1
   a:PKEY: id-ecPublicKey, 384 (bit); sigalg: RSA-SHA256
   v:NotBefore: Mar 13 00:00:00 2024 GMT; NotAfter: Mar 12 23:59:59 2027 GMT
-----BEGIN CERTIFICATE-----
MIIEVzCCAj+gAwIBAgIRAIOPbGPOsTmMYgZigxXJ/d4wDQYJKoZIhvcNAQELBQAw
TzELMAkGA1UEBhMCVVMxKTAnBgNVBAoTIEludGVybmV0IFNlY3VyaXR5IFJlc2Vh
cmNoIEdyb3VwMRUwEwYDVQQDEwxJU1JHIFJvb3QgWDEwHhcNMjQwMzEzMDAwMDAw
WhcNMjcwMzEyMjM1OTU5WjAyMQswCQYDVQQGEwJVUzEWMBQGA1UEChMNTGV0J3Mg
RW5jcnlwdDELMAkGA1UEAxMCRTUwdjAQBgcqhkjOPQIBBgUrgQQAIgNiAAQNCzqK
a2GOtu/cX1jnxkJFVKtj9mZhSAouWXW0gQI3ULc/FnncmOyhKJdyIBwsz9V8UiBO
VHhbhBRrwJCuhezAUUE8Wod/Bk3U/mDR+mwt4X2VEIiiCFQPmRpM5uoKrNijgfgw
gfUwDgYDVR0PAQH/BAQDAgGGMB0GA1UdJQQWMBQGCCsGAQUFBwMCBggrBgEFBQcD
ATASBgNVHRMBAf8ECDAGAQH/AgEAMB0GA1UdDgQWBBSfK1/PPCFPnQS37SssxMZw
i9LXDTAfBgNVHSMEGDAWgBR5tFnme7bl5AFzgAiIyBpY9umbbjAyBggrBgEFBQcB
AQQmMCQwIgYIKwYBBQUHMAKGFmh0dHA6Ly94MS5pLmxlbmNyLm9yZy8wEwYDVR0g
BAwwCjAIBgZngQwBAgEwJwYDVR0fBCAwHjAcoBqgGIYWaHR0cDovL3gxLmMubGVu
Y3Iub3JnLzANBgkqhkiG9w0BAQsFAAOCAgEAH3KdNEVCQdqk0LKyuNImTKdRJY1C
2uw2SJajuhqkyGPY8C+zzsufZ+mgnhnq1A2KVQOSykOEnUbx1cy637rBAihx97r+
bcwbZM6sTDIaEriR/PLk6LKs9Be0uoVxgOKDcpG9svD33J+G9Lcfv1K9luDmSTgG
6XNFIN5vfI5gs/lMPyojEMdIzK9blcl2/1vKxO8WGCcjvsQ1nJ/Pwt8LQZBfOFyV
XP8ubAp/au3dc4EKWG9MO5zcx1qT9+NXRGdVWxGvmBFRAajciMfXME1ZuGmk3/GO
koAM7ZkjZmleyokP1LGzmfJcUd9s7eeu1/9/eg5XlXd/55GtYjAM+C4DG5i7eaNq
cm2F+yxYIPt6cbbtYVNJCGfHWqHEQ4FYStUyFnv8sjyqU8ypgZaNJ9aVcWSICLOI
E1/Qv/7oKsnZCWJ926wU6RqG1OYPGOi1zuABhLw61cuPVDT28nQS/e6z95cJXq0e
K1BcaJ6fJZsmbjRgD5p3mvEf5vdQM7MCEvU0tHbsx2I5mHHJoABHb8KVBgWp/lcX
GWiWaeOyB7RP+OfDtvi2OsapxXiV7vNVs7fMlrRjY1joKaqmmycnBvAq14AEbtyL
sVfOS66B8apkeFX2NY4XPEYV4ZSCe8VHPrdrERk2wILG3T/EGmSIkCYVUMSnjmJd
VQD9F6Na/+zmXCc=
-----END CERTIFICATE-----
---
Server certificate
subject=CN = wikipedia.com
issuer=C = US, O = Let's Encrypt, CN = E5
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: ECDSA
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 2856 bytes and written 399 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 256 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
DONE
```


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

### Extension fields

It also appears that the x509 v3 extensions on the certificate state what a certificate is allowed to be used for.

I manually copied the entire certificate contents for both the end entity and the intermediate (the sections between "-----BEGIN CERTIFICATE-----" and "-----END CERTIFICATE-----" in the `openssl s_client` output) into files named *end-entity* and *intermediate* for this exercise:

The end entity cert;
```bash
$ cat end-entity | openssl x509 -noout -ext keyUsage,extendedKeyUsage,basicConstraints
X509v3 Key Usage: critical
    Digital Signature
X509v3 Extended Key Usage:
    TLS Web Server Authentication, TLS Web Client Authentication
X509v3 Basic Constraints: critical
    CA:FALSE
```

The intermediate cert;
```bash
$ cat intermediate | openssl x509 -noout -ext keyUsage,extendedKeyUsage,basicConstraints
X509v3 Key Usage: critical
    Digital Signature, Certificate Sign, CRL Sign
X509v3 Extended Key Usage:
    TLS Web Client Authentication, TLS Web Server Authentication
X509v3 Basic Constraints: critical
    CA:TRUE, pathlen:0
```

  And the root;
  ```bash
$ cat /etc/ssl/certs/ISRG_Root_X1.pem | \
  openssl x509 -noout -ext keyUsage,extendedKeyUsage,basicConstraints
X509v3 Key Usage: critical
    Certificate Sign, CRL Sign
X509v3 Basic Constraints: critical
    CA:TRUE
```

This highlights the other observable difference between root, intermediate and end entity certificates:

- Root and intermediate certs will have the CA boolean set to true in the basicConstraints field, and Certificate Sign set in the Key Usage field
- Intermediate certificates may have a pathlen value set in the basicConstraints field denoting whether the certificate should be used to sign intermediates or not
- End-entity certificates will have the CA boolean set to false for basicConstraints, and will not have Certificate Sign set in the keyUsage field

The full and authoritative details of these (and all the other) certificate fields is contained within [rfc5280](https://datatracker.ietf.org/doc/html/rfc5280)

> An important concept here - certificate authorities clearly should conform to these rules to maintain their status (and it appears standards/practice conformity is audited through [CABF](https://cabforum.org/about/information/auditors-and-assessors/audit-criteria/)).<br><br>However, ultimately the client (browser etc.) **must** ensure that the certificate chain they are validating maintains these properties as a line of defence against non-conformity from CAs for whatever reason.<br><br>We will go into this in a bit more detail later in this post, but this BlackHat presentation from the early 2000s ([PDF](https://www.blackhat.com/presentations/bh-dc-09/Marlinspike/BlackHat-DC-09-Marlinspike-Defeating-SSL.pdf) and [video](https://www.youtube.com/watch?v=MFol6IMbZ7Y)) touch on, in the early sections of the presentation, issues at the time emerging from clients not verifying the integrity of the extensions in the chain.
{: .prompt-info }

## what happened before these extensions were added?

This is really hard to find out - I've spent a good few hours searching for anything that outlines the historic approach (prior to the v3 extensions) that stopped anyone who possesses a public CA signed cert from using their private key to sign a subordinate cert, effectively passing it off as an intermediate.

Given that x509 V1 & V2 look to (almost) pre-date the advent of any significant use of SSL/TLS through the early years of the 2000s, it may be that this issue wasn't considered as significant, with x509 V3 extensions being in place as SSL/TLS became prevalent.

## Can I sign a certificate with another that's not intended to be a root/intermediate

To do this, I'll need to convince openssl to ignore the basicConstraints field as part of signing....