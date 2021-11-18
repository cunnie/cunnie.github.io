---
title: "Concourse CI on Kubernetes (GKE), Part 5: Vault"
date: 2021-11-18T03:46:53-08:00
draft: true
images:
- https://d1q6f0aelx0por.cloudfront.net/product-logos/library-vault-logo.png
---

{{< figure src="https://d1q6f0aelx0por.cloudfront.net/product-logos/library-vault-logo.png" alt="Vault logo" >}}

In our previous post, we configured our GKE Concourse CI server, which was the
capstone of the series. But we were wrong: _this post_ is the capstone in the
series. In this post, we install Vault and configure our Concourse CI server to
use Vault to retrieve secrets.

### Installation

Most of these instructions are derived from the Hashicorp tutorial, _[Vault on
Kubernetes Deployment
Guide](https://learn.hashicorp.com/tutorials/vault/kubernetes-raft-deployment-guide?in=vault/kubernetes)_.

Create a DNS A record which points to the IP address of your GKE load balancer.
In our case, we created `vault.nono.io` which points to `104.155.144.4`.

Create the _vault_ namespace and deploy the TLS issuers to that namespace.
**Replace `brian.cunnie@gmail.com` with your email address**:

```bash
kubectl create namespace vault
kubectl apply -f <(
  curl -o- https://cert-manager.io/docs/tutorials/acme/example/staging-issuer.yaml |
  sed 's/user@example.com/brian.cunnie@gmail.com/') -n vault
kubectl apply -f <(
  curl -o- https://cert-manager.io/docs/tutorials/acme/example/production-issuer.yaml |
  sed 's/user@example.com/brian.cunnie@gmail.com/') -n vault
```

Let's create `vault-values.yml`, which contains the necessary customizations for
our Vault server.
**Replace `vault.nono.io` with your DNS record**:

```yaml
injector:
  enabled: false
server:
  ingress:
    enabled: true
    labels:
      traffic: external
    annotations:
      cert-manager.io/issuer: "letsencrypt-prod"
      kubernetes.io/ingress.class: "nginx"
    hosts:
      - host: vault.nono.io
        paths: []
    tls:
    - hosts:
      - vault.nono.io
      secretName: vault.nono.io
ui:
  enabled: true
```

Let's use Helm to install Vault:

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault \
  --namespace vault \
  -f vault-values.yml \
  --wait
```

Let's initialize our pristine Vault server:

```bash
curl \
  --request POST \
  --data '{"secret_shares": 1, "secret_threshold": 1}' \
  https://vault.nono.io/v1/sys/init | jq
```

Record `root_token` and `keys`; you'll need them to unseal the Vault and to log
in, respectively.

Now let's unseal the vault...

```bash
export VAULT_TOKEN=s.QmByxxxxxxxxxxxxxxxxxxxx
export VAULT_ADDR=https://vault.nono.io
curl \
    --request POST \
    --data '{"key": "5a302397xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"}' \
    $VAULT_ADDR/v1/sys/unseal | jq
 # check initialization status
curl $VAULT_ADDR/v1/sys/init
```

### What We Did Wrong

Hashicorp
[warns](https://learn.hashicorp.com/tutorials/vault/kubernetes-raft-deployment-guide?in=vault/kubernetes#vault-ui):

> Vault should not be deployed in a public internet facing environment

We're doing the exact opposite of what they suggest. If we're going to the
trouble of deploying Vault, we want to make sure we can use it from everywhere,
security be damned; we don't want to sprinkle separate Vault deployments on each
of our environments.

Hashicorp also
[warns](https://learn.hashicorp.com/tutorials/vault/kubernetes-raft-deployment-guide?in=vault/kubernetes#configure-vault-helm-chart):

> For a production-ready install, we suggest that the Vault Helm chart is
> installed in high availability (HA) mode

We're installing in _standalone_ mode, not _HA_ mode. We think _HA_ mode is
overkill. Also, our GKE cluster is small because we pay for it out-of-pocket,
and we don't have the resources to spend on _HA_ mode.

### Addendum: Motivation

"Why go to all this trouble to install Vault? Wasn't the Concourse server
enough?" you might ask.

The reason we installed Vault was that there were two things we wanted to
accomplish that assumed Vault (or another secret-store such as
[CredHub](https://github.com/cloudfoundry/credhub)):

1. We wanted a Concourse pipeline "[to build and publish a container
   image](https://blog.concourse-ci.org/how-to-build-and-publish-a-container-image/)".
   The tutorial's pipelines assumed a secret-store. We didn't have one. We didn't
   want to be losers that didn't have a secret-store; we wanted to be one of the
   cool kids.

1. We wanted a Concourse pipeline that used [Platform
   Automation](https://docs.pivotal.io/platform-automation/v5.0/index.html) to
   deploy a [Tanzu Ops
   Manager](https://tanzu.vmware.com/components/ops-manager). We're part of the
   development team that supports Ops Manager, and we like having our own
   testbed (instead of the corporate ones) to develop/troubleshoot. Note:
   there's nothing wrong with the corporate testbeds—they're pretty awesome—but
   we like having our own personal testbed, as well as our own personal
   Concourse CI server.

"Why use Vault instead of CredHub? Isn't CredHub is more tightly integrated into
the Cloud Foundry ecosystem?" you might also ask. The answer is simple: we like
a GUI. Yes, we know that using the GUI not as macho as using the CLI, but so
what?  A well-designed GUI can present data in a more digestible fashion than a
CLI.

Also, we wanted to learn how to use Vault, and this was a good opportunity.
Besides, Vault was written in Golang, which is more fashionable than CredHub's
Java (and starts more quickly, too).
