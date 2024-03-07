---
title: The Underground Guide to Cloud Foundry Acceptance Tests
date: 2022-07-04T12:46:03-07:00
draft: false
images:
- https://upload.wikimedia.org/wikipedia/commons/2/29/Postgresql_elephant.svg
---

{{< figure src="https://user-images.githubusercontent.com/1020675/177218158-146d64ab-80fd-45e7-b116-fdff2da22620.png" alt="Cloud Foundry logo" >}}

The [Cloud Foundry Acceptance
Tests](https://github.com/cloudfoundry/cf-acceptance-tests) are the gold
standard to test the proper functioning of your Cloud Foundry deployment. This
guide tells you how to run them. When in doubt, refer to the
[README](https://github.com/cloudfoundry/cf-acceptance-tests).

#### Quick Start

```bash
cd ~/workspace/
git clone git@github.com:cloudfoundry/cf-acceptance-tests.git
cd cf-acceptance-tests
. ./.envrc
cp example-cats-config.json cats-config.json
export CONFIG=cats-config.json
cf api api.cf.nono.io # or whatever your Cloud Foundry's API endpoint is
cf login -u admin
cf create-space -o system system # don't worry if it's already created
cf t -o system -s system
cf enable-feature-flag diego_docker # necessary if you're running the Docker tests (`"include_docker": true`)
cf enable-feature-flag service_instance_sharing # necessary if you're running the sharing tests (`"include_service_instance_sharing": true`)
```

If you don't have the Cloud Foundry CLI (command line interface), follow the
[installation
instructions](https://docs.cloudfoundry.org/cf-cli/install-go-cli.html).
Install the latest version (v8).

Let's configure our `cats-config.json`. You should know the values for all the
replacements except `credhub_secret`; we'll explain how to get that next:

```diff
-IN DEVELOPMENT
 {
-  "api": "api.DOMAIN.com",
-  "apps_domain": "DOMAIN.com",
+  "api": "api.cf.nono.io",
+  "apps_domain": "foundry.fun",
   "admin_user": "admin",
-  "admin_password": "PASSWORD",
+  "admin_password": "MySecretAdminPassword",
   "credhub_mode": "assisted",
-  "credhub_client": "CREDHUB_CLIENT",
-  "credhub_secret": "CREDHUB_SECRET",
+  "credhub_client": "credhub_admin_client",
+  "credhub_secret": "XDBlXvmH7aN3IG2czVfzvitu5dTHIj",
   "artifacts_directory": "logs",
   "skip_ssl_validation": true,
   "timeout_scale": 1,
```

By default you're running the CredHub tests, so you need `credhub_secret`, but
getting it is a multi-step affair. First, get the credentials to communicate
with the BOSH Director's CredHub:

```bash
bosh int --path /instance_groups/name=bosh/jobs/name=uaa/properties/uaa/scim/users/name=credhub_cli_user bosh-vsphere.yml
```

_[where `bosh-vsphere.yml` is your BOSH Director's manifest]_

The output should look something like the following:

```
groups:
- credhub.read
- credhub.write
name: credhub_cli_user
password: SomeKrazyPassword
```

Now that we have the password ("SomeKrazyPassword"), let's authenticate to the
BOSH Director's CredHub using the [CredHub
CLI](https://github.com/cloudfoundry/credhub-cli):

```bash
credhub api bosh-vsphere.nono.io:8844 --skip-tls-validation # assuming your BOSH Director's hostname is "bosh-vsphere.nono.io"
credhub login --username=credhub_cli_user --password=SomeKrazyPassword
credhub find -n / # we don't need this, but it gives us a complete list of creds
credhub get -n /bosh-vsphere/cf/credhub_admin_client_secret # your path might be different, e.g. "/bosh/TAS/credhub_admin_client_secret"
  id: 1b238b69-72ac-4a73-a124-b67da71da572
  name: /bosh-vsphere/cf/credhub_admin_client_secret
  type: password
  value: XDBlXvmH7aN3IG2czVfzvitu5dTHIj
  version_created_at: "2021-11-18T23:25:53Z"
```

Aha! Now in our `cats-config.json`, set `"credhub_secret": "XDBlXvmH7aN3IG2czVfzvitu5dTHIj"`.

But we're still not done: we need to create a security group that allows the
apps to communicate with Cloud Foundry's CredHub (separate & distinct from the
BOSH Director's CredHub) (yes, it's confusing).

```bash
cf create-security-group credhub <(echo '[{"protocol":"tcp","destination":"10.0.0.0/8","ports":"8443,8844","description":"credhub"}]')
cf bind-running-security-group credhub
cf bind-staging-security-group credhub
```

Now let's run our tests!

```bash
bin/test -nodes=6
```

#### Troubleshooting Docker Failures

```text
[Fail] [docker] Docker Application Lifecycle running a docker app with a start command [BeforeEach] retains its start command through starts and stops
/home/cunnie/workspace/cf-acceptance-tests/docker/docker_lifecycle.go:53

[Fail] [docker] Docker Application Lifecycle running a docker app without a start command [BeforeEach] handles docker-defined metadata and environment variables correctly
/home/cunnie/workspace/cf-acceptance-tests/docker/docker_lifecycle.go:80

[Fail] [docker] Docker Application Lifecycle running a docker app without a start command [BeforeEach] when env vars are set with 'cf set-env' prefers the env vars from cf set-env over those in the Dockerfile
/home/cunnie/workspace/cf-acceptance-tests/docker/docker_lifecycle.go:80
```

Means you've forgotten to `cf enable-feature-flag diego_docker`.

#### Troubleshoot Service Binding Failures (CredHub)

```text
[Fail] [credhub] service bindings during staging [BeforeEach] [assisted credhub]  has CredHub references in VCAP_SERVICES interpolated
/home/cunnie/workspace/cf-acceptance-tests/credhub/service_bindings.go:123

[Fail] [credhub] service bindings during staging [BeforeEach] [non-assisted credhub]  still contains CredHub references in VCAP_SERVICES
/home/cunnie/workspace/cf-acceptance-tests/credhub/service_bindings.go:123

[Fail] [credhub] service bindings during runtime service bindings to credhub enabled broker [assisted credhub]  [BeforeEach] the broker returns credhub-ref in the credentials block
/home/cunnie/workspace/cf-acceptance-tests/credhub/service_bindings.go:123

[Fail] [credhub] service bindings during runtime service bindings to credhub enabled broker [assisted credhub]  [BeforeEach] the bound app gets CredHub refs in VCAP_SERVICES interpolated
/home/cunnie/workspace/cf-acceptance-tests/credhub/service_bindings.go:123
```

Means you've either set the wrong `credhub_secret` in `cats-config.json` or you forgot to create & bind the `credhub` security group.


## Corrections & Updates

*2024-03-06*

Added instructions to download the CF CLI. Also included
instructions to enable service instance sharing.
