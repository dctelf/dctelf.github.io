---
title: What is the end to end process that takes place when a client validates a service certificate
date: 2024-12-06 18:00:00 +/-0000
categories: [Authentication]
tags: [x509, ca, signing, authentication]     # TAG names should always be lowercase
---

I embarked on writing up a post on mTLS client authentication, and effectively started writing a post that first introduces the more common webPKI model of a client authenticating the identity of a service.  Given that this is to some extent an independent topic, I've elected to split that post out as a standalone summary.

## Questions to explore

To further understanding, I'd like to explore the following questions and where applicable demonstrate answers:

- what properties of the service certificate are used to authenticate the service by the client?
- which certificate extensions are applicable to service authentication?

##  The service/server identification model

The model for service (/server) authentication has evolved from the outset of the use of TLS - the typically accepted practices are now articulated within [RFC9525](https://datatracker.ietf.org/doc/html/rfc9525).   This RFC in [section 6](https://datatracker.ietf.org/doc/html/rfc9525#name-verifying-service-identity) outlines the high level themes of service identity verification.  

The primary concepts are:

- That a service has a *presented identifier* (e.g. a DNS name), which typically a client uses to access that service (e.g. it's the DNS name of the website the client/browser is connecting to).  
- This presented identifier (/DNS name) is also contained with a trusted CA signed certificate with a field (the subjectAltName field) of the certificate.  
- The client verifies that the certificate & chain is valid (dates, revocations, signing integrity, extensions on cert use hold etc.)
- The client verifies that the identifier (DNS name) being accessed matches that of the identifier name in the verified certificate
- Along side this, there is an inferred trust in the Certificate Authority that they have satisfied themselves that the certificate has been issued to a service that can demonstrate some form of ownership/control of the *identifier* - [RFC 8555:Introduction](https://datatracker.ietf.org/doc/html/rfc8555) and [the CABF Baseline Requirements section 3.2.2.4](https://cabforum.org/working-groups/server/baseline-requirements/documents/CA-Browser-Forum-TLS-BR-2.1.1.pdf). This forms part of the broad trust model of WebPKI.

#### End to end model
![Service authentication model](assets/img/2024-12-06-svc-auth-model.png){: width="972" height="589"}
_Service authentication model_

<span style="color:#97D077">**Establishing trust between parties in advance**</span>
1. **CA root certificate creation:** A certificate authority generates it's keypair and issues a root (self signed) certificate
2. **WebPKI acceptance and distribution:** OS & Browser vendors trust this CA and are satisfied with it's processes/controls - they agree to distribute the CA certificate to the clients they control/support
3. **Client trust store inclusion:** The CA certificate is installed within the clients truststore (e.g from creation of the original installation media byt the vendor, or through an update etc.)
4. **Service CSR creation:** The service creates a keypair and generates a certificate signing request (with the service identifier, in this case the DNS name, set in the subjectAltName field) and sends to the certificate authority
5. **demonstration of DNS control:** The CA validates by some mechanisms (see [RFC 8555:Introduction](https://datatracker.ietf.org/doc/html/rfc8555)) that the requester has a level of control of the domain in the subjectAltName field and only issues a certificate if so.
6. **Certificate signing:** The CA signs the service certificate, typically with an intermediate certificate signed by the root, and returns the signed certificate (and intermediate in a certificate bundle) back to the service
7. **Signed certificate installation:** The service *installs* the signed certificate (/bundle) on it's service (e.g. on a webserver, or loadbalancer etc.).

    <span style="color:#DDC5C4">**At time of client to service connection**</span>
8. **Client service request:** Client **uses** the service identifier to access the service (e.g. the DNS name in the hyperlink, or typed into the address bar)
9. **Service certificate response:** The service returns the certificate bundled (containing the CA signed certificate) back to the client
10. **Certificate validation:** The client verifies that the certificate & chain is valid (dates, revocations, signing integrity, extensions on cert use hold etc.), validates that the subjectAltName field contains the service identifier (/dnsName) used to access the service and **only** if both validate then allows the connection to progress

> I have omitted some details from this flow for simplicity - e.g. the more complex architectures of certificate authorities & registration authorities, certificate revocation mechanisms and supplemental verifications such as pinning checks etc.
{: .prompt-info }

### Certificate properties

As with the post on [intermediate certificates](/posts/what-makes-an-intermediate-certificate-intermediate/) I'll use the [wikipedia.com](https://wikipedia.com) and look at the certificate chain it presents with `openssl s_client`.

From this output...

```bash
$ echo "GET /" | \
openssl s_client -showcerts -connect www.wikipedia.com:443 -verify_return_error
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

.... I manually copied the entire certificate contents for both the end entity and the intermediate (the sections between "-----BEGIN CERTIFICATE-----" and "-----END CERTIFICATE-----") into files named *end-entity* and *intermediate* for this exercise:

The end entity cert subject and subjectAltName;
```bash
$ cat end-entity | openssl x509 -noout -subject -ext subjectAltName
subject=CN = wikipedia.com
X509v3 Subject Alternative Name:
    DNS:*.en-wp.com, DNS:*.en-wp.org, DNS:*.mediawiki.com, DNS:*.voyagewiki.com, DNS:*.voyagewiki.org, DNS:*.wiikipedia.com, DNS:*.wikibook.com, DNS:*.wikibooks.com, DNS:*.wikiepdia.com, DNS:*.wikiepdia.org, DNS:*.wikiipedia.org, DNS:*.wikijunior.com, DNS:*.wikijunior.net, DNS:*.wikijunior.org, DNS:*.wikipedia.com, DNS:en-wp.com, DNS:en-wp.org, DNS:mediawiki.com, DNS:voyagewiki.com, DNS:voyagewiki.org, DNS:wiikipedia.com, DNS:wikibook.com, DNS:wikibooks.com, DNS:wikiepdia.com, DNS:wikiepdia.org, DNS:wikiipedia.org, DNS:wikijunior.com, DNS:wikijunior.net, DNS:wikijunior.org, DNS:wikipedia.com
```
Note that [overview section of rfc9525](https://datatracker.ietf.org/doc/html/rfc9525#name-overview-of-recommendations) states that **only** the DNS names in the subjectAltName extension field are to be used.  Historically the DN within the subject field of the certificate held the service identifier name - I imagine many clients may continue to accept a DNS name in the subject field for the purposes of backwards compatibility.  [Appendix A. Changes from RFC 6125](https://datatracker.ietf.org/doc/html/rfc9525#name-changes-from-rfc-6125) clarify that this behaviour was updated as of rfc9525.

> We can see in this certificate output that the DNS form of the service identifier (wikipedia.com) is present in the subjectAltName field.  Interestingly (not something I have seen often) rfc9525 outlines the format of other service identifier types that can be held in the subjectAltName field e.g. IP addresses, an SRV name etc.
{: .prompt-info }

### Primary applicable certificate extensions

Alongside the core reliance on subjectAltName to match the service identifier as described above, other certificate fields and extensions are used by clients to verify the requisite degree of trust/assurance in the identity such as;

- **The hierarchical trust up to the CA root through signing** (selected lines from the cert)

```text
Signature Algorithm: ecdsa-with-SHA384
Issuer: C = US, O = Let's Encrypt, CN = E5
Signature Value:
    30:64:02:30:20:10:97:61:a4:72:d9:64:d4:05:f5:d5:76:fa:
    b1:1e:4c:03:58:ea:62:c2:41:9f:ab:38:c5:9d:28:88:fc:f5:
    23:5f:22:19:fe:64:aa:8c:b6:d1:a9:d0:8a:3a:a9:98:02:30:
    6f:a9:f6:a9:f3:97:a8:99:62:20:8f:6d:b8:2b:5b:c6:6d:60:
    e4:fc:ab:b7:21:3c:fd:eb:c3:ec:95:06:8a:a6:d8:3d:fc:6f:
    78:86:15:64:89:6a:aa:4f:e9:fa:b8:5f
```

- **key usage v3 extensions**

Section [4.2.1.3](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.3) and [4.2.1.12](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.12) of rfc5280 outline the context of these extensions.

```text
X509v3 Key Usage: critical
    Digital Signature
X509v3 Extended Key Usage:
    TLS Web Server Authentication, TLS Web Client Authentication
```
- **validity dates**

```text
Validity
    Not Before: Oct 31 23:18:47 2024 GMT
    Not After : Jan 29 23:18:46 2025 GMT
```

- **Certificate constraints**

Section [4.2.1.9](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.9) of rfc5280 on constraints.

```text
X509v3 Basic Constraints: critical
    CA:FALSE
```

Alongside these, there are other certificate attributes that form some role in client assurance of identity and validity...

- Certificate Policies
- Certificate transparency
- OCSP information
- CRL information

> As a footnote - having just completed this post, I've spotted that this [section](https://www.feistyduck.com/library/openssl-cookbook/online/openssl-command-line/examining-public-certificates.html) of the [openssl cookbook](https://www.feistyduck.com/library/openssl-cookbook/) (which is a free online companion to the excellent [bulletproof TLS and PKI](https://www.feistyduck.com/books/bulletproof-tls-and-pki/) book) follows a very similar outline to the summary above, giving further information on the anatomy of x509 public certificates.
{: .prompt-info }