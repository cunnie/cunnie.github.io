---
title: "Concourse CI on Kubernetes (GKE), Part 3: TLS"
date: 2021-08-11T16:55:31-07:00
draft: true
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
I merely condensed it & made it more opinionated.

### Installation

Let's [add the Jetstack Helm Repository](
https://github.com/jetstack/cert-manager/releases/latest/download/cert-manager.yaml):

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
  --version v1.5.0 \
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

Let's download and edit the Ingress (I've already configured `gke.nono.io` to
point to the GCP/GKE load balancer at 104.155.144.4):
```bash
curl -o ingress-kuard.yml -L https://netlify.cert-manager.io/docs/tutorials/acme/example/ingress.yaml
sed -i '' "s/example.example.com/gke.nono.io/g" ingress-kuard.yml
kubectl apply -f ingress-kuard.yml
```

Let's use curl to check (note the cert is still self-signed at this point):
```bash
curl -kivL -H 'Host: gke.nono.io' 'http://104.155.144.4'
```

#### 6. [Configure Let’s Encrypt Issuer](https://cert-manager.io/docs/tutorials/acme/ingress/#step-6-configure-let-s-encrypt-issuer)

Let's deploy the staging & production issuers:
```bash
kubectl apply -f <(
  curl -o- https://cert-manager.io/docs/tutorials/acme/example/staging-issuer.yaml |
  sed 's/user@example.com/brian.cunnie@gmail.com/')
kubectl apply -f <(
  curl -o- https://cert-manager.io/docs/tutorials/acme/example/production-issuer.yaml |
  sed 's/user@example.com/brian.cunnie@gmail.com/')
kubectl describe issuer letsencrypt-staging
kubectl describe issuer letsencrypt-prod
```

#### 7. [Step 7 - Deploy a TLS Ingress Resource](https://cert-manager.io/docs/tutorials/acme/ingress/#step-7-deploy-a-tls-ingress-resource)

Let's deploy the ingress resource using annotations to obtain the certificate:
```bash
kubectl apply -f <(
  curl -o- https://cert-manager.io/docs/tutorials/acme/example/ingress-tls.yaml |
  sed 's/example.example.com/gke.nono.io/')
kubectl get certificate # takes ~30s to become ready ("READY" == "True")
kubectl describe certificate quickstart-example-tls
kubectl describe secret quickstart-example-tls
```

Browse to <https://gke.nono.io> and notice that although the cert is still
invalid it's no longer self-signed; instead, it's issued by the Let's Encrypt
staging CA.

Let's do the production certificate:

```bash
kubectl apply -f <(
  curl -o- https://cert-manager.io/docs/tutorials/acme/example/ingress-tls-final.yaml |
  sed 's/example.example.com/gke.nono.io/')
kubectl delete secret quickstart-example-tls # triggers the process to get a new certificate
kubectl get certificate # takes ~30s to become ready ("READY" == "True")
kubectl describe certificate quickstart-example-tls
kubectl describe secret quickstart-example-tls
```

And now browse: <https://gke.nono.io/>

---

### References

- cert-manager documentation: <https://cert-manager.io/docs/>

### Footnotes

**<a id="external_address">external address</a>**

You can also acquire the external address via the command line (don't forget to
change "blabbertabber" to your project's name):

