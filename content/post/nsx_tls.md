---
title: "How to Install a Commercial TLS Certificate on NSX Manager"
date: 2023-07-02T08:05:37-08:00
short: |
    How to install a Transport Layer Security (TLS) certificate issued by a
    commercial Certificate Authority (CA) on a VMware NSX 4.1
title: How to Install a TLS Certificate on NSX 4.1
draft: false
---

If you don't like seeing the "Your connection is not private" or "Warning: Potential Security Risk Ahead" when you browse to your NSX Manager, then you may want to install a TLS certificate from a commercial CA (Certificate Authority). This post tells you how.

{{< figure src="https://github.com/cunnie/cunnie.github.io/assets/1020675/971b47cc-dc82-4c26-9adc-6b3fa20883c3" alt="NSX Manager certificate hierarchy" caption="This NSX manager has a certificate from issued from Sectigo. Note that the padlock in the address bar shows no warning and the certificate's chain-of-trust can be examined.">}}


### 0. Create Key and CSR

Set the environment variable `CN` to the fully-qualified domain name (FQDN) of your NSX Manager, in our case, "nsx.nono.io":

```bash
# change "nsx.nono.io" to be your NSX Manager's fully-qualified domain name:
export CN=nsx.nono.io # "CN" is the abbreviation for "Common Name"
```

We'll also need to set the environment variable `USER_PASS` to the admin user & password of the NSX Manager, separated by a colon:

```bash
export USER_PASS='admin:NonoIo!23NonoIo'
```

We'll use the environment variables `CN` and `USER_PASS` in subsequent commands.

Create your key and your CSR (Certificate Signing Request). Don't be tempted to use elliptic-curve cryptography even though that's what all the cool kids are doing these days. Instead, stick with the tried-and-true RSA.

<!-- 
openssl genrsa -out ${CN}.key 2048

openssl ecparam -name P-256 -genkey -out ${CN}.key
  - If you're using EC, you need to modify your public key to **remove the EC
    Parameters section** to avoid the error, "Invalid PEM data received for
    private key. (Error code: 2004)" when you try to save. Here's the section I
    removed from my private key:

(
  echo  "{\"pem_encoded\": \"$(cat ${CN//./_}.crt ${CN//./_}.ca-bundle \
    | awk '{printf "%s\\\\n", $0}')\","; \
  echo "\"private_key\": \"$(awk '{printf "%s\\\\n", $0}' < nsx.nono.io.key)\"}"
) \
  | jq . \
  > ${CN}.json

```
-----BEGIN EC PARAMETERS-----
BggqhkjOPQMBBw==
-----END EC PARAMETERS-----
```

- System → Certificates → Trusted CA Bundles → Import CA Bundle
  - make sure your CA Bundle ends in `.crt`. I had to `mv nsx.nono.io.ca-bundle
    nsx.nono.io.ca-bundle.crt`

### 2. Trim Your Key

You'll want to remove your "EC PARAMETERS" from your key and save the new file (in our case, we saved it as `nsx.nono.io-trimmed.key`):

```diff
- -----BEGIN EC PARAMETERS-----
- BggqhkjOPQMBBw==
- -----END EC PARAMETERS-----
  -----BEGIN EC PRIVATE KEY-----
  MHcCAQEEIP4JAZF9uCrtwSaiY6NYTA1mKwXzHYb5grw42+5YC7/EoAoGCCqGSM49
 ```

Here's a quick way using `awk`:

```bash
awk '/BEGIN EC PRIVATE KEY/,/END EC PRIVATE KEY/ {print}' \
  < ${CN}.key > ${CN}-trimmed.key
```

If you don't trim your key, you'll receive an "Error: Invalid PEM data received for private key. (Error code: 2004)" when you import your key into NSX.

```
curl -s -k --user "${USER_PASS}" "https://${CN}/policy/api/v1/infra/certificates/72483a4e-e15a-425e-be40-3c96c0b84a5d?details=true" \
  | jq -r '.details[].is_valid'
```
You may see the word "true" appear twice: once for the certificate, once for the issuer.

