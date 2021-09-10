---
title: "Disk Controller Benchmarks: VMware Paravirtual's vs. LSI Logic's"
date: 2021-09-08T10:45:28-07:00
draft: true
---

Our benchmarks show that VMware's Paravirtual SCSI (small computer system
interface) controller offered a 2-3% performance increase in IOPS (I/O
(input/output) operations per second) over the LSI Logic Parallel SCSI
controller at the cost of a similar decrease in sequential performance (both
read & write). Additionally the Paravirtual SCSI controller (pvscsi) had a
slight reduction in CPU (central processing unit) usage on the host best-case
scenario is a 3% lower CPU usage).

{{< figure src="https://docs.google.com/spreadsheets/d/e/2PACX-1vTddYLAn6UFpesWIPH5S6ptr9sm3ECcHxf5aYobpfKqT1pdp8IyTZu4D9yV7SOmwQEVkhgwpy5xnlUW/pubchart?oid=881167435&format=image" alt="IOPS" caption="On average the Paravirtual Driver's IOPS performance is 2.5% better than the LSI Logic's">}}

{{< figure src="https://docs.google.com/spreadsheets/d/e/2PACX-1vTddYLAn6UFpesWIPH5S6ptr9sm3ECcHxf5aYobpfKqT1pdp8IyTZu4D9yV7SOmwQEVkhgwpy5xnlUW/pubchart?oid=941146114&format=image" alt="Sequential Read Throughput" caption="Although not consistently faster, the LSI Logic averages 2% faster than the Paravirtual on sequential read operations">}}

{{< figure src="https://docs.google.com/spreadsheets/d/e/2PACX-1vTddYLAn6UFpesWIPH5S6ptr9sm3ECcHxf5aYobpfKqT1pdp8IyTZu4D9yV7SOmwQEVkhgwpy5xnlUW/pubchart?oid=2035776217&format=image" alt="Sequential Write Throughput" caption="The LSI Logic's sequential write performance is consistently faster than the Paravirtual's by an average of 4%">}}

#### Host CPU Utilization

{{< figure src="https://user-images.githubusercontent.com/1020675/132873236-1d446a53-6e7b-49bb-996a-2fd016c4f7d9.png" alt="LSI Logic Host CPU Utilization" >}}

{{< figure src="https://user-images.githubusercontent.com/1020675/132873272-08d29b13-b1ba-4401-8c3d-94d087b421f3.png" alt="VMware Paravirtual Host CPU Utilization" >}}

{{< figure src="https://user-images.githubusercontent.com/1020675/132879128-cc093daa-18b5-4a81-8085-aa5b721a4ddd.jpg" alt="Samsung NVMe SSD 960 Pro" >}}

### Why These Benchmarks Are Flawed

- **We only tested one type of datastore**. We tested an NVMe (non-volatile
  memory express) datastore, but it would have been interesting to test VMware's
  vSAN (virtual storage area network).

- **We only ran the benchmark for an hour**. Had we run the benchmark longer, we
  would have seen different performance curves. For example, when we ran the
  benchmarks back-to-back we saw degradation of the throughput from almost 2
  GB/s to slightly more than 1 GB/s on the sequential read & write tests. We
  attribute this degradation to saturation of the Samsung controller.

- **We never modified the kernel settings to enhance pvscsi performance**. For
  example, went with the default settings for queue depth for device (64) and
  adapter (254), but this VMware [knowledgebase (KB)
  article](https://kb.vmware.com/s/article/2053145) suggests increasing those to
  254 and 1024, respectively.

- **We only tested Linux**. Windows performance may be different.

- **We benchmarked slightly different versions of Linux**. We tested against two
  very-close versions of Ubuntu Bionic, but they weren't the _exact_ same
  version. Had we used completely identical versions, we may have seen different
  performance numbers.

### References

- _[Which vSCSI controller should I choose for
  performance?](https://blogs.vmware.com/vsphere/2014/02/vscsi-controller-choose-performance.html),
  Mark Achetemichuk,_ "PVSCSI and LSI Logic Parallel/SAS are essentially the
  same when it comes to overall performance capability [for customers not
  producing 1 million IOPS]"

- _[Achieving a Million I/O Operations per Second from a Single VMware vSphere®
  5.0
  Host](https://www.vmware.com/content/dam/digitalmarketing/vmware/en/pdf/techpaper/1M-iops-perf-vsphere5.pdf)_,
  "a PVSCSI adapter provides 8% better throughput at 10% lower CPU cost". Our
  benchmarks show 2.5%, 3% respectively.

- _[Large-scale workloads with intensive I/O patterns might require queue depths
  significantly greater than Paravirtual SCSI default values
  (2053145)](https://kb.vmware.com/s/article/2053145)_, "... increase PVSCSI
  queue depths to 254 (for device) and 1024 (for adapter)"
