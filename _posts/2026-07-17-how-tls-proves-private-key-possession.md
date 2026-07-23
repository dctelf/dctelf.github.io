---
title: "How does TLS prove private key possession?"
date: 2026-07-17
categories: []
tags: []
math: true
---

I thought it worth exploring how TLS provides mechanisms to prove to the client (or service in the mTLS context) that the service holds the private key matching the public key within the certificate.  

This is a fundamental aspect of asymmetric cryptography - without this capabiilty, anyone could take a copy of the public certificate and pass off as the service - this would undermine the authentication aspects of TLS.

## Questions to explore

- What stages of the TLS protocol handshake demonstrate private key ownership
- What is actually signed to demonstrate ownership, and why
- What signing algorithms are used - is this the same as certificate trust chain signatures
- Does mTLS perform this process twice, or is it more involved/complicated
- Has TLS always behaved the way TLS 1.3 does, or have there been shifts in the protocol

## The TLS connection 

For the purposes of the initial exploration of this topic, I'll stick with the TLS 1.3 protocol approach, but later in this post I'll dive a little into the changes that have taken place in signature algorithms and schemes - in particular [this post](https://timtaubert.de/blog/2016/07/the-evolution-of-signatures-in-tls/) is a useful summary of the evolution from TLS 1.0 through TLS 1.3.

[This section (4.5.2)](https://datatracker.ietf.org/doc/html/rfc9846#name-certificate-verify) of [rfc9846](https://datatracker.ietf.org/doc/html/rfc9846) for TLS 1.3 (that recently superceded [section (4.4.3)](https://datatracker.ietf.org/doc/html/rfc8446#section-4.4.3) of [rfc8446](https://datatracker.ietf.org/ doc/html/rfc8446) ) outlines the specific stage in the handshake that performs the proof of private key posession.

The excellent [Illustrated TLS 1.3 Connection](https://tls13.xargs.org/#server-certificate-verify/annotated) page also clealry shows where in the protocol this event happens, and provides a description of what is signed as part of this message.

## Setting up the test CA and connection

I'll re-use the same CA steps from earlier posts [here](/posts/what-makes-an-intermediate-certificate-intermediate/) and [here](/posts/rsa-x509-signing/).

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
            30:6b:57:eb:97:3f:1e:17:9a:82:57:e4:c0:02:5c:3f:e5:b3:2d:ea
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = dummyCA
        Validity
            Not Before: Jul 20 20:34:26 2026 GMT
            Not After : Jul 20 20:34:26 2027 GMT
        Subject: CN = dummyCA
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:94:83:a5:08:e3:50:2c:66:b7:88:0f:e7:92:7f:
                    97:21:d7:ab:f6:75:fe:0b:41:eb:27:0e:f9:c9:e3:
                    c7:02:e9:49:c6:12:8b:df:d6:03:61:31:d2:d6:38:
                    7c:a8:03:a1:f3:52:aa:e2:fc:4c:e4:3a:01:09:f5:
                    18:07:cb:45:22:70:d7:dc:90:a1:de:f0:9c:d2:10:
                    0c:54:d2:42:ec:2e:a6:37:d2:28:a3:23:d3:6a:7f:
                    ad:50:4f:4e:99:1a:61:81:b4:4c:58:75:c2:ad:cc:
                    f0:90:9f:48:ff:88:14:f1:0b:8f:c3:91:42:6d:1a:
                    51:06:98:56:d0:33:29:76:8f:dc:68:d6:94:28:f5:
                    a0:57:58:f9:50:b8:19:33:5f:3e:8c:ba:f7:9a:d6:
                    69:05:72:c2:75:80:fd:57:4d:ca:ac:c4:9b:94:dd:
                    7a:3d:01:0d:d9:2b:b4:6b:5d:02:f1:3e:d3:bb:b4:
                    5d:5f:9a:48:03:6b:fc:90:b5:bd:17:30:47:49:44:
                    22:2b:0f:41:a7:a8:e1:9f:32:63:b3:b8:c1:aa:46:
                    ba:6c:15:c9:8a:d1:9c:d4:f9:a2:ad:c3:f7:52:0b:
                    ac:5d:39:76:fb:ac:8f:50:03:8f:f8:8b:ef:1b:86:
                    d9:58:69:36:bf:4f:ea:87:59:32:f4:15:ba:04:ef:
                    e7:77
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                78:72:36:B8:BA:46:61:CF:E6:D0:66:FD:9C:E0:13:8F:B4:4A:3B:26
            X509v3 Authority Key Identifier:
                78:72:36:B8:BA:46:61:CF:E6:D0:66:FD:9C:E0:13:8F:B4:4A:3B:26
            X509v3 Basic Constraints: critical
                CA:TRUE
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        6d:be:c6:bc:1a:46:27:b6:70:e0:b1:e1:c3:21:63:7a:16:a3:
        54:ae:36:f7:80:06:45:ba:98:39:dc:90:6d:c0:7c:28:09:76:
        fe:62:80:b4:91:60:05:cd:ef:bc:47:57:c1:ef:67:07:13:8b:
        79:e2:68:4c:3d:14:ab:6f:88:55:b6:a6:0e:eb:00:68:7b:1b:
        75:9f:0f:e1:12:b7:8d:df:6e:bc:5d:82:97:b9:84:24:93:7a:
        e5:5a:45:e5:9a:25:3b:6e:16:eb:53:b7:c0:ca:13:dc:b8:dd:
        90:31:95:d4:d5:fa:45:dd:d6:2b:67:22:79:0b:72:44:f3:4a:
        24:97:d2:bf:97:e7:ac:13:cb:b4:ed:64:c2:dc:1c:b2:ad:95:
        a3:d8:3a:be:d7:24:f0:0f:14:00:5e:b4:44:aa:9d:be:c8:8a:
        8c:9e:f4:84:0b:c3:e8:49:57:d8:bb:33:a5:c9:4a:a8:98:c1:
        8c:25:82:a6:94:81:e1:cd:c0:c1:59:5b:d1:72:f3:77:09:2e:
        1c:63:85:7f:93:4c:1e:76:2a:a5:2f:23:71:aa:f2:ae:b7:a3:
        1d:72:99:86:f4:2d:1a:bd:73:12:66:05:6d:1d:be:b3:fe:a8:
        7d:95:7d:4d:d2:e6:6d:75:f1:6a:8a:02:54:14:58:63:26:32:
        5d:07:2b:a8
```

</details>

I'll also show the values of the private key — this is for demonstration purposes, so sharing it here is fine.

<details markdown="1">
<summary>
<i>Expand for full `-text` openssl output of this private key</i>

</summary>

```text
$ openssl rsa -in CA-key.key -text
Private-Key: (2048 bit, 2 primes)
modulus:
    00:94:83:a5:08:e3:50:2c:66:b7:88:0f:e7:92:7f:
    97:21:d7:ab:f6:75:fe:0b:41:eb:27:0e:f9:c9:e3:
    c7:02:e9:49:c6:12:8b:df:d6:03:61:31:d2:d6:38:
    7c:a8:03:a1:f3:52:aa:e2:fc:4c:e4:3a:01:09:f5:
    18:07:cb:45:22:70:d7:dc:90:a1:de:f0:9c:d2:10:
    0c:54:d2:42:ec:2e:a6:37:d2:28:a3:23:d3:6a:7f:
    ad:50:4f:4e:99:1a:61:81:b4:4c:58:75:c2:ad:cc:
    f0:90:9f:48:ff:88:14:f1:0b:8f:c3:91:42:6d:1a:
    51:06:98:56:d0:33:29:76:8f:dc:68:d6:94:28:f5:
    a0:57:58:f9:50:b8:19:33:5f:3e:8c:ba:f7:9a:d6:
    69:05:72:c2:75:80:fd:57:4d:ca:ac:c4:9b:94:dd:
    7a:3d:01:0d:d9:2b:b4:6b:5d:02:f1:3e:d3:bb:b4:
    5d:5f:9a:48:03:6b:fc:90:b5:bd:17:30:47:49:44:
    22:2b:0f:41:a7:a8:e1:9f:32:63:b3:b8:c1:aa:46:
    ba:6c:15:c9:8a:d1:9c:d4:f9:a2:ad:c3:f7:52:0b:
    ac:5d:39:76:fb:ac:8f:50:03:8f:f8:8b:ef:1b:86:
    d9:58:69:36:bf:4f:ea:87:59:32:f4:15:ba:04:ef:
    e7:77
publicExponent: 65537 (0x10001)
privateExponent:
    0b:4c:80:af:ce:6b:79:15:4f:7d:40:88:83:b2:c5:
    52:c3:cf:c7:6e:6e:a7:78:9a:65:5c:54:50:b1:cd:
    a0:41:13:65:c8:5f:6f:e6:1e:57:b4:ac:af:b3:98:
    78:47:de:78:5e:9f:b5:a9:30:48:64:c9:53:72:9c:
    23:6b:a9:94:d7:34:f5:08:e3:e7:cc:32:82:20:ca:
    6f:61:97:c9:d4:3a:bd:20:76:0b:03:5c:c0:4b:7a:
    6a:13:be:8d:13:5e:bb:b9:75:dd:7d:08:14:a4:f4:
    e0:6b:dd:e7:e2:f8:84:e6:36:47:d0:b3:57:0d:9b:
    80:7e:f2:8b:e0:78:95:16:7b:23:bb:26:b0:1b:5f:
    be:56:82:fa:99:22:b1:42:7e:82:79:e5:9a:f0:58:
    96:35:b6:ee:27:6a:dc:26:48:7a:e1:cc:f7:60:3d:
    d7:0d:da:8b:24:54:b5:e4:81:9b:6a:eb:b0:09:5c:
    9b:9a:d8:06:9f:f8:3d:9e:5e:77:ba:ab:7c:59:ee:
    9c:0b:7e:4f:84:0b:7d:5b:76:07:ef:61:ee:96:86:
    23:41:6d:3a:c4:e9:55:bd:76:a9:42:6e:0c:fe:dd:
    33:d8:f5:2d:22:76:3e:9c:1d:28:a5:80:f0:a1:17:
    f6:86:56:c9:ff:b7:53:46:42:ae:cd:3a:77:e3:c7:
    e1
prime1:
    00:ca:65:5c:18:ee:b2:81:01:c2:72:e8:f2:6e:b0:
    23:2e:b4:38:ad:7c:62:c3:73:88:ca:86:67:0e:cd:
    14:4b:21:24:86:89:d9:e3:80:bd:b8:55:da:55:50:
    9e:e4:a3:2f:1e:b9:83:8b:8c:7b:dc:b7:e9:58:30:
    49:c1:eb:9d:e8:0e:55:bf:1c:88:bf:c3:49:de:9e:
    ec:24:1d:d4:d1:d7:ed:dd:a3:62:29:25:7b:9c:80:
    20:18:61:60:6e:a2:ad:03:d1:8d:8d:a0:a8:73:b2:
    ec:d2:79:cd:a5:7d:f0:1c:41:bb:4e:5f:bb:2f:8c:
    7e:ca:a1:7c:7f:15:de:63:29
prime2:
    00:bb:d9:0d:b6:b8:c5:21:ea:93:11:97:b8:a9:42:
    95:4f:27:74:6c:2b:15:b6:47:8d:7d:be:19:fb:80:
    34:89:72:3f:41:72:41:72:6b:90:fc:88:d1:6e:30:
    24:51:72:7d:03:35:11:e0:aa:2c:b6:2e:73:31:cc:
    6a:f3:27:d1:0d:3b:ae:0e:22:a5:9a:7c:3a:4c:f7:
    7f:45:78:3e:69:4d:72:30:62:09:58:81:28:32:83:
    1e:ef:b7:6f:95:72:a2:6c:06:ae:d6:40:3b:b1:95:
    c7:94:7b:d5:84:24:01:4c:41:2d:2c:3b:83:00:09:
    a9:53:37:98:a6:cd:10:e9:9f
exponent1:
    62:3f:43:ae:a2:a8:29:f1:75:b7:9c:16:9a:de:8b:
    a5:8f:3c:78:12:8a:4a:c0:59:a5:9e:0a:86:e7:cc:
    33:10:1a:8f:e8:78:c9:73:e4:24:88:20:5d:0b:ae:
    a5:e4:04:ea:90:39:27:d3:81:08:ca:89:ce:12:5a:
    ab:74:b9:89:3c:f4:28:ba:2c:33:92:13:d8:aa:22:
    8d:01:a2:1e:5f:08:0b:6f:d5:25:8e:19:6c:05:d2:
    0e:a3:ae:50:e6:4c:c0:2e:c7:dc:f9:20:ec:50:ed:
    9e:da:1b:96:7b:04:c4:62:b0:0e:c2:6f:b6:0c:28:
    3c:2a:99:a9:83:2f:19:c9
exponent2:
    26:a5:6a:1f:dc:75:9a:1b:b3:74:1c:1d:be:9c:d7:
    30:f8:b2:08:0a:f9:25:8e:24:fa:e8:a0:59:d0:af:
    7e:53:85:d6:06:16:96:de:b0:6e:74:0b:7a:3a:e7:
    4d:e6:5a:f7:cc:f4:47:9f:5b:21:83:fe:e9:10:e0:
    33:f4:4e:1b:05:db:32:47:48:80:b6:ec:1b:a7:93:
    84:8c:4f:72:c4:9f:28:7b:12:e7:25:73:4a:a9:15:
    35:46:2c:eb:b7:30:d9:3e:aa:bb:a3:6d:64:84:a7:
    11:d2:44:44:32:50:1e:0b:0e:ab:19:f7:42:8b:ba:
    4d:47:93:dd:45:35:24:8b
coefficient:
    71:0f:b4:54:3e:1c:dc:13:08:68:1a:b4:33:6a:ce:
    d1:91:1a:57:ac:1c:3d:a6:e3:5f:de:9e:52:f1:a3:
    b6:14:06:88:d1:ee:88:84:51:14:70:ad:1d:a8:17:
    93:9c:2c:7a:44:87:ec:eb:ee:e5:da:22:0b:ef:63:
    60:8f:2b:d5:79:8d:32:62:b7:7e:ec:a2:88:a2:72:
    cb:f9:00:ba:72:36:88:79:c9:e7:db:93:ed:a5:69:
    41:a7:dc:5b:e2:56:c3:d7:3e:fb:8c:12:cf:8e:af:
    c4:0b:b3:4f:70:a2:59:6a:4e:db:f0:ec:2a:d4:a4:
    d3:39:5a:44:72:96:3c:74
writing RSA key
-----BEGIN PRIVATE KEY-----
MIIEvAIBADANBgkqhkiG9w0BAQEFAASCBKYwggSiAgEAAoIBAQCUg6UI41AsZreI
D+eSf5ch16v2df4LQesnDvnJ48cC6UnGEovf1gNhMdLWOHyoA6HzUqri/EzkOgEJ
9RgHy0UicNfckKHe8JzSEAxU0kLsLqY30iijI9Nqf61QT06ZGmGBtExYdcKtzPCQ
n0j/iBTxC4/DkUJtGlEGmFbQMyl2j9xo1pQo9aBXWPlQuBkzXz6Muvea1mkFcsJ1
gP1XTcqsxJuU3Xo9AQ3ZK7RrXQLxPtO7tF1fmkgDa/yQtb0XMEdJRCIrD0GnqOGf
MmOzuMGqRrpsFcmK0ZzU+aKtw/dSC6xdOXb7rI9QA4/4i+8bhtlYaTa/T+qHWTL0
FboE7+d3AgMBAAECggEAC0yAr85reRVPfUCIg7LFUsPPx25up3iaZVxUULHNoEET
Zchfb+YeV7Ssr7OYeEfeeF6ftakwSGTJU3KcI2uplNc09Qjj58wygiDKb2GXydQ6
vSB2CwNcwEt6ahO+jRNeu7l13X0IFKT04Gvd5+L4hOY2R9CzVw2bgH7yi+B4lRZ7
I7smsBtfvlaC+pkisUJ+gnnlmvBYljW27idq3CZIeuHM92A91w3aiyRUteSBm2rr
sAlcm5rYBp/4PZ5ed7qrfFnunAt+T4QLfVt2B+9h7paGI0FtOsTpVb12qUJuDP7d
M9j1LSJ2PpwdKKWA8KEX9oZWyf+3U0ZCrs06d+PH4QKBgQDKZVwY7rKBAcJy6PJu
sCMutDitfGLDc4jKhmcOzRRLISSGidnjgL24VdpVUJ7koy8euYOLjHvct+lYMEnB
653oDlW/HIi/w0nenuwkHdTR1+3do2IpJXucgCAYYWBuoq0D0Y2NoKhzsuzSec2l
ffAcQbtOX7svjH7KoXx/Fd5jKQKBgQC72Q22uMUh6pMRl7ipQpVPJ3RsKxW2R419
vhn7gDSJcj9BckFya5D8iNFuMCRRcn0DNRHgqiy2LnMxzGrzJ9ENO64OIqWafDpM
939FeD5pTXIwYglYgSgygx7vt2+VcqJsBq7WQDuxlceUe9WEJAFMQS0sO4MACalT
N5imzRDpnwKBgGI/Q66iqCnxdbecFprei6WPPHgSikrAWaWeCobnzDMQGo/oeMlz
5CSIIF0LrqXkBOqQOSfTgQjKic4SWqt0uYk89Ci6LDOSE9iqIo0Boh5fCAtv1SWO
GWwF0g6jrlDmTMAux9z5IOxQ7Z7aG5Z7BMRisA7Cb7YMKDwqmamDLxnJAoGAJqVq
H9x1mhuzdBwdvpzXMPiyCAr5JY4k+uigWdCvflOF1gYWlt6wbnQLejrnTeZa98z0
R59bIYP+6RDgM/ROGwXbMkdIgLbsG6eThIxPcsSfKHsS5yVzSqkVNUYs67cw2T6q
u6NtZISnEdJERDJQHgsOqxn3Qou6TUeT3UU1JIsCgYBxD7RUPhzcEwhoGrQzas7R
kRpXrBw9puNf3p5S8aO2FAaI0e6IhFEUcK0dqBeTnCx6RIfs6+7l2iIL72NgjyvV
eY0yYrd+7KKIonLL+QC6cjaIecnn25PtpWlBp9xb4lbD1z77jBLPjq/EC7NPcKJZ
ak7b8Owq1KTTOVpEcpY8dA==
-----END PRIVATE KEY-----
```

</details>

Using `openssl s_server` to start a service listening on port 4443 and showing all protocol messages on the console.

```bash
$ openssl s_server -accept 4443 -cert dummyCA.pem \
-key dummyCA.key -tls1_3 -msg
```

I can then call this with `openssl s_client`, similarly showing all protocol messages

```bash
$ openssl s_client -connect localhost:4443 -tls1_3 -msg -CAfile ./dummyCA.pem
```

From the client side (truncating the output) we can see the contents of the certificateVerify message - this is the data containing the signature passed from the server to the client that, when verified by the client using the public key from the certificate, proves that the server posesses the equivalent private key.

```bash
~~~
<<< TLS 1.3, Handshake [length 0108], CertificateVerify
    0f 00 01 04 08 04 01 00 23 27 bb 80 e9 b4 61 0d
    12 d2 b1 5b 6a 4f ea aa 41 5b e9 ab 41 6c 43 2f
    c7 ac 1e ee 52 bc cb bb 54 72 51 7a 50 a1 ed ce
    c3 5f c4 e6 7f f3 2e 54 54 73 71 09 eb 43 26 17
    21 f0 72 f1 87 e5 d5 77 97 aa 81 7e 0b 55 4b 09
    0f a8 ed 90 61 3e d6 c6 64 a1 88 db 5f a7 23 02
    a6 6c 85 51 d4 43 99 02 fa ca e6 25 53 78 c1 2a
    cd 25 17 20 a1 8f 3e f1 f5 09 d7 96 cb 5a d6 12
    b8 01 6f 97 ff 54 26 46 e8 63 85 af 9f 61 56 57
    98 3c 17 6d 46 9c 51 49 e3 6e 29 e5 8d 00 c6 48
    80 44 37 3c 0d 60 d5 7c e7 70 b1 8c 81 b1 d4 27
    c5 58 c1 af a4 37 b4 d7 6e 44 73 d7 66 fd cf de
    71 67 c5 26 8a 94 23 97 b6 da df 28 d5 c4 4e b6
    7f a9 64 b4 74 0b 64 d5 87 76 d3 77 b8 62 d6 75
    3e d9 c9 3c 09 fd 89 ff e8 8b 82 57 cf f2 32 37
    57 03 99 26 9e 78 46 ed b5 7c 0b 74 1a 7d 89 15
    18 d6 a0 f0 cb 3d a4 d1
~~~
```

> I had assumed that this part of the handshake would be visible from a packet capture however, on review of some pcaps, it was apparent that the CertificateVerify message (along with the contents of the presented certificate) weren't clearly visible - but were visible in a TLS 1.2 handshake.  After some reading I know understand that TLS 1.3 introduced a significant change in the transmission of handshake contents - in TLS 1.2, the contents of the CertificateVerify message (along with other parts of the handshake, such as the service certificate) were transmitted in the clear.<br /><br />TLS 1.3 was designed to encrypt more of the handshake message (see [slide 4 of the TLS 1.3 wishlist from Eric Rescorla at IETF 87](https://www.ietf.org/proceedings/87/slides/slides-87-tls-5.pdf) and some of the clouflare blog summaries [here](https://blog.cloudflare.com/encrypted-client-hello/#handshake-encryption-in-tls) and [here](https://blog.cloudflare.com/rfc-8446-aka-tls-1-3/) ) - the core purpose is for privacy purposes as sharing the certificate(s) alongside other elements of the handshake in the clear exposes a variety of identity and connection context that introduces a variety of risks.<br /><br />As a side note, this remains a problem beyond the TLS handshake due to SNI - [this post](https://blog.cloudflare.com/encrypted-client-hello/#background) gives a summary, alongside the mitigation through adoption of Encrypted Client Hello (ECH).
{: .prompt-info }

## What CertificateVerify signs

For an RSA signed certificate, to prove that the server possesses the private key, the client and the server must agree a set of data already shared between both parties that the server will apply some encoding scheme, then sign with the private key - the client can then verify this by taking the signature, raising to the power of the public key (modulo N) and then performing the inverse of the encoding scheme utilising the shared data to determine whether the siganuture was valid.  Only (or rather, with an extremely high probabibility) by being in posession of the private key can the server generate such a valid signature.  This previous [post](/posts/rsa-x509-signing/) demonstrates the equivalent RSA signature verificaiton process as used when verifying a certificate chain signature under the PKCS#1 v1.5 encoding scheme - TLS 1.3 .



## Why the transcript, not a nonce

## The input structure: padding, context string, and transcript hash

## Signature algorithm choices: PSS for handshake messages, PKCS#1 v1.5 for chain certificates

## mTLS: the same mechanism, symmetrically applied

## Contrast with TLS 1.2: implicit proof via RSA decryption
