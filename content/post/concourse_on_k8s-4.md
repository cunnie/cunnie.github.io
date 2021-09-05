---
title: "Concourse CI on Kubernetes (GKE), Part 4: Concourse"
date: 2021-09-01T18:39:26-07:00
draft: true
images:
- https://pbs.twimg.com/profile_images/971772210821070848/jnsjSQcw_400x400.jpg
---

{{< figure src="https://pbs.twimg.com/profile_images/971772210821070848/jnsjSQcw_400x400.jpg" alt="Concourse logo" >}}

In our previous post, we configured our GKE (Google Kubernetes Engine) to
use Let's Encrypt TLS certificates. In this post, the capstone of our series, we
install Concourse CI.

### Installation

These instructions are a more-opinionated version of the canonical instructions
for the Concourse CI Helm chart found here:
<https://github.com/concourse/concourse-chart>.

#### First Install with Helm

We use `helm` to install Concourse. We first add the helm repo, and then install
it. We take the opportunity to bump the default login time from 24 hours to ten
days (`duration=240h`) because we hate re-authenticating every morning to our
Concourse every morning.  **Replace `gke.nono.io` with your DNS record**:

```bash
kubectl delete ingress kuard # to free up https://gke.nono.io
helm repo add concourse https://concourse-charts.storage.googleapis.com/
helm install gke-nono-io concourse/concourse \
  --set concourse.web.externalUrl=https://gke.nono.io \
  --set concourse.web.auth.duration=240h \
  --set 'web.ingress.enabled=true' \
  --set 'web.ingress.annotations.cert-manager\.io/issuer=letsencrypt-prod' \
  --set 'web.ingress.annotations.kubernetes\.io/ingress.class=nginx' \
  --set 'web.ingress.hosts={gke.nono.io}' \
  --set 'web.ingress.tls[0].hosts[0]=gke.nono.io' \
  --set 'web.ingress.tls[0].secretName=gke.nono.io' \

```

We wait approximately 30 seconds for the acquisition of the TLS certificate for
gke.nono.io, then browse to our site <https://gke.nono.io>. If we get an HTTP
status 503, then wait another ten seconds and try again.

### Second Install: now with GitHub OAuth

We want to authenticate with GitHub, so we follow [these
instructions](https://docs.github.com/en/developers/apps/building-oauth-apps/creating-an-oauth-app)
to set up a GitHub Oauth app. Better yet, read the instructions below.

We browse to GitHub → dropdown menu on our user → Settings → Developer Settings
→ OAuth Apps → New OAuth App.

Here's how we filled out ours. **Make sure to replace
`gke.nono.io` with your URL**. We want the correct callback URL:

{{< figure src="https://user-images.githubusercontent.com/1020675/132111425-a20812dc-78c6-41bb-ae9f-5e6fde7c1d6f.png" alt="GitHub OAuth Application #1" >}}

We click "Register Application", which brings us to the next screen, where we
get the Client ID (`2317874f614900d21bdd`) and then click "Generate a new client
secret" to get the Client secret (`08d670a1916193d9294705e223fb0a09d0fccb08`).
Don't forget to click "Update Application"!

{{< figure src="https://user-images.githubusercontent.com/1020675/132111554-3cccc044-85b6-40e2-9f15-32c43a988c7e.png" alt="GitHub OAuth Application #1" >}}

Now we can add the four GitHub OAuth-related lines to our `helm install`
command. **Replace the GitHub org `blabbertabber` and the GitHub Client ID and
Client Secret with the ones you've created**:

```bash
helm delete gke-nono-io # we want to clear out the old Concourse CI
helm install gke-nono-io concourse/concourse \
  --set concourse.web.externalUrl=https://gke.nono.io \
  --set concourse.web.auth.duration=240h \
  --set 'web.ingress.enabled=true' \
  --set 'web.ingress.annotations.cert-manager\.io/issuer=letsencrypt-prod' \
  --set 'web.ingress.annotations.kubernetes\.io/ingress.class=nginx' \
  --set 'web.ingress.hosts={gke.nono.io}' \
  --set 'web.ingress.tls[0].hosts[0]=gke.nono.io' \
  --set 'web.ingress.tls[0].secretName=gke.nono.io' \
  \
  --set concourse.web.auth.mainTeam.github.org=blabbertabber \
  --set concourse.web.auth.github.enabled=true \
  --set secrets.githubClientId=2317874f614900d21bdd \
  --set secrets.githubClientSecret=08d670a1916193d9294705e223fb0a09d0fccb08 \

```

We wait our 30 seconds, and then browse to our URL: <https://gke.nono.io>. We
log in with GitHub Auth. We authorize our app. We download & install our `fly`
CLI. We create the following simple pipeline file, `simple.yml`:

```yaml
jobs:
- name: simple
  plan:
  - task: simple
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: fedora
      run:
        path: true
```

We connect to our new Concourse server:

```bash
fly -t gke login -c https://gke.nono.io # click the link, see "login successful!"
fly -t gke sp -p simple -c simple.yml
```

### Extra Credit: Creating an External Concourse Worker

Let's say, hypothetically, we have a home vSphere setup, where VMs and disk are
essentially free. Furthermore, let's say that cost is a very big consideration
because these VMs are paid for out-of-pocket (i.e. we're buying them, our
corporation isn't). In that case, we might want to create a large worker VM on
our vSphere setup.

An alternative motivation for setting up an external Concourse worker would be
network access: a Concourse worker within a corporate network would have
access to resources that a GKE worker wouldn't, e.g.: an internal GitLab
instance.

To create an external Concourse worker, we must first create a Kubernetes
service that allows the external Concourse worker to contact the Concourse
server on port 2222. Below is our version. Remember to **replace `34.135.26.144`
with your load balancer's IP and any occurrence of `nono-io` with something more
appropriate to your setup**. Name the file `concourse-service.yml`:

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: ci-nono-io-web-worker-gateway-lb
  namespace: default
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  loadBalancerIP: 34.135.26.144
  ports:
  - name: concourse-worker
    port: 2222
    protocol: TCP
    targetPort: tsa
  selector:
    app: gke-nono-io-web
```

Now apply it.

```bash
kubectl apply -f concourse-service.yml
```

#### Pro-tip

Rather than having an onerous number of `--set` arguments to our `helm install`
command, we find it easier to modify the corresponding settings in the
[`values.yml`](https://github.com/concourse/concourse-chart/blob/master/values.yaml)
file and pass it in instead, i.e. `helm install -f values.yml ...`

---

### References

- Concourse CI Helm chart: <https://github.com/concourse/concourse-chart>
- Helm Chart Install: Advanced Usage of the “Set” Argument: <https://itnext.io/helm-chart-install-advanced-usage-of-the-set-argument-3e214b69c87a>
- Creating an OAuth App on GitHub:
  <https://docs.github.com/en/developers/apps/building-oauth-apps/creating-an-oauth-app>
