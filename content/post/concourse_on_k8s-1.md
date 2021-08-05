---
title: "Concourse CI on Kubernetes (GKE), Part 1: Terraform"
date: 2021-08-05T11:41:34-07:00
draft: true
images:
- https://d15shllkswkct0.cloudfront.net/wp-content/blogs.dir/1/files/2020/06/gke.png
---

{{< figure src="https://d15shllkswkct0.cloudfront.net/wp-content/blogs.dir/1/files/2020/06/gke.png" alt="GKE logo" >}}

Let's deploy [Concourse](https://concourse-ci.org/), a continuous-integration,
continuous delivery (CI/CD) application (similar to
[Jenkins](https://www.jenkins.io/) and [CircleCI](https://circleci.com/)).

We'll deploy it to [Google Cloud](https://console.cloud.google.com/), to our
[Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine) (GKE).

In this section, we'll use [HashiCorp](https://www.hashicorp.com/)'s
[Terraform](https://www.hashicorp.com/products/terraform) to create our cluster.
