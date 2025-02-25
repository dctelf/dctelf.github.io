---
title: Approaches to certificate revocation - part 2
date: 2025-02-25 08:00:00 +/-0000
categories: [Revocation]
tags: [x509, revocation, pki]
math: true
---

Following on from [part 1](/posts/certificate-revocation1/) where I explored the formative CRL and OCSP approaches to certificate revocation, I though it worthwhile to explore some of the revocation approaches that have introduced a return to the use of CRLs - these approaches entail browser vendors distributing some form of revocation list using alternative methods.

## Questions to explore

- What are the differing approaches taken by browser vendors
- specifically how do these approaches work
  - Google CRLSets ( & blocklist)
  - Mozilla CRLite ( & OneCRL )
- conceptually how do these approaches work
  - How do browser vendors know about revocations to pass this information on
  - Why not ship the browser applications with these lists embedded
  - What do these approaches achieve from a revocation handling perspective
- can I find example revocations within these mechanisms
 

## Differing approaches

The newer revocation approaches described here appear not to be IETF standards based, and are instead led from (differing, but similar) browser vendor needs and goals - as such, there isn't an authoritative list of all approaches.  Instead, I've found a few sources that describe the approaches taken by prominent browser vendors.

There is a section at the top of the crt.sh page for certificates showing revocation statuses (this taken from the page for the [current at time of writing wikipedia.com certificate](https://crt.sh/?id=16299019531) ):

![crt.sh screenshot of revocations for wikipedia.com](/assets/img/2025-02-25-revocation2-crtsh-screenshot.png){: width="972" height="589"}
_crt.sh screenshot of revocations for wikipedia.com_

Similarly, posts such as [this one from Let's Encrypt](https://letsencrypt.org/2022/09/07/new-life-for-crls/) describe their changes to publish CRLs in support of the various browser shifts to browser-summarized CRLs.

This suggests that the approaches taken are;

- Mozilla Firefox: OneCRL and CRLite
- google Chrome: CRLsets and blocklist
- Microsoft Edge: dissalowedcert.stl

I couldn't find a description of the approach taken by Apple for Safari, although do note that the [Apple Root Certificate Program requirements](https://www.apple.com/certificateauthority/ca_program.htm), section 2.1.2 states a need for CA providers to publish CRL distribution points through the [Common CA Database](https://www.ccadb.org/) which suggests that Apple also follow some sort of CRL distribution approach.

I've found it difficult to confirm whether the public CRL lists published by CAs (often, URLs for these are stored within the `X509v3 CRL Distribution Points` extension in certificates) are the same as those shared between CAs and Browser vendors via groups such as CCADB - I think the CCADB ["All Certificate Information (root and intermediate) in CCADB (CSV)" file](https://ccadb.my.salesforce-sites.com/ccadb/AllCertificateRecordsCSVFormatv2) may be the authoritative source allowing any public consumer to access the same CRL lists as made available to browser vendors.

## Specific models

### Mozilla CRLite & OneCRL

The approaches taken by Mozilla appear to have the best documentation on background both conceptually and technically - there are a series of 4 detailed blog posts (all can be found with this [tag on the mozilla security blog](https://blog.mozilla.org/security/tag/crlite/)).  The [first article](https://blog.mozilla.org/security/2020/01/09/crlite-part-1-all-web-pki-revocations-compressed/) gives the background on both technical and conceptual topics.  This [Mozilla wiki page](https://wiki.mozilla.org/CA/Revocation_Checking_in_Firefox) also outlines the concepts more broadly on revocation checking, although that page includes references to dates and events which may now be out of date, and may not give an authoritative current state of revocation within Firefox.

My understanding is that Firefox currently operates a mix of approaches, and is adapting to changes such as the move away from OCSP by some CAs - from a CRL-style perspective, the 2 approaches in use are;

- [OneCRL](https://blog.mozilla.org/security/2015/03/03/revoking-intermediate-certificates-introducing-onecrl/)
- [CRLite](https://blog.mozilla.org/security/2020/01/09/crlite-part-1-all-web-pki-revocations-compressed/)

#### OneCRL

The OneCRL section on the Mozilla Firefox Wiki best described the purpose and intent of OneCRL.

![OneCRL description screenshot from wiki.mozilla.org](/assets/img/2025-02-25-onecrl-screenshot.png){: width="972" height="589"}
_OneCRL description screenshot from wiki.mozilla.org_

The primary observation I've take from this is that OneCRL, due to the low volume of revocations of root and intermediate certificates (which are typically seen as impactful but infrequent events in the world of WebPKI security), involves relatively simple technologies which underpin the distribution of the list of certificate identifiers.

The mechanism of distribution appears to be based upon a process of polling the CCADB list of CA intermediates, parsing the list, then distributing revocation data to Firefox installs through the Firefox [RemoteSettings](https://remote-settings.readthedocs.io/en/latest/) mechanism  - much of this is inferred through a variety of docs and sources for firefox such as the [OneCRL Tools](https://github.com/mozilla/OneCRL-Tools) & [CCADB Tools](https://github.com/mozilla/CCADB-Tools) Github repos.

All of the RemoteSettings data published is listed [here](https://mozilla-services.github.io/remote-settings-permissions/), with the current list of OneCRL revocations as published by Firefox RemoteSettings held [here](https://firefox.settings.services.mozilla.com/v1/buckets/security-state/collections/onecrl/changeset?_expected=0)

From the browser side, this can be verified after [enabling the browser toolbox](https://firefox-source-docs.mozilla.org/devtools-user/browser_toolbox/index.html) within Firefox - once enabled, the Browser Toolbox can be used to view the content of RemoteSettings values held within the browser.

![OneCRL entry location within Firefox storage](/assets/img/2025-02-25-onecrl-toolbox-attachments-location-screenshot.png){: width="650"}
_OneCRL entry location within Firefox storage_

From a CT monitoring perspective, the crt.sh tool has a filter to [show only those certificates which are included within the OneCRL list](https://crt.sh/mozilla-onecrl).

> There are around 1600 revoked intermediates distributed through this mechanism (from the CCADB list, through Firefox infrastructure, and onwards down to the browsers).  This list has an (understandably) slow rate of change, and the volume of data is fairly low, hence this feels like an effective mechanism to handle revocation of intermediates.<br /><br />I started to explore in further detail how the internals of the Firefox RemoteSettings mechanism works, but this was starting to drift from the core question on revocation approaches.<br /><br />**Given that there are no obvious ways to find a certificate that was signed by a now revoked intermediate (where that intermediate has ended up in the OneCRL list, so not "test" revocations) it is hard to demonstrate this mechanism working "in anger"**
{: .prompt-info }


#### CRLite

The Mozila CRLite approach is well documented, with a series of [blog posts](https://blog.mozilla.org/security/2020/01/09/crlite-part-1-all-web-pki-revocations-compressed/) and extensive documentation on the [CRLite github repo](https://github.com/mozilla/crlite) - this [Real Word Crypto conference session](https://www.youtube.com/watch?v=kED3K57EV6w) (and [slides](https://rwc.iacr.org/2020/slides/Thyla.pdf) ) is particularly informative, as is the [FAQ on the github repo](https://github.com/mozilla/crlite/wiki#faq).  CRLite appears to be based on a formal paper submitted to the IEEE Symposium on Security & Privacy: [CRLite: A Scalable System for Pushing All TLS Revocations to All Browsers](https://www.ccs.neu.edu/home/cbw/static/pdf/larisch-oakland17.pdf)

In essence, the CRLite model is an attempt to publish a form of **all** non-expired, revoked certificates to browsers allowing for revocation checking without the need for the browser to contact OCSP responders, rely on stapled OCSP responses or download all encountered CRLs -  I won't attempt to summarise these posts and docs any further here as they are well written and clearly explain the purpose of CRLite.  

The bloom filter based approach is interesting however, and worthy of a deeper dive, so I intend to write a future post that explores bloom filters in more detail.

As outline in the Mozilla posts, CRLite acquires the list of certificates (and therefore CRLs) through monitoring CT logs - I'll address the CT log mechanism in a future post as well, but from the perspective of CRLite, this offers a list of all certificates that Firefox would potentially accept (noting that Firefox, like other browsers, now requires that presented certificates contain signed certificate timestamp [SCT] entries).

There are a number of tools created by Mozilla to work with the CRLite ecosystem such as [moz_crlite_query](https://github.com/mozilla/moz_crlite_query) and [rust-query-crlite](https://github.com/mozilla/crlite/tree/main/rust-query-crlite).

> Initially the moz_crlite_query tool looked to have a larger set of features to explore the state of revocation data, however I had a number of issues with the tool, including issues within dependencies which themselves have [long outstanding pull requests](https://github.com/mozilla/filter-cascade/pull/26).<br /><br />From a [comment](https://github.com/mozilla/moz_crlite_query/issues/29#issuecomment-1145342610) on a related issue from another mozilla repo, it looks like the moz_crlite_query tool is no longer maintained, and changes in the upstream data set from Mozilla have broken the functionality (I started to make some changes to get this working again, but after encountering further issues though it best to stay on topic and work with the more current rust-query-crlite tool instead).
{: .prompt-info }

Using `rust-query-crlite` I can evaluate some of the demo certificates - note that I had a bit of a hard time getting this working as the docs were a touch unclear on how to do the first creation/update of the filter, but with a bit of trial and error I realised this just entailed trying to very a certificate (either locally or via the tools https connection option) with the update flag set.

Getting the tool downloaded and setup;

```bash
$ git clone https://github.com/mozilla/crlite.git
Cloning into 'crlite'...
remote: Enumerating objects: 4492, done.
remote: Counting objects: 100% (916/916), done.
remote: Compressing objects: 100% (416/416), done.
remote: Total 4492 (delta 558), reused 500 (delta 499), pack-reused 3576 (from 3)
Receiving objects: 100% (4492/4492), 6.38 MiB | 2.84 MiB/s, done.
Resolving deltas: 100% (2517/2517), done.
$ cd crlite/rust-query-crlite/
$ cargo build

[[[truncated]]]

$ cd ~
$ ./crlite/rust-query-crlite/target/debug/rust-query-crlite --update prod -vvv https wikipedia.com
INFO - Fetching cert-revocations records from remote settings https://firefox.settings.services.mozilla.com/v1/buckets/security-state/collections/
INFO - Fetching 20250223-0-default.filter.delta from https://firefox-settings-attachments.cdn.mozilla.net/security-state-staging/cert-revocations/694f2988-c59f-43b2-8246-f66d80152cc5.delta
[[[ truncated - around 20 more delta files fetched]]]
INFO - Fetching https://ccadb.my.salesforce-sites.com/mozilla/MozillaIntermediateCertsCSVReport
DEBUG - Loaded certificate from wikipedia.com
DEBUG - Issuer DN: C=US, O=Let's Encrypt, CN=E5
DEBUG - Serial number: 04f018ee5c9aeda1453112cc158dcacc1418
DEBUG - Issuer SPKI hash: 3586d4ecf070578cbd27aedce20b964e48bc149faeb9dad72f46b857869172b8
INFO - wikipedia.com Good
```

This process creates a directory within the cwd called `crlite_db`, and within there stores;

- A `default-filter` file, approx 3.7MB in size, dated 11/02/2025
- 31 delta files all around 100KB in size, with filenames suggesting 4 deltas per day, up to the current date
- A `crlite.intermediates` file of around 2.4MB

```bash
20250211-2-default.filter        20250216-1-default.filter.delta  20250219-0-default.filter.delta
20250213-0-default.filter.delta  20250216-2-default.filter.delta  20250219-1-default.filter.delta
20250214-0-default.filter.delta  20250216-3-default.filter.delta  20250220-0-default.filter.delta
20250214-1-default.filter.delta  20250217-0-default.filter.delta  20250220-1-default.filter.delta
20250214-2-default.filter.delta  20250217-1-default.filter.delta  20250221-0-default.filter.delta
20250214-3-default.filter.delta  20250217-2-default.filter.delta  20250221-1-default.filter.delta
20250215-0-default.filter.delta  20250217-3-default.filter.delta  20250222-0-default.filter.delta
20250215-1-default.filter.delta  20250218-0-default.filter.delta  20250222-1-default.filter.delta
20250215-2-default.filter.delta  20250218-1-default.filter.delta  20250223-0-default.filter.delta
20250215-3-default.filter.delta  20250218-2-default.filter.delta  crlite.intermediates
20250216-0-default.filter.delta  20250218-3-default.filter.delta
```

This aligns to the approach described in the various blog posts on CRLite - the only exception is the intermediates file which I'm not entirely clear on the purpose of (given the active role of OneCRL as described above).

Now that we have a current filter set held locally, we can run queries - I'll re-use the digicert demo certificates as used in the [previous CRL and OCSP post](/posts/certificate-revocation1/):

```bash
$ ./rust-query-crlite -vvv https digicert-tls-rsa4096-root-g5.chain-demos.digicert.com
DEBUG - Loaded certificate from digicert-tls-rsa4096-root-g5.chain-demos.digicert.com
DEBUG - Issuer DN: C=US, O=DigiCert, Inc., CN=DigiCert G5 TLS RSA4096 SHA384 2021 CA1
DEBUG - Serial number: 0d2454ccc0a54b4228950bd6c066ec41
DEBUG - Issuer SPKI hash: e51d01e1494f7aa98682d7b053dfb4414603bcef7e50d3786322fc4a21ce615a
INFO - digicert-tls-rsa4096-root-g5.chain-demos.digicert.com Good
$ ./rust-query-crlite -vvv https digicert-tls-rsa4096-root-g5-expired.chain-demos.digicert.com
DEBUG - Loaded certificate from digicert-tls-rsa4096-root-g5-expired.chain-demos.digicert.com
DEBUG - Issuer DN: C=US, O=DigiCert, Inc., CN=DigiCert G5 TLS RSA4096 SHA384 2021 CA1
DEBUG - Serial number: 0b01d4f563652678bde3ff5c2166361b
DEBUG - Issuer SPKI hash: e51d01e1494f7aa98682d7b053dfb4414603bcef7e50d3786322fc4a21ce615a
WARN - digicert-tls-rsa4096-root-g5-expired.chain-demos.digicert.com Expired
$ ./rust-query-crlite -vvv https digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com
DEBUG - Loaded certificate from digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com
DEBUG - Issuer DN: C=US, O=DigiCert, Inc., CN=DigiCert G5 TLS RSA4096 SHA384 2021 CA1
DEBUG - Serial number: 0be540a854ce51e943e94a50061b52cb
DEBUG - Issuer SPKI hash: e51d01e1494f7aa98682d7b053dfb4414603bcef7e50d3786322fc4a21ce615a
ERROR - digicert-tls-rsa4096-root-g5-revoked.chain-demos.digicert.com Revoked
```
This all looks exactly as we would hope/expect - CRLite has created an updated set of filter data which we can query real CA signed certificates in various states against, and we see valid results reflecting those certificate statuses.

Within Firefox itself though, on the builds I tested this with (135.0.1 on Ubuntu and Windows), by default the browser didn't error with the revoked digicert cert.  Within `about:config` it looks like the current default setting is to have CRLite turned off.  Enabling this by setting `security.remote_settings.crlite_filters.enabled` to `true` made CRLite spark into life, and I could reliably have Firefox display an error (specifically `Error code: SEC_ERROR_REVOKED_CERTIFICATE`) for all certificates that I could also find as revoked with the `rust-query-crlite` tool.  There are modes of operation for CRLite once enabled, also configurable, determining whether to strictly accept CRLite status only, or attempt CRLite status then fallback to OCSP etc.

> This mechanism appears to work reliably for the small number of demo certificates I could evaluate easily.  I was surprise to see that, 5 years after the introduction of CRLite, this is not something enabled by default.  Given that both the need for presence in CT logs and the existence of CRLs is a pre-requisite for root programmes across all common browsers, and that CCADB now co-ordinates CA info to browser vendors, I'm struggling to envisage the rationale for this mechanism not being used as the primary revocation handling approach in Firefox.<br /><br />I imagine there must be a reason why Mozilla have held back on enforcing CRLite, but I struggled to find any posts, comments or discussions that described the current status and thinking from Mozilla on revocation handling.
{: .prompt-info }

#### A note on CRLs

Differing CAs appear to take slightly differing approaches to the inclusion and publishing of CRL data that could end up in CRLite.

The DigiCert example appears to be straightforward - the end-entity certificates include a CRL endpoint, and that CRL contains the revocation status of the certificate in question...

```bash
$ openssl x509 -in revoked-end-entity.pem -serial -ext crlDistributionPoints -noout
serial=0BE540A854CE51E943E94A50061B52CB
X509v3 CRL Distribution Points:
    Full Name:
      URI:http://crl3.digicert.com/DigiCertG5TLSRSA4096SHA3842021CA1-1.crl
    Full Name:
      URI:http://crl4.digicert.com/DigiCertG5TLSRSA4096SHA3842021CA1-1.crl
$ curl http://crl3.digicert.com/DigiCertG5TLSRSA4096SHA3842021CA1-1.crl --output DigiCertG5TLSRSA4096SHA3842021CA1-1.crl
$ openssl crl -in DigiCertG5TLSRSA4096SHA3842021CA1-1.crl -text -noout | grep -A 4 0BE540A854CE51E943E94A50061B52CB
    Serial Number: 0BE540A854CE51E943E94A50061B52CB
        Revocation Date: Feb 21 16:55:45 2025 GMT
        CRL entry extensions:
            X509v3 CRL Reason Code:
                Superseded
```

Additionally, this CRL distribution point matches that held within the [CCADB list of all root and intermediate certificates](https://www.ccadb.org/resources) for the signing intermediate of this revoked end entity.

The Let's Encrypt model is more complex, but may be understandable due to the shift towards shorter life certificates by Let's Encrypt, and the programmatic means through ACME to create and revoke certificates - the CRLs could become very long very quickly and could be prone to abuse/stuffing causing issues for clients that still attempt to download CRLs in full to check revocation statuses.

The Let's encrypt revoked end entity demo certificate does not have a CRL endpoint extension value set...

```bash
$ openssl x509 -in lenc-revoked.pem -serial -ext crlDistributionPoints -noout
serial=0320AE25BABAA106254F7F32B928258043E5
No extensions in certificate
```

Going up a level to the signing intermediate...

```bash
$ openssl x509 -in lenc-interm.pem -serial -ext crlDistributionPoints -noout
serial=80A97348EF2768A9E3F6BB43C0F9C629
X509v3 CRL Distribution Points:
    Full Name:
      URI:http://x2.c.lencr.org/
```

However, that CRL is empty...

```bash
$ curl http://x2.c.lencr.org/ --output
e6.c.lencr.org-1.crl  lenc-interm.pem       lenc-revoked.pem      lenc.crl              lencr.crl
$ curl http://x2.c.lencr.org/ --output lenc.crl
$ openssl crl -in lenc.crl -text -noout
Certificate Revocation List (CRL):
        Version 2 (0x1)
        Signature Algorithm: ecdsa-with-SHA384
        Issuer: C = US, O = Internet Security Research Group, CN = ISRG Root X2
        Last Update: Dec 11 00:00:00 2024 GMT
        Next Update: Nov 10 23:59:59 2025 GMT
        CRL extensions:
            X509v3 Authority Key Identifier:
                7C:42:96:AE:DE:4B:48:3B:FA:92:F8:9E:8C:CF:6D:8B:A9:72:37:95
            X509v3 CRL Number:
                105
            X509v3 Issuing Distribution Point: critical
                Only CA Certificates

No Revoked Certificates.
    Signature Algorithm: ecdsa-with-SHA384
    Signature Value:
        30:66:02:31:00:af:0c:ff:fc:77:39:02:f0:2f:a8:e5:42:d8:
        1d:c7:6f:ef:87:a7:df:e5:a3:bb:3d:f2:19:11:38:83:b0:f4:
        46:fe:9c:44:c8:42:fa:75:b1:66:61:f6:66:06:7e:1f:1d:02:
        31:00:ed:19:49:69:6a:f7:0d:e4:8b:ef:c9:39:46:f8:b5:47:
        3f:a8:ca:48:bd:4c:40:c9:34:61:05:0b:f7:d6:65:a6:fe:2e:
        86:7e:47:10:04:33:5b:61:56:89:44:2c:07:88

```

If we instead look at the [CCADB list](https://www.ccadb.org/resources), we can see for this intermediate, there is a long sharded list of 128 CRLs endpoints...

```
"[
""http://e6.c.lencr.org/1.crl"",
""http://e6.c.lencr.org/2.crl"",

[[[truncated]]]

""http://e6.c.lencr.org/127.crl"",
""http://e6.c.lencr.org/128.crl""
]"
```

I'm not entirely clear on the approach taken by Let's Encrypt to shard the CRLs - I think it's probably covered somewhere in their [boulder CA repo](https://github.com/letsencrypt/boulder).  Given that the CRLite mechanism works end to end, I've not bothered finding the right shard (or downloading them all and finding my specific revoked certificate), but worth being aware that there is a divergence between CRL endpoint addresses stored in certificate extensions, and the reality of where a CA may publish their CRLs.  The browser vendors (e.g. Mozilla with CRLite & OneCRL in this case) would use the CCADB listing as the authoritative source of CRL locations.


### Google CRLsets & blocklist

The approaches taken within Chrome/Chromium appear to be quite a bit less complex than the Mozilla FIrefox models - there appears to be 2 mechanisms;

- blocklist - a set of certificates that are considered blocked/revoked by Chromium which I believe are shipped as part of the release of Chrome
- CRLsets - a selective list of certificates which the Chrome team choose to distribute dynamically via the Chrome Component Updater mechanism

#### blocklist

The blocklist mechanism is described within [this README in the Chromium source repo](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/net/data/ssl/blocklist/README.md), which also contains some useful background on the certificates that they choose to block with this mechanism.

The [directory this README sits in](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/net/data/ssl/blocklist) also contains a copy of each revoked certificate.

Given that these are long revoked certificates, related to some notable CA issue or private key loss, it's not really possible to demonstrate this in action (barring adding a new certificate into the blocklist source code file, then compiling Chromium myself).

#### CRLsets


This [page within the Chromium Security site](https://www.chromium.org/Home/chromium-security/crlsets/) describes the CRLsets mechanism, and links out to a post on [Adam Langley's blog](https://www.imperialviolet.org/2012/02/05/crlsets.html) which delves further into the rationale for the approach around CRLsets.

Notably, the page on the Chromium site highlights that the process for generating the CRLset is not public, so unlike CRLite, we do not have a clear description of the mechanism.  That page does however describe crawling the CCADB list of CRLs, and the use of certificate transparency logs in general terms, so it's fair to assume that the mechanism is fairly similar to CRLite.  THe key difference appears to be in the selection of certificates to include - it appears that Chrome have taken the approach of minimising the size/length of the CRLsets data distributed to browsers by selecting only a subset of revoked certificates.  Adam Langley's post suggests interaction with CAs to get *"the most important revocations"* inferring a degree of selectivity in this process.

Adam Langley's Github repo contains [crlset-tools](https://github.com/agl/crlset-tools) which allows us to download and parse CRLsets data.

```bash
$ git clone https://github.com/agl/crlset-tools.git
Cloning into 'crlset-tools'...
remote: Enumerating objects: 37, done.
remote: Counting objects: 100% (7/7), done.
remote: Compressing objects: 100% (6/6), done.
remote: Total 37 (delta 1), reused 6 (delta 1), pack-reused 30 (from 1)
Receiving objects: 100% (37/37), 12.07 KiB | 1.01 MiB/s, done.
Resolving deltas: 100% (11/11), done.
$ cd crlset-tools/
$ go build crlset.go
$ ./crlset fetch > crlset240225
Downloading CRLSet version 9574
$ ./crlset dump crlset240225 | head
Sequence: 9574
Parents: 179

03b4392598a10a3ff5695cf02a5775586b170f564a808a4d41568578a184e329
  179cc56dc82dbded5573c7999997f646
  4491ca5825be79842b29b0c37286215f
  5bb8dd672998a2d5c6f3cdd446aac3a2
03cb44b933d7e14551e52ddbfc335a4d57bf65a703667b57ac961de31e3a106d
  01313ac3
  329cab52a80a2e3f
  ```

From the shape of the dump output, we can count the number of revoked certificate serial numbers contained within this current set (840), and the number of issuers in the set (179).

```bash
$ ./crlset dump crlset240225 | egrep '^^[a-f0-9]+' | wc -l
179
$ ./crlset dump crlset240225 | egrep '^\s+[a-f0-9]+' | wc -l
840
```

The data served by google as the CRLset is not based on a structure such as a bloom filter, rather is a structure of bytes as outlined in [this chromium source file](https://chromium.googlesource.com/chromium/src/+/HEAD/net/cert/crl_set.cc).  Given that the set contains only 840 or so certificate serial numbers, there is no need for the more complex data structures warranted by the CRLite model.

We can also see in Chrome through the `chrome://components` page  that the most recent sets are being downloaded (this page describes the [Component Updater](https://chromium.googlesource.com/chromium/src/+/lkgr/components/component_updater/README.md)  mechanism that is used for a variety of purposes within Chrome to acquire updated data sets including CRLsets)

![Components screenshot showing crlset](/assets/img/2025-02-25-revocation2-crlsets-screenshot.png){: width="400"}
_Components screenshot showing crlset_



### Microsoft disallowedcert.

I've elected not to spend too much time digging into the structures and approaches that support the Microsoft revocation mechanism - from the crt.sh page, there is mention of a disallowedcert.stl mechanism - it's difficult to find authoritative information on any Microsoft site on how this is created/what the criteria or process involved etc.

This [thread on the security stackexchange](https://security.stackexchange.com/questions/197256/understanding-microsoft-list-of-disallowed-certificates) gave the best description I could find of the mechanism.  I was able to inspect the contents of the disallowedcert.stl file after base64 encoding, then uploading the [very useful CyberChef tool](https://gchq.github.io/CyberChef/) - this also generates a [deep link to the recipe including the full base64 encoded input](https://gchq.github.io/CyberChef/#recipe=From_Base64('A-Za-z0-9%2B/%3D',true,false)To_Hex('Space',0)Parse_ASN.1_hex_string(0,32)&input=TUlJV3lBWUpLb1pJaHZjTkFRY0NvSUlXdVRDQ0ZyVUNBUUV4RHpBTkJnbGdoa2dCWlFNRUFnRUZBRENDQnlRR0NTc0dBUVFCZ2pjSw0KQWFDQ0J4VXdnZ2NSTUF3R0Npc0dBUVFCZ2pjS0F4NEVPRVFBYVFCekFHRUFiQUJzQUc4QWR3QmxBR1FBUXdCbEFISUFkQUJmQUVFQQ0KZFFCMEFHOEFWUUJ3QUdRQVlRQjBBR1VBWHdBeEFBQUFBZ2dCMjBaMlhzVDYwaGNOTWpReE1qQTBNVGd3TURFMFdqQU9CZ29yQmdFRQ0KQVlJM0Nnc1BCUUF3Z2dhY01CSUVFQ1g3ZWwyRzl5OWVaeWlQZVhNRi9wUXdFZ1FRYnkxRFpjRUNIMXVMWSs4VEs4T3pZREFTQkJDdA0KRWR1M2JKenhxNW1ZellRdXdYWnpNQklFRU4rOTF5K1p3N1pLZVg1YXlXMVp2bFl3RWdRUXhtZ1ZTK2xlRnEyOE1ocThNVzQ0U2pBUw0KQkJBM09TNkRQY1lGM1hzNEpFYzVrNTdqTUJJRUVERjUva3RYSnRqYktxODkrVmpKYTVjd0VnUVF3MXFYeUE5b2ZjUEJDTWFqTTV0bw0KUmpBU0JCQWhHS1RHOXhqUHg5YlllSXhUZE5NcE1CSUVFRkpxT2NCTkZZWXRRbi9aSmE4RE5wQXdFZ1FRUERiaGFLdk1oWlpqN1VlZw0Kd0ZydWVUQVNCQkFCbm4xVzFnMjVyZXhBdVdleHZMcWZNQklFRURiTjZacTRjMytHS0h4WU53VEpYaFl3RWdRUUpwa0tkMWgrMkdRQg0KaE1TVFpxeXdkVEFTQkJEMm5TS3VIdFlWc2JuamtPTVF1N3N4TUJJRUVPdnBDdEVCMDRBcmlreVJQS3p1YWxjd0VnUVFIaVh5VHQrdw0Kd0FRdHUyKzlQQlpQdURBU0JCQXMrNktlS1VHOWFQWnJ5SzB3cU0rZ01CSUVFQVJTR3VDUHgvS2ppeXp6OUN5aXN1OHdFZ1FRQTNzMw0KSVIvZmVFUDY1SjhRbGNuTWZEQVNCQkROaEluY2dXOTdiZFFKSVh0NjZCSGhNQklFRU9HK3lqcGJtNVBBZG9hSzVuSlpZV1F3RWdRUQ0KOUlTc1k0VHBXaEJXR1VqcmhrdlBmakFTQkJERkpNQU9SN3RqU0FaUm52dUNJYXh1TUJJRUVDSkJYaGlCSWhQcExCUmlpeTZkdVRNdw0KRWdRUXZVL3ZHK3lMWnREbTJDOHhwcDltalRBU0JCQmZSc0puVG1yWlJIend1TEdoYUJlMU1CSUVFTWpxVUNnRjZwSHM4aVV3blhhQQ0KMmlBd0VnUVExS1ZpbEd2Yk02emc3dUtkbVFFcVZUQVNCQkN4QVJPaFVVc3J4Sm1PU00vOFE5S3lNQklFRUhXWWdBVG9EKy9hS0N0eA0KNU90TVNpY3dFZ1FRNnRyRGF2YVBLbENGOFJadW9CVUxVakFTQkJBZzRZY1NiRGFjTzU0cGRBYXVnc0RkTUJJRUVPV2l0YXRhbDlOZw0Kcys0TzFCSlJTbWN3RWdRUUJBQnFITlZOMk5yVmNMTXRCV1MwNnpBU0JCQldHZTZyNit5S0o5YTFBYkpiU2pjek1CSUVFQllyR1JHWQ0Kc3Y5N1B6Rys1UWRWd1ZNd0VnUVFadVUwOXpqVCtQMzlWTmprWWV1a3FUQVNCQkNuQ1JIR1lXL09OdHdoNHY5WU1FU0dNQklFRUVhYg0KK3N4dXNFZGFPMzZzQWdjdCtDZ3dFZ1FRN1BVOVdodzJDRkgzZ1krVU40ZTFCekFTQkJEZ3R1Q0FucStidEMzRHJtNk8rVWJUTUJJRQ0KRUVwNERlRjZaYjU5dFE0NG52eXdsaU13RWdRUXliUWx3clFQNkU4YXFEVWZyNERWN2pBU0JCRCs5TkU4bzJMelMrVXA5Z3duWFhGUg0KTUJJRUVOaTdvcEVraEZxTlhTYmRKM2Y3eEpFd0VnUVFIcGNwV2ZYYUZuQmtQMEFJOHhnaVVUQVNCQkJQc0NYSTU4TnJGVTFFeEtWUQ0KNEJNd01CSUVFR0QzZlRHSUdqUHZuR2pWM3UxSGp6a3dFZ1FRTzRQa3pjamRGWENLb3RlMUVDTU1hREFTQkJBT0Y5TzFpNVFGMXJwdQ0KUU5BSjJIamxNQklFRU1Cbmhwd2lrK2gzRXNiUUNLZkVTNWN3RWdRUWJIZlZ0bzN1UVA3WFVlNmk0cWxKdkRBU0JCQ0M1MXc2RjFITQ0KTDZCS05OVmZSWit4TUJJRUVDRzgzM0R1ZDZubnFLUjR1ZHFpcWRNd0VnUVFzMGhYOXI3ZUtwY3FSVmZrRGZreTBqQVNCQkM5L1E2Ug0Kb2xqQTg5WkNSSlFVMTdEN01CSUVFRU41S2I0cm1BVExHci9PM1ZhUVFVUXdFZ1FRbEVLYWx0OWs5MlB4UFFiR1J2OW95VEFTQkJCeQ0KcjNKMDFVRFd5ejU2UG1uY2RtUDNNQklFRUZYdmdpSi9TNVB1OUVybGU5elhyd1F3RWdRUTZtTXZaM0F0aTY0aGlYM0FWb3Q2SnpBUw0KQkJCbTVOZExEOTh6V2VRdkdVZzFkRnRWTUJJRUVMVkovbDBnV2xmZE4xYnRLekxUV2Iwd0VnUVFkakM4KzRmbi9BTXN2UDlESW5aaw0KbnpBU0JCQ1dhak5QRzRsMUsyNmJQV051KzgwRU1CSUVFR0xhOUcxRmh0R0lCSitmSThCaUxHY3dFZ1FRZXp4d2xCd05wVEc2ZnpOcg0KS2tyZ1FUQVNCQkNqOGJXSW5CUXEvNHJKTzBuYis4c1lNRElFTUc1YnBhTG1HOURUZ3h3c3FqT3VmT1hKWHBLcjhGa3YrQU9NemJ4cg0KUytZZE54alJNSS9raVNDZEt5Zzg1TCtPSlRBeUJERGdJRTRQZTZCdUgzVmdpK0c5WW1mRjRMWVRaTXc3VG9jZlloQlVmUXU0TnRxWg0KYmFIVlNXc1VzWnRMYVY2dmFsY3dNZ1F3M0lJTXBJMlhOVjhvQXYwNFFUZUR3QjA4TzQwQ3F0QUhDZVp0VzF2dWZiQWhoT2E2SlhHVQ0KSDVVbCtLdG82VHh0TURJRU1LR2hQVmxOSjNHSGxEWi84OXR4QUVBNWNmYUdoWDBSdUJIUXZwYTZETUIxYWkwK1pLN1A1cThJV0VTSQ0KdHZsdk5EQXlCRERpQlFJa0FxV3BReFhMblZjN3pxblJFdkZacFY4TW5TYnFlZUNIdEtvblVUNWdTTEFJanV4ZUQrWUt4UzNGTW1Bdw0KTWdRd1Q4WUZ3eUwzYjAxbFVYbmJpV0tmT1NqRGJaSzJQMkNiM1ZncFE0ZVliaElVZ2xRYlA4SkdxNXY5d2Riay92ZWpvSUlOR2pDQw0KQmdVd2dnUHRvQU1DQVFJQ0V6TUFBQUJ5TmJhZkV4UFVFNHNBQUFBQUFISXdEUVlKS29aSWh2Y05BUUVMQlFBd2dZRXhDekFKQmdOVg0KQkFZVEFsVlRNUk13RVFZRFZRUUlFd3BYWVhOb2FXNW5kRzl1TVJBd0RnWURWUVFIRXdkU1pXUnRiMjVrTVI0d0hBWURWUVFLRXhWTg0KYVdOeWIzTnZablFnUTI5eWNHOXlZWFJwYjI0eEt6QXBCZ05WQkFNVElrMXBZM0p2YzI5bWRDQkRaWEowYVdacFkyRjBaU0JNYVhOMA0KSUVOQklESXdNVEV3SGhjTk1qUXdPVEV5TWpBeE56UXlXaGNOTWpVd09URXhNakF4TnpReVdqQ0JpVEVMTUFrR0ExVUVCaE1DVlZNeA0KRXpBUkJnTlZCQWdUQ2xkaGMyaHBibWQwYjI0eEVEQU9CZ05WQkFjVEIxSmxaRzF2Ym1ReEhqQWNCZ05WQkFvVEZVMXBZM0p2YzI5bQ0KZENCRGIzSndiM0poZEdsdmJqRXpNREVHQTFVRUF4TXFUV2xqY205emIyWjBJRU5sY25ScFptbGpZWFJsSUZSeWRYTjBJRXhwYzNRZw0KVUhWaWJHbHphR1Z5TUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUF0OXNBOGd6Z0diYnE2bXZYSzB0Mg0KVmJzbnczWUYzMTlGK1ZMTlhlUW9MM3RrZTgzNkMyTXhuc2pEalFicjl1UVUrUDFNU29MM3JkNHVESCtqT1EzbStNZ0xMQTFvYVM5VA0KZ3lUM05pbmxseFF0RXFqRFpIY28wQy9jR05pUW5UbFR4WGl1dnplaHZiUDFmWTlMVEpOcWNrSG95T0k5dDVWVjdTektSQlBnRk1iSA0KSlovaG9LUFpHVDZCcm5DY2gxeERMbStaU0RUa3lWS0hpbVlyWVVad2ZXTStmbTlORjdGL28ycUlmL0JUNndUaGxsUEdDMmZHSG5zWg0KVStRcXdPS1cwc0tmZmNrUXlUMFFuVGQzYnZRMUFTbktJN09SWWNpZDQyZUZua2pOOHVHcytLUmNtT0d6dlpqZ2tFcGNDODBxQXBrRA0KVTEvdUpIdU8xSVliS0JIQkR3SURBUUFCbzRJQmFqQ0NBV1l3RlFZRFZSMGxCQTR3REFZS0t3WUJCQUdDTndvRENUQWRCZ05WSFE0RQ0KRmdRVUxCb2NRU0JiUE1ETjdBVzN4TzVielRIMDZ2SXdSUVlEVlIwUkJENHdQS1E2TURneEhqQWNCZ05WQkFzVEZVMXBZM0p2YzI5bQ0KZENCRGIzSndiM0poZEdsdmJqRVdNQlFHQTFVRUJSTU5Nakk1T0RnM0t6VXdNamsyT0RBZkJnTlZIU01FR0RBV2dCUkI4Q0hIN2NTSA0KK29OMS93b00zQzNzcUdxcldUQlpCZ05WSFI4RVVqQlFNRTZnVEtCS2hraG9kSFJ3T2k4dlkzSnNMbTFwWTNKdmMyOW1kQzVqYjIwdg0KY0d0cEwyTnliQzl3Y205a2RXTjBjeTlOYVdORFpYSk1hWE5EUVRJd01URmZNakF4TVMwd015MHlPUzVqY213d1hRWUlLd1lCQlFVSA0KQVFFRVVUQlBNRTBHQ0NzR0FRVUZCekFDaGtGb2RIUndPaTh2ZDNkM0xtMXBZM0p2YzI5bWRDNWpiMjB2Y0d0cEwyTmxjblJ6TDAxcA0KWTBObGNreHBjME5CTWpBeE1WOHlNREV4TFRBekxUSTVMbU55ZERBTUJnTlZIUk1CQWY4RUFqQUFNQTBHQ1NxR1NJYjNEUUVCQ3dVQQ0KQTRJQ0FRQnZiWldFeEJUdkIyblpkRkppRy9FSEhITFZzb3pjYmtZNk9COElaaVhwLzdmNTlsYUpnakJBWS8zcWRyOFhZRXEwNEEraA0KYkdpRWQ0ZE5LLys5TW1uU1RmS2R0eVV5TXBRalN0NXRKWU8zajRydDgyZW4zN2FZYzM4Rjd2YUI2bndkdkZ5dnVDZ1dtaENkNGtwcg0KM2E4M3VFTURNS3p6Y3ZVRTBHOTVoelQ5b1FRSkpBUnNXUjRpeEplTGdrN3ErMmRhRFBxL1NDNEQzV3NyRG9LWlg0TWZ0QzVyNnFxbA0KME9GemozOEF2RHV1T1E1eFRjT251dU9ISWxGTVJpVFV2Yk5sN0F4ck9jRVkrKzJJM1hraXNNWituYnUweEIwOTJYMEhnRFlNbDQ1Ng0KL2ZkdUZTRWQ4ekw4ajZBcUYxRm9wd0VLYVBpRERQc0hjWW1kdTA3N3FLalpST29zdjZWeXIxME01UndXN2xFTCtNN1lCQW5EK1RBNw0KNlRURnVvNHp0bVBOQ2MwSHZ5RnhQYmFZVkFaMkdua1kzUzdCVS90ZG1JZzNJY1RGQU1PSUZaTVdWSkZYU3B3RlBUTWljaXlqV3ZPRw0KNUxXeTE2ai9GemtFdEpyZHFlQWFORlcvc0NUUnBaMHExd3Jic1NCK0FxeU9pLzRmdjJyY1d4UmREaVY5NVh0MnlZK2lHd3ZheXJDUg0KelNFUDZsaFdsWW9jNklMUUZOWGRnbHFROUNCcUcxb2g5VkJwK2NrcTBRU2NpYi9QNERNejZpT09TZUZ0R1pGWWxDTlBqOFhNZm1OWA0KUE9UWVBJd3RFS1E1T0JYYWlrYXRIV2VWSHpnYndLNlBvL1ZtSmFyV2NHVU1ZNmtiZ2VEMjBjcWxYdS95cUVWdFFBbERpVUxFM3U1NQ0KZEhMZ0t6Q0NCdzB3Z2dUMW9BTUNBUUlDQ21FUmJKSUFBQUFBQUFjd0RRWUpLb1pJaHZjTkFRRUxCUUF3Z1lneEN6QUpCZ05WQkFZVA0KQWxWVE1STXdFUVlEVlFRSUV3cFhZWE5vYVc1bmRHOXVNUkF3RGdZRFZRUUhFd2RTWldSdGIyNWtNUjR3SEFZRFZRUUtFeFZOYVdOeQ0KYjNOdlpuUWdRMjl5Y0c5eVlYUnBiMjR4TWpBd0JnTlZCQU1US1UxcFkzSnZjMjltZENCU2IyOTBJRU5sY25ScFptbGpZWFJsSUVGMQ0KZEdodmNtbDBlU0F5TURFd01CNFhEVEV4TURNeU9URTROVGd6T1ZvWERUSTJNRE15T1RFNU1EZ3pPVm93Z1lFeEN6QUpCZ05WQkFZVA0KQWxWVE1STXdFUVlEVlFRSUV3cFhZWE5vYVc1bmRHOXVNUkF3RGdZRFZRUUhFd2RTWldSdGIyNWtNUjR3SEFZRFZRUUtFeFZOYVdOeQ0KYjNOdlpuUWdRMjl5Y0c5eVlYUnBiMjR4S3pBcEJnTlZCQU1USWsxcFkzSnZjMjltZENCRFpYSjBhV1pwWTJGMFpTQk1hWE4wSUVOQg0KSURJd01URXdnZ0lpTUEwR0NTcUdTSWIzRFFFQkFRVUFBNElDRHdBd2dnSUtBb0lDQVFDNGhIcUEvVTBucDlMdmdqYnhVd1dkdmtKdA0KampFSVI4dk5zNE1LUzA0ekdpNTgzWEtCamM2US9Ed0Z5eTgwaFRpUFBCeG9iV0JSVWsyczJtMHJmck56UjZ2UzNKVnhTakdCV3FFZg0KcTRJbVJTMk02SVI0dlNEd0RjYjFyaWFISGxhT1ZUd0lNREtVbENLVEM2V3d4bDNtTFlFNXplbkhydWpZU1hGSnE1RjB1RStOTDBleg0KUDlDVGcxd0NHdDVMdUxJOE4rbVQ2bkpibU1manJCamc1bjVLd1lFcy9TSVVkblBoYU53Z0NjRHpSczBqSnNoRklzckh2SFQ4aWY5WA0KNE0rOWpyQXI3eWJXZDZzYTlHZEI4VjRNY2FvQ2YxN0FncW9KaSt5SmlFSDFBMEpwMlI5RjJWYytCSlpLMVRLMzBXRW1hTWZCc2FEZw0KZWdWT3RXM0NndUF1dHVka254WjlsU3FHTXRBaHlGMzR5d1V3SHJrQ21HeXprMnZJZzJjaFhkWmxtQ0JrM2N1L1I1di9HUHJ4a042bg0KZU0xN0JJWitKNHEzbFp3bTNiR1cvRS9nUUNDRGFOM3NNL0lxb0FlbjY1SDZyQTlSUVlqeHhZZEJUSWRIWXAxWXdKNS91eEo5M3RPZg0KL2NISEZMMS9tTkJYbStIamJGZmhaVi93M0N1Y29WVENWaW9WWk11cVR1VDl3K2gzaVAvYkRhK1FuOWRvZ1FFdmxPR3Z4dVRHZHR0MQ0KMnQvUUVrenlpVFp2U0lDQldOMFhDU2dyVmF5VEkrV09NV1d0RFk2VDAzR25nUlNZNmF5cUJWanUxMFJETUcwZHg3ckNmL1ZJeE9XZw0KamxXT3RBbkFBY09kSFViMS9rYTFPZ0NJSTdYd3lrSE5PdzNHOXNwQUJPcWI1WWcybndJREFRQUJvNElCZkRDQ0FYZ3dFQVlKS3dZQg0KQkFHQ054VUJCQU1DQVFBd0hRWURWUjBPQkJZRUZFSHdJY2Z0eElmNmczWC9DZ3pjTGV5b2FxdFpNQmtHQ1NzR0FRUUJnamNVQWdRTQ0KSGdvQVV3QjFBR0lBUXdCQk1Bc0dBMVVkRHdRRUF3SUJoakFQQmdOVkhSTUJBZjhFQlRBREFRSC9NQjhHQTFVZEl3UVlNQmFBRk5YMg0KVnN1UDZLSmNZbWpSUFpTUVc5Zk9taGpFTUZZR0ExVWRId1JQTUUwd1M2QkpvRWVHUldoMGRIQTZMeTlqY213dWJXbGpjbTl6YjJaMA0KTG1OdmJTOXdhMmt2WTNKc0wzQnliMlIxWTNSekwwMXBZMUp2YjBObGNrRjFkRjh5TURFd0xUQTJMVEl6TG1OeWJEQmFCZ2dyQmdFRg0KQlFjQkFRUk9NRXd3U2dZSUt3WUJCUVVITUFLR1BtaDBkSEE2THk5M2QzY3ViV2xqY205emIyWjBMbU52YlM5d2Eya3ZZMlZ5ZEhNdg0KVFdsalVtOXZRMlZ5UVhWMFh6SXdNVEF0TURZdE1qTXVZM0owTURjR0ExVWRKUVF3TUM0R0NDc0dBUVVGQndNREJnb3JCZ0VFQVlJMw0KQ2dNQkJnb3JCZ0VFQVlJM0NnTUpCZ29yQmdFRUFZSTNDZ01UTUEwR0NTcUdTSWIzRFFFQkN3VUFBNElDQVFDQzk2bWxzNy9seUZsQg0KSnpRUFlweEI4S3NyZmZtbnFNaW9EMTFEdnEzeW1mai8rL1o1VUVRTVVPcEMyNTBCNmFWSmVTZ3BFejVBTm5RVzI0OGd6STB0VVJEYw0KSzBFMmZMYlFRQk9iSEFBM1RJRnBhTEVhZ3BZN2FYWEg1VFRZUHR4YUNhdlR2Nm12eEFodjQwZkdNdThsNlFzTFZSS1U3NGNVR2RMaA0KSWQ0M3o2Nm1ORjBqS1FRRWJXM25HdDFFTUhsMEdvMk13ZmsrK2Ewd2E2TzBhblJKT1ZzM0xRRVo3Z01wM2E1S0wvbUNyMGdmRkpxYw0KUFNVeFZ1bzZwMDIzL1lzLy9yOTNOb3RWNWJNUVVPNVUxTDlyMkNyeU0zZ3V2ekg1TmhIdk1BdjVURU9ENDF1cFhCMWJwbllGdVBCMQ0KVCttNEh6WkVwbjltMEVzTkdGVWVkQzRuSitVbVFvTnV1NlR2ZS9ua21MM1ZPNG5UV0pLNDBjMFdmemwrWmlVTjI0Tlp2MWNmbTlMcA0KRzNJblhXc3owZjZpa2t4UlBjYk1sRHBXLytzUVFTN2RrbFBORVBFZE51c0VHdHMxMlpHMm1XQVA0QXVzWnd4RUZwd0NSOHUzUnBaSg0KRDk4RHNRK3REaEt0U0FVMjRTMC91MXJnbE5TWGt1ays1dXNsR2NzejhkK1RZSkNtdVE1VzlpanBRc2NlRUZ1bUxnKzUyNmxrMTQ3bA0KTTlLZFE0SkxiamRtdVExblYxUmFTQTdqaXZzZjdRb212QTAwMGdwSGhXRXFJN0hnVklwUUZGYUZ3UDh0OTJtWlJIMGE5RTE4R0E3aA0KQndmdUNXWlNTbm9hWXFUbGk4K0Zvb2FLY1pDeGZkWVIwMUVlMmx6bnpOWVNFSGFvcmsrVHRXVEp2ZTNjK3pHQ0FsY3dnZ0pUQWdFQg0KTUlHWk1JR0JNUXN3Q1FZRFZRUUdFd0pWVXpFVE1CRUdBMVVFQ0JNS1YyRnphR2x1WjNSdmJqRVFNQTRHQTFVRUJ4TUhVbVZrYlc5dQ0KWkRFZU1Cd0dBMVVFQ2hNVlRXbGpjbTl6YjJaMElFTnZjbkJ2Y21GMGFXOXVNU3N3S1FZRFZRUURFeUpOYVdOeWIzTnZablFnUTJWeQ0KZEdsbWFXTmhkR1VnVEdsemRDQkRRU0F5TURFeEFoTXpBQUFBY2pXMm54TVQxQk9MQUFBQUFBQnlNQTBHQ1dDR1NBRmxBd1FDQVFVQQ0Kb0lHUE1CZ0dDU3FHU0liM0RRRUpBekVMQmdrckJnRUVBWUkzQ2dFd0x3WUpLb1pJaHZjTkFRa0VNU0lFSU93YlhLVEpEaG1CZ3hIYw0KZ2Y5MzBZZVZScDQxZTJraUNueWxQVmFZU0l0TE1FSUdDaXNHQVFRQmdqY0NBUXd4TkRBeW9CU0FFZ0JOQUdrQVl3QnlBRzhBY3dCdg0KQUdZQWRLRWFnQmhvZEhSd09pOHZkM2QzTG0xcFkzSnZjMjltZEM1amIyMHdEUVlKS29aSWh2Y05BUUVCQlFBRWdnRUFuQU5SaWpQaA0KWmcxLzRibkdNQ2lpZWdrN3M1NVNZM1lqSjE2NTh6b0JNNTZKZ1JpeS9qU2FMK3BuU2pUbllCRlJlWTgzbnBxQ1lrYlZFU1ljdFFHWg0KOGRoeGxoakRFMGRJWU5sd0FqbnhsbzQ3UTdGV3dJem82YjVqREFmanpBaGFRWDNDTVBDSkN0SHoxNUd2c01tODdHYi9XYnBzVUJ6eA0KVWxiaHNIdTQ4T0tFb1pQZ000MTFPRmRlSHhlc090T2Z1SGM0TjRZYW8zWk9vOFAvb1R1VjNucEVOQ0xqVk9sUXlBKzIxU2VDem9sMA0KQzEyQmt2K3RmaStIanB2M1pFMlZwTU1qTkFDK1NHOVhLMllGN2VFS3IvWXpXSGc3ZCt2NHMyMXVxRFJtRGZJd0o5RnJ6M01jcXRXSw0KYUFXWldXZFUvZ0xQVTV6VXNHSThsZTJsSkdMT2VRPT0&ieol=CRLF) which is useful.

I attempted to find these md5 hashes of revoked certificates, but crt.sh does not hold the md5 hashes.  I also trued the censys.io certificate search function which looks to support md5 but didn't have much luck either.  The stackexchange discussion does highlight that many of these certificates are quite old and predate CT, so perhaps thy don't appear in logs.

This mechanism feels somewhere between the chrome blocklist approach (as it looks to only tackle a small number of highly significant revocations) and the CRLsets approach as the updates are distributed periodically through updates to clients.

## Revocation conclusions

It feels all a bit of a mess.

There was a quote towards the end of the [session at Real Word Crypto by Thyle van der Merwe](https://www.youtube.com/watch?v=kED3K57EV6w) in relation to CRLite that felt appropriate...

**It's an excellent solution to a problem we shouldn't have**

I feel this sentiment applies to many of the mechanisms described in this post (and the previous post) - simpler revocation mechanisms such as `must-staple` exist, and when considering the foundational principles of trust and PKI, it feels like all options are twists and turns on the road to short lived certificates.

Ultimately, revocation is in place to handle a temporal challenge - a CA has issued a thing at a point in time with an expiry date, but in the intervening period something may have gone wrong.  Revocation techniques are in place to allow clients to know about this event, but through a variety of paths these all involve interactions between the CA, the service provider, the client, and intermediaries like browser vendors to ultimately communicate to the client that the CA has decided the certificate is invalid in advance of it's expiry date.  

This will always be a trade-off (as the guaranteed option of asking the CA for every connection clearly undoes the distributed nature of PKI/public key signing  - we may as well use Kerberos if we are pursing a centralised model) however the mechanisms outlined in these posts ultimately just shorten this time window (e.g. through periodic OCSP acquisition, or periodic generation of distributed CRLs in some form to clients).  

If we're going to this effort anyway, shorter lived certificates seem far (far) simpler.







