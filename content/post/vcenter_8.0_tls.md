---
authors:
- cunnie
categories:
- vSphere
date: 2022-11-02T10:16:22Z
draft: false
short: |
    We install a Transport Layer Security (TLS) certificate issued by a
    commercial Certificate Authority (CA) on a VMware VCSA 8.0 while avoiding
    several pitfalls.
title: How to Install a TLS Certificate on vCenter Server Appliance (VCSA) 8.0
---

{{< figure src="https://user-images.githubusercontent.com/1020675/199730887-ac22531d-8741-4120-b6c0-3e793bc00587.png" >}}

## Quickstart

First, create your key and your CSR (Certificate Signing Request). In the
following example, we are creating a CSR for our vCenter host,
"vcenter-80.nono.io":

```bash
CN=vcenter-80.nono.io # "CN" is the abbreviation for "Common Name"
openssl genrsa -out $CN.key 3072
openssl req \
  -new \
  -key $CN.key \
  -out $CN.csr \
  -sha256 \
  -subj "/C=US/ST=California/L=San Francisco/O=nono.io/OU=homelab/CN=${CN}/emailAddress=brian.cunnie@gmail.com" \
  -config <(cat <<EOF
[ req ]
distinguished_name = req_distinguished_name
req_extensions     = req_ext
[ req_distinguished_name ]
[ req_ext ]
subjectAltName = @alt_names
[alt_names]
DNS.1   = ${CN}
EOF
)
```

You'll have two files, `vcenter-80.nono.io.key` and `vcenter-80.nono.io.csr`.

