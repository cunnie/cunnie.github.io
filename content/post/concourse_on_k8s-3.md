---
title: "Concourse CI on Kubernetes (GKE), Part 3: TLS"
date: 2021-08-11T16:55:31-07:00
draft: false
images:
- https://letsencrypt.org/images/letsencrypt-logo-horizontal.svg
---

{{< figure src="https://letsencrypt.org/images/letsencrypt-logo-horizontal.svg" alt="Let's Encrypt logo" >}}

In our previous blog post, we configured ingress to our Kubernetes cluster but
were disappointed to discover that the TLS certificates were self-signed. In
this post we'll remedy that by installing cert-manager, the Cloud native
certificate management tool.

Disclaimer: most of this blog post was lifted whole cloth from the
most-excellent [cert-manager documentation](https://cert-manager.io/docs/).
We merely condensed it & made it more opinionated.

### Installation

Let's add the Jetstack Helm Repository:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

Let's [install cert-manager](https://cert-manager.io/docs/installation/helm/#4-install-cert-manager):

```bash
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.6.0 \
  --set installCRDs=true
```

### [Verifying Installation](https://cert-manager.io/docs/installation/verify/#manual-verification)

Do we see all three pods?

```bash
kubectl get pods --namespace cert-manager
```

Now let's create an issuer to test the webhook:
```bash
cat <<EOF > test-resources.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
    - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
EOF
```

And now let's apply those resources:

```bash
kubectl apply -f test-resources.yaml
sleep 10
kubectl describe certificate -n cert-manager-test | grep "has been successfully"
kubectl delete -f test-resources.yaml
```

#### 4. [Deploy an Example Service](https://cert-manager.io/docs/tutorials/acme/ingress/#step-4-deploy-an-example-service)

Let's install the sample services to test the controller:
```bash
kubectl apply -f https://netlify.cert-manager.io/docs/tutorials/acme/example/deployment.yaml
kubectl apply -f https://netlify.cert-manager.io/docs/tutorials/acme/example/service.yaml
```

Let's download and edit the Ingress (we've already configured `gke.nono.io` to
point to the GCP/GKE load balancer at 34.135.26.144). **Replace `gke.nono.io`
with the DNS record of your load balancer** set up in the previous blog post:

```bash
curl -o ingress-kuard.yml -L https://netlify.cert-manager.io/docs/tutorials/acme/example/ingress.yaml
sed -i '' "s/example.example.com/gke.nono.io/g" ingress-kuard.yml
kubectl apply -f ingress-kuard.yml
```

Let's use curl to check the
GKE load balancer. **Replace `gke.nono.io` with the DNS record
of your load balancer** set up in the previous blog post:

```bash
curl -kivL -H 'Host: gke.nono.io' 'http://gke.nono.io'
```

You should see output similar to the following (note that the cert is still
self-signed):

```
...
* Server certificate:
*  subject: O=Acme Co; CN=Kubernetes Ingress Controller Fake Certificate
```

#### 6. [Configure Let’s Encrypt Issuer](https://cert-manager.io/docs/tutorials/acme/ingress/#step-6-configure-let-s-encrypt-issuer)

Let's deploy the staging & production issuers.  **Replace
`brian.cunnie@gmail.com` with your email address**:

```bash
kubectl apply -f <(
  curl -o- https://cert-manager.io/docs/tutorials/acme/example/staging-issuer.yaml |
  sed 's/user@example.com/brian.cunnie@gmail.com/')
kubectl apply -f <(
  curl -o- https://cert-manager.io/docs/tutorials/acme/example/production-issuer.yaml |
  sed 's/user@example.com/brian.cunnie@gmail.com/')
 # check to make sure they were deployed:
kubectl describe issuer letsencrypt-staging
kubectl describe issuer letsencrypt-prod
```

#### 7. [Step 7 - Deploy a TLS Ingress Resource](https://cert-manager.io/docs/tutorials/acme/ingress/#step-7-deploy-a-tls-ingress-resource)

Let's deploy the ingress resource using annotations to obtain the certificate.
**Replace `gke.nono.io` with the DNS record of your load balancer** set up in
the previous blog post:

```bash
kubectl apply -f <(
  curl -o- https://cert-manager.io/docs/tutorials/acme/example/ingress-tls.yaml |
  sed 's/example.example.com/gke.nono.io/')
kubectl get certificate # takes ~30s to become ready ("READY" == "True")
kubectl describe certificate quickstart-example-tls
kubectl describe secret quickstart-example-tls
```

Let's use curl again to check the GKE load balancer's certificate. **Replace
`gke.nono.io` with the DNS record of your load balancer** set up in the previous
blog post:

```bash
curl -kivL -H 'Host: gke.nono.io' 'http://gke.nono.io'
```

You should see output similar to the following:

```
...
* Server certificate:
*  subject: CN=gke.nono.io
*  start date: Sep  1 22:02:55 2021 GMT
*  expire date: Nov 30 22:02:54 2021 GMT
*  issuer: C=US; O=(STAGING) Let's Encrypt; CN=(STAGING) Artificial Apricot R3
```

Great! We have the staging cert, but that's not quite good enough—we want a real
certificate. Let's upgrade to the production certificate. As usual, **Replace
`gke.nono.io` with the DNS record of your load balancer** set up in the previous
blog post:

```bash
kubectl apply -f <(
  curl -o- https://cert-manager.io/docs/tutorials/acme/example/ingress-tls-final.yaml |
  sed 's/example.example.com/gke.nono.io/')
kubectl delete secret quickstart-example-tls # triggers the process to get a new certificate
kubectl get certificate # takes ~30s to become ready ("READY" == "True")
kubectl describe certificate quickstart-example-tls
kubectl describe secret quickstart-example-tls
```

Let's use curl one more time to check the GKE load balancer's certificate.
**Replace `gke.nono.io` with the DNS record of your load balancer** set up in
the previous blog post:

```bash
curl -kivL -H 'Host: gke.nono.io' 'http://gke.nono.io'
```

You should see output similar to the following:

```
...
* Server certificate:
*  subject: CN=gke.nono.io
*  start date: Sep  1 22:11:36 2021 GMT
*  expire date: Nov 30 22:11:35 2021 GMT
*  issuer: C=US; O=Let's Encrypt; CN=R3
*  SSL certificate verify ok.
```

And now browse (replacing `gke.nono.io` with the DNS record of your load
balancer): <https://gke.nono.io/>. Yes, we get an HTTP 503 status, but our
certificate is valid!

#### Stay Tuned!

Stay tuned for our next installment, where we install Concourse CI on GKE.

---

### References

- cert-manager documentation: <https://cert-manager.io/docs/>

### Updates/Errata

**2021-11-13** Bumped the _cert-manager_ version 1.5.0 → 1.6.0