-->

```bash
openssl genrsa -out ${CN}.key 2048
openssl req \
  -new \
  -key ${CN}.key \
  -out ${CN}.csr \
  -sha256 \
  -nodes \
  -subj "/C=US/ST=California/L=San Francisco/O=nono.io/CN=${CN}/emailAddress=brian.cunnie@gmail.com" \
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

You'll have two files, `nsx.nono.io.key` and `nsx.nono.io.csr` (substitute your NSX Manager's FQDN as appropriate).

Extra credit: double-check your CSR at [SSLShopper](https://www.sslshopper.com/csr-decoder.html). It's important that your "Subject Alternative Names" matches your hostname (e.g. "nsx.nono.io").

### 1. Acquire Your Certificate

Acquire a certificate for your host from a Commercial CA. In our example, we acquired a certificate for our host `nsx.nono.io` from [SSls.com](https://ssls.com), and we purchased their least-expensive offering, the _PositiveSSL 1 domain Comodo SSL_.

_[We do not endorse either SSLs.com or Sectigo (formerly Comodo); We encourage you to use the reseller and the Certificate Authority (CA) with which you are most comfortable. We have no financial interest in either SSLs.com or Sectigo]._

SSLs.com sends us two files, `nsx_nono_io.crt` and `nsx_nono_io.ca-bundle` (again, substitute your NSX Manager's FQDN as appropriate).

### 2. Create Your Certificate + CA Bundle

You'll need to catenate your certificate onto its CA bundle (certificate chain). We did the following:

```bash
cat ${CN//./_}.crt \
  ${CN//./_}.ca-bundle \
  > ${CN}.chain.crt
```

### 3 Sectigo / Comodo Users: Fix Your Root Certificate

If you bought your certificate from Sectigo, your CA Bundle probably has a "bad" root certificate. "Bad" means a root certificate that has been cross-signed with another root certificate that was self-signed with the weak SHA-1 algorithm. NSX Manager will prevent you from setting this certificate. Here's how to fix.

- Edit your `.chain.crt` file (e.g. `nsx.nono.io.chain.crt`)
- Extract the bottom-most certificate to `/tmp/bad-root.pem`
- Check if it's a cross-signed root certificate by seeing if the issuer is different than the subject:
```bash
openssl x509 -text -noout < /tmp/bad-root.pem | egrep -i "issuer|subject"
```
```
Issuer: C = GB, ST = Greater Manchester, L = Salford, O = Comodo CA Limited, CN = AAA Certificate Services
Subject: C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
```
- We confirm the CN of the _Subject_, "USERTrust RSA Certification Authority", is different than the CN of the _Issuer_, "AAA Certificate Services"
- We take the CN of the _Subject_, in the above example "USERTrust RSA Certification Authority", and search for it on https://crt.sh
- We see several entries, but the only one we're interested in is the one whose _Issuer Name_ contains "USERTrust RSA Certification Authority", we click on its link: <https://crt.sh/?id=1199354>.
- Click on "Download Certificate: PEM", save it to `/tmp/good-root.pem`
- Edit your `.chain.crt` file, replacing the bad root certificate with the good root certificate.

### 4. Import Your Certificate + Bundle and Key

- System → Certificates → Certificates → Import → Certificate
  - Name: Use your NSX managers FQDN, e.g. *nsx.nono.io*
  - Service Certificate: **No**
  - Certificate Contents: upload your combined certificate + bundle, e.g. *nsx.nono.io.chain.crt*
  - Private key: upload your  private key, e.g. *nsx.nono.io.key*

{{< figure src="https://github.com/cunnie/cunnie.github.io/assets/1020675/dda3acd2-0aeb-4ebe-951c-fc69a69be414" alt="NSX Manager import certificate" caption="NSX manager Import Certificate web page">}}

If you get an error, "Error: Certificate chain validation failed. Make sure a valid chain is provided in order leaf,intermediate,root certificate. (Error code: 2076)", it probably means you didn't fix your root certificate.

If you get an error "Error: Invalid PEM data received for private key. (Error code: 2004)", it probably means you tried to use ECC instead of RSA even though I warned you not to.

We need to find the id that NSX Manager assigned the certificate. Run the following command:

```bash
curl -s -k --user "$USER_PASS" "https://${CN}/api/v1/trust-management/certificates" | \
  jq -r ".results[] | select(.display_name == \"${CN}\") | .id"
```

It returns our certificate's id (your id will be different):

```text
72483a4e-e15a-425e-be40-3c96c0b84a5d
```

If it doesn't return _any_ certificate ID, then you probably forget to set "Service Certificate" to "No" when you imported it. To fix, delete your certificate via the web interface & re-import.

Validate the certificate (make sure the "status" is "OK""). Be sure to replace our certificate id with yours:

```bash
curl -s -k --user "$USER_PASS" "https://${CN}/api/v1/trust-management/certificates/72483a4e-e15a-425e-be40-3c96c0b84a5d?action=validate"
```
The above `curl` should return the following:

```json
{
  "status" : "OK"
}
```

However, if the `curl` command returns the following error message, it means you insisted on using an ECC key in spite of my warning, but you thought you were smarter and trimmed the "-----BEGIN EC PARAMETERS-----" from your key to get past the earlier error, and now finally you see the error of your ways and are ready to throw in the towel and use an RSA key:

```json
{
  "status" : "REJECTED",
  "error_message" : "Certificate was rejected: KeyUsage does not allow key encipherment"
}
```

Tech note: ECC keys use key agreement instead of key encipherment, which is a "[more secure way to protect session keys and include features such as forward secrecy](https://stackoverflow.com/a/66963485/2510873)", but NSX insists on key encipherment, and, by corollary, insists on RSA.

Find the node id by querying the API. Look for the `node_uuid`.

```
curl -s -k --user "$USER_PASS" "https://${CN}/api/v1/node"
```

Here's our output. Our `node_uuid` is `ab413d42-5050-2404-b8b8-bce8b244d217`.

```json
{
  "_schema": "NodeProperties",
  "_self": {
    "href": "/node",
    "rel": "self"
  },
  "cli_coredump_config": {
    "global_file_limit": 2,
    "global_frequency_threshold": 600,
    "process_config": []
  },
  "cli_history_size": 100,
  "cli_output_central": false,
  "cli_output_datetime": true,
  "cli_timeout": 600,
  "export_type": "UNRESTRICTED",
  "fully_qualified_domain_name": "nsx.nono.io",
  "hostname": "nsx",
  "kernel_version": "5.15.92-nn2-server",
  "motd": "",
  "node_type": "NSX Manager",
  "node_uuid": "ab413d42-5050-2404-b8b8-bce8b244d217",
  "node_version": "4.1.0.2.0.21761695",
  "product_version": "4.1.0.2.0.21761691",
  "system_datetime": "2023-07-02T16:16:01Z",
  "system_time": 1688314561103,
  "timezone": "Etc/UTC"
}
```

Now we're ready to set our certificate. In the following command, replace our certificate id ("724...") with yours, our `node_uuid` ("ab413...") with yours:

```bash
curl -X POST -s -k --user "${USER_PASS}" \
  -X POST \
  "https://${CN}/api/v1/trust-management/certificates/72483a4e-e15a-425e-be40-3c96c0b84a5d?action=apply_certificate&service_type=API&node_id=ab413d42-5050-2404-b8b8-bce8b244d217"
```

There's no output from this `curl`.

If you refresh your NSX manager's webpage, you should see the unadorned padlock. Congratulations, you have set your certificate.

### References

- [NSX API Reference](https://developer.vmware.com/apis/1583/nsx-t)
- [NSX-T Data Center Administration Guide: Importing and Replacing Certificates](https://docs.vmware.com/en/VMware-NSX-T-Data-Center/3.2/administration/GUID-904871E8-8FC1-42EB-94F0-59DB12EC583F.html)
