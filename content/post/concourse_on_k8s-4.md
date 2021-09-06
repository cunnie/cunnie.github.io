---
title: "Concourse CI on Kubernetes (GKE), Part 4: Concourse"
date: 2021-09-01T18:39:26-07:00
draft: false
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

We want to authenticate against our GitHub organization, "blabbertabber", so we
browse to our organization (<https://github.com/blabbertabber>) → Settings →
Developer Settings → OAuth Apps → New OAuth App.

_Note: "[Note that the client must be created under an organization if you want
to authorize users based on organization/team
membership](https://concourse-ci.org/github-auth.html)."_

Here's how we filled out ours. **Make sure to replace `gke.nono.io` with your
URL**. The authorization callback URL is particularly important; don't mess it
up:

{{< figure src="https://user-images.githubusercontent.com/1020675/132111425-a20812dc-78c6-41bb-ae9f-5e6fde7c1d6f.png" alt="GitHub OAuth Application #1" >}}

We click "Register Application", which brings us to the next screen, where we
get the Client ID (`2317874f614900d21bdd`) and then click "Generate a new client
secret" to get the Client secret (`08d670a1916193d9294705e223fb0a09d0fccb08`).
Don't forget to click "Update Application"!

{{< figure src="https://user-images.githubusercontent.com/1020675/132131649-678f961b-978a-4055-8388-0b67debbc62e.png" alt="GitHub OAuth Application #2" >}}

Now we can add the five GitHub OAuth-related lines to our `helm install`
command. **Replace the GitHub org `blabbertabber` and the GitHub Client ID and
Client Secret with the ones you've created**:

_[Note: we take the opportunity to disable `localAuth` so that strangers can't
log into our Concourse server using the account "test" with the password
"test".]_

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
  --set concourse.web.localAuth.enabled=false \
  --set concourse.web.auth.mainTeam.github.org=blabbertabber \
  --set concourse.web.auth.github.enabled=true \
  --set secrets.githubClientId=5e4ffee9dfdced62ebe3 \
  --set secrets.githubClientSecret=549e10b1680ead9cafa30d4c9a715681cec9b074 \

```

We wait our 30 seconds, and then browse to our URL: <https://gke.nono.io>. We
log in with GitHub Auth. We authorize our app. We download & install our `fly`
CLI. Then we log in:

```bash
fly -t gke login -c https://gke.nono.io
 # click the link
 # click "Authorize blabbertabber"
 # see "login successful!"
```

We create the following simple pipeline file, `simple.yml`:

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
        path: "true"
```

Let's `fly` our new pipeline:

```bash
fly -t gke set-pipeline -p simple -c simple.yml
fly -t gke expose-pipeline -p simple
fly -t gke unpause-pipeline -p simple
```

We browse to our Concourse and see the sweet green of success (it'll take a
minute or two to run):

{{< figure src="https://user-images.githubusercontent.com/1020675/132144890-29253cd7-18bb-4c99-8466-10f4013000ec.png" alt="A successful green build!" >}}

Yay! We're done.

#### Pro-tip

Rather than having an onerous number of `--set` arguments to our `helm install`
command, we find it easier to modify the corresponding settings in the
[`values.yml`](https://github.com/concourse/concourse-chart/blob/master/values.yaml)
file and pass it to our invocation of `helm`, i.e. `helm install -f values.yml ...`

---

### References

- Concourse CI Helm chart: <https://github.com/concourse/concourse-chart>
- Helm Chart Install: Advanced Usage of the “Set” Argument: <https://itnext.io/helm-chart-install-advanced-usage-of-the-set-argument-3e214b69c87a>
- Creating an OAuth App on GitHub:
  <https://docs.github.com/en/developers/apps/building-oauth-apps/creating-an-oauth-app>
