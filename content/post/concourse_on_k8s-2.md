---
title: "Concourse CI on Kubernetes (GKE), Part 2: Ingress"
date: 2021-08-07T14:49:17-07:00
draft: false
images:
- https://www.nginx.com/wp-content/uploads/2018/08/NGINX-logo-rgb-large.png
---

{{< figure src="https://www.nginx.com/wp-content/uploads/2018/08/NGINX-logo-rgb-large.png" alt="nginx logo" >}}

In our previous blog post, we set up our Kubernetes cluster and deployed a pod
running nginx, but the experience was disappointing—we couldn't browse to our
pod. Let's fix that by deploying the nginx Ingress controller.

### Acquire the External IP Address (Elastic IP)

We'll use the [Google Cloud console](https://console.cloud.google.com) to
acquire the external address
<sup>[[external address](#external_address)]</sup>
for our load balancer.

Navigate to _VPC network → External IP addresses → Reserve Static Address:_

- Name: gke-nono-io _(or "gke-" and whatever your domain is, with dashes not dots)_
- Description: Ingress for GKE


{{< figure src="https://user-images.githubusercontent.com/1020675/128951949-aac47759-8ac8-426b-b256-90c237a3f5b5.png" alt="acquire external IP" >}}

In our example, we acquire the IP address, `34.135.26.144`.

### Create DNS Record to Point to Acquired IP Address

You'll need a DNS domain for this part. In our examples, we use the domain
"nono.io", so whenever you see "nono.io", substitute & replace your domain.
Similarly, whenever you see "34.135.26.144", substitute your external IP
address.

Adding a DNS record is outside the scope of the humble blog post (we use BIND,
but these days services such as AWS's Route 53 are all the rage).

We create the DNS address record "gke.nono.io" to point to "34.135.26.144";
Let's test to make sure it's set up properly:

```bash
dig gke.nono.io +short # should return 34.135.26.144
```

### Create Kubernetes Ingress nginx Manifest Files

We're going to shamelessly copy the canonical Ingress nginx manifest files and
modify them to include our static IP address:

```bash
curl -L https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.48.1/deploy/static/provider/cloud/deploy.yaml \
  -o nginx-ingress-controller.yml
nvim nginx-ingress-controller.yml
```

We need to add our IP address to our load balancer Kubernetes service. search
for the string "LoadBalancer" and add the IP address as shown below (don't
include the plus sign "+" in your file):

```diff
 spec:
   type: LoadBalancer
   externalTrafficPolicy: Local
+  loadBalancerIP: 34.135.26.144
   ports:
     - name: http
       port: 80
```

Let's apply our changes:

```bash
kubectl apply -f nginx-ingress-controller.yml
```

Let's wait for the change to have completed

```bash
 kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

Let's browse to our endpoint: <http://gke.nono.io>.  We see the
nginx "404 Not Found" status page, but that's reassuring: it means we've
properly set up the nginx controller, but haven't yet set up Ingress to our
existing pods.

Before we set up Ingress, let's check our HTTPS endpoint: <https://gke.nono.io>.

Wait, what is this? We're seeing an unsettling message, "Warning: Potential
Security Risk Ahead" (Chrome users may see "Your connection is not private";
Safari users, "This Connection Is Not Private").  We're upset—we
don't want to be seen as losers who are using self-signed TLS certificates; we
want to be winners who are using certificates from Let's Encrypt.

#### Stay Tuned!

Stay tuned for our next installment, where we configure Let's Encrypt
certificates for our TLS (Transport Layer Security) endpoints.

---

### References

- The canonical Kubernetes documentation for deploying the nginx Ingress
  controller on GKE <https://kubernetes.github.io/ingress-nginx/deploy/#gce-gke>

### Footnotes

**<a id="external_address">external address</a>**

You can also acquire the external address via the command line (don't forget to
change "blabbertabber" to your project's name):

```bash
gcloud compute addresses create gke-nono-io --project=blabbertabber --description=Ingress\ for\ GKE --region=us-central1
```

Or, for the truly advanced among you, you can modify your terraform templates to
acquire the address for you. The terraform site has [great
documentation](https://registry.terraform.io/modules/terraform-google-modules/address/google/latest#external-ip-address),
and here's the snippet you'll need:

```terraform
module "address-fe" {
  source  = "terraform-google-modules/address/google"
  version = "0.1.0"

  names  = [ "gke-nono-io"]
  global = true
}
```
