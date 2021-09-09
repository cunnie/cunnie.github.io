---
title: "Disk Controller Benchmarks: VMware Paravirtual's vs. LSI Logic's"
date: 2021-09-08T10:45:28-07:00
draft: true
---


### Why These Benchmarks Are Flawed

- **We only ran the benchmark for an hour**. Had we run the benchmark longer, we
  would have seen different performance curves. For example, when we ran the
  benchmarks back-to-back we saw degradation of the throughput from almost 2
  GB/s to slightly more than 1 GB/s on the sequential read & write tests. We
  attribute this degradation to saturation of the Samsung controller.

- **We never modified the kernel settings to enhance performance**. For example,
  went with the default settings for queue depth.

- **We benchmarked slightly different versions of Linux**. We tested against two
  very-close versions of Ubuntu Bionic, but they weren't the _exact_ same
  version. Had we used completely identical versions, we may have seen different
  performance numbers.

- **We only tested Linux**. Windows performance may be different.

- **We only tested one type of datastore**. We tested an NVMe (non-volatile
  memory express) datastore, but it would have been interesting to test VMware's
  vSAN (virtual storage area network).
