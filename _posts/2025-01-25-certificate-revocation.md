---
title: What are the approaches to certificate revocation?
date: 2025-01-24 18:00:00 +/-0000
categories: [Revocation]
tags: [x509, revocation, pki]
math: true
---

Certificate revocation has always felt a bit opaque to me - I'm aware that the from a WebPKI perspective, browser behaviour has been varied and has not necessarily made revocation checking a reliable feature of the trust model.  I'd like to understand a bit more of the internal workings of revocation mechanisms and what the current CA and browser approach is.

## Questions to explore

- What are certificate revocation lists (CRLs)?
    - How are CRL lists formed and validated?
- What is the Online Certificate Status Protocol (OCSP)?
    - What is the request and response form for the protocol?
- What is OCSP stapling?
- What's the prevalent approach these days - are CRLs making a return?


## Certificate revocation lists (CRLs)

### CRL examples

I spent some time looking for examples of demonstration services (e.g. the very useful [badssl](https://badssl.com/)) or certificate authorities (e.g. [Let's encrypt](https://letsencrypt.org/certificates/) ) - due to the changing prevalent approaches, the availability of revocations published in CRLs is variable - further details below.

The most reliable example set of working, CA produced CRLs I could find for demonstration purposes are published by [Digicert](https://www.digicert.com/kb/digicert-root-certificates.htm) - note that I'm not advocating the use of Digicert, but their demo certs are clearly published for this sort of evaluation and have populated CRLs.

I'll use the `DigiCert TLS RSA4096 Root G5` demo sites, specifically:

- the active demo certificate [https://digicert-tls-rsa4096-root-g5.chain-demos.digicert.com/](https://digicert-tls-rsa4096-root-g5.chain-demos.digicert.com/)
- the revoked certificate [https://digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com/](https://digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com/)

Following the hierarchical process from a client connecting to a service (using `openssl s_client`).

```shell
$ echo "GET /" | openssl s_client \
-connect digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com:443 \
-showcerts 2>&1 | \
egrep "^depth"
```
```text
depth=2 C = US, O = "DigiCert, Inc.", CN = DigiCert TLS RSA4096 Root G5
depth=1 C = US, O = "DigiCert, Inc.", CN = DigiCert G5 TLS RSA4096 SHA384 2021 CA1
depth=0 jurisdictionC = US, jurisdictionST = Utah, businessCategory = Private Organization, serialNumber = 5299537-0142, C = US, ST = Utah, L = Lehi, O = "DigiCert, Inc.", CN = digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com
```

<details markdown="1">
<summary>
<i>Expand for full `-showcerts` openssl output of this connection</i>

</summary>

```text
$ echo "GET /" | openssl s_client \
-connect digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com:443 \
-showcerts
CONNECTED(00000003)
depth=2 C = US, O = "DigiCert, Inc.", CN = DigiCert TLS RSA4096 Root G5
verify return:1
depth=1 C = US, O = "DigiCert, Inc.", CN = DigiCert G5 TLS RSA4096 SHA384 2021 CA1
verify return:1
depth=0 jurisdictionC = US, jurisdictionST = Utah, businessCategory = Private Organization, serialNumber = 5299537-0142, C = US, ST = Utah, L = Lehi, O = "DigiCert, Inc.", CN = digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com
verify return:1
---
Certificate chain
 0 s:jurisdictionC = US, jurisdictionST = Utah, businessCategory = Private Organization, serialNumber = 5299537-0142, C = US, ST = Utah, L = Lehi, O = "DigiCert, Inc.", CN = digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com
   i:C = US, O = "DigiCert, Inc.", CN = DigiCert G5 TLS RSA4096 SHA384 2021 CA1
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Jan  6 00:00:00 2025 GMT; NotAfter: Feb  4 23:59:59 2025 GMT
-----BEGIN CERTIFICATE-----
MIIIkTCCBnmgAwIBAgIQDoi0a+K6cIv3TymPbhYuWjANBgkqhkiG9w0BAQsFADBY
MQswCQYDVQQGEwJVUzEXMBUGA1UEChMORGlnaUNlcnQsIEluYy4xMDAuBgNVBAMT
J0RpZ2lDZXJ0IEc1IFRMUyBSU0E0MDk2IFNIQTM4NCAyMDIxIENBMTAeFw0yNTAx
MDYwMDAwMDBaFw0yNTAyMDQyMzU5NTlaMIHuMRMwEQYLKwYBBAGCNzwCAQMTAlVT
MRUwEwYLKwYBBAGCNzwCAQITBFV0YWgxHTAbBgNVBA8MFFByaXZhdGUgT3JnYW5p
emF0aW9uMRUwEwYDVQQFEww1Mjk5NTM3LTAxNDIxCzAJBgNVBAYTAlVTMQ0wCwYD
VQQIEwRVdGFoMQ0wCwYDVQQHEwRMZWhpMRcwFQYDVQQKEw5EaWdpQ2VydCwgSW5j
LjFGMEQGA1UEAxM9ZGlnaWNlcnQtdGxzLXJzYTQwOTYtcm9vdC1nNS1yZXZva2Vk
LmNoYWluLWRlbW9zLmRpZ2ljZXJ0LmNvbTCCASIwDQYJKoZIhvcNAQEBBQADggEP
ADCCAQoCggEBANWQXL5GClrN5ymSzn6NGzg90D9Y5J1oGkB1VpzFKVs2oGW1YtOr
N5ASjpS39CHy8UBMwUTtW7+M+TTOZa5Dt/Jb5IN19qVHYeueBIXi/caWgk3kHDG/
T8h5gSPBhpP1ya132ELga7Oq/c24ddTvV7L0FB4rSdKH7CxshvhXNjElwdFVCOvy
yXqCnolr6xeYuetRGWwE5CDe0srDdPNxvN/DlHeqq+qNOuAKV12pFwOL/NRjujk5
Pc0v2wYipjefLzqPxM9rVyVPJXf4qvO9oLqs8e2c3bcyO8oBDRaXvw65kwNT05vJ
wfO0DmjvNkXD+XpPCRhEq0ZS2OIH2tNxMeUCAwEAAaOCA74wggO6MB8GA1UdIwQY
MBaAFK66lDO67zdNC9cY70rkoQ28B7ZzMB0GA1UdDgQWBBR1tUXatukFuWg989VL
JaMmHKl/fjBIBgNVHREEQTA/gj1kaWdpY2VydC10bHMtcnNhNDA5Ni1yb290LWc1
LXJldm9rZWQuY2hhaW4tZGVtb3MuZGlnaWNlcnQuY29tMEoGA1UdIARDMEEwCwYJ
YIZIAYb9bAIBMDIGBWeBDAEBMCkwJwYIKwYBBQUHAgEWG2h0dHA6Ly93d3cuZGln
aWNlcnQuY29tL0NQUzAOBgNVHQ8BAf8EBAMCBaAwHQYDVR0lBBYwFAYIKwYBBQUH
AwEGCCsGAQUFBwMCMIGbBgNVHR8EgZMwgZAwRqBEoEKGQGh0dHA6Ly9jcmwzLmRp
Z2ljZXJ0LmNvbS9EaWdpQ2VydEc1VExTUlNBNDA5NlNIQTM4NDIwMjFDQTEtMS5j
cmwwRqBEoEKGQGh0dHA6Ly9jcmw0LmRpZ2ljZXJ0LmNvbS9EaWdpQ2VydEc1VExT
UlNBNDA5NlNIQTM4NDIwMjFDQTEtMS5jcmwwgYUGCCsGAQUFBwEBBHkwdzAkBggr
BgEFBQcwAYYYaHR0cDovL29jc3AuZGlnaWNlcnQuY29tME8GCCsGAQUFBzAChkNo
dHRwOi8vY2FjZXJ0cy5kaWdpY2VydC5jb20vRGlnaUNlcnRHNVRMU1JTQTQwOTZT
SEEzODQyMDIxQ0ExLTEuY3J0MAwGA1UdEwEB/wQCMAAwggF9BgorBgEEAdZ5AgQC
BIIBbQSCAWkBZwB3AE51oydcmhDDOFts1N8/Uusd8OCOG41pwLH6ZLFimjnfAAAB
lDxBhrEAAAQDAEgwRgIhALyyYeby0C4+3FVz1fTJFHmAfqjAhvTimD+uGxRkfwGa
AiEA9/QWN1cYsWN9Bvj2Weh08wHIF4jdiJBrT+SnSOOaZZwAdQBzICIPCBaK+fPE
posKsmqaSgDu9XeFighNBQDUpUJEWQAAAZQ8QYanAAAEAwBGMEQCIB7P5N2eCp96
vBdmk46zqNE+M1XJULkN5pKOAJlGZdoBAiBJ1OhB5dsPm3XNOA2jg4KRoqWiDqws
E6Q5LO39r83c0AB1AObSMWNAd4zBEEEG13G5zsHSQPaWhIb7uocyHf0eN45QAAAB
lDxBhsQAAAQDAEYwRAIge/xfS+pXPz7gT7sBf7irBW9rEbViohcIkzFavEH2BMEC
IAaYumnUDwt6+w7gE2tbAxeXPNQ7oM0fxl9Xrk11/N2MMA0GCSqGSIb3DQEBCwUA
A4ICAQA2CWF9cktmj86m7m32Yvx/4c0IIciw32z+tLlIlETtCWPkmjCgDPg/wfwo
G8chKiJOedR4g0KuknF+4nqwnGEqObFPT1O+Y/rj7sMHpwI+bPhmG+OMDlAPRCbF
6Qf+wyBy1QCN32ij7cOkBIKjw12mgNnb7eUJ32N37wv/JxSUZ6GnuMq7DVWEfMf8
aynoxVfbVDIKfO/6IVWaneh9daDwO+vlABHArRSLzkwm7wWdA8TgZoU8fx6M1uYJ
t5b6BEY0EDOB/KJQPUkFb3dwaAvZc3iLN1zACqBAotU2gY1S3pTlMy4YHxZpCqel
dF6gae39Bt+CTEcYInduSxZob7+rBV/naDf/qwwBi0w3T76sWTK7a23WCDao0sKb
BktkYgIitWUXI4LNyb/DtikN7OOF1CMtG9/69MuvZgqeCQL21CeGE94oM5EuKESu
j7GRkJ6rXlr9VEOm1nPBlDJf1JoskBecrEz0RvUmAR6C0G0a1ZZZZSGh2t1j6VQB
aw19FZrF88V9obAc99DrgdpP6fuWrQRZ5hAzRXpJ0IZ9SfKG4P2+gTiUuMonHbhL
OA7LabFXDpaqruMLurQbb69Pg1aPrwf/JiSgWbChnj5f1JZoxa9er+4hw/NUIsge
9D9vqbAi+oA3vKX39GiLZVixe9XDbIfgKIBB99qrSLAwsoVHpg==
-----END CERTIFICATE-----
 1 s:C = US, O = "DigiCert, Inc.", CN = DigiCert G5 TLS RSA4096 SHA384 2021 CA1
   i:C = US, O = "DigiCert, Inc.", CN = DigiCert TLS RSA4096 Root G5
   a:PKEY: rsaEncryption, 4096 (bit); sigalg: RSA-SHA384
   v:NotBefore: Apr 14 00:00:00 2021 GMT; NotAfter: Apr 13 23:59:59 2031 GMT
-----BEGIN CERTIFICATE-----
MIIGuzCCBKOgAwIBAgIQDmRY51TsnMe6yDIx1flNWDANBgkqhkiG9w0BAQwFADBN
MQswCQYDVQQGEwJVUzEXMBUGA1UEChMORGlnaUNlcnQsIEluYy4xJTAjBgNVBAMT
HERpZ2lDZXJ0IFRMUyBSU0E0MDk2IFJvb3QgRzUwHhcNMjEwNDE0MDAwMDAwWhcN
MzEwNDEzMjM1OTU5WjBYMQswCQYDVQQGEwJVUzEXMBUGA1UEChMORGlnaUNlcnQs
IEluYy4xMDAuBgNVBAMTJ0RpZ2lDZXJ0IEc1IFRMUyBSU0E0MDk2IFNIQTM4NCAy
MDIxIENBMTCCAiIwDQYJKoZIhvcNAQEBBQADggIPADCCAgoCggIBAMnB9AAUezLy
ZIJWqclXdCRODlmfOnlRYNvE3OR/WaHipH5TnngNfRz4nT97RDaZx9Mc5Sdo9c/R
7N7juzv/5XSQIRYp488dRYQOuej2QkdfDKHDBunwqa7shUpm+2NXouQbQqNi3Tkf
NURyVFntX+tL014KdjNr0w5KZXAzCsHTie0ZPb9JMn4MQfzKcfHcLDtytiHxCE5M
OjEpm6O4YDQgEoPvBXTvjD/SQu4JM+f4jRhPWFzdssuTEcyGy9po/0h2xoCP/exh
e8EabVWhqSgz5UWAG1Wn1gLsFzQwR3fYy882OiRJ+uqisbi+QIsD5Dhgmxfo4Voo
vUcvairFFdDwQp7nTwPL1p5iZ6u2OuZS/HI0d1Yj5p9aYpi6nZR+S+iJD0hBqtSw
EHmmAFzGgTwDBJqHKzanwvsyiUeGP7lmd323M8ZcRKGvRBjH8Caa4vZpkcYaKGAV
K/gDUr5vEUAfs30r0NC8cQpPLaLfkA/+tJOvSpRPiPkHk/I7xMq1bJ4zcdlqZ6CC
5hw76litkmt2mFFCg0/JtApRQavIfoaEvwrqxwTxg29ZjvE2tWo8YhwPRf4mogjQ
XsbgXLksXThVv5oVjhzKrrydwU1KNbzmmZNRWesFGYhq/aUCEmpROkhvcHqj7233
6gjU8xeBTxbB1KZM3oT9R9JoH3CGYZmrAgMBAAGjggGKMIIBhjASBgNVHRMBAf8E
CDAGAQH/AgEAMB0GA1UdDgQWBBSuupQzuu83TQvXGO9K5KENvAe2czAfBgNVHSME
GDAWgBRRMxztNkCvF9MlzWlo8q9OIz6zQTAOBgNVHQ8BAf8EBAMCAYYwHQYDVR0l
BBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMCMHoGCCsGAQUFBwEBBG4wbDAkBggrBgEF
BQcwAYYYaHR0cDovL29jc3AuZGlnaWNlcnQuY29tMEQGCCsGAQUFBzAChjhodHRw
Oi8vY2FjZXJ0cy5kaWdpY2VydC5jb20vRGlnaUNlcnRUTFNSU0E0MDk2Um9vdEc1
LmNydDBGBgNVHR8EPzA9MDugOaA3hjVodHRwOi8vY3JsMy5kaWdpY2VydC5jb20v
RGlnaUNlcnRUTFNSU0E0MDk2Um9vdEc1LmNybDA9BgNVHSAENjA0MAsGCWCGSAGG
/WwCATAHBgVngQwBATAIBgZngQwBAgEwCAYGZ4EMAQICMAgGBmeBDAECAzANBgkq
hkiG9w0BAQwFAAOCAgEAm5n6ds96TsvAgH4+P/HASmAPv4gJGTkgXjflouTbHJqx
FAOsExKNfs0G7BlrdqcFnIOcNR20t0hDvikyZvLT9yBAJYB3uAoYZP/7HIkdkjQB
1EfX2A9VLzEhAitWYnv4A1vwIVgI7nKaSTz2wGd0Be6cAu+4xxjKieCYAQduI14t
ebbHEapLulqRuvemZc/dAIeAVVrC01GKhPkGrb2iAXCRmZzo9IYh59l4u8K9LoHb
7Hf4Ff2uDUgIa1Zabg4wXjwYvoAhxPhRa/5lSn/tGCRg/JSpi3dF0Hry1EIueLL3
q3QfX4kjs6qTqWSHGL98sLjWDowRQWDAbgjylkGAVZBG4aqvlgEhKCLNtgzdN250
4XWLJbcCj1F1Bi+WoXSGlpm3x+1otDJgkSnCjXs0uRxrHsB8kWzMR+WpTROqsDVQ
/9+NAbtkLNFc6RdRHPBnvNrah8Z8tfKcGJrzjncmyBae1BdFhZ09keXQ+Zj5Fu/o
P8gHuJhmwIPsIE5tEs33q1BUXaP6vQU+v/lmvb6Vy2xj3U091m9cYSYt1xObnOLS
HeL48nhq1fIhJQfXWX2neiOmZxeVi+g0nZ9Ws7MpARDSTBsR8NBsWBBVcmTUKJhv
g/cqkTWFeprxB41VkxJc79K3VL8yo01JYIUfAiTwNecb6/mvBAV/yOzGCYSqff0=
-----END CERTIFICATE-----
---
Server certificate
subject=jurisdictionC = US, jurisdictionST = Utah, businessCategory = Private Organization, serialNumber = 5299537-0142, C = US, ST = Utah, L = Lehi, O = "DigiCert, Inc.", CN = digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com
issuer=C = US, O = "DigiCert, Inc.", CN = DigiCert G5 TLS RSA4096 SHA384 2021 CA1
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA
Server Temp Key: ECDH, prime256v1, 256 bits
---
SSL handshake has read 4438 bytes and written 489 bytes
Verification: OK
---
New, TLSv1.2, Cipher is ECDHE-RSA-AES256-GCM-SHA384
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-RSA-AES256-GCM-SHA384
    Session-ID: BF0B1D9F71BCA21BEEB1C4DC85082088226AFCBE2409135867868F3E5CECE2C5
    Session-ID-ctx:
    Master-Key: 12176CAF18D65E8B68875984C5DEC1B92C661A82508ECE56EC6376FEE853FAED5C1CB3B664B75749CF7E578F4E76DA41
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    Start Time: 1738072427
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: no
---
DONE
```

</details>

I also used this full output to manually copy the base64 PEM encoded certificates from the served chain into `end-entity` and `intermediate` files, and copied in the root from my local machine trust-store:

```shell
$ ls
DigiCert_TLS_RSA4096_Root_G5.pem  end-entity.pem  intermediate.pem
$ openssl x509 -in end-entity.pem -noout -subject
subject=jurisdictionC = US, jurisdictionST = Utah, businessCategory = Private Organization, serialNumber = 5299537-0142, C = US, ST = Utah, L = Lehi, O = "DigiCert, Inc.", CN = digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com
$ openssl x509 -in intermediate.pem -noout -subject
subject=C = US, O = "DigiCert, Inc.", CN = DigiCert G5 TLS RSA4096 SHA384 2021 CA1
$ openssl x509 -in DigiCert_TLS_RSA4096_Root_G5.pem -noout -subject
subject=C = US, O = "DigiCert, Inc.", CN = DigiCert TLS RSA4096 Root G5

```

From the end-entity certificate, we can see that the certificate contains Certificate Revocation List distribution point URLs:

```bash
$ openssl x509 -in end-entity.pem -noout -ext crlDistributionPoints
X509v3 CRL Distribution Points:
    Full Name:
      URI:http://crl3.digicert.com/DigiCertG5TLSRSA4096SHA3842021CA1-1.crl
    Full Name:
      URI:http://crl4.digicert.com/DigiCertG5TLSRSA4096SHA3842021CA1-1.crl
```

> At this point, I intended to first show the use of CRLs by asking openssl, as part of the s_client connection to download, verify then check the CRL - however openssl appears to have a maximum download size (this may be a [bug](https://github.com/openssl/openssl/issues/8581#issuecomment-513919367)...) that causes the verification request to fail.
{: .prompt-warning }

```bash
$ echo "GET /" | openssl s_client -connect digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com:443 \
-showcerts \
-verify_return_error \
-CAfile DigiCert_TLS_RSA4096_Root_G5.pem \
-crl_download -crl_check 2>&1 \
| head -5

Unable to load CRL via CDP
40378B6A277F0000:error:1E800075:HTTP routines:check_set_resp_len:max resp len exceeded:../crypto/http/http_client.c:491:length=4715231, max=102400
40378B6A277F0000:error:1E800067:HTTP routines:OSSL_HTTP_REQ_CTX_exchange:error receiving:../crypto/http/http_client.c:912:server=http://crl3.digicert.com:80
depth=0 jurisdictionC = US, jurisdictionST = Utah, businessCategory = Private Organization, serialNumber = 5299537-0142, C = US, ST = Utah, L = Lehi, O = "DigiCert, Inc.", CN = digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com
verify error:num=3:unable to get certificate CRL
```

However, we can still undertake a CRL check with s_client by first downloading the CRL....

```bash
curl http://crl3.digicert.com/DigiCertG5TLSRSA4096SHA3842021CA1-1.crl \
--output DigiCertG5TLSRSA4096SHA3842021CA1-1.crl
$ openssl crl -inform DER -in DigiCertG5TLSRSA4096SHA3842021CA1-1.crl -text -noout | head
Certificate Revocation List (CRL):
        Version 2 (0x1)
        Signature Algorithm: sha384WithRSAEncryption
        Issuer: C = US, O = "DigiCert, Inc.", CN = DigiCert G5 TLS RSA4096 SHA384 2021 CA1
        Last Update: Jan 27 22:01:25 2025 GMT
        Next Update: Feb  3 22:01:25 2025 GMT
        CRL extensions:
            X509v3 Authority Key Identifier:
                AE:BA:94:33:BA:EF:37:4D:0B:D7:18:EF:4A:E4:A1:0D:BC:07:B6:73
            X509v3 CRL Number:
```

...then passing this as an argument with `-CRL file`...

```bash
$ echo "GET /" | openssl s_client -connect digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com:443 \
-showcerts \
-verify_return_error \
-CAfile DigiCert_TLS_RSA4096_Root_G5.pem \
-CRL DigiCertG5TLSRSA4096SHA3842021CA1-1.crl \
-crl_check 2>&1 | \
egrep '^Verification'
Verification error: certificate revoked
```

Comparing this with the DigiCert end point that exposes a valid (non-revoked) certificate...

```bash
$ echo "GET /" | openssl s_client -connect digicert-tls-rsa4096-root-g5.chain-demos.digicert.com:443 \
-showcerts \
-verify_return_error \
-CAfile DigiCert_TLS_RSA4096_Root_G5.pem \
-CRL DigiCertG5TLSRSA4096SHA3842021CA1-1.crl \
-crl_check 2>&1 | \
egrep '^Verification'
Verification: OK
```

### CRL structure

[RFC5280](https://datatracker.ietf.org/doc/html/rfc5280) covers the definition of the structure of both certificates and certificate revocation lists.

From the standard, the shape of CRLs is very similar to that of x509 certificates, with ASN.1 structures and encoding options through DER and PEM.  It does look like the default form of the file obtainable through the CRL distribution point URIs is DER.

`openssl crl -text` can be used in a similar fashion to `openssl x509 -text` to display the content of the CRL in a human readable form:

```bash
$ openssl crl -inform der -in DigiCertG5TLSRSA4096SHA3842021CA1-1.crl -text -noout
Certificate Revocation List (CRL):
        Version 2 (0x1)
        Signature Algorithm: sha384WithRSAEncryption
        Issuer: C = US, O = "DigiCert, Inc.", CN = DigiCert G5 TLS RSA4096 SHA384 2021 CA1
        Last Update: Jan 27 22:01:25 2025 GMT
        Next Update: Feb  3 22:01:25 2025 GMT
        CRL extensions:
            X509v3 Authority Key Identifier:
                AE:BA:94:33:BA:EF:37:4D:0B:D7:18:EF:4A:E4:A1:0D:BC:07:B6:73
            X509v3 CRL Number:
                1380
            X509v3 Issuing Distribution Point: critical
                Full Name:
                  URI:http://crl3.digicert.com/DigiCertG5TLSRSA4096SHA3842021CA1-1.crl
                  URI:http://crl4.digicert.com/DigiCertG5TLSRSA4096SHA3842021CA1-1.crl
  Revoked Certificates:
    Serial Number: 0F66F50A08647C84CADDEA81A6BAFC17
        Revocation Date: Dec 23 17:41:55 2023 GMT
[######  rest of this list removed for brevity ##########]
    Signature Algorithm: sha384WithRSAEncryption
    Signature Value:
        86:0a:97:41:25:bb:e8:bb:f9:67:9c:61:fd:f3:f9:5b:16:4e:
        4f:7d:5a:d3:aa:76:a4:97:6f:27:6e:02:14:25:15:1d:bd:df:
        49:5a:d7:10:6a:f5:48:52:91:b9:f7:b2:27:07:5f:96:84:c8:
        5b:bf:ca:ed:a4:71:83:01:57:58:fd:5c:c1:8e:fc:d0:c5:57:
        33:94:2e:61:57:1d:d5:a2:9c:e6:28:47:79:cf:53:9a:90:85:
        f1:12:65:0f:22:05:93:56:9b:f9:cc:90:a1:7e:d2:c1:b9:0e:
        92:7f:93:39:69:4c:d5:2e:d6:6a:d1:e7:27:94:94:22:68:d0:
        57:b9:ba:a9:91:8e:bf:4d:c4:3e:9a:57:b8:c0:74:ec:27:fa:
        23:4c:3e:51:ac:ef:cb:ee:67:bd:4f:5e:dd:7f:25:5b:ab:47:
        f4:33:35:58:85:0d:e3:59:c0:d3:f7:9f:dc:b3:12:02:7a:07:
        80:28:9f:8d:35:5f:f4:9f:2f:58:1a:7d:74:ca:b7:c4:b1:9f:
        4e:11:e3:4f:ca:2d:4a:78:c1:a5:f8:18:0b:c7:68:89:81:82:
        96:af:6f:02:32:20:cb:36:3a:e2:31:71:69:b9:c3:22:f5:22:
        a3:c6:ed:ac:be:51:17:45:47:5b:05:6b:56:0f:c0:2e:9d:b3:
        c9:25:e8:74:74:b9:bb:2b:70:48:2e:87:42:88:c1:2e:16:6b:
        e7:37:4d:c5:84:b5:f5:be:4c:5d:40:45:1a:4a:26:0b:69:7c:
        e5:d4:f7:e4:0d:fb:49:49:0a:66:c6:b7:14:0a:69:d6:b5:f2:
        26:fc:11:3f:ad:6d:ae:58:97:6d:9f:d5:9a:61:24:92:9e:ef:
        0c:41:74:4a:fd:5e:20:f3:28:eb:59:a1:41:c6:2f:e8:3c:b9:
        5d:fb:32:18:c6:ae:38:de:29:ec:52:6b:36:eb:9e:d3:ee:dc:
        b7:28:a8:90:a7:6f:81:eb:96:f1:2d:a0:64:4b:b5:81:a4:98:
        78:79:27:ff:76:4b:77:59:14:85:22:08:9e:6c:4d:e0:bf:d0:
        27:7a:ef:d6:ec:32:91:b0:1c:25:cf:f2:c2:5b:a1:4c:cf:84:
        7a:d3:fc:a4:1c:93:72:b0:47:c3:a6:fa:c6:2a:0a:2b:57:84:
        97:e3:1e:7f:3f:b6:12:2a:3e:68:86:5f:e3:21:d9:c0:dd:48:
        d8:ef:a6:02:94:a6:76:c0:7d:9d:7c:03:09:ad:97:03:14:3d:
        03:ba:75:0b:bc:1c:d8:54:8e:f8:a9:93:a3:b5:05:0a:5f:be:
        f9:f3:8e:c8:d8:4f:78:20:f4:66:98:53:c0:8e:f2:d7:10:aa:
        b3:75:7d:0c:fb:de:3e:61

```

The tools and libraries that support the creation and signing of CA certificates all include the equivalent ability to generate and CRLs.

The approach to signing of CRLs (to ensure integrity and authenticity) is effectively identical to the approaches for signing of certificates - this approach is covered through the same RFCs as outlined in my post covering [RSA signing](/posts/rsa-x509-signing/#specification-of-the-signing-algorithm) e.g. using the same PKCS #1 V1.5 encoding.  Similarly for validation, the CRL signature can be verified against the applicable signing intermediate and ultimately CA root.

### manually checking for a certificate in a CRL file

Checking for the presence of an end-entity certificate within a CRL entails 2 steps;

- verifying that the signature of the CRL is valid
- checking whether the serial number of the end-entity certificate is present in the CRL

Verifying that the signature on the CRL file is valid against the intermediate.

```bash
$ openssl crl -inform der -in DigiCertG5TLSRSA4096SHA3842021CA1-1.crl -CAfile intermediate.pem -noout
verify OK
```

and for completeness verifying that the intermediate is signed by the trusted root.

```bash
$ openssl verify -CAfile DigiCert_TLS_RSA4096_Root_G5.pem intermediate.pem
intermediate.pem: OK
```

We now have a chain of trust for the CRL up to the trusted CA root (that also mirrors the same chain of trust for the certificate we wish to check the revocation status of) - we can now confidently search for the serial number of the end-entity certificate within the trusted CRL.

```bash
$ openssl x509 -in end-entity.pem -noout -serial
serial=0E88B46BE2BA708BF74F298F6E162E5A

$ openssl crl -inform der -in DigiCertG5TLSRSA4096SHA3842021CA1-1.crl -text -noout | \
grep -A 1 0E88B46BE2BA708BF74F298F6E162E5A
    Serial Number: 0E88B46BE2BA708BF74F298F6E162E5A
        Revocation Date: Jan  6 15:52:07 2025 GMT
```

> This confirms that our end-entity certificate is present in a valid list of revoked certificates, and we can consider the end-entity certificate revoked.
{: .prompt-info }


## Online Certificate Status Protocol (OCSP)

### OCSP examples

Initially, I'll re-use the `DigiCert TLS RSA4096 Root G5` demo sites (as above for CRLs) to dive into OCSP, specifically:

- the active demo certificate [https://digicert-tls-rsa4096-root-g5.chain-demos.digicert.com/](https://digicert-tls-rsa4096-root-g5.chain-demos.digicert.com/)
- the revoked certificate [https://digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com/](https://digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com/)

As OCSP is a bit more prevalent (currently), I can also use the [badssl](https://badssl.com/) and [Let's encrypt](https://letsencrypt.org/certificates/) revoked certificates, but to start off with, will use the DigiCert examples.

I'll re-use the same process as above for CRLs, and will re-use the end-entity and intermediate certificates extracted from the output of `openssl s_client` which I've stored locally;

```bash
$ ls
DigiCert_TLS_RSA4096_Root_G5.pem  end-entity.pem  intermediate.pem
$ openssl x509 -in end-entity.pem -noout -subject
subject=jurisdictionC = US, jurisdictionST = Utah, businessCategory = Private Organization, serialNumber = 5299537-0142, C = US, ST = Utah, L = Lehi, O = "DigiCert, Inc.", CN = digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com
$ openssl x509 -in intermediate.pem -noout -subject
subject=C = US, O = "DigiCert, Inc.", CN = DigiCert G5 TLS RSA4096 SHA384 2021 CA1
$ openssl x509 -in DigiCert_TLS_RSA4096_Root_G5.pem -noout -subject
subject=C = US, O = "DigiCert, Inc.", CN = DigiCert TLS RSA4096 Root G5
```

From the end-entity certificate, we can see the OCSP URI:

```bash
$ openssl x509 -in end-entity.pem -noout -ext authorityInfoAccess
Authority Information Access:
    OCSP - URI:http://ocsp.digicert.com
    CA Issuers - URI:http://cacerts.digicert.com/DigiCertG5TLSRSA4096SHA3842021CA1-1.crt
```


> I found it difficult to find the short names used when calling `openssl x509 -ext` to extract a specific value such as in the command above. Relative to the `-text` output that uses slightly more verbose names (e.g. Authority Information Access => authorityInfoAccess).  After a bit of hunting through man pages, spotted that these are captured in full in `man x509v3_config`
{: .prompt-warning }


We can then use this OCSP URI with the `openssl ocsp` command, and the applicable certificates, to verify the revocation status of the end-entity certificate.

```bash
$ openssl ocsp -issuer intermediate.pem -cert end-entity.pem -url http://ocsp.digicert.com
WARNING: no nonce in response
Response verify OK
end-entity.pem: revoked
        This Update: Jan 29 04:21:02 2025 GMT
        Next Update: Feb  5 03:21:02 2025 GMT
        Reason: superseded
        Revocation Time: Jan  6 15:52:07 2025 GMT
```
> This output shows that the response verified OK (that is, the response signature was checked and validated as signed by the same intermediate as the certificate being checked).<br /><br />The output also shows that the end-entity certificate was revoked as we expected, with some additional context on the reason for revocation, when the certificate was revoked, and (less relevant for revoked certificate), when the OCSP status was updated, and when it will next be updated.  This is far more relevant when caching results of non-revoked certificates.<br /><br />I'll touch on the `no nonce in response` output later.
{: .prompt-info }

Trying a valid/non-revoked certificate for comparison.

```bash
$ openssl s_client -connect digicert-tls-rsa4096-root-g5.chain-demos.digicert.com:443 -showcerts
```

I then extracted the end-entity certificate for this valid (non-revoked) certificate into the file `active-end-entity.pem`

```bash
$ openssl x509 -in active-end-entity.pem -noout -subject
subject=jurisdictionC = US, jurisdictionST = Utah, businessCategory = Private Organization, serialNumber = 5299537-0142, C = US, ST = Utah, L = Lehi, O = "DigiCert, Inc.", CN = digicert-tls-rsa4096-root-g5.chain-demos.digicert.com
```

I can then run the OCSP check for this valid certificate showing the differences in result.

```bash
$ openssl ocsp -issuer intermediate.pem -cert active-end-entity.pem -url http://ocsp.digicert.com
WARNING: no nonce in response
Response verify OK
active-end-entity.pem: good
        This Update: Jan 29 08:51:02 2025 GMT
        Next Update: Feb  5 07:51:02 2025 GMT
```

> This output shows that the response verified OK (that is, the response signature was checked and validated as signed by the same intermediate as the certificate being checked).<br /><br />The output also shows that the end-entity certificate is `good`  and when the OCSP status was updated, and when it will next be updated.  This allows the client to know that, if caching this repsonse, there is no point in re-checking until Feb 5th.<br /><br />I'll touch on the `no nonce in response` output later.
{: .prompt-info }

### OCSP request structure

[rfc6960](https://datatracker.ietf.org/doc/html/rfc6960) defines the format of OCSP requests and responses - over a [GET HTTP call](https://datatracker.ietf.org/doc/html/rfc6960#appendix-A.1), the request is formed as a base 64 encoding of the DER encoded ASN1 structure containing a variety of fields.  The `openssl ocsp` command gives the option to show a text representation of the request.

```bash
$ openssl ocsp -issuer intermediate.pem \
-cert end-entity.pem \
-url http://ocsp.digicert.com \
-req_text

OCSP Request Data:
    Version: 1 (0x0)
    Requestor List:
        Certificate ID:
          Hash Algorithm: sha1
          Issuer Name Hash: F5065C39ECFD480EAD998A227A1FE9C55950EA18
          Issuer Key Hash: AEBA9433BAEF374D0BD718EF4AE4A10DBC07B673
          Serial Number: 0E88B46BE2BA708BF74F298F6E162E5A
    Request Extensions:
        OCSP Nonce:
            0410AAA8128F20973CBDCEDAB64FA040393E
WARNING: no nonce in response
Response verify OK
end-entity.pem: revoked
        This Update: Jan 29 04:21:02 2025 GMT
        Next Update: Feb  5 03:21:02 2025 GMT
        Reason: superseded
        Revocation Time: Jan  6 15:52:07 2025 GMT
```
RFC6090 describes the purpose of these request fields, and they are broadly self explanatory (e.g. the use of an optional [nonce](https://datatracker.ietf.org/doc/html/rfc6960#section-4.4.1) value to prevent replay attacks, declaring the hash algorithm to be used, and the serial number of the certificate in question).

The only values passed that are not immediately clear on their purpose are the issuer attributes - [RFC6090 section 4.1.2](https://datatracker.ietf.org/doc/html/rfc6960#section-4.1.2) describes why a hash of both the issuer name **and** issuer key is passed, but it's not immediately obvious why any attributes of the issuer are required in the first place.  The rfc offers an option to [delegate](https://datatracker.ietf.org/doc/html/rfc6960#section-2.6) OCSP signature authority to a different CA, so this isn't some attempt to bind the OCSP signer to the certificate signer.  Having done some reading, the only purpose I can see in passing this information is to allow the OCSP responder to uniquely identify the certificate in question within it's certificate status database - serial numbers of certificates for a particular CA (that is, root certificate) must be unique, but OCSP responders can serve responses for multiple CAs, roots & intermediates.  By passing the issuer name/serial number hash in the request, the OCSP responder can unambiguously identify the unique certificate in question.

### OCSP response structure

The response structure is also covered in [rfc6960](https://datatracker.ietf.org/doc/html/rfc6960#section-4.2), noting that the OCSP response structure is CA signed allowing authenticity and integrity to be validated.  As mentioned above, this could be signed by the same issuing certificate as the end-entity being checked, or the authority to sign OCSP responses could be [delegated](https://datatracker.ietf.org/doc/html/rfc6960#section-2.6) by the CA to a different authority.

The content of the response can also be viewed using the `openssl ocsp` command.

```bash
$ openssl ocsp -issuer intermediate.pem -cert end-entity.pem -url http://ocsp.digicert.com -resp_text
OCSP Response Data:
    OCSP Response Status: successful (0x0)
    Response Type: Basic OCSP Response
    Version: 1 (0x0)
    Responder Id: AEBA9433BAEF374D0BD718EF4AE4A10DBC07B673
    Produced At: Jan 30 04:37:05 2025 GMT
    Responses:
    Certificate ID:
      Hash Algorithm: sha1
      Issuer Name Hash: F5065C39ECFD480EAD998A227A1FE9C55950EA18
      Issuer Key Hash: AEBA9433BAEF374D0BD718EF4AE4A10DBC07B673
      Serial Number: 0E88B46BE2BA708BF74F298F6E162E5A
    Cert Status: revoked
    Revocation Time: Jan  6 15:52:07 2025 GMT
    Revocation Reason: superseded (0x4)
    This Update: Jan 30 04:21:02 2025 GMT
    Next Update: Feb  6 03:21:02 2025 GMT

    Signature Algorithm: sha384WithRSAEncryption
    Signature Value:
        32:4e:3b:a7:21:5a:47:7e:a4:f7:d5:9b:6e:42:8e:a3:91:65:
        c8:0c:92:0f:17:c1:66:96:3d:03:79:24:a4:3e:75:02:63:71:
        06:f4:bd:b7:43:5b:c0:4a:0c:ae:61:4f:3e:58:95:3f:b6:39:
        0a:ec:57:ee:b6:41:3c:92:31:3f:cf:5a:57:d7:f7:63:90:f3:
        ec:ca:2b:f6:75:77:07:72:b2:50:8c:d3:b3:df:e2:76:d9:0a:
        33:29:c0:72:69:46:cb:93:a9:68:6e:68:48:e3:ad:1d:12:bd:
        0e:7e:2d:09:93:2f:f5:79:30:5b:44:c3:06:1a:a0:1c:51:83:
        60:37:9a:5d:59:ff:a4:a8:c4:02:13:e1:f4:b9:80:18:69:2f:
        91:cc:80:c4:bc:fa:ab:e9:91:40:ed:0f:31:37:51:74:99:38:
        b8:15:27:d9:d3:c9:6f:50:66:74:df:6d:27:22:3a:a8:3f:55:
        c8:a5:c2:09:b0:98:70:23:70:52:61:1d:da:d6:a3:1c:ba:b4:
        63:63:2e:ba:9b:0e:e4:76:9c:a4:aa:9e:0c:7e:d3:bf:18:9f:
        1a:45:a3:c3:3a:60:08:8e:f9:4e:da:61:bb:b0:7e:05:21:3c:
        ac:82:0e:e6:14:49:9a:e2:c0:49:57:4b:dc:88:67:8d:0a:6a:
        1d:96:76:31:4d:b4:25:55:a9:7f:d4:be:2b:19:c7:46:0d:8f:
        b7:22:0a:34:bd:aa:99:f4:37:13:e4:0e:16:0f:7f:00:7f:8b:
        0f:8c:36:97:1f:d9:3f:c7:5b:5e:ba:97:0d:d6:63:ed:31:be:
        60:f3:96:b8:ff:12:d8:5d:6e:92:68:e5:8b:f5:03:3f:4e:97:
        6f:6d:27:57:22:18:2b:52:78:84:74:66:41:fe:13:54:70:e2:
        a9:12:82:c9:f6:c7:97:e5:41:cc:4a:07:de:ec:b6:90:10:3c:
        04:05:e3:8c:bc:33:14:2f:59:00:47:1e:88:fb:5e:ed:86:be:
        84:20:2a:d2:6f:78:0b:a0:f7:5a:ad:ed:10:ec:1b:8f:18:a4:
        79:b7:13:96:ec:8f:fe:40:e4:6e:79:a6:14:d6:e4:f2:72:09:
        25:8d:8c:55:2b:fc:a6:0a:26:48:77:a7:c7:99:f4:9d:76:40:
        c2:02:c5:9c:cb:ae:b9:63:09:95:a3:ad:d7:ed:8c:66:f7:76:
        0a:a7:84:30:03:fe:d4:4e:de:17:5d:c4:e2:73:c1:29:e1:5a:
        5b:44:b1:53:44:0b:04:89:2f:1b:3a:d7:b2:81:9b:c8:01:3a:
        34:36:df:4a:eb:eb:1f:a4:29:21:03:8f:4f:7d:d6:cf:d8:51:
        a9:6c:62:62:de:b9:6e:74
WARNING: no nonce in response
Response verify OK
end-entity.pem: revoked
        This Update: Jan 30 04:21:02 2025 GMT
        Next Update: Feb  6 03:21:02 2025 GMT
        Reason: superseded
        Revocation Time: Jan  6 15:52:07 2025 GMT
```

## OCSP stapling

OCSP stapling removed the need for the client to interact with any other party out-with the service which is presenting the certificate in the first place - this removes the privacy concerns inherent OCSP - effectively by a client interacting with an OCSP responder, the client is sharing a full trail of all the services it has connected to with a CA or their delegated OCSP responder providers.

In TLS, this is achieved through the service passing on the (CA signed) OCSP response as part of the initial certificate response from service to client - the mechanism has changed slightly between TLS 1.2 (where the mechanism is described in [rfc6961](https://datatracker.ietf.org/doc/html/rfc6961)) and TLS 1.3 where the mechanism is described in the overarching TLS [rfc8446 - section 4.4.2.1](https://datatracker.ietf.org/doc/html/rfc8446#section-4.4.2.1).  The status response cannot be forged by the intermediate in this model, as the OCSP response is signed by the CA, which the client can validate.

Periodically, the service will obtain the OCSP status of a certificate which it serves, and will cache and use this until some point in time no later than the `next update` time on the response.

This [cloudflare blog post](https://blog.cloudflare.com/high-reliability-ocsp-stapling/) gives a good illustration of the mechanism, and described approaches they use to cache and serve stapled OCSP responses.


### Stapling examples - DigiCert demo

Initially, I worked through an example using the revoked demo DigiCert certificate as above (note that, as I've written this post over a few days, DigiCert have renewed the certificates, so the end-entity used here is a more recent certificate with different serial numbers etc.).

```bash
$ openssl x509 -in end-entity.pem -noout -serial
serial=04DC61BC20A2C9B6D05207CC4C52212F
$ openssl ocsp -issuer intermediate.pem -cert end-entity.pem -url http://ocsp.digicert.com -req_text
OCSP Request Data:
    Version: 1 (0x0)
    Requestor List:
        Certificate ID:
          Hash Algorithm: sha1
          Issuer Name Hash: F5065C39ECFD480EAD998A227A1FE9C55950EA18
          Issuer Key Hash: AEBA9433BAEF374D0BD718EF4AE4A10DBC07B673
          Serial Number: 04DC61BC20A2C9B6D05207CC4C52212F
    Request Extensions:
        OCSP Nonce:
            0410EA5F520257339701F4D2979EE242AFC1
WARNING: no nonce in response
Response verify OK
end-entity2.pem: revoked
        This Update: Jan 31 12:51:02 2025 GMT
        Next Update: Feb  7 11:51:02 2025 GMT
        Reason: superseded
        Revocation Time: Jan 29 14:43:10 2025 GMT

```

We can see that, from the OCSP responder, the certificate is revoked as expected.

> This next step confused me for some time - initially I used FireFox to connect to the DigiCert page that serves this revoked certificate (aware that FireFox currently requests OCSP stapled certificates and will display errors if a revoked status is received).<br /><br />However, for this certificate no error was shown, and FireFox indicated the certificate was currently valid!<br /><br />After opening wireshark and looking through some packet captures, I spotted that the the stapled OCSP response stated that the certificate was good - the following `openssl s_client` output with the use of the `-status` option reflects this.  This highlights one of the issues of OCSP stapling (and revocation approaches in general) - the certificate is clearly revoked from the online OCSP response, but it looks like DigiCert generate and cache an OCSP response that states the certificate is good.<br /><br />I think this may be a bug/glitch in DigiCert's approach to demo certificates - the cached OCSP response was generated at the time the certificate was first created, but the certificate is renewed before the `next update` time on the OCSP response with another certificate that has a stapled `good` OCSP response.
{: .prompt-warning }

```bash
$ echo "GET /" | openssl s_client -connect digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com:443 -showcerts -status
[[[truncated]]]
verify return:1
OCSP response:
======================================
OCSP Response Data:
    OCSP Response Status: successful (0x0)
    Response Type: Basic OCSP Response
    Version: 1 (0x0)
    Responder Id: AEBA9433BAEF374D0BD718EF4AE4A10DBC07B673
    Produced At: Jan 29 00:00:00 2025 GMT
    Responses:
    Certificate ID:
      Hash Algorithm: sha1
      Issuer Name Hash: F5065C39ECFD480EAD998A227A1FE9C55950EA18
      Issuer Key Hash: AEBA9433BAEF374D0BD718EF4AE4A10DBC07B673
      Serial Number: 04DC61BC20A2C9B6D05207CC4C52212F
    Cert Status: good
    This Update: Jan 29 00:00:00 2025 GMT
    Next Update: Feb  4 23:00:00 2025 GMT
[[[truncated]]]

```

### Stapling examples - badssl

The [badssl site](https://badssl.com) offers a [revoked](https://revoked.badssl.com) test endpoint/certificate.  The badssl service uses Let's Encrypt as a CA (I've copied and saved the end entity certificate from an `openssl s_client` call).

```bash
$ openssl x509 -in badssl_revoked_end-entity.pem -issuer -noout
issuer=C = US, O = Let's Encrypt, CN = E6
```

Currently, Let's Encrypt's approach is that they:

- don't publish revoked certificates in CRLs
- publish revocation information on OCSP endpoints
- support the use of stapling/the "must staple" extension

This is all in a [state of change](https://letsencrypt.org/2024/12/05/ending-ocsp/) at the time of writing but for now, the current approach holds.

Although stapling *could* be used for this badssl demo certificate, the service does not provide a stapled OCSP response when called - I don't believe there are currently any OCSP/CA related technical impediments, but implementation of stapling is a choice of the service provider.

```bash
$ echo "GET /" | \
openssl s_client -connect revoked.badssl.com:443 \
-showcerts \
-status 2>&1 | \
grep OCSP
OCSP response: no response sent
```

We can make a call to the OCSP responder to determine that the certificate is in fact revoked.

```bash
$ openssl ocsp \
-issuer badssl_revoked_intermediate.pem \
-cert badssl_revoked_end-entity.pem \
-url http://e6.o.lencr.org \
-req_text
OCSP Request Data:
    Version: 1 (0x0)
    Requestor List:
        Certificate ID:
          Hash Algorithm: sha1
          Issuer Name Hash: D47A388041E8E98D07387CECF6B6D8F20FA56431
          Issuer Key Hash: 0DC5CCFD9BEE1405A14C3082A53E5E8AC35809D2
          Serial Number: 04ADF72B0F8A8BF99BE39C1148427D07DF52
    Request Extensions:
        OCSP Nonce:
            041018B54DB1ECBBBE3A54965BF2F64F46EC
WARNING: no nonce in response
Response verify OK
badssl_revoked_end-entity.pem: revoked
        This Update: Feb  2 04:38:00 2025 GMT
        Next Update: Feb  9 04:37:58 2025 GMT
        Reason: keyCompromise
        Revocation Time: Dec 19 19:19:49 2024 GMT
```

> I had some confusion with Firefox when connecting to the `revoked.badssl.com` endpoint.  Firefox errored indicating that the certificate was revoked, but from tests such as above, and from following a packet capture in Wireshark, I couldn't see any calls to the OCSP endpoint<br /><br />Initially I though this may be down to the use of [crlite](https://blog.mozilla.org/security/2020/01/09/crlite-part-1-all-web-pki-revocations-compressed/) by FireFox (which I will cover in another post) but as let's encrypt don't currently publish CRLs, this wasn't the revocation detection mechanism involved.<br /><br />It turned out that this was all down to the method of testing - I didn't have a packet capture of the first time I connected to the URL, and FireFox caches OCSP responses.  By restarting FireFox and taking a packet capture from the outset, I could see the TLS connection starting, then a call to the OCSP endpoint immediately following.  Once receiving the OCSP revoked response, FireFox stopped communication with the service and displayed the error page.
{: .prompt-info }

### Stapling example - bbc.co.uk

To illustrate a working, non-revoked OCSP stapled response, I tried a few common sites before landing on one that worked - [bbc.co.uk](https://bbc.co.uk).

The following truncated output illustrates a correctly working stapled status response with the response provided by an authorised delegated OCSP responder (see the `X509v3 Extended Key Usage` value in the certificate contained within the stapled OCSP response)

```bash
$ echo "GET /" | openssl s_client -connect bbc.co.uk:443 -showcerts -status

OCSP response:
======================================
OCSP Response Data:
    OCSP Response Status: successful (0x0)
    Response Type: Basic OCSP Response
    Version: 1 (0x0)
    Responder Id: D677FE585DB30E7D5A90EADAB153FFA8BA74CCA6
    Produced At: Feb  2 02:26:58 2025 GMT
    Responses:
    Certificate ID:
      Hash Algorithm: sha1
      Issuer Name Hash: 6B7064FE6A7443DC2D6D5B79ECACA7AE5C2EC33F
      Issuer Key Hash: F8EF7FF2CD7867A8DE6F8F248D88F1870302B3EB
      Serial Number: 62C34811A4083CC562DF7E91
    Cert Status: good
    This Update: Feb  2 02:26:58 2025 GMT
    Next Update: Feb  6 02:26:57 2025 GMT
[[[truncated]]]
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            70:5c:38:47:54:fd:5c:7f:0e:d8:07:f4
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=BE, O=GlobalSign nv-sa, CN=GlobalSign RSA OV SSL CA 2018
        Validity
            Not Before: Dec  8 07:41:15 2024 GMT
            Not After : Mar 10 07:41:14 2025 GMT
        Subject: C=BE, O=GlobalSign nv-sa/serialNumber=201906110004, CN=gsrsaovsslca2018CA OCSP Responder
[[[truncated]]]
        X509v3 extensions:
            X509v3 Authority Key Identifier:
                F8:EF:7F:F2:CD:78:67:A8:DE:6F:8F:24:8D:88:F1:87:03:02:B3:EB
            OCSP No Check:

            X509v3 Extended Key Usage:
                OCSP Signing
            X509v3 Subject Key Identifier:
                D6:77:FE:58:5D:B3:0E:7D:5A:90:EA:DA:B1:53:FF:A8:BA:74:CC:A6
            X509v3 Key Usage: critical
                Digital Signature
[[[truncated]]]
======================================
---
Certificate chain
 0 s:C = GB, ST = London, L = London, O = BRITISH BROADCASTING CORPORATION, CN = www.bbc.com
   i:C = BE, O = GlobalSign nv-sa, CN = GlobalSign RSA OV SSL CA 2018
[[[truncated]]]
 1 s:C = BE, O = GlobalSign nv-sa, CN = GlobalSign RSA OV SSL CA 2018
   i:OU = GlobalSign Root CA - R3, O = GlobalSign, CN = GlobalSign
[[[truncated]]]
 2 s:OU = GlobalSign Root CA - R3, O = GlobalSign, CN = GlobalSign
   i:C = BE, O = GlobalSign nv-sa, OU = Root CA, CN = GlobalSign Root CA
[[[truncated]]]

```




## Prevalent approach as of Jan 2025

In the process of understanding this topic, I've relied on a number of very useful posts and articles which are well worth a read:

- [https://scotthelme.co.uk/lets-encrypt-to-end-ocsp-support-in-2025/](https://scotthelme.co.uk/lets-encrypt-to-end-ocsp-support-in-2025/)
- [https://letsencrypt.org/2024/12/05/ending-ocsp/](https://letsencrypt.org/2024/12/05/ending-ocsp/)
- [https://www.feistyduck.com/newsletter/issue_121_the_slow_death_of_ocsp](https://www.feistyduck.com/newsletter/issue_121_the_slow_death_of_ocsp)
- [https://blog.mozilla.org/security/2020/01/09/crlite-part-1-all-web-pki-revocations-compressed/](https://blog.mozilla.org/security/2020/01/09/crlite-part-1-all-web-pki-revocations-compressed/)
- [https://blog.cloudflare.com/high-reliability-ocsp-stapling/](https://blog.cloudflare.com/high-reliability-ocsp-stapling/)

Broadly, my observations are;

- it's very hard to find a working example of a stapled & revoked certificate - none of the CA demo revoked certificate endpoints stapled (or stapled a correct) revoked OCSP response - I was hoping to see how browsers handled such a response (without setting up through my own demoCA)
- CRLs died off some years ago, favouring OCSP... now OCSP is dying off, favouring CRLs, but using an alternate distribution mechanism to endpoints such as CRLite & CRLsets
- it's unclear how CRLite/CRLsets apply outwith the typical browser/browser engine models

A key point - many of these revocation standards/protocols etc. feel like they are dancing round an a fairly obvious solution which OCSP stapling re-enforces - short lived certificates seem like answer.  

With OCSP stapling, services are effectively asking their CAs for a signed object, with a shorter expiry time than the *real* certificate, that declares whether the certificate remains valid or not, and offering that to clients alongside the certificate itself.Given that identity verification for issuance (which I covered in [this post](/posts/service-certificates/#the-serviceserver-identification-model)) can now be an automated, and rapidly checked process, it feels like short lived certificates will be the far simpler approach, typically avoiding the need for revocation checks at all.
