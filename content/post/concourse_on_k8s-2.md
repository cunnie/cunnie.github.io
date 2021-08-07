---
title: "Concourse CI on Kubernetes (GKE), Part 2: Ingress"
date: 2021-08-07T14:49:17-07:00
draft: true
images:
- https://www.nginx.com/wp-content/uploads/2018/08/NGINX-logo-rgb-large.png
---

{{< figure src="https://www.nginx.com/wp-content/uploads/2018/08/NGINX-logo-rgb-large.png" alt="nginx logo" >}}

In our previous blog post, we set up our Kubernetes cluster and deployed a pod
running nginx, but the experience was disappointing—we couldn't browse to our
pod. Let's fix that by deploying the nginx Ingress controller.

---

### References

- The canonical Kubernetes documentation for deploying the nginx Ingress
  controller on GKE <https://kubernetes.github.io/ingress-nginx/deploy/#gce-gke>
