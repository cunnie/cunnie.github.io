---
title: "For Sale"
date: 2025-06-29T06:40:15-07:00
draft: true
---

## NAS (TrueNAS) Server 8-core, 128 GiB ECC, 2 x 10 Gbe SFP+, 30TB Raw

- [Intel Xeon D-1537 8-Core 1.7GHz 35W](https://www.intel.com/content/www/us/en/products/sku/91196/intel-xeon-processor-d1537-12m-cache-1-70-ghz/specifications.html)
- [Supermicro X10SDV-8C-TLN4F+ Mini-ITX](https://www.supermicro.com/products/motherboard/Xeon/D/X10SDV-8C-TLN4F_.cfm)
  - 2 x 10 Gbe **SFP+** connectors, cables and SFP+ included.
  - IPMI (no monitor, keyboard needed)
- 128 GiB ECC (4 × D760R Samsung DDR4-2666 32GB ECC Registered)
- [LSI SAS 9211-8i 6Gb/s SAS Host Bus Adapter](https://docs.broadcom.com/docs/12352062)
- 7 x Seagate 4TB NAS HDD ST4000VN000
- 1 x spare Seagate 4TB IronWolf ST4000VN006
- 1 x 128 GiB SATA DOM for OS
- [Samsung SSD 960 PRO 2TB NVMe m.2 2280](https://semiconductor.samsung.com/consumer-storage/internal-ssd/960pro/)
- [LIAN LI PC-Q25B Black Aluminum Mini-ITX Tower Computer Case](https://www.newegg.com/lian-li-mini-itx-tower-aluminum-computer-case-black-pc-q25b/p/N82E16811112339?srsltid=AfmBOopZj5lUxwUrnKqfw8I3sazkJ-GfzBwvriCTJyyzbcK2N-HZbuRx)

## ESXi Server, 8-core, 256 GiB ECC, 2 x 10GBase-T, 2TB

- [Intel Xeon D-2141I, 8-Core, 16 Threads, 65W](https://www.intel.com/content/www/us/en/products/sku/136430/intel-xeon-d2141i-processor-11m-cache-2-20-ghz/specifications.html)
- [Supermicro X11SDV-8C+-TLN2F+](https://www.supermicro.com/en/products/motherboard/X11SDV-8C+-TLN2F)
  - 2 x 10GBase-T
  - IPMI (no monitor, keyboard needed)
- 4 x 64GB (256GB total) DDR4 2133 ECC LRDIMM
- Samsung SSD 870 EVO 2TB m.2 NVMe 2280
- [Fractal Design Core 500](https://www.fractal-design.com/products/cases/core/core-500/black/)

## [NVIDIA T4 GPU](https://www.nvidia.com/en-us/data-center/tesla-t4/)

- 16 GiB
- 70W
- small PCIe form factor
- Noctua PWM fan & shroud included

```
nvidia-smi
Sun Jun 29 07:37:05 2025
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.230.02             Driver Version: 535.230.02   CUDA Version: 12.2     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  Tesla T4                       Off | 00000000:01:00.0 Off |                    0 |
| N/A   71C    P8              12W /  70W |      2MiB / 15360MiB |      0%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+

+---------------------------------------------------------------------------------------+
| Processes:                                                                            |
|  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
|        ID   ID                                                             Usage      |
|=======================================================================================|
|  No running processes found                                                           |
+---------------------------------------------------------------------------------------+
```
