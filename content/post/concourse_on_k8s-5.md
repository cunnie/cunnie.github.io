---
title: "Concourse CI on Kubernetes (GKE), Part 5: Vault"
date: 2021-11-18T03:46:53-08:00
draft: false
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

Let's create `vault-values.yml`, which contains the necessary customizations
for our Vault server.  **Replace `vault.nono.io` with your DNS record**:

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

Let's initialize our pristine Vault server. We want only one key
(`"secret_shares": 1`) to unseal the vault; we're not a nuclear missile silo
that needs three separate keys to trigger a launch.

```bash
curl \
  --request POST \
  --data '{"secret_shares": 1, "secret_threshold": 1}' \
  https://vault.nono.io/v1/sys/init | jq
```

Record `root_token` and `keys`; you'll need them to unseal the Vault and log
in, respectively.
**Replace `VAULT_KEY` and `VAULT_TOKEN` with your `keys` and `root_token`**:

```bash
export VAULT_KEY=5a302397xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
export VAULT_TOKEN=s.QmByxxxxxxxxxxxxxxxxxxxx
export VAULT_ADDR=https://vault.nono.io
curl \
    --request POST \
    --data '{"key": "'$VAULT_KEY'"}' \
    $VAULT_ADDR/v1/sys/unseal | jq
 # check initialization status
curl $VAULT_ADDR/v1/sys/init
```

### Concourse Integration

