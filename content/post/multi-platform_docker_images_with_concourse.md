---
title: "Creating Multi-Platform Docker Images with Concourse"
date: 2022-11-25T08:13:55-08:00
draft: false
---

[Concourse CI/CD](https://concourse-ci.org/) (continuous integration/continuous
delivery) can create multi-platform Docker images. This blog post describes how.

A multi-platform docker image is one that contains "[variants for
different architectures](https://docs.docker.com/build/building/multi-platform/)".

Docker images are often created for a single architecture ("instruction set
architecture" or
"[ISA](https://en.wikipedia.org/wiki/Instruction_set_architecture)"), typically
Intel's/AMD's [x86-64](https://en.wikipedia.org/wiki/X86-64), but with the
advent of ARM64-based offerings such as AWS's
[Graviton](https://aws.amazon.com/ec2/graviton/) and Apple's
[M1](https://en.wikipedia.org/wiki/Apple_M1)/[M2](https://en.wikipedia.org/wiki/Apple_M2),
It's becoming more common to build multi-platform images to avoid the heavy
emulation performance penalty (typically >10x) when running an image on a
different architecture. Multi-platform images enable a developer, for example,
to run a container just as fast on their Apple M1 laptop as their GCP (Google
Cloud Platform) Kubernetes cluster.

### Quick Start

Make sure you're on **Concourse 7.9** or later; it has the
[registry-image](https://github.com/concourse/registry-image-resource) resource
necessary to create multi-platform images.

Create a Concourse resource, "multi-platform-docker-image". This is the Docker
image you'll be building. Using the example below, make the following changes:

- replace
  [cunnie/multi-platform](https://hub.docker.com/repository/docker/cunnie/multi-platform)
  with the name of the multi-platform Docker image that you intend to create.
- replace `username: cunnie` with your Docker username.
- replace `((docker_token))` with your [Docker hub access
  token](https://docs.docker.com/docker-hub/access-tokens/)

```yaml
resources:
- name: multi-platform-docker-image
  type: registry-image
  icon: docker
  source:
    repository: cunnie/multi-platform # ← Replace with the name of your Docker image
    username: cunnie # ← Replace with your Docker Hub username
    password: ((docker_token)) # ← Replace with your Docker Hub token (or password)
    tag: latest
```

Create a job to build the Docker image using the following as an example. Make
the following changes:

- replace [`CONTEXT:
  deployments/multi-platform-docker`](https://github.com/cunnie/deployments/tree/c8207ebc06bf2adb4fabe9632d81416f69ce00ae/multi-platform-docker)
  with the path to your `Dockerfile`'s location. In this example, I have a
  Concourse input resource "deployments", and it has a subdirectory
  "multi-platform-docker/", and that subdirectory has a "Dockerfile".

Note:
- `IMAGE_PLATFORM` defines the platforms to build. If you're not sure, use
  `linux/arm64,linux/amd64`
- `OUTPUT_OCI: true` is required for multi-platform
- `image: image/image` is required for OCI output

```yaml
jobs:
- name: build-and-push-multi-platform-docker-image
  plan:
  - get: deployments
    trigger: true
  - task: build-image-task
    privileged: true
    config:
      platform: linux
      image_resource:
        type: registry-image
        source:
          repository: concourse/oci-build-task
      inputs:
      - name: deployments
      outputs:
      - name: image
      params:
        CONTEXT: deployments/multi-platform-docker
        IMAGE_PLATFORM: linux/arm64,linux/amd64
        OUTPUT_OCI: true
      run:
        path: build
  - put: multi-platform-docker-image
    params:
      image: image/image
```

Putting it all together, we have the following:

- The Concourse
  [pipeline](https://ci.nono.io/teams/main/pipelines/multi-platform-docker)
- The Concourse pipeline
  [definition](https://github.com/cunnie/deployments/blob/22b50f6c60e8486e2e73fce996ab6ce6fa836ed1/ci/pipeline-multi-platform-docker.yml)
  (YAML)
- The
  [Dockerfile](https://github.com/cunnie/deployments/blob/c8207ebc06bf2adb4fabe9632d81416f69ce00ae/multi-platform-docker/Dockerfile)
- The generated [Docker
  image](https://hub.docker.com/repository/docker/cunnie/multi-platform)

The resulting Docker image is simple: it prints out the architecture of the
underlying kernel (i.e. "aarch64" for ARM, "x86_64" for Intel) and then exits.
See for yourself:

```shell
docker run -it --rm cunnie/multi-platform
```

### Gotchas

Update your Concourse pipelines to use the new resource `registry-image`
instead of the old, deprecated `docker-image`, otherwise your pipelines will
pull the old (pre-multi-platform) image, and you'll be sad.

```diff
jobs:
  name: unit
  plan:
  - get: sslip.io
    trigger: true
  - config:
      image_resource:
        source:
          repository: cunnie/fedora-golang-bosh
-       type: docker-image
+       type: registry-image
```

### Advanced Topics

If your Docker image needs to be built slightly differently for different
platforms, use the
[`TARGETARCH`](https://docs.docker.com/engine/reference/builder/#automatic-platform-args-in-the-global-scope)
environment variable. In the following
[Dockerfile](https://github.com/cunnie/sslip.io/blob/d900b738c7dd6da3b20e806d3e0e87c783e45d26/k8s/Dockerfile-sslip.io-dns-server#L28-L31)
snippet, we use the `TARGETARCH` to download and install tha Golang-compiled
executables for the appropriate architecture:

```docker
ARG TARGETARCH # amd64, arm64 (so I can run on AWS graviton2)
RUN curl -L https://github.com/cunnie/sslip.io/releases/download/2.6.1/sslip.io-dns-server-linux-$TARGETARCH \
    -o /usr/sbin/sslip.io-dns-server; \
  chmod 755 /usr/sbin/sslip.io-dns-server
```

## Corrections & Updates

*2023-04-18*

Gotcha: update your pipelines to use the new `registry-image` instead of the
old `docker-image`, lest your pipelines don't pull the new image.

*2022-12-04*

I updated the post to require Concourse 7.9, which allowed me to remove the
custom Concourse resource type.
