---
title: What makes an intermediate certificate intermediate?
date: 2024-12-02 18:00:00 +/-0000
categories: [PKI]
tags: [x509, ca, signing]     # TAG names should always be lowercase
---

Having spent a little time creating dummy CAs, signing certificates and viewing lots of `openssl x509 -text` output, I came to the realisation that I wasn't entirely clear which x509 certificate attributes I should expect to see that denote the purpose of a certificate.

## Questions to explore

To further understanding, I'd like to explore the following questions and where applicable demonstrate the answers:

- What are the observable differences between root, intermediate, and end-entity certificates
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

From a root certificate perspective, the way `openssl s_client` displays this isn't intuitive (it only displays the certificate bundle presented by the service, but not the root), additionally I have a suspicion that openssl (`OpenSSL 3.0.13`) is always using my machines (Ubuntu) trust store and validating the full chain, so there is no easy way to see the difference between a chain that isn't validated against a root vs. one that is, with openssl.

To show the similar fields from the matching root;

```bash
$ cat /etc/ssl/certs/ISRG_Root_X1.pem | openssl x509 -text -noout | egrep 'Issuer:|Subject:'
        Issuer: C = US, O = Internet Security Research Group, CN = ISRG Root X1
        Subject: C = US, O = Internet Security Research Group, CN = ISRG Root X1
```

Clearly there is a cryptographic mechanism of signing that makes the trust in all this hang together, but the hierarchy is reflected in the chain of subject and issuer - joining these 2 outputs together demonstrates one answer to the first question from above - What are the observable differences between end-entity, intermediate and root certificates:

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

