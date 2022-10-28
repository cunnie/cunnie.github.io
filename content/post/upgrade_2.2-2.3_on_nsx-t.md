---
authors:
- rowan
- yeshwant
- cunnie
categories:
- NSX-T
- Operations Manager
- BOSH
date: Thu Sep  6 10:44:30 PDT 2018
draft: false
short: |
  How to upgrade an NSX-T-based PAS 2.2 → 2.3 foundation without downtime
title: Safely Upgrading PAS 2.2 with NSX-T Load Balancers
---

When customers with vSphere+NSX-T-based foundations apply a stemcell update,
update a tile, or upgrade PAS (Pivotal Application Service) from 2.2 to 2.3,
their Cloud Foundry may become unreachable as their NSX-T static load balancer
server pools have been emptied.

This blog post describes a method to ensure availability during upgrades. We
use a combination of customized Operations Manager [resource
configs](https://docs.pivotal.io/pivotalcf/2-2/customizing/config-er-vmware.html#resources)
and BOSH [VM Extensions](https://bosh.io/docs/terminology/#vm-extension).

The sample workflow in this post is for upgrading PAS 2.2 to PAS 2.3 with an
Operations Manager upgrade; however, it can also be adapted to stemcell or
tile upgrades as well.

_Operations Manager 2.3 introduces the capability to manage the full lifecycle
of the membership of the NSX-T load balancer pools, which relieves customers of
the responsibility of manually assigning VMs to server pools.  This allows
uninterrupted availability of the BOSH instance group VMs that use NSX-T load
balancers even during upgrades, re-deploys, and IP address re-assignment._

_To enable these features without downtime for deployments using Operations
Manager 2.2, one must migrate membership from out-of-band VM assignment to
BOSH-managed assignment prior to upgrading to Operations Manager 2.3. This blog
post describes that migration process._

## 0. Procedure

- Review manually-created static <a href="#dynamic"><sup>[or Dynamic]</sup></a> NSX-T Load Balancer Server Pools
- Craft BOSH VM Extensions
- Craft Operations Manager resource configs
- Upgrade Operations Manager to version 2.3
- Stage VM extensions with the Operations Manager API
- Prepare to upgrade Pivotal Application Service to 2.3.0
- Stage new resource configs with the Operations Manager API
- Apply changes (deploy)

## 1. Review Manually-Created Static NSX-T Load Balancer Server Pools

We log into our NSX Manager to review the Server Pool configuration of our Static Load Balancers:

{{< figure src="https://user-images.githubusercontent.com/1020675/45176304-68843000-b1c4-11e8-84ea-14011b80de79.png" >}}

<div class="alert alert-warning" role="alert">

You may not have four pools; it's possible you may only have one. In that case,
adapt the instructions below to the number of load balancer server pools you have.<br /><br />

For example, you may not have a TCP Router server pool if you don't use the TCP
routing feature of PAS.

</div>

In the table below, we describe the purpose of each of the load balancer server
pools. The most important load balancers are the first two; they allow HTTP(S)
access to our applications. The "Job" column is the name of the job that is
load balanced by this server pool; they can be viewed in the Pivotal Application
Service "Status" page in Operations Manager. The "BOSH instance group" column is
the name of the instance group for this job within BOSH.

| Server Pool               | Description                                  | Port | Job         | BOSH instance group |
|---------------------------|----------------------------------------------|------|-------------|---------------------|
| PAS-GoRouter443ServerPool | HTTPS access to cf apps                      | 443  | Router      | `router`            |
| PAS-GoRouter80ServerPool  | HTTP access to cf apps                       | 80   | Router      | `router`            |
| PAS-SSHProxyServerPool    | `cf ssh`                                     | 2222 | Diego Brain | `diego_brain`       |
| PAS-TCPRouterServerPool   | TCP access to non-HTTP(S) cf apps (optional) | *    | TCP Router  | `tcp_router`        |

\* - the TCP router server pool uses a port range, e.g. 1024-1124, and does not load balance a single port.

We double-check to make sure the IP addresses of the VMs backing each
server pool correspond to the IP addresses of the jobs. For example, we check
the IP addresses backing the **PAS-GoRouter443ServerPool** server pool:

{{< figure src="https://user-images.githubusercontent.com/1020675/45178047-8902b900-b1c9-11e8-9fbc-cfd28a055078.png" >}}


We cross-reference those IP addresses to the IPs of the Router VMs in our foundation:

{{< figure src="https://user-images.githubusercontent.com/1020675/45177840-eea27580-b1c8-11e8-9b00-98e720550c6a.png" >}}

We cross-reference all server pools.

## 2. Craft BOSH VM Extensions

For each BOSH instance group, we prepare a JSON file describing a VM extension
that will be applied to that instance group.

Here is an example of the VM extension configuration for the Router job:

```
{
  "cloud_properties": {
    "nsxt": {
      "lb": {
        "server_pools": [
          {
            "name": "PAS-GoRouter443ServerPool",
            "port": 443
          },
          {
            "name": "PAS-GoRouter80ServerPool",
            "port": 80
          }
        ]
      }
    }
  },
  "name": "http_https_lb"
}
```

The `server_pools` array should contain a list of JSON objects with the name of
the NSX-T load balancer server pool, and where applicable, a port.  When
configuring load balancers that use a port range, such as the TCP router load
balancer, the port should be omitted.

The `name` (e.g. `http_https_lb`) is arbitrary but must match the `name` used
in the the resource config in the following section.

In our case, we create 3 VM extensions: `router_vm-extension.json`,
`diego_brain_vm-extension.json`, `tcp_router_vm-extension.json`.

## 3. Craft Operations Manager resource configs

For each BOSH instance group, we prepare a JSON  (JavaScript Object Notation)
file describing the resource config that BOSH will use when redeploying the
instance group during the upgrade.

### 3.0 Authenticate To Use the Operations Manager API

We [authenticate](http://docs.pivotal.io/pivotalcf/2-3/opsman-api/#authentication)
in order to obtain our UAA access token.

```bash
FOUNDATION_URL= # your ops manager URL goes here, e.g. https://pcf.example.com
UAA_ACCESS_TOKEN= # your token goes here, a very long string. Very long.
```

### 3.1 Determine the GUID of the PAS Foundation

We use the Operations Manager API to determine the GUID of the PAS foundation:

```bash
curl "$FOUNDATION_URL/api/v0/staged/products" -H "Authorization: Bearer $UAA_ACCESS_TOKEN"
```

This will return a long string of JSON. We're looking for this part:
`[..., {"installation_name":"some-guid","guid":"some-guid","type":"cf","product_version":"2.2.2"}, ...]`.
This is our `guid`: `cf-ebd5ce1f7b11714cbb94`.

### 3.2 Determine the GUID of the Jobs That Belong in Server Pools

```bash
CF_GUID= # the GUID we got from the previous step
curl "$FOUNDATION_URL/api/v0/staged/products/$CF_GUID/jobs" -H "Authorization: Bearer $UAA_ACCESS_TOKEN"
```

This also returns a long string of JSON. We want to find the GUIDs for the jobs
(BOSH instance groups) `router`, `diego_brain`, and `tcp_router` (the
corresponding GUIDs in our foundation are `router-1a40bb1433cd790d3920`,
`diego_brain-247c1f4a616b5d43546e`, and `tcp_router-93c7a99605c109f53f8c`).

If you have `jq` installed,
you may use the following command to isolate the important components:
`jq -r '.jobs[] | select(.name=="router" or .name=="diego_brain" or .name=="tcp_router")'`.

<!-- {"jobs":[{"name":"consul_server","guid":"consul_server-8ad14bfd8618c97de8ce"},{"name":"nats","guid":"nats-6be11c3f099126c60ef1"},{"name":"nfs_server","guid":"nfs_server-2dfef76a89ec89887cce"},{"name":"mysql_proxy","guid":"mysql_proxy-86ca13c9e7ae8f4c5576"},{"name":"mysql","guid":"mysql-c2156702dce38b439e75"},{"name":"backup-restore","guid":"backup-restore-0399ae58267bf0056c44"},{"name":"diego_database","guid":"diego_database-cb7e5f6c141e0ada5654"},{"name":"uaa","guid":"uaa-3a9a51f5f6f77e6a9da5"},{"name":"cloud_controller","guid":"cloud_controller-62d932d15269c0a93834"},{"name":"ha_proxy","guid":"ha_proxy-441ee77ed12ee8b86dff"},{"name":"router","guid":"router-1a40bb1433cd790d3920"},{"name":"mysql_monitor","guid":"mysql_monitor-a736ca63cd9ab8aa913c"},{"name":"clock_global","guid":"clock_global-889e0927aa33ad48cdab"},{"name":"cloud_controller_worker","guid":"cloud_controller_worker-2ab643dc6b2f5f71fc7f"},{"name":"diego_brain","guid":"diego_brain-247c1f4a616b5d43546e"},{"name":"diego_cell","guid":"diego_cell-11b55994dd4964379ac1"},{"name":"loggregator_trafficcontroller","guid":"loggregator_trafficcontroller-f1e490f8f159450cca7f"},{"name":"syslog_adapter","guid":"syslog_adapter-491cfdfce6ef86abed17"},{"name":"syslog_scheduler","guid":"syslog_scheduler-e0eb55476c1557f84d35"},{"name":"doppler","guid":"doppler-aa3abe9d191298ecb78f"},{"name":"tcp_router","guid":"tcp_router-93c7a99605c109f53f8c"},{"name":"credhub","guid":"credhub-2c2724686e7bb260e32b"}]} -->

### 3.3 Get Existing Resource Configs Attached to the Jobs

We run the following command for the GUID of each job we're interested in:

```
curl "$FOUNDATION_URL/api/v0/staged/products/$CF_GUID/jobs/$JOB_GUID/resource_config" -H "Authorization: Bearer $UAA_ACCESS_TOKEN" > $JOB_NAME-resource-config.json
```

Where `CF_GUID` is set to the GUID from step 3.1, `JOB_GUID` is set to the GUID
from step 3.2, and `JOB_NAME` is set to the name of the BOSH instance group for
this job.

This produces a result like `{"instance_type":{"id":"automatic"},"instances":2,"nsx_security_groups":null,"nsx_lbs":[],"additional_vm_extensions":[]}`.

### 3.4 Write New Resource Configs For Each Job

Edit each file from the previous step to add the name of the corresponding load
balancer (see table in step 1), in quotes, to the `additional_vm_extensions`
array. <a href="#resource_config"><sup>[Resource Config]</sup></a>

We do this once for each load-balanced job (`router`, `diego_brain`, and `tcp_router`).

## 4. Upgrade Operations Manager to Version 2.3

Following the steps in the [Upgrading Pivotal Cloud Foundry](https://docs.pivotal.io/pivotalcf/2-2/customizing/upgrading-pcf.html)
documentation, we upgrade Operations Manager to version 2.3.

***Do not*** follow steps to upgrade the PAS tile at this time.

If you are using the VMware NSX-T tile, and it is version 2.2 or earlier, you should now stage version 2.3 of that tile.

***Do not*** press "Apply Changes" during this step.

## 5. Stage VM Extensions with the Operations Manager API

For each VM extension JSON file we wrote in step 2, we run the following `curl` command:

```bash
curl "$FOUNDATION_URL/api/v0/staged/vm_extensions" -X POST -H "Authorization: Bearer $UAA_ACCESS_TOKEN" -d "@${VM_EXTENSION_FILE}" -H "Content-Type: application/json"
```

Where `${VM_EXTENSION_FILE}` is the path to the file. We expect to see an HTTP status 200 and an empty JSON object `{}` returned for each call.

## 6. Stage Pivotal Application Service Tile Version 2.3.0

Following the steps in the [Upgrading Pivotal Cloud Foundry](https://docs.pivotal.io/pivotalcf/2-2/customizing/upgrading-pcf.html)
documentation, we stage PAS tile version 2.3.0.

***Do not*** press "Apply Changes" during this step.

## 7. Stage New Resource Configs with the Operations Manager API

For each resource config JSON file we wrote in step 3.4, we run the following `curl` command:

```bash
curl "$FOUNDATION_URL/api/v0/staged/products/${CF_GUID}/jobs/${JOB_GUID}/resource_config" -X PUT -H "Authorization: Bearer $UAA_ACCESS_TOKEN" -d "@${RESOURCE_CONFIG_FILE}" -H "Content-Type: application/json"
```

Where `${RESOURCE_CONFIG_FILE}` is the path to the file. We expect to see an HTTP status 200 and an empty JSON object `{}` returned for each call.

## 8. Apply Changes (Deploy)

Press "Review Pending Changes" in the Operations Manager 2.3 UI, then press "Apply Changes".

At the end of this step, you will have a PAS 2.3.0 foundation, with networking
optionally provided by VMware NSX-T tile 2.3.0, where each job VM is located in
the appropriate NSX-T load balancer server pool.

## 9. Gotchas

An incident occurred where the NSX-T load balancer was unable to forward traffic
to the newly-deployed gorouters.

Rebooting one of the NSX-T Edges restored the flow of traffic from the NSX-T
load balancer to the gorouters. We are unsure of the root cause; however, since
existing load balancer pools continued to function, we suspect the Edge had
become incapable of honoring updates.

## 10. Troubleshooting

We find the Traceflow (*Tools → Traceflow*) networking tool
invaluable when debugging network failures. In the screenshot below, we examine
the `gorouter/0` VM's ability to communicate with the load balancer (IP address
10.144.15.4) on port 443 (HTTPS). In this case, we determined the gorouter's CID
using the  `bosh vms` command, but we could have just as easily determined it by
looking it up on the *Status* page of the PAS tile on Operations Manager:

{{< figure src="https://user-images.githubusercontent.com/1020675/48797866-2beead80-ecb8-11e8-8be2-ceb62b5a503c.png" >}}

## References

VM Extensions we used for our deployment:

- [Router](https://github.com/cunnie/deployments/blob/1ed754e555722df3ce5f503d4d4d27820ac0a2ad/nsx-t/gorouter-443-80-vm-extension.json)
- [Diego Brain](https://github.com/cunnie/deployments/blob/1ed754e555722df3ce5f503d4d4d27820ac0a2ad/nsx-t/ssh-proxy-vm-extension.json)
- [TCP Router](https://github.com/cunnie/deployments/blob/1ed754e555722df3ce5f503d4d4d27820ac0a2ad/nsx-t/tcp-router-vm-extension.json)

Our BASH commands we followed when we upgraded: [script](https://github.com/cunnie/deployments/blob/018c61d891a68bb78f3750b466165b00f3176817/nsx-t/scripts.md).

## Acknowledgements

Josh Gray of the PEZ Team was instrumental in discovering the behavior and
providing resources to test remediation. The BOSH vSphere CPI Team provided
invaluable support.

Bryan Kelly of Cerner provided invaluable feedback, pointing out that this
process is relevant not only to 2.2 → 2.3 upgrades but also to stemcell upgrades
and tile upgrades.

## Corrections & Updates

*2018-11-20*

Added _Gotchas_ and _Troubleshooting_ sections after suggestions from Bryan Kelly.

## Footnotes

<a id="dynamic"><sup>[or Dynamic]</sup></a> This blog post focuses on static
pools, but the procedure is identical for dynamic pools since the BOSH
vSphere CPI ["looks up the NSGroup [of the server pool] and adds the VM to the
NSGroup"](https://bosh.io/docs/vsphere-cpi/#resource-pools).

<a id="resource_config"><sup>[resource_config]</sup></a> In a prettified JSON file, our change would look like the following:

```diff
  {
    "instance_type": {
      "id": "automatic"
    },
    "instances": 2,
    "nsx_security_groups": null,
    "nsx_lbs": [],
    "additional_vm_extensions": [
+     "http_https_lb"
    ]
  }
```
