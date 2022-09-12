---
title: "Tuning HAProxy in a vSphere Environment"
date: 2022-09-10T09:33:02-07:00
draft: false
images:
- https://user-images.githubusercontent.com/1020675/189493113-19fbba73-fbc6-4028-aafa-62bb81102cd7.svg
---

<!-- https://gohugo.io/content-management/shortcodes/#figure -->

{{< figure src="https://docs.google.com/drawings/d/e/2PACX-1vRrDRY6cxfp4AOz2pEPSXHcM2KWnwtWnC_tSoQNwUkRKxs-oxdWo51GBxDaaR7dNfTYbB-gioM3bkjX/pub?w=1999&amp;h=662" height="100%" width="100%" alt="Network Diagram" caption="Network Diagram. We want to maximize the throughput from the blue box (the client) to the green box (HAProxy)">}}

### Summary

We were able to push through almost 450 MB/sec through HAProxy (which
terminated our SSL) by carefully matching our 4-core HAProxy with 2 x 4-core
Gorouters (which were on a _much_ slower ESXi host).

### Results

| Bandwidth MB/second | Configuration                   |
|--------------------:|---------------------------------|
| 201.27MB            | 1 HAProxy: 1 vCPU               |
| 136.47MB            | 1 HAProxy: 2 vCPUs              |
| 270.56MB            | 2 Gorouters: 1 vCPU             |
| 350.48MB            | 2 Gorouters: 2 vCPUs            |
| 447.49MB            | 1 HAProxy, 2 Gorouters: 4 vCPUs |

#### 0. HAProxy with 1 vCPU

HAProxy had only 1 vCPU during this iteration, and the CPU was maxed to 100%
during the test (according to `htop`). We suspect that TLS was the culprit for
much of the traffic: HAProxy terminated TLS traffic inbound, and initiated TLS
to the Gorouters.

#### 1. HAProxy with 2 vCPUs

Surprisingly, adding a second vCPU (2 vCPUs, 1 socket) made things _worse_;
bandwidth dropped >30%.

Although the HAProxy never maxed-out its CPU, the Gorouter did, pinned at 100%
during the duration of the test (`htop`).

#### 2. Two Gorouters

Previously we only had 1 Gorouter backing the HAProxy, but now we doubled it to
2 Gorouters. The doubling of VMs notwithstanding, the Gorouters' CPUs were
still pegged at 100%.

#### 2. Two Gorouters with 2 vCPUs

Previously our Gorouters had 1 vCPU. We doubled it to 2 vCPUs. We saw that our
HAProxy's 2 cores were pegged at 100%, and the Gorouters' 2 cores were almost
maxed-out.

Note that the Gorouters are on a Xeon which can be charitably characterized as
"the world's slowest Xeon". In other words, don't extrapolate the ratios of
HAProxies to Gorouters from this particularly lopsided setup.

#### 2. One HAProxy, Two Gorouters with 4 vCPUs

We were able to push through almost 450 MB/s, but the HAProxy's CPU was
almost, but not quite, 100%. We've reached the hardware limits of our
homelab—we don't have enough physical cores to tune any further, so we bring
this post to a close.

### References

- Test suite: <https://github.com/wg/wrk>
- Test suite invocation:
```bash
./wrk -t8 -c32 -d60s https://10.9.250.10/10mb -H "Host: dora.foundry.fun"
```
- Our 10MB app: <https://github.com/cloudfoundry/cf-acceptance-tests>
- Our change to the code (thanks Matthew Kocher!):
```diff
--- a/assets/dora/dora.rb
+++ b/assets/dora/dora.rb
@@ -39,6 +39,11 @@ class Dora < Sinatra::Base
     end
   end

+  get '/10mb' do
+    require 'securerandom'
+    $ten_mb ||= SecureRandom.random_bytes(10 * 1024 * 1024)
+  end
+
```

<!--
{{< figure src="https://user-images.githubusercontent.com/1020675/189493113-19fbba73-fbc6-4028-aafa-62bb81102cd7.svg" height="100%" width="100%" alt="HAProxy logo" >}}
-->