Much of this section was shamelessly copied from the excellent canonical
Concourse documentation, _[The Vault credential
manager](https://concourse-ci.org/vault-credential-manager.html)_.

#### Configure the Secrets Engine

Create a key-value `concourse/` path in Vault for Concourse to access its
secrets:

```bash
vault secrets enable -version=1 -path=concourse kv
```

Create `concourse-policy.hcl` so that our Concourse server has access to that
path:

```hcl
path "concourse/*" {
  policy = "read"
}
```

Let's upload that policy to Vault:

```bash
vault policy write concourse concourse-policy.hcl
```

#### Configure a Vault `approle` for Concourse

Let's enable the `approle` backend on Vault:

```bash
vault auth enable approle
```

Let's create the Concourse `approle`:

```bash
vault write auth/approle/role/concourse policies=concourse period=1h
```

We need the `approle`'s `role_id` and `secret_id` to set in our Concourse
server:

```bash
vault read auth/approle/role/concourse/role-id
  # role_id    045e3a37-6cc4-4f6b-4312-36eed80f7adc
vault write -f auth/approle/role/concourse/secret-id
  # secret_id  59b8015d-8d4a-fcce-f689-xxxxxxxxxxxx
```

#### Configure Concourse to Use Vault

Let's configure our Concourse deployment to use Vault. We append yet even more
arguments to our already-gargantuan <sup>[[values_file](#values_file)]</sup>
command to deploy Concourse.
**Replace `gke-nono-io` with the name of your Helm release, `gke.nono.io` with
the hostname of your Concourse server**, and replace all the various and sundry
credentials, too:


```bash
helm upgrade gke-nono-io concourse/concourse \
  --set concourse.web.externalUrl=https://gke.nono.io \
  --set concourse.web.auth.duration=240h \
  --set 'web.ingress.enabled=true' \
  --set 'web.ingress.annotations.cert-manager\.io/issuer=letsencrypt-prod' \
  --set 'web.ingress.annotations.kubernetes\.io/ingress.class=nginx' \
  --set 'web.ingress.hosts={gke.nono.io}' \
  --set 'web.ingress.tls[0].hosts[0]=gke.nono.io' \
  --set 'web.ingress.tls[0].secretName=gke.nono.io' \
  \
  --set-file secrets.sessionSigningKey=secrets/session_signing_key \
  --set-file secrets.hostKey=secrets/tsa_host_key \
  --set-file secrets.hostKeyPub=secrets/tsa_host_key.pub \
  --set-file secrets.workerKey=secrets/worker_key \
  --set-file secrets.workerKeyPub=secrets/worker_key.pub \
  \
  --set secrets.localUsers="" \
  --set concourse.web.localAuth.enabled=false \
  --set concourse.web.auth.mainTeam.github.org=blabbertabber \
  --set concourse.web.auth.github.enabled=true \
  --set secrets.githubClientId=5e4ffee9dfdced62ebe3 \
  --set secrets.githubClientSecret=549e10b1680ead9cafa30d4c9a715681cec9b074 \
  \
  --set concourse.web.vault.enabled=true \
  --set concourse.web.vault.url=https://vault.nono.io:443 \
  --set concourse.web.vault.authBackend=approle \
  --set concourse.web.vault.useAuthParam=true \
  --set secrets.vaultAuthParam="role_id:045e3a37-6cc4-4f6b-4312-36eed80f7adc\\,secret_id:59b8015d-8d4a-fcce-f689-xxxxxxxxxxxx" \
  \
  --wait
```

_Gotchas: you need `:443` at the end of the vault URL
<sup>[[443](#443)]</sup>,
and you need the double-backslash before the comma in the `vaultAuthParam`
<sup>[[double_backslash](#double_backslash)]</sup>._

#### Putting It Together

Let's create a secret which we'll interpolate into our pipeline. We create the
key under the Vault path `concourse/main` (`main` is our Concourse team's name.
If you're not sure what your Concourse team name, it's probably `main`):

```bash
vault kv put concourse/main/ozymandias-secret value="Look on my Works, ye Mighty, and despair\!"
```

Let's create a simple Concourse pipeline definition, `pipeline-ozymandias.yml`

```yaml
jobs:
- name: ozymandias-job
  plan:
  - task: ozymandias-task
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: fedora
      run:
        path: echo
        args:
        - "Ozymandias says:"
        - ((ozymandias-secret))
```

Let's `fly` our new pipeline. **Replace `nono` with your Concourse target's
name**:

```bash
fly -t gke set-pipeline -p ozymandias-pipeline -c pipeline-ozymandias.yml
fly -t gke expose-pipeline -p ozymandias-pipeline
fly -t gke unpause-pipeline -p ozymandias-pipeline
fly -t gke trigger-job -j ozymandias-pipeline/ozymandias-job
```

Let's browse to our job, expand `ozymandias-task` by clicking on it, and allow
ourselves to luxuriate in the sweet smell of success:

{{< figure src="https://user-images.githubusercontent.com/1020675/143608009-7b215d58-15c3-463c-be81-d93b39a510cb.png" alt="Vault logo" >}}

If you instead see an aborted Concourse job with the error, `failed to
interpolate task config: undefined vars: ozymandias-secret`, you probably have
mangled the authentication credentials (`vaultAuthParam`) setting.

### What We Did Wrong

Hashicorp
[warns](https://learn.hashicorp.com/tutorials/vault/kubernetes-raft-deployment-guide?in=vault/kubernetes#vault-ui):

> Vault should not be deployed in a public internet facing environment

We're doing the exact opposite of what they suggest. If we're going to the
trouble of deploying Vault, we want to make sure we can use it from everywhere,
security be damned; we don't want to waste our time sprinkling separate Vault
deployments like magic fairy dust on each of our environments.

Hashicorp also
[warns](https://learn.hashicorp.com/tutorials/vault/kubernetes-raft-deployment-guide?in=vault/kubernetes#configure-vault-helm-chart):

> For a production-ready install, we suggest that the Vault Helm chart is
> installed in high availability (HA) mode

We're installing in _standalone_ mode, not _HA_ mode. We think _HA_ mode is
overkill for our use case. Also, our GKE cluster is small because we pay for it
out-of-pocket, and we don't have the resources to spend on _HA_ mode.

### Addendum: Updating

Here's the correct process for updating:

1. Update vault. Then unseal the vault.
2. Update Concourse.

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

### Updates/Errata

**2022-07-04** Added the order in which to apply Vault & Concourse updates.

### Footnotes

**<a id="values_file">values_file</a>**

Are you sick of the gargantuan command to deploy Concourse? Then do what we
do—use a Helm values file (e.g. `concourse-values.yml`). You can see ours
[here](https://github.com/cunnie/deployments/blob/1a925698535610c45d15befbb4ad6e262c62b80c/terraform/gcp/gke/concourse-values.yml#L6-L10).
With that file our command to deploy Concourse becomes manageably smaller:

```bash
helm upgrade gke.nono.io concourse/concourse \
  -f concourse-values.yml \
  ...
```

**<a id="443">443</a>**

You may think, "That `:443` at the end of `https://vault.nono.io:443` is
redundant & superfluous; I'll redact that on my version. Any programmer worth
his salt knows that the `https` scheme defaults to port 443."

Well I got news for you, Sunshine: you need that `:443`; if you skip it, you'll
get the dreaded, "failed to interpolate task config: timed out to login to
vault" error when your Concourse task attempts to interpolate a variable.

In some ways it's similar to the `openssl s_client -connect vault.nono.io:443`
command which insists that you specify the port even though 99% of the time that
port is going to be 443. What, you're not familiar with the `openssl s_client
-connect` command? Well stick around, it'll come in useful when you have to
debug certs on someone else's server.

**<a id="double_backslash">double_backslash</a>**

You need the double backslash before the comma; if you don't have it, the
following error will rear its ugly head when you attempt `helm upgrade`:

```
Error: INSTALLATION FAILED: failed parsing --set data: key "secret_id:59b8015d-8d4a-fcce-f689-xxxxxxxxxxxx" has no value
```
