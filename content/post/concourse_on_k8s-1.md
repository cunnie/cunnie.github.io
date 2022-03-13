---
title: "Concourse CI on Kubernetes (GKE), Part 1: Terraform"
date: 2021-08-06T16:38:00-07:00
draft: false
images:
- https://d15shllkswkct0.cloudfront.net/wp-content/blogs.dir/1/files/2020/06/gke.png
---

{{< figure src="https://d15shllkswkct0.cloudfront.net/wp-content/blogs.dir/1/files/2020/06/gke.png" alt="GKE logo" >}}

Let's deploy [Concourse](https://concourse-ci.org/), a continuous-integration,
continuous delivery (CI/CD) application (similar to
[Jenkins](https://www.jenkins.io/) and [CircleCI](https://circleci.com/)).

We'll deploy it to [Google Cloud](https://console.cloud.google.com/), to our
[Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine) (GKE).

In this post, we'll use [HashiCorp](https://www.hashicorp.com/)'s
[Terraform](https://www.hashicorp.com/products/terraform) to create our cluster.

We assume you've already installed the terraform command-line interface (CLI)
and created a Google Cloud account.

```bash
mkdir -p ~/workspace/gke
cd ~/workspace/gke
```

Next we download the terraform templates and terraform vars file:

```bash
curl -OL https://raw.githubusercontent.com/cunnie/deployments/6b230118399f4326094b4d60e21cda32e8c6f321/terraform/gcp/gke/gke.tf
curl -OL https://raw.githubusercontent.com/cunnie/deployments/6b230118399f4326094b4d60e21cda32e8c6f321/terraform/gcp/gke/vpc.tf
curl -OL https://raw.githubusercontent.com/cunnie/deployments/6b230118399f4326094b4d60e21cda32e8c6f321/terraform/gcp/gke/terraform.tfvars
curl -OL https://raw.githubusercontent.com/cunnie/deployments/6b230118399f4326094b4d60e21cda32e8c6f321/terraform/gcp/gke/outputs.tf
```

At this point we hear cries of protest, "What?! Downloading dubious files
from sketchy software developers on the internet? Files whose provenance is
murky at best?"

Let us reassure you: the provenance of these files is crystal-clear: they have
been patterned after templates from HashiCorp's excellent tutorial, [Provision a
GKE Cluster (Google
Cloud)](https://learn.hashicorp.com/terraform/kubernetes/provision-gke-cluster),
and the companion git repo,
<https://github.com/hashicorp/learn-terraform-provision-gke-cluster>.
<sup>[[provenance](#provenance)]</sup>

Let's login with `gcloud`:

```bash
gcloud auth application-default login
```

(if you get a `command not found` error, then it means you need to install
Google Cloud's CLI; the [HashiCorp
tutorial](https://learn.hashicorp.com/tutorials/terraform/gke) has great
instructions.)

Let's customize our `terraform.tfvars` file. At the very least, change the
`project_id` to your Google Cloud's project's ID. If you're not sure what that
is, you can find it on the Google console:

{{< figure src="https://user-images.githubusercontent.com/1020675/128548349-bc7a3952-29e8-4869-ba19-c7a0ca81989b.png" alt="Project ID on Google Cloud Console" >}}

Let's use neovim (or your editor of choice):

```bash
nvim terraform.tfvars
```

Let's change the Project ID to "my-google-project-id" (assuming that's your
Google Project's name, which it isn't):

```diff
-project_id = "blabbertabber"
-friendly_project_id = "nono"
+project_id = "my-google-project-id"
+friendly_project_id = "my-google-project-id"
```

We're ready to terraform!

```bash
terraform init
terraform apply
  # answer "yes" when asked, "Do you want to perform these actions?"
```

The `terraform apply` takes ~10 minutes to complete. Now let's get our cluster
credentials:

```bash
gcloud container clusters get-credentials $(terraform output -raw kubernetes_cluster_name) --zone $(terraform output -raw zone)
```

We have a cluster at this point—let's test by deploying nginx:

```bash
kubectl run nginx --image=nginx
kubectl get pods
```

You should see the following output:

```
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          2m21s
```

#### Save that `terraform.tfstate` file!

Save the `terraform.tfstate` file; if you lose it, it becomes much more
difficult to make changes to your terraform infrastructure (you'll have to
manually tear it down & start from scratch).

We won't tell you how or where to save it, but we will tell you that we've
chosen to save ours in a [public GitHub
repo](https://github.com/cunnie/deployments/blob/551509bd1cbf2f62de8fca3a3f6cbf0f5a0ec976/terraform/gcp/gke/terraform.tfstate).
This is a bad idea! `terraform.tfstate` files often contain secrets which you do
not want to make public. Ours doesn't appear to contain any secrets, and we like
to have it publicly viewable for instructional purposes, but we may have made a
terrible mistake by publishing it.

#### Stay Tuned!

Stay tuned for the next installment, where we configure load balancers and
install Concourse CI.

---

### References

- HashiCorp's excellent tutorial, _Provision a GKE Cluster (Google Cloud)_
  <https://learn.hashicorp.com/terraform/kubernetes/provision-gke-cluster>
- Companion GitHub repository,
  <https://github.com/hashicorp/learn-terraform-provision-gke-cluster>
- <https://github.com/cloudfoundry/bosh-community-stemcell-ci-infra> is the
  repo that contains the scripts to spin up Concourse CI on GKE, and may be of
  interest to those who would like a more automated way of spinning up
  Concourse on GKE. This repo is actively maintained, and is used by the team
  that produces the BOSH Bionic stemcells. "It also uses config connector
  instead of Terraform for managing the cloud SQL instance
  <https://cloud.google.com/config-connector/docs/overview>." Thanks, Ruben
  Koster!
- <https://github.com/pivotal-cf/pci-infrastructure/tree/master/k8s> "sets up a
  GKE cluster using Terraform, Vault with Google KMS [Key Management Service]
  integration, Concourse with Vault integration, as well as cert-manager,
  Kubernetes ingress-nginx and external DNS that hooks into Google Cloud's DNS
  to create our external IP." Thanks Brian Rieger!
- <https://github.com/skyscrapers/terraform-concourse> is a good resource for
  those interested in deploying to AWS instead of GCP. Thanks [Ringo De
  Smet](https://twitter.com/ringods)!
- [How to Download & Install Terraform on Windows, Linux,
  Mac](https://spacelift.io/blog/how-to-install-terraform) is a tutorial for
  installing the terraform CLI. Martyna Łokuciejewska asked me nicely to link
  to it, so I did.

### Updates/Errata

**2022-03-12** Added a reference to installing the Terraform CLI.

**2022-01-02** Pointed out that the modifications to the terraform
configuration enable the creation of a Zonal cluster, which qualifies for the
GKE free tier.

**2021-09-30** Added an additional reference for those interested in deploying
to AWS.

**2021-09-16** Added two additional references for more complete/more automated
ways to spin up Concourse on GKE.

### Footnotes

**<a id="provenance">provenance</a>**

This begs the question, "If we're patterning our templates after HashiCorp's,
why not use HashiCorp's directly? Why change the templates?"

Our templates are $74.40 per month cheaper than Hashicorp's. Specifically, our
templates create a Zonal cluster; Hashicorp's create a Regional cluster. A Zonal
cluster [qualifies for the GKE free
tier](https://cloud.google.com/kubernetes-engine/pricing):

> The GKE free tier provides $74.40 in monthly credits per billing account that
> are applied to zonal and Autopilot clusters. If you only use a single Zonal or
> Autopilot cluster, this credit will at least _cover the complete cost of that
> cluster_ each month

But if that's not a good reason for you, then by all means use Hashicorp's
templates—they're great templates!

We've made other tweaks to the templates as well, for example, we split the
templates into a virtual private cloud (VPC) (`vpc.tf`) template and a Google
Kubernetes Engine (gke) (`gke.tf`) template.  It seemed like a good idea at the
time. Also, we didn't want to spend a lot of money, so instead of three
instances in the region, we modified the template to place two instances in the
same availability zone (creating a Zonal cluster).

_[`e2-medium` instances [cost $24.46 /
month](https://cloud.google.com/compute/vm-instance-pricing) in the region
`us-central1`. We didn't want to spend the extra $25 for a third instance.]_

Finally, we didn't like the name of our Google Cloud project ("blabbertabber"):
it was too long & referred to a project we had mothballed months ago. We wanted
a shorter and friendlier name ("nono"), and we were loath to create a brand new
Google Cloud project, so we modified the templates to include a "friendly"
project name.