> I believe that, although CAs are highly likely to conform to the basic principles of setting these extension values, the client must always verify the integrity of the chain in terms of these field values.<br><br>I will go into this in a bit more detail later in this post, but this BlackHat presentation from the early 2000s ([PDF](https://www.blackhat.com/presentations/bh-dc-09/Marlinspike/BlackHat-DC-09-Marlinspike-Defeating-SSL.pdf) and [video](https://www.youtube.com/watch?v=MFol6IMbZ7Y)) touch on, in the early sections of the presentation, issues at the time emerging from clients not verifying the integrity of the extensions in the chain.
{: .prompt-info }

## what happened before these extensions were added?

This is really hard to find out - I've spent a good few hours searching for anything that outlines the historic approach (prior to the v3 extensions) that stopped anyone who possesses a public CA signed cert from using their private key to sign a subordinate cert, effectively passing their CA signed certificate off as an intermediate.

Given that x509 V1 & V2 look to (almost) pre-date the advent of any significant use of SSL/TLS through the early years of the 2000s, it may be that this issue wasn't considered significant, with x509 V3 extensions being in place as SSL/TLS became prevalent.  Either way, I can't find a definitive answer, but this appears to be an issue resolved with v3 certs.

## Can I sign a certificate with another that's not intended to be a root/intermediate

I'll start off by creating a dummy CA using the openssl default config.

```bash
$ cd ~/dummyCA/
$ openssl req -x509 -nodes -newkey rsa:2048 -keyout dummyCA.key \
  -sha256 -days 365 -out dummyCA.pem \
  -subj "/CN=dummyCA/"
$ ls
dummyCA.key  dummyCA.pem
```

<details markdown="1">
<summary>
<i>Expand for full `-text` openssl output of this cert</i>

</summary>

```text
$ openssl x509 -in dummyCA.pem -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            3e:44:4c:e5:37:d0:56:4b:2a:ac:4b:4e:77:e5:f2:e2:69:64:27:14
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = dummyCA
        Validity
            Not Before: Dec  8 17:16:03 2024 GMT
            Not After : Dec  8 17:16:03 2025 GMT
        Subject: CN = dummyCA
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:a2:79:0d:9e:1a:a9:82:4d:cc:33:3f:db:bc:43:
                    ab:64:e1:9b:3e:f3:92:d0:da:18:e0:8c:8e:73:e8:
                    b0:93:a7:d7:ae:df:d8:92:29:d4:a0:50:ca:5e:2e:
                    c5:6e:6f:c8:35:89:80:b0:a7:9d:53:2d:64:24:35:
                    08:91:e0:99:77:48:2e:da:93:f0:ea:25:3c:5c:de:
                    a8:a2:d5:e6:b2:f3:a1:15:68:ce:a4:37:26:cf:bb:
                    ba:43:02:d5:ac:7f:a2:43:3c:8e:fb:19:89:05:49:
                    55:07:59:5c:79:b1:b4:51:f8:d1:d4:b3:87:73:bd:
                    b3:9a:9c:e2:8e:b5:76:d5:0b:b5:3f:df:4b:cf:4b:
                    2f:2d:ad:fc:7e:0a:e7:b9:79:ea:20:ec:ca:b6:0b:
                    83:ed:20:e4:1a:18:83:19:f5:73:12:00:5f:a8:7b:
                    b6:f7:18:86:3a:31:d3:39:37:d3:ac:37:0d:ea:f0:
                    33:1e:dc:e1:74:61:8b:50:70:7c:09:60:d4:8f:54:
                    68:c5:a6:4a:1d:d9:c6:05:a7:95:ac:e2:5c:f4:59:
                    a7:61:86:26:63:0a:4f:6e:39:76:d4:a1:34:e7:5c:
                    83:03:d0:a6:fd:27:65:78:a4:27:91:0e:bb:75:00:
                    f9:40:23:32:d1:48:c7:6d:31:9c:0e:9a:98:88:e2:
                    68:9f
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                3A:1A:83:4F:F4:57:F3:E4:39:9C:33:00:48:76:8F:64:67:92:78:40
            X509v3 Authority Key Identifier:
                3A:1A:83:4F:F4:57:F3:E4:39:9C:33:00:48:76:8F:64:67:92:78:40
            X509v3 Basic Constraints: critical
                CA:TRUE
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        74:10:4c:69:6d:25:47:2f:96:0f:75:79:9b:cc:41:a8:47:7d:
        69:33:5f:9a:94:c0:44:b9:60:b9:30:fe:d5:5d:0e:1e:40:bd:
        13:5b:0a:d3:ae:38:21:8c:2b:30:5f:59:51:f9:04:57:4b:48:
        4f:e7:1f:6b:a6:85:44:5b:32:29:20:9a:2e:97:17:81:68:6f:
        7b:a2:48:54:05:2d:74:12:b0:f0:72:fd:39:07:a7:b4:b9:bd:
        c5:61:cd:21:3f:9e:5c:c9:58:f1:a5:a2:91:97:a9:44:36:2c:
        6c:67:50:e6:ac:d2:dd:45:96:77:a7:62:91:2e:cf:cd:6a:b1:
        b4:71:8b:a2:32:45:0e:9c:aa:bc:e7:e6:09:84:04:43:7e:3f:
        46:43:0f:4b:99:f8:80:d1:4f:d1:b4:b7:f0:1a:a2:d8:76:c5:
        e2:57:75:eb:82:a1:f3:73:d3:77:40:e4:a3:74:99:8c:5f:ec:
        e0:d4:52:d7:b4:a5:89:bf:ed:5c:13:b4:8e:f9:a6:02:c7:f9:
        5a:f4:4a:6f:e5:5b:8e:d0:50:27:3a:98:f3:eb:07:a5:ef:9c:
        16:0d:a8:40:d0:91:3e:1a:4e:c7:a1:27:66:42:12:11:11:ec:
        6d:4c:74:d1:02:57:d2:a6:04:98:51:6a:f1:66:13:a5:0b:8c:
        21:fd:12:38
```

</details>


Next, I'll create a CSR for the end-entity (in this example, I won't bother with an intermediate certificate).

```bash
$ openssl req -nodes -newkey rsa:2048 -keyout end-entity.key \
   -out end-entity.csr -subj "/CN=end-entity/"
```

<details markdown="1">
<summary>
<i>Expand for full `-text` openssl output of this CSR</i>

</summary>

```text
$ openssl req -in end-entity.csr -noout -text
Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: CN = end-entity
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:b9:3a:98:e4:73:d8:4b:be:66:9a:d5:15:de:bd:
                    41:aa:76:cd:2e:4f:b9:e2:5d:1e:6b:f0:de:7c:9f:
                    38:14:4b:52:56:3b:15:ac:aa:59:74:68:f8:d5:0b:
                    d4:36:7e:84:e8:b1:0d:16:9a:3d:10:6b:84:5b:ec:
                    13:56:46:06:1b:2f:54:9f:a0:d0:43:96:a3:d3:40:
                    9f:de:7d:6d:c9:46:c5:56:28:b1:d0:1c:2d:a1:74:
                    40:4a:6b:f1:b8:62:7a:05:d3:c9:b3:67:4e:09:84:
                    bf:24:a7:57:97:d0:10:09:20:38:f6:7a:10:6c:19:
                    de:5e:25:0a:cf:00:76:7d:b2:e8:79:6e:62:94:99:
                    ec:15:7b:b6:7d:39:4e:cd:ed:68:06:93:db:7f:b1:
                    36:5d:c2:b1:d6:71:b7:15:65:9d:51:cb:be:43:0c:
                    cd:c7:77:bb:d9:72:2b:4a:ab:c4:0b:fd:21:a9:49:
                    d6:53:a3:1d:40:39:a5:0c:05:c5:a9:d9:36:d7:17:
                    05:5d:9d:72:32:14:fc:66:3d:b5:09:42:fb:49:49:
                    8f:8e:89:c0:2b:60:f1:bc:79:b5:62:80:79:35:f2:
                    2c:11:f4:03:41:6a:f8:65:8c:63:53:c7:b2:b8:df:
                    2f:f9:d4:14:c0:e2:a9:b1:6c:8a:ce:61:72:04:d4:
                    b9:27
                Exponent: 65537 (0x10001)
        Attributes:
            (none)
            Requested Extensions:
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        19:e4:ea:13:69:1e:cb:51:9a:e5:a4:9c:b4:dc:2b:bd:c7:1b:
        f6:5e:9c:f9:b9:75:65:9b:53:d2:c7:6f:96:d7:3b:b7:93:09:
        91:d7:50:20:3e:32:38:3c:f8:36:96:73:b0:64:a6:9a:ae:f4:
        15:4c:9d:e5:df:7a:98:d3:65:85:e2:ba:77:06:08:a9:c6:fa:
        25:7b:0e:95:76:39:fc:76:aa:be:c6:82:b3:c9:a5:63:bc:3a:
        38:48:d2:c6:42:8f:be:3a:c7:7b:03:a1:3c:b5:24:90:99:20:
        7a:10:26:05:26:b3:09:be:31:74:1d:1a:8c:23:f5:b6:ee:55:
        87:24:a8:87:c9:02:84:7e:62:6a:e7:d2:95:80:06:30:fc:f8:
        cd:14:ca:d5:11:47:ca:68:3a:ad:51:13:74:9d:53:43:b1:ef:
        98:af:6e:c9:1f:5b:88:c5:41:97:94:e8:36:45:fe:ef:d5:9b:
        8b:40:30:c8:09:9b:a8:e6:b6:6a:83:2d:24:ac:08:69:99:d5:
        2f:22:3d:f8:d1:6a:45:ad:41:f6:6d:7d:3d:fd:f0:ca:ee:31:
        28:1a:ea:86:3b:93:1d:33:fc:66:d0:6e:76:d2:24:e5:76:57:
        5c:fc:5b:1d:50:67:9e:8d:99:b9:88:e0:39:90:59:34:ab:7e:
        83:b7:8a:c0
```
</details>

Next, I'll sign with my dummyCA, using an extension file to ensure we generate a v3 certificate with the CA:FALSE basicConstraint set ...

```bash
$ cat v3.ext
authorityKeyIdentifier = keyid,issuer
basicConstraints = critical, CA:FALSE
keyUsage = digitalSignature
$ openssl x509 -req -in end-entity.csr -CA dummyCA.pem -CAkey dummyCA.key \
  -CAcreateserial -out end-entity.pem -days 364 -sha256 -extfile v3.ext
Certificate request self-signature ok
subject=CN = end-entity
```

<details markdown="1">
<summary>
<i>Expand for full `-text` openssl output of this signed certificate</i>

</summary>

```text
$ openssl x509 -in end-entity.pem -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            12:c8:b2:0b:cb:e3:4e:a7:3e:a7:12:8f:f2:79:98:e2:df:6f:72:83
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = dummyCA
        Validity
            Not Before: Dec  8 18:10:54 2024 GMT
            Not After : Dec  7 18:10:54 2025 GMT
        Subject: CN = end-entity
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:b9:3a:98:e4:73:d8:4b:be:66:9a:d5:15:de:bd:
                    41:aa:76:cd:2e:4f:b9:e2:5d:1e:6b:f0:de:7c:9f:
                    38:14:4b:52:56:3b:15:ac:aa:59:74:68:f8:d5:0b:
                    d4:36:7e:84:e8:b1:0d:16:9a:3d:10:6b:84:5b:ec:
                    13:56:46:06:1b:2f:54:9f:a0:d0:43:96:a3:d3:40:
                    9f:de:7d:6d:c9:46:c5:56:28:b1:d0:1c:2d:a1:74:
                    40:4a:6b:f1:b8:62:7a:05:d3:c9:b3:67:4e:09:84:
                    bf:24:a7:57:97:d0:10:09:20:38:f6:7a:10:6c:19:
                    de:5e:25:0a:cf:00:76:7d:b2:e8:79:6e:62:94:99:
                    ec:15:7b:b6:7d:39:4e:cd:ed:68:06:93:db:7f:b1:
                    36:5d:c2:b1:d6:71:b7:15:65:9d:51:cb:be:43:0c:
                    cd:c7:77:bb:d9:72:2b:4a:ab:c4:0b:fd:21:a9:49:
                    d6:53:a3:1d:40:39:a5:0c:05:c5:a9:d9:36:d7:17:
                    05:5d:9d:72:32:14:fc:66:3d:b5:09:42:fb:49:49:
                    8f:8e:89:c0:2b:60:f1:bc:79:b5:62:80:79:35:f2:
                    2c:11:f4:03:41:6a:f8:65:8c:63:53:c7:b2:b8:df:
                    2f:f9:d4:14:c0:e2:a9:b1:6c:8a:ce:61:72:04:d4:
                    b9:27
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Authority Key Identifier:
                3A:1A:83:4F:F4:57:F3:E4:39:9C:33:00:48:76:8F:64:67:92:78:40
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Key Usage:
                Digital Signature
            X509v3 Subject Key Identifier:
                48:4F:E9:2C:38:CF:01:EC:3C:EB:57:4E:CC:65:91:91:EF:F8:56:CA
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        9d:54:ba:25:bd:c3:4a:55:6d:5c:7d:d4:d5:e5:9e:5d:10:55:
        bf:55:25:37:99:49:f6:ed:5b:ef:53:f7:ec:a3:bb:a8:d3:a8:
        5e:55:e9:3a:a2:5d:92:d1:5b:c1:69:8b:9b:26:e5:89:99:ae:
        35:43:fd:ea:72:ef:d2:45:da:e7:9e:a0:44:85:4f:55:73:fb:
        e9:7f:ec:75:47:c4:ba:de:55:d2:6c:65:51:ef:72:62:2a:2f:
        5e:b5:7f:a4:99:28:c2:66:28:55:d5:77:19:ca:24:7e:7d:22:
        c2:0c:64:d8:c4:fa:79:4b:bb:63:5c:7e:bc:35:46:7c:a7:32:
        3d:38:35:e6:18:9d:d6:9a:0e:98:7a:59:14:fb:fc:23:8a:25:
        6c:cd:91:18:05:89:92:62:82:39:f0:9a:33:c2:44:1b:bd:3e:
        b7:4d:91:3a:3d:a2:67:9c:91:66:8e:e8:e1:14:52:2b:ea:d7:
        df:f9:e2:4e:07:c3:b5:20:3a:7a:ae:c5:76:2c:06:67:a9:5c:
        2d:8f:95:bf:00:9d:be:71:8c:9a:e6:c7:40:a6:88:8b:5e:e4:
        d1:9e:24:df:60:02:89:ae:c6:f2:1c:9c:8e:ab:27:76:37:77:
        68:91:56:8d:46:ab:79:e8:18:25:26:ac:ca:58:a9:53:01:de:
        2f:ca:d9:59
```

</details>

We can double check that this root and end entity chain verifies...

```bash
$ openssl verify -show_chain -CAfile dummyCA.pem end-entity.pem
end-entity.pem: OK
Chain:
depth=0: CN = end-entity (untrusted)
depth=1: CN = dummyCA
```

Now, in a real world scenario, we would have access to our end-entity private key and this signed end-entity certificate, but would not have access to the CA's private key (or any intermediate private keys), so let's delete our CA private key...

```bash
$ rm dummyCA.key
```

Next, let's create a *fake* end-entity certificate, i.e. one we will aim to sign with out CA signed end-entity certificate.

```bash
$ openssl req -nodes -newkey rsa:2048 -keyout next-end-entity.key \
   -out next-end-entity.csr -subj "/CN=next-end-entity/"
```

<details markdown="1">
<summary>
<i>Expand for full `-text` openssl output of this CSR</i>

</summary>

```text
$ openssl req -in next-end-entity.csr -noout -text
Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: CN = next-end-entity
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:9d:c8:d1:d4:4c:9a:e8:ce:39:60:0e:9f:5c:db:
                    1a:82:c7:15:9d:93:2b:05:e4:9c:a5:7f:1f:7d:a7:
                    c2:8c:ed:a4:74:da:7d:45:ef:a1:38:6e:a9:bb:52:
                    2a:35:18:97:53:c9:30:b9:ea:8e:72:be:ac:c0:29:
                    48:57:33:fe:6a:63:03:9b:fd:71:1b:98:d4:8c:ef:
                    92:43:a9:38:4f:54:b6:29:0c:2b:a9:3d:d1:44:e1:
                    c4:dd:d5:97:a8:cb:ac:77:3e:e1:ab:7f:ff:17:9c:
                    f8:46:87:4c:2b:94:bc:d7:10:09:e1:7c:48:2e:53:
                    d1:39:25:1c:19:a4:30:ac:76:46:7e:45:33:7c:f2:
                    6a:e9:db:8b:1a:57:bb:cb:6c:be:7f:f5:a2:4d:6c:
                    e4:35:1b:f7:81:c8:50:3d:ec:ea:50:ee:d1:6f:90:
                    b9:e9:43:37:22:5c:22:88:44:68:46:e1:76:ad:09:
                    ff:bf:06:bd:d6:f9:cd:06:c7:cf:10:b1:ff:96:da:
                    a3:db:45:21:9d:bc:2e:ae:f8:48:62:92:19:13:33:
                    d5:84:92:85:05:b4:08:79:a5:8e:b0:78:c2:a1:6d:
                    6d:a3:7c:17:1f:f0:b9:54:66:b2:3c:02:70:9c:e3:
                    25:ff:f6:ba:df:6d:57:11:46:7f:47:2f:ed:04:c5:
                    c1:c3
                Exponent: 65537 (0x10001)
        Attributes:
            (none)
            Requested Extensions:
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        5d:74:69:da:46:83:65:4a:ae:c0:25:ba:a9:8f:d2:ea:aa:9d:
        2e:bb:2f:37:87:02:38:c1:e6:ef:d4:aa:7a:cb:2a:21:aa:c1:
        fe:d1:75:7c:29:64:d4:23:31:d1:b1:01:d9:bb:bb:a3:3a:a6:
        ff:15:48:2f:42:b8:c6:10:be:11:cd:ef:b1:92:00:fa:23:c5:
        5b:33:40:a3:d5:5f:90:45:29:87:d9:b6:72:a5:77:89:33:03:
        c4:b1:95:29:65:88:a5:99:18:b9:f7:de:25:7a:60:ff:57:44:
        b7:43:0c:8f:bc:8d:8f:9b:c0:03:75:37:26:d7:d2:bb:cb:f9:
        dc:a7:69:95:bc:c9:39:ca:7c:88:71:98:db:b1:66:c4:4b:f5:
        22:5b:25:60:59:c2:ad:4a:41:2f:64:b0:9a:f9:3a:89:24:07:
        61:72:ac:19:47:e2:2c:b1:71:4d:12:14:be:d9:a6:a0:f4:05:
        1b:8b:e1:7e:4e:47:e6:8b:f3:a4:0e:bd:af:1f:cc:ba:e6:1a:
        e2:89:ab:16:f9:31:9a:c6:e8:78:1a:62:80:e9:dd:de:26:33:
        42:48:7e:15:81:de:1e:98:95:6e:fc:98:5a:e5:9e:63:0d:1a:
        e4:db:de:67:66:02:32:45:a6:3d:1a:73:03:46:31:2d:24:4d:
        f8:e6:7a:d0
```

</details>

Next, I'll make an attempt to sign this certificate, but I'll try to use my CA signed cert as the CA (this has "CA:FALSE" set, so will be interesting to see what openssl does) - I'll use the same v3.ext as above...

```bash
$ openssl x509 -req -in next-end-entity.csr -CA end-entity.pem -CAkey end-entity.key \
  -CAcreateserial -out next-end-entity.pem -days 364 -sha256 -extfile v3.ext
Certificate request self-signature ok
subject=CN = end-entity
```

I was expecting openssl to put up more of a fight here (e.g. some warning or error), but this looks to have signed fine, disregarding the CA:FALSE value.

```bash
$ openssl x509 -in next-end-entity.pem -noout -text | egrep '^\s+Issuer:|^\s+Subject:'
        Issuer: CN = end-entity
        Subject: CN = next-end-entity
```


<details markdown="1">
<summary>
<i>Expand for full `-text` openssl output of this signed certificate</i>

</summary>

```text
$ openssl x509 -in next-end-entity.pem -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            38:49:c3:4b:cc:2c:b3:6e:03:7a:91:a9:96:45:11:6d:8d:bd:44:75
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = end-entity
        Validity
            Not Before: Dec  8 20:03:02 2024 GMT
            Not After : Dec  7 20:03:02 2025 GMT
        Subject: CN = next-end-entity
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:9d:c8:d1:d4:4c:9a:e8:ce:39:60:0e:9f:5c:db:
                    1a:82:c7:15:9d:93:2b:05:e4:9c:a5:7f:1f:7d:a7:
                    c2:8c:ed:a4:74:da:7d:45:ef:a1:38:6e:a9:bb:52:
                    2a:35:18:97:53:c9:30:b9:ea:8e:72:be:ac:c0:29:
                    48:57:33:fe:6a:63:03:9b:fd:71:1b:98:d4:8c:ef:
                    92:43:a9:38:4f:54:b6:29:0c:2b:a9:3d:d1:44:e1:
                    c4:dd:d5:97:a8:cb:ac:77:3e:e1:ab:7f:ff:17:9c:
                    f8:46:87:4c:2b:94:bc:d7:10:09:e1:7c:48:2e:53:
                    d1:39:25:1c:19:a4:30:ac:76:46:7e:45:33:7c:f2:
                    6a:e9:db:8b:1a:57:bb:cb:6c:be:7f:f5:a2:4d:6c:
                    e4:35:1b:f7:81:c8:50:3d:ec:ea:50:ee:d1:6f:90:
                    b9:e9:43:37:22:5c:22:88:44:68:46:e1:76:ad:09:
                    ff:bf:06:bd:d6:f9:cd:06:c7:cf:10:b1:ff:96:da:
                    a3:db:45:21:9d:bc:2e:ae:f8:48:62:92:19:13:33:
                    d5:84:92:85:05:b4:08:79:a5:8e:b0:78:c2:a1:6d:
                    6d:a3:7c:17:1f:f0:b9:54:66:b2:3c:02:70:9c:e3:
                    25:ff:f6:ba:df:6d:57:11:46:7f:47:2f:ed:04:c5:
                    c1:c3
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Authority Key Identifier:
                48:4F:E9:2C:38:CF:01:EC:3C:EB:57:4E:CC:65:91:91:EF:F8:56:CA
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Key Usage:
                Digital Signature
            X509v3 Subject Key Identifier:
                B2:E8:5E:19:E8:F7:AD:54:ED:B6:3F:76:4D:44:B7:4C:30:A1:C5:07
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        18:ec:8e:03:ab:1e:f7:b1:fd:89:73:7d:1c:4a:2d:c1:e4:0d:
        24:8c:67:73:b1:8d:a0:db:17:16:6e:b2:36:c0:9c:a0:2d:46:
        75:83:17:b0:5e:15:4d:1d:60:d9:3e:a0:2c:8e:ef:62:f3:8d:
        cc:fc:2b:c0:27:9e:9a:65:d1:a4:af:47:82:d4:01:49:7a:a9:
        d6:65:bf:ee:4d:2c:61:72:7c:28:1b:28:e8:03:25:c5:28:f3:
        64:64:d4:62:b9:cd:eb:d7:0f:4f:9a:2c:95:da:95:6f:ce:56:
        1e:63:c0:b5:88:16:c9:03:ff:b8:8f:85:e6:22:1c:c1:4a:d1:
        31:08:04:fe:ce:08:4c:a2:f7:e2:9a:e2:be:b9:fd:0d:74:f1:
        12:93:a9:86:46:f2:94:5b:e0:bb:15:1f:f0:85:b9:73:e7:f9:
        7b:69:71:31:48:57:a4:22:7a:bf:31:77:80:f0:45:cd:91:76:
        b6:aa:07:f0:d0:8d:bf:67:0a:7f:40:a3:cd:29:e3:48:e5:11:
        7a:55:6a:9b:02:52:0d:bc:29:09:65:8c:1c:48:63:8a:7a:dc:
        ef:9b:d6:ff:d9:b1:83:3c:c7:4b:5d:4e:c3:f0:6e:ca:96:9e:
        52:50:4b:2c:5e:b3:97:35:8a:9a:ea:bb:38:8f:09:eb:83:fc:
        85:58:74:cd
```

</details>

But, attempting to use openssl verify throws an error due to the key usage field of the (fake) *intermediate* ...

```bash
$ openssl verify -show_chain -CAfile dummyCA.pem -untrusted end-entity.pem next-end-entity.pem
CN = end-entity
error 79 at 1 depth lookup: invalid CA certificate
CN = end-entity
error 32 at 1 depth lookup: key usage does not include certificate signing
error next-end-entity.pem: verification failed
```

So, in answering the original question;

- Can I sign a certificate with another that's not intended to be a root/intermediate e.g. asking openssl to ignore these v3 extensions?

Yes - I can use openssl to sign a certificate with another signed certificate that has CA:FALSE set in it's extensions, and I didn't have to ask openssl to ignore anything, it signed without question.  I'm curious if there are additional extensions that might create different behaviour, so in future I might generate a CA signed cert (from let's encrypt or similar) to check if the behaviour is the same.

