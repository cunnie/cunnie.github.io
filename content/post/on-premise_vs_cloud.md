---
title: "On-premise is Almost Four Times Cheaper * than the Cloud"
date: 2023-01-04T19:42:50-08:00
draft: false
---

_* If you don't count the amount of time spent maintaining the on-premise equipment._

### Abstract

My 48-VM (virtual machine) homelab configuration costs me approximately
**$430/month** in hardware, electricity, virtualization software, and internet,
but an equivalent configuration on AWS (Amazon Web Services) would cost
**$1,660/month** (almost four times as expensive)!

Disclosures:

- I work for VMware, which sells on-premise virtualization software (i.e.
  vSphere).
- I didn't put a dollar value on the time spent maintaining on-premise because
  I had a hard time assigning a dollar value. For one thing, I don't track how
  much time I spend maintaining my on-premise equipment. For another, I enjoy
  maintaining on-premise equipment, so it doesn't feel like work.

### Shortcomings of On-Premise

- **Time and Effort**: Before you leap into on-premise, you need to ask
  yourself the following, "Am I interested, and do I have the time, to maintain
  my own infrastructure?" If you like swapping out broken hard drives,
  troubleshooting failed power supplies, creating VLANs, building firewalls,
  configuring backups, and flashing BIOS—if you like getting your hands
  dirty—then on-premise is for you.
- **Only one IPv4 address**: This is a big drawback. Who gets the sole IPv4
  (73.189.219.4) address's HTTPS port—the Kubernetes cluster or the Cloud
  Foundry foundation? In my case, the Cloud Foundry foundation won that battle.
  On the IPv6 front there's no scarcity: Xfinity has allocated me
  2601:646:100:69f0/60 (eight /64 subnets!).
- **Poor upload speed**: Although my Xfinity download speed at 1.4 Gbps can
  rival the cloud VMs', the anemic 40 Mbps upload speed can't. I don't host
  large files on my on-premise home lab. This may not be a problem if your
  internet connection has symmetric speeds (e.g. fiber).
- **Scalability**: I can't easily scale up my home lab. For example, my 15 amp
  outlet won't support more than what it already has (2 ESXi hosts, 1 TrueNAS
  ZFS fileserver, two switches, an access point, a printer). Similarly, my
  modestly-sized San Francisco apartment's closet doesn't have room to
  accommodate additional hardware.
- **Widespread outages**: When I upgraded my TrueNAS ZFS fileserver that
  supports the VMs, I had to power-off every single VM. Only then could I
  safely upgrade the fileserver.
- **Ground-up Rebuilds**: One time I made the mistake of _not_ powering down my
  48 VMs before rebooting my fileserver, and I spent a significant portion of
  my winter break recovering corrupted VMs (re-installing vCenter, rebuilding
  my Unifi console from scratch).

### How I Calculated the AWS Costs