Extra credit: double-check your CSR at
[GoDaddy](https://ssltools.godaddy.com/views/csrDecoder). It's important that
your "Subject Alternative Names" matches your hostname (e.g.
"vcenter-80.nono.io").

Acquire a certificate for your host from a Commercial CA. In our example, we
acquired a certificate for our host `vcenter-80.nono.io` from
[SSls.com](https://ssls.com), and we purchased their least-expensive offering,
the _PositiveSSL 1 domain Comodo SSL_.

_[We do not endorse either SSLs.com or Sectigo (formerly
Comodo); We encourage you to use the reseller and the Certificate Authority
(CA) with which you are most comfortable]_.

SSLs.com sends us two files, `vcenter-80.nono.io.crt` and
`vcenter-80_nono_io.ca-bundle`

Do the following:

- On your vCenter, navigate to **Menu → Administration → Certificates →
  Certificate Management**
- On the **__MACHINE_CERT** tile, click **Actions**, select **Import and
  Replace Certificate**.
- Select **Replace with external CA certificate(requires private key)**.
  - **Machine SSL Certificate**: click **Browse File** and select
    `vcenter-80.nono.io.crt`
  - **Chain of trusted root certificates**: click **Browse File** and select
    `vcenter-80_nono_io.ca-bundle`
  - **Private Key**: click **Browse File** and select
    `vcenter-80.nono.io.key`
- Click **Replace**.

The change should take effect immediately, though it may take a minute for the
website to be ready.

If you get errors such as "Please provide the strong signature algorithm
certificate", then refer to the [Troubleshooting](#troubleshooting) section
below.

### Troubleshooting

> Error occurred while fetching tls: Provided certificate using the weak
> signature algorithm. Please provide the strong signature algorithm
> certificate

I encountered this error, and to fix it I had to edit the CA Bundle
(certificate chain) and replace a cross-signed root certificate with a
self-signed root certificate. Here's the [CA
Bundle](https://raw.githubusercontent.com/cunnie/docs/main/tls/vcenter-80_nono_io.ca-bundle)
that I used and that worked for me.

Technical details: Sectigo included a variant of their root certificate
"[USERTrust RSA Certification Authority](https://crt.sh/?id=1282303295)" that
had been cross-signed (issued) by the old "[AAA Certificate
Services](https://crt.sh/?id=331986)" which had been self-signed with the weak
SHA-1 algorithm, which Google and others have been
[deprecating](https://security.googleblog.com/2014/09/gradually-sunsetting-sha-1.html)
since 2016.

Sectigo has another variant of that same root certificate "[USERTrust RSA
Certification Authority](https://crt.sh/?id=1199354)", which is self-signed
with the strong SHA-384 algorithm. This is the variant you should use.

> Error occurred while fetching tls: The provided MACHINE_SSL certificate and
> provided private key are not valid.

The private key doesn't match the certificate. Here are two ways it happened to
me:

1. I had vCenter generate the CSR on my behalf, and Sectigo took so long to
   issue the certificate that I thought something went wrong and I had vCenter
   generate a new CSR, not realizing that would trigger a new key, but by then
   Sectigo came through with the certificate for the _old_ CSR, and I didn't
   realize that the certificate didn't match the key.
2. I reissued the certificate, but the .zip file from Sectigo had the old
   vcenter-80.nono.io certificate in it. The fix: I cut-and-paste the one from
   the email content instead (I didn't use the wrong certificate from the .zip
   file).

### How to Determine if a Certificate is a Self-Signed Certificate

To determine if a certificate is a self-signed certificate, confirm the subject
is the same as the issuer. In the following example, we use two common TLS
command line tools (`cfssl` and `openssl`).

First we use `cfssl`, whose output is JSON, which we pipe to `jq` to extract
the Common Name of the issuer and subject and the signature algorithm. We
choose the problematic certificate ("USERTrust RSA Certification Authority")
that has been signed by a root cert that in turn is self-signed with the weak
SHA-1 algorithm ("AAA Certificate Services"):

```bash
curl https://crt.sh/?d=1282303295 | \
  cfssl certinfo -cert - | \
  jq -r '{"Issuer": .issuer.common_name, "Subject": .subject.common_name, "Signature Algorithm": .sigalg }'
```
produces:
```json
{
  "Issuer": "AAA Certificate Services",
  "Subject": "USERTrust RSA Certification Authority",
  "Signature Algorithm": "SHA384WithRSA"
}
```

Now let's look at its issuer's ("AAA Certificate Services") certificate, which is a root
certificate, and which is self-signed with the weak SHA-1 algorithm:

```bash
curl https://crt.sh/?d=331986 | \
  cfssl certinfo -cert - | \
  jq -r '{"Issuer": .issuer.common_name, "Subject": .subject.common_name, "Signature Algorithm": .sigalg }'
```
produces:
```json
{
  "Issuer": "AAA Certificate Services",
  "Subject": "AAA Certificate Services",
  "Signature Algorithm": "SHA1WithRSA"
}
```

Aha! That's our smoking gun: "SHA1WithRSA" is the culprit, the reason we've
been getting the "Provided certificate using the weak signature algorithm"
error.

Let's explore the self-signed variant (the root certificate variant) of the
"USERTrust RSA Certification Authority", this time with `openssl`. We can use a
simple `egrep` to extract the information we need:

```bash
curl https://crt.sh/?d=1199354 | \
  openssl x509 -noout -text | \
  egrep "Issuer:|Subject:|Signature Algorithm:"
```
produces:
```
Signature Algorithm: sha384WithRSAEncryption
Issuer: C=US, ST=New Jersey, L=Jersey City, O=The USERTRUST Network, CN=USERTrust RSA Certification Authority
Subject: C=US, ST=New Jersey, L=Jersey City, O=The USERTRUST Network, CN=USERTrust RSA Certification Authority
Signature Algorithm: sha384WithRSAEncryption
```

_(We don't know why "Signature Algorithm" is repeated)._

We can see that this is a self-signed root certificate with a strong, SHA-384
signature algorithm. This is the root certificate we should use in our
certificate chain (CA bundle).

### References

- Self-signed (not cross-signed) USERTrust RSA Certification Authority
  certificate <https://crt.sh/?id=1199354>, [.pem](https://crt.sh/?d=1199354)
- Sectigo root certificates:
  <https://sectigo.com/knowledge-base/detail/Sectigo-Root-Certificates/kA03l000000c4KV>
- Information on Sectigo's certificate chain hierarchy:
  <https://support.sectigo.com/articles/Knowledge/Sectigo-Chain-Hierarchy-and-Intermediate-Roots>
- Location of log file for cert-related errors on vCenter:
  `/var/log/vmware/certificatemanagement/certificatemanagement-svcs.log`
- To reset all certificates on VCSA 8:
  <https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-authentication/GUID-5572C39C-1556-4ACC-B12D-26E3BCBC4D56.html>
- VMware discussion of the dreaded "Please provide the strong signature
  algorithm certificate" error:
  <https://communities.vmware.com/t5/vCenter-Server-Discussions/SSL-Certificate-Error-vCenter-8/td-p/2933907>
- Getting rid of unneeded root certificates:
  <https://communities.vmware.com/t5/VMware-vCenter-Discussions/Cleanup-old-trusted-root-certificates-from-PSC/m-p/472299/highlight/true#M5344>
- Mozilla's list of approximately 150 root certificates:
  <https://wiki.mozilla.org/CA/Included_Certificates>
- How to Install a TLS Certificate on vCenter Server Appliance (VCSA) 7:
  <https://vnote42.net/2020/04/15/replace-machine-certificate-in-vsphere-7/>
- Our assets:
  - CSR
    ([vcenter-80.nono.io.csr](https://raw.githubusercontent.com/cunnie/docs/main/tls/vcenter-80.nono.io.csr))
  - Certificate
    ([vcenter-80.nono.io.crt](https://raw.githubusercontent.com/cunnie/docs/main/tls/vcenter-80.nono.io.crt))
  - CA Bundle/Certificate Chain
    ([vcenter-80_nono_io.ca-bundle](https://raw.githubusercontent.com/cunnie/docs/main/tls/vcenter-80_nono_io.ca-bundle))

### Corrections & Updates

*2022-11-05*

Expanded the "How to Determine if a Certificate is a Self-Signed Certificate"
section to include the signature algorithm, and included the problematic root
cert to drive home the cause of the error.

*2023-01-05*

We mistakenly told users to upload the CA bundle when they should have uploaded
the key. Thanks @obsidianindy.