## Is enforcement ultimately a responsibility of the validating party

Having worked through these examples above, and given that no other party plays a part when taking an already signed certificate and using it to sign a subordinate, it is apparent that ultimate responsibility lies with the client to verify that various aspects of a certificate chain meet the expected/required characteristics, and that errors should be thrown where an issue is encountered.

There may be other features of the public PKI architecture which may also play a role in identifying certificates for domains not signed by expected parties (for now I'm guessing some elements of pinning or CT logs) - I'll attempt to explore this in a later post and may come back to this topic.

## How do different clients behave when attempting to validate a broken certificate chain 

Trying openssl s_server/s_client first... Starting a service with s_server:

```bash
$ openssl s_server -key next-end-entity.key -cert next-end-entity.pem \
  -cert_chain end-entity.pem -accept 4433 -www
Using default temp DH parameters
ACCEPT

```

Then connecting with s_client and finding the verification errors...

```bash
$ echo "GET /" | openssl s_client -connect localhost:4433 -CAfile dummyCA.pem 2>&1 | \
  egrep '^verify error:|^Verification error'
verify error:num=79:invalid CA certificate
verify error:num=26:unsuitable certificate purpose
verify error:num=32:key usage does not include certificate signing
Verification error: key usage does not include certificate signing
```

Clearly, verification errors are identified.

I'll try with Chrome next, although this is a little more involved - I need to address 2 things to ensure I only see the errors caused by the certificate chain signing anomaly, rather than other unrelated errors:

1. I'll need to create a new *next-end-entity* certificate that has the subjectAltName field set with `localhost`
2. I'll need to install my dummyCA root in my truststore

For this first item I'll go back and create a new *next-end-entity* certificate for *next-end-entity2* with the subjectAltName field added...

```bash
$ openssl req -nodes -newkey rsa:2048 -keyout next-end-entity2.key \
   -out next-end-entity2.csr -subj "/CN=next-end-entity2/" \
   -addext "subjectAltName = DNS:localhost"
```

<details markdown="1">
<summary>
<i>Expand for full `-text` openssl output of this CSR</i>

</summary>

```text
$ openssl req -in next-end-entity2.csr -noout -text
Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: CN = next-end-entity2
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c5:1c:c6:70:ec:20:7c:cf:0c:93:01:d3:2a:67:
                    ce:24:f7:5c:07:20:95:48:6e:df:1d:2b:ff:85:c8:
                    d2:12:e8:3a:13:71:02:38:e5:93:a5:06:5b:44:74:
                    0b:f1:9c:5c:e7:2e:34:8e:1f:bd:75:ef:d2:3a:01:
                    1d:15:38:32:db:72:5b:dd:17:f0:a3:bc:1b:47:97:
                    6e:80:38:76:0e:08:0a:3a:80:93:35:70:97:57:20:
                    49:26:15:5f:58:58:e6:0a:64:77:8e:ac:fb:0d:8c:
                    55:7e:57:dd:70:83:d1:aa:21:fa:bd:43:f6:a5:80:
                    bd:e9:d3:c0:2b:35:48:29:0d:84:3c:7c:33:a9:e3:
                    f1:b3:b9:6b:f7:e8:a8:5c:00:da:da:bf:97:e2:c8:
                    77:cc:bf:b9:bf:86:64:be:36:81:4a:88:03:e0:ca:
                    e9:3a:49:bb:1e:e8:10:0e:71:ab:75:be:64:b3:cb:
                    cf:22:ac:3a:f3:d6:bb:25:6c:5a:f6:1a:51:16:39:
                    7e:d9:68:6c:e1:ec:25:31:7d:4b:9b:eb:7a:cd:ba:
                    6b:9e:78:89:4b:2c:dc:e8:eb:4d:b7:01:ed:39:e5:
                    09:43:c2:95:bf:23:8a:aa:91:74:15:eb:d9:5e:89:
                    38:40:b1:eb:11:5b:d8:fc:c1:cb:6c:48:39:b9:86:
                    2b:13
                Exponent: 65537 (0x10001)
        Attributes:
            Requested Extensions:
                X509v3 Subject Alternative Name:
                    DNS:localhost
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        69:70:91:c0:71:c6:2d:e3:a7:ce:3c:a5:fb:1d:db:33:f6:ab:
        46:c0:ee:ed:29:a8:ee:25:75:97:14:34:c9:ef:99:47:95:44:
        26:89:5a:f2:e4:fb:e6:f7:86:66:ae:0d:21:13:4d:72:9f:d1:
        11:e6:6a:3a:ff:5f:d7:33:5c:70:91:d7:f6:6d:79:32:0e:c2:
        ac:02:e7:e7:11:5e:6f:2a:7f:ad:b1:8e:e3:74:01:f9:f7:c4:
        7a:12:bf:cf:63:c2:8d:39:0c:3a:0d:1c:72:d8:8c:5a:d6:9b:
        11:e9:64:4c:67:c7:cb:77:d9:d1:12:4e:e2:28:5f:cb:f2:4b:
        dc:26:20:12:30:07:c2:f5:17:bd:21:59:a2:48:4f:6d:bf:88:
        93:ea:94:f5:ff:74:de:b9:a4:72:59:92:60:eb:a5:55:6e:23:
        3b:82:3a:e5:7f:bd:25:30:25:e3:59:dd:88:03:43:a7:1a:1b:
        fe:7d:e4:71:f4:46:47:90:ec:2d:20:10:1f:a0:7e:72:ae:e9:
        da:ad:95:07:a5:e8:a6:31:c3:79:5c:78:64:d1:44:77:c9:d3:
        53:4a:c9:e3:63:fe:73:1a:1d:1c:4e:26:58:cd:9e:b8:b5:39:
        1d:0c:9b:9b:b4:d6:00:7b:7c:9a:01:26:f9:47:46:c5:f0:95:
        07:d5:a7:98
```

</details>

Next, I'll  sign this certificate using my CA signed cert as the CA - I'll use the same v3.ext as above but ask to copy the CSR extensions to bring in the subjectAltName field...

```bash
$ openssl x509 -req -in next-end-entity2.csr -CA end-entity.pem -CAkey end-entity.key \
  -CAcreateserial -out next-end-entity2.pem -days 364 -sha256 \
  -extfile v3.ext -copy_extensions copy

```

<details markdown="1">
<summary>
<i>Expand for full `-text` openssl output of this signed certificate</i>

</summary>

```text
$ openssl x509 -in next-end-entity2.pem -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            38:49:c3:4b:cc:2c:b3:6e:03:7a:91:a9:96:45:11:6d:8d:bd:44:77
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = end-entity
        Validity
            Not Before: Dec  8 21:24:13 2024 GMT
            Not After : Dec  7 21:24:13 2025 GMT
        Subject: CN = next-end-entity2
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c5:1c:c6:70:ec:20:7c:cf:0c:93:01:d3:2a:67:
                    ce:24:f7:5c:07:20:95:48:6e:df:1d:2b:ff:85:c8:
                    d2:12:e8:3a:13:71:02:38:e5:93:a5:06:5b:44:74:
                    0b:f1:9c:5c:e7:2e:34:8e:1f:bd:75:ef:d2:3a:01:
                    1d:15:38:32:db:72:5b:dd:17:f0:a3:bc:1b:47:97:
                    6e:80:38:76:0e:08:0a:3a:80:93:35:70:97:57:20:
                    49:26:15:5f:58:58:e6:0a:64:77:8e:ac:fb:0d:8c:
                    55:7e:57:dd:70:83:d1:aa:21:fa:bd:43:f6:a5:80:
                    bd:e9:d3:c0:2b:35:48:29:0d:84:3c:7c:33:a9:e3:
                    f1:b3:b9:6b:f7:e8:a8:5c:00:da:da:bf:97:e2:c8:
                    77:cc:bf:b9:bf:86:64:be:36:81:4a:88:03:e0:ca:
                    e9:3a:49:bb:1e:e8:10:0e:71:ab:75:be:64:b3:cb:
                    cf:22:ac:3a:f3:d6:bb:25:6c:5a:f6:1a:51:16:39:
                    7e:d9:68:6c:e1:ec:25:31:7d:4b:9b:eb:7a:cd:ba:
                    6b:9e:78:89:4b:2c:dc:e8:eb:4d:b7:01:ed:39:e5:
                    09:43:c2:95:bf:23:8a:aa:91:74:15:eb:d9:5e:89:
                    38:40:b1:eb:11:5b:d8:fc:c1:cb:6c:48:39:b9:86:
                    2b:13
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Alternative Name:
                DNS:localhost
            X509v3 Authority Key Identifier:
                48:4F:E9:2C:38:CF:01:EC:3C:EB:57:4E:CC:65:91:91:EF:F8:56:CA
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Key Usage:
                Digital Signature
            X509v3 Subject Key Identifier:
                BD:5F:C1:84:2B:B3:27:AE:5E:5B:D9:A7:B1:3E:6B:6B:6E:38:68:80
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        13:78:a6:9c:49:3e:56:a3:a2:30:31:16:fa:4a:80:af:33:e3:
        eb:6f:ae:be:96:89:6b:b1:66:2e:c0:be:93:83:56:9e:39:32:
        39:84:a7:6f:70:e1:1d:fa:26:11:90:a5:75:97:40:37:0e:e4:
        ad:82:3f:ca:28:a9:b6:da:64:e9:fc:e3:6d:47:f9:bb:93:80:
        d8:f5:3e:3d:fa:2c:cf:d4:02:5e:63:84:b0:37:94:23:c3:f0:
        b7:6e:e3:0a:bb:9c:50:1c:96:d8:15:a8:1c:a0:0b:f9:7a:c9:
        50:12:4b:9d:a3:b0:6a:0c:4b:ef:ed:e5:93:31:a5:02:5c:57:
        6f:4b:57:4e:0f:20:17:c1:c5:cb:ed:0e:0b:50:7e:a7:a3:9c:
        42:c0:3d:78:ab:97:1f:0b:85:b6:42:ba:4b:f1:25:d0:f1:ec:
        bf:f4:a4:1c:c8:41:5a:f5:ba:c0:f2:41:0d:18:da:60:62:0b:
        0c:36:d7:1a:4e:6e:10:2f:91:f9:d8:59:f2:5c:f7:21:16:f8:
        e0:46:c7:85:b2:33:23:97:64:aa:f7:b1:38:1c:73:1b:8a:ec:
        f1:cd:8a:3a:45:89:7f:1a:fd:25:c6:40:e0:26:aa:2a:a5:4c:
        a1:6c:ea:14:05:fc:d7:5f:6f:ad:b0:c2:3f:e3:99:13:ed:8a:
        47:9b:99:40
```

</details>

We now have an end-entity certificate, signed with our invalid intermediate, CA signed cert, with subjectAltName set as localhost.

Starting s_server with the new end-entity certificate `next-end-entity2.pem`

```bash
$ openssl s_server -key next-end-entity2.key -cert next-end-entity2.pem \
  -cert_chain end-entity.pem -accept 4433 -www
Using default temp DH parameters
ACCEPT
```

I added the dummyCA file into my machines truststore (no need to add screenshots here - fairly standard practice) then connected from Chrome and Firefox to the s_server service...

#### Chrome
![Chrome error](/assets/img/2024-12-02-cert-error1.png){: width="972" height="589" .w-75 .normal}
_Chrome error_

#### Firefox
![Firefox error](/assets/img/2024-12-02-cert-error2.png){: width="972" height="589" .w-75 .normal}
_Firefox error_

Notably, in both examples, neither browser offers the option to ignore this error and proceed anyway (perhaps indicative of how much of a mess an invalid signing certificate is).

So, in answering the original question;

- How do different clients behave when attempting to validate a broken certificate chain

All of `openssl s_client`, Chrome and Firefox throw various errors, with the browsers not offering *ignore and proceed anyway* type behaviour by the user for this form of error.