---
title: "Disk Controller Benchmarks: VMware Paravirtual's vs. LSI Logic Parallel's"
date: 2021-11-19T08:12:28-07:00
draft: false
---

Is it worth switching your VMware vSphere VM's SCSI (small computer system
interface) from the LSI Logic Parallel controller to the VMware Paravirtual SCSI
controller? Except for ultra-high-end database servers (> 1M IOPS ( input/output
operations per second)), the answer is "no"; the difference is negligible.

Our benchmarks show that VMware's Paravirtual SCSI (small computer system
interface) controller offered a 2-3% performance increase in IOPS (I/O
(input/output) operations per second) over the LSI Logic Parallel SCSI
controller at the cost of a similar decrease in sequential performance (both
read & write). Additionally the Paravirtual SCSI controller (pvscsi) had a
slight reduction in CPU (central processing unit) usage on the host (best-case
scenario is 3% lower CPU usage).

### The Benchmarks

The Paravirtual has better IOPS than the LSI Logic, but worse sequential
throughput.

{{< figure src="https://docs.google.com/spreadsheets/d/e/2PACX-1vTddYLAn6UFpesWIPH5S6ptr9sm3ECcHxf5aYobpfKqT1pdp8IyTZu4D9yV7SOmwQEVkhgwpy5xnlUW/pubchart?oid=881167435&format=image" alt="IOPS" caption="On average the Paravirtual Driver's (red) IOPS performance is 2.5% better than the LSI Logic's">}}

{{< figure src="https://docs.google.com/spreadsheets/d/e/2PACX-1vTddYLAn6UFpesWIPH5S6ptr9sm3ECcHxf5aYobpfKqT1pdp8IyTZu4D9yV7SOmwQEVkhgwpy5xnlUW/pubchart?oid=941146114&format=image" alt="Sequential Read Throughput" caption="Although not consistently faster, the LSI Logic averages 2% faster than the Paravirtual on sequential read operations">}}

{{< figure src="https://docs.google.com/spreadsheets/d/e/2PACX-1vTddYLAn6UFpesWIPH5S6ptr9sm3ECcHxf5aYobpfKqT1pdp8IyTZu4D9yV7SOmwQEVkhgwpy5xnlUW/pubchart?oid=2035776217&format=image" alt="Sequential Write Throughput" caption="The LSI Logic's sequential write performance is consistently faster than the Paravirtual's by an average of 4%">}}

#### Host CPU Utilization

The ESXi host CPU utilization was trickier to measure—we had to eyeball it. You
can see the two charts below, taken while we were running our benchmarks. Our
guess is that the Paravirtual driver used ~3% less CPU than the LSI Logic.

Note that 3% is a best-case scenario (you're unlikely to get 10% improvement):
the benchmarks were taken on what we affectionately refer to as "The world's
slowest Xeon", i.e. the [Intel Xeon
D-1537](https://ark.intel.com/content/www/us/en/ark/products/91196/intel-xeon-processor-d-1537-12m-cache-1-70-ghz.html),
which clocks in at an anemic 1.7GHz (side note: We purchased it for its low TDP
(Thermal Design Power), not its speed). In other words, this processor is so
slow that any improvement in CPU efficiency is readily apparent.

{{< figure src="https://user-images.githubusercontent.com/1020675/132873236-1d446a53-6e7b-49bb-996a-2fd016c4f7d9.png" alt="LSI Logic Host CPU Utilization (lower is better)" >}}

{{< figure src="https://user-images.githubusercontent.com/1020675/132873272-08d29b13-b1ba-4401-8c3d-94d087b421f3.png" alt="VMware Paravirtual Host CPU Utilization (lower is better)" >}}

### Benchmark Setup

We paired a slow CPU with a fast disk:

- [Samsung SSD 960 2TB M.2 2280
PRO](http://www.samsung.com/semiconductor/minisite/ssd/product/consumer/ssd960/)
- [Supermicro
X10SDV-8C-TLN4F+](http://www.supermicro.com/products/motherboard/Xeon/D/X10SDV-8C-TLN4F_.cfm)
motherboard with a soldered-on 1.7 GHz 8-core [Intel Xeon Processor
D-1537](https://ark.intel.com/products/91196/Intel-Xeon-Processor-D-1537-12M-Cache-1_70-GHz),
and 128 GiB RAM.

We used [gobonniego](https://github.com/cunnie/gobonniego/) to benchmark the
performance.

We ran each benchmark for [1
hour](https://github.com/cunnie/deployments/blob/f3e771bbe83d9858675b2d9bf86f69b917489e34/gobonniego/pvscsi.yml#L20).

We configured the VMs with [4 vCPUs, 1 GiB
RAM](https://github.com/cunnie/deployments/blob/f3e771bbe83d9858675b2d9bf86f69b917489e34/gobonniego/vsphere-cc.yml#L14-L16)
and a [20 GiB
drive](https://github.com/cunnie/deployments/blob/f3e771bbe83d9858675b2d9bf86f69b917489e34/gobonniego/vsphere-cc.yml#L7).
We used 4 CPUs so that the Xeon's slow speed wouldn't handicap the benchmark
results (we wanted the disk to be the bottleneck, not the CPU).

{{< figure src="https://user-images.githubusercontent.com/1020675/132879128-cc093daa-18b5-4a81-8085-aa5b721a4ddd.jpg" alt="Samsung NVMe SSD 960 Pro" caption="The Samsung NVMe SSD 960 Pro is a gold standard for NVMe disk performance. ">}}

### Why These Benchmarks Are Flawed

In the spirit of full disclosure, we'd like to point out the shortcomings of our
benchmarks:

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

- [Raw Benchmark
  results](https://github.com/cunnie/cloud_storage_benchmarks/tree/78af7a9e1ec90438f75745293e788761d3e16816/pvscsi)
  (JSON-formatted).