First, I pulled a list of my VMs and their hardware configuration (number of CPUs (cores), amount of RAM (Random Access Memory))
I used the following [`govc`](https://github.com/vmware/govmomi/blob/main/govc/README.md) command:

```bash
export GOVC_USERNAME=administrator@vsphere.local GOVC_PASSWORD='some-password' GOVC_URL=vcenter-80.nono.io
govc ls -json '/*/vm/*' | jq -r '.elements[].Object.Config
    | select(.Name == null | not)
    | {Name, "NumCPU": .Hardware["NumCPU"], "MemoryMB": .Hardware["MemoryMB"] }
    | join("\t")'
```

The output is a tab-separated file with three columns: VM Name, cores, RAM.
Here's a typical line from my Ubuntu Jammy-based VM with 4 cores and 2GiB of RAM:

```
jammy.nono.io	4	2048
```

I imported  the data into a [Google
Sheet](https://docs.google.com/spreadsheets/d/e/2PACX-1vSkzEYj22iSzGd-uSN_zK4QgVtQU0KQy4XyIBf0oPWvMhduiXfnCQZpcLeaDZs8F0mdrTtVNg61dpgZ/pubhtml?gid=1036799965&single=true),
and then found the closest available instance in the AWS region us-east-1 using
the following criteria:

- Use the closest instance type, but round down in AWS's favor (this post isn't
  a hit piece). For example, my biggest VM, *nsx.nono.io*, has 6 vCPUs and 24
  GiB RAM, but AWS doesn't offer that class of instance, so instead we drop
  down to the closest comparable instance, the _t4g.xlarge_ with 4 vCPUs and 16
  GiB of RAM.
- When there are multiple instance types which match the hardware
  configuration, use the instance type most favorable (cheapest) to AWS. As a
  result, many of the instances are *Graviton*-based. *Graviton* is AWS's
  custom implementation of the ARM (Advanced RISC Machines) architecture, and
  its instances cost less than an equivalent x86-based instance. For example,
  for a 2-vCPU 4 GiB instance, we use the *t4g.medium* instance type
  (Graviton-based, as noted by the "g") which costs $15.40 per month and which
  is 19% percent cheaper than an equivalent *t3.medium* at $19.05 per month.
- Use a 1-year Reserved Instance pricing instead of On-Demand pricing, which
  again favors AWS. For example, a *t4g.medium* with a 1-year reservation costs
  $15.40 per month which is 37% cheaper than an On-Demand which costs $24.54. I
  don't use the 3-year Reserved Instance pricing because I used it once and was
  trapped in an instance type I didn't want for two years.

| Machine Name                            | Num vCPU | RAM (MiB) | AWS Instance | 1 yr reserved instance $/mo |
|-----------------------------------------|---------:|----------:|--------------|----------------------------:|
| unifi.nono.io                           |         1|      1024 | t2.micro     |                        5.26 |
| vm-7f223465-fee6-4cce-bd36-e208ab134eef |         1|      1024 | t2.micro     |                        5.26 |
| vm-b2dff011-7c0a-4b9a-8c40-d6c6b8a806e6 |         1|      1024 | t2.micro     |                        5.26 |
| vm-c685c9f0-b221-4369-b021-11657db9d242 |         1|      1024 | t2.micro     |                        5.26 |
| cc-worker_cf_4565d3c15290               |         1|      4096 | m6g.medium   |                        17.59|
| controller-0.nono.io                    |         1|      4096 | m6g.medium   |                        17.59|
| controller-1.nono.io                    |         1|      4096 | m6g.medium   |                        17.59|
| controller-2.nono.io                    |         1|      4096 | m6g.medium   |                        17.59|
| credhub_cf_cb824be566b2                 |         1|      4096 | m6g.medium   |                        17.59|
| doppler_cf_9d4b65e4b9fa                 |         1|      4096 | m6g.medium   |                        17.59|
| log-api_cf_6f75cfc1d54d                 |         1|      4096 | m6g.medium   |                        17.59|
| nats_cf_f5ba1300f093                    |         1|      4096 | m6g.medium   |                        17.59|
| scheduler_cf_6b106f0f539a               |         1|      4096 | m6g.medium   |                        17.59|
| tcp-router_cf_6a78efed748e              |         1|      4096 | m6g.medium   |                        17.59|
| worker-0.nono.io                        |         1|      4096 | m6g.medium   |                        17.59|
| worker-1.nono.io                        |         1|      4096 | m6g.medium   |                        17.59|
| worker-2.nono.io                        |         1|      4096 | m6g.medium   |                        17.59|
| om.tas.nono.io                          |         1|      8192 | r6g.medium   |                        23.21|
| api_cf_19bd0834f739                     |         2|      2048 | t4g.small    |                        7.67 |
| database_cf_740768a383a0                |         2|      2048 | t4g.small    |                        7.67 |
| diego-api_cf_645d9f6fed02               |         2|      2048 | t4g.small    |                        7.67 |
| singleton-blobstore_cf_469b8edcd414     |         2|      2048 | t4g.small    |                        7.67 |
| uaa_cf_358f9461cbf2                     |         2|      2048 | t4g.small    |                        7.67 |
| vm-075fc31a-db73-42fc-9af0-b4755e24b4df |         2|      4096 | t4g.medium   |                        15.40|
| vm-1a2fef42-b2ba-4714-a1c6-6fe856a1dde6 |         2|      8192 | t4g.large    |                        30.73|
| vm-a5887419-c158-4a2e-afca-7dfc59f63a3e |         2|      8192 | t4g.large    |                        30.73|
| vm-e19d536a-8d38-48dc-a57b-8f8eeaa664a1 |         2|      8192 | t4g.large    |                        30.73|
| jammy.nono.io                           |         4|      2048 | t4g.small    |                        7.67 |
| haproxy_cf_239724625ef9                 |         4|      4096 | t4g.medium   |                        15.40|
| minikube.nono.io                        |         4|      4096 | t4g.medium   |                        15.40|
| router_cf_6d1ae08baa9b                  |         4|      4096 | t4g.medium   |                        15.40|
| router_cf_d060cf838699                  |         4|      4096 | t4g.medium   |                        15.40|
| edge-0                                  |         4|      8192 | c6g.xlarge   |                        62.56|
| edge-1                                  |         4|      8192 | c6g.xlarge   |                        62.56|
| worker_sslipio_7308d8fd0554             |         4|      8192 | c6g.xlarge   |                        62.56|
| diego-cell_cf_2f95c43dcdc1              |         4|      16384| t4g.xlarge   |                        61.54|
| diego-cell_cf_7e3d791ed94c              |         4|      16384| t4g.xlarge   |                        61.54|
| fed.nono.io                             |         4|      16384| t4g.xlarge   |                        61.54|
| isolated-diego-cell_cf_c3cd3c843ee8     |         4|      16384| t4g.xlarge   |                        61.54|
| log-cache_cf_2af63a960266               |         4|      16384| t4g.xlarge   |                        61.54|
| vm-317535ae-3699-4b66-b679-cfa26ecaee4d |         4|      16384| t4g.xlarge   |                        61.54|
| vm-d3143c15-9cf2-45e7-b08c-e6bea0f6065d |         4|      16384| t4g.xlarge   |                        61.54|
| windows2019-cell_cf_61eda5c3d90d        |         4|      16384| t4g.xlarge   |                        61.54|
| windows2019-cell_cf_d0340ccccfb3        |         4|      16384| t4g.xlarge   |                        61.54|
| nesxi-0.nono.io                         |         4|      32768| r6g.xlarge   |                        92.71|
| nesxi-1.nono.io                         |         4|      32768| r6g.xlarge   |                        92.71|
| nesxi-2.nono.io                         |         4|      32768| r6g.xlarge   |                        92.71|
| nesxi-template                          |         4|      32768| r6g.xlarge   |                        92.71|
| nsx.nono.io                             |         6|      24576| t4g.xlarge   |                        61.54|
|                                         |          |           |              |                             |
| TOTAL                                   |          |           |              |                     1,662.05|

### How I Calculated the On-Premise Monthly Cost

I added up the hardware I've purchased, then divided by the number of months it
has been in use to determine its monthly cost. Admittedly this is not a
GAAP-approved (generally accepted accounting principles) method of assigning a
monthly cost, but I'm an engineer not an accountant.

| Hardware                                                            | Date Purchased | One-time Cost | $/Month |
|---------------------------------------------------------------------|----------------|---------------|---------|
| Fileserver: iSCSI TrueNAS server custom build                       | 11/12/2014     | 2,434.50      | 24.58   |
| ESXi host: Supermicro X10SDV-8C-TLNF4+ Intel Xeon D-1537 128 GiB    | 9/14/2017      | 2,287.68      | 35.20   |
| ESXi host: Supermicro X10SDV-8C-TLNF4+ Intel Xeon D-1537 128 GiB    | 9/26/2017      | 2,285.13      | 35.38   |
| Network switch: Qnap QSW-1208-8C-US 12-Port unmanaged 10 Gbe Switch | 9/27/2018      | 828.07        | 15.75   |
| ESXi host: Supermicro X11SDV-8C+-TLN2F-B Intel Xeon D-2141 256 GiB  | 3/27/2019      | 2,430.52      | 52.13   |
| Network Switch: Ubiquiti 24-port Gbe + 2 x 10 Gbe uplink            | 7/10/2019      | 649.92        | 15.05   |
| Network cable modem: ARRIS SURFboard S33                            | 10/14/2021     | 184.64        | 11.54   |
| Network Firewall: Supermicro A2SDi-H-TF-B                           | 11/25/2021     | 1,063.11      | 72.71   |
| Network Switch: Ubiquiti Switch Flex XG                             | 10/15/2021     | 324.79        | 20.34   |
| Electricity: assume half of latest bill, 117.24                     |                |               | 58.62   |
| Internet (Comcast $70/mo 1.2Gpbs/40Mbps)                            |                |               | 70.00   |
| Virtualization Software (VMUG advantage, $200/yr)                   |                |               | 16.67   |
|                                                                     |                |               |         |
| TOTAL                                                               |                |               | 427.98  |


### Why I didn't Include the Cost of Storage (Disks)

The costs of disk was overshadowed by the cost of cores and RAM, and I was
loath to include another metric. For example, a *t4g.medium* instance costs
$15.40/month, and adding a 30 GiB gp3 EBS (Elastic Block Store) adds a
paltry $2.40 (15% increase).

This decision to overlook the cost of disk plays in AWS's favor.

### "Why Do You Hate the Cloud?"

I don't hate the cloud—I love the cloud! I maintain personal servers on AWS,
Azure, Digital Ocean, Google Cloud Platform (GCP) (a GKE cluster!), and
Hetzner, and I've had them for years. And this is a good excuse to list them
all:

| Name            | IaaS (Infrastructure as a Service)  | Purpose             |
|-----------------|-------------------------------------|---------------------|
| ns-aws          | AWS (Ubuntu, dual-stack)            | DNS, NTP, web       |
| ns-azure        | Azure (Ubuntu)                      | DNS, NTP, web       |
| ns-gce          | GCP (GKE, kubernetes)               | DNS, NTP, Vault, CI |
| nono.io         | Hetzner (FreeBSD, dual-stack)       | DNS, NTP            |
| ns-digitalocean | Digital Ocean (FreeBSD, dual-stack) | DNS, NTP            |
