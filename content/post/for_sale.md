---
title: "For Sale"
date: 2025-06-29T06:40:15-07:00
draft: true
---

## NAS (TrueNAS) Server 8-core, 128 GiB ECC, 2 x 10 Gbe SFP+, 30TB Raw

- [Intel Xeon D-1537 8-Core 1.7GHz 35W](https://www.intel.com/content/www/us/en/products/sku/91196/intel-xeon-processor-d1537-12m-cache-1-70-ghz/specifications.html)
- [Supermicro X10SDV-8C-TLN4F+ Mini-ITX](https://www.supermicro.com/products/motherboard/Xeon/D/X10SDV-8C-TLN4F_.cfm)
  - 2 x 10 Gbe **SFP+** connectors, cables and SFP+ included.
    - MAC Address of first 10Gbe: `ac:1f:6b:2d:15:92`
  - IPMI (no monitor, keyboard needed)
- 128 GiB ECC (4 × D760R Samsung DDR4-2666 32GB ECC Registered)
- [LSI SAS 9211-8i 6Gb/s SAS Host Bus Adapter](https://docs.broadcom.com/docs/12352062)
- 7 x Seagate 4TB NAS HDD ST4000VN000
- 1 x spare Seagate 4TB IronWolf ST4000VN006
- 1 x 128 GiB SATA DOM for OS
- [Samsung SSD 960 PRO 2TB NVMe m.2 2280](https://semiconductor.samsung.com/consumer-storage/internal-ssd/960pro/)
- [LIAN LI PC-Q25B Black Aluminum Mini-ITX Tower Computer Case](https://www.newegg.com/lian-li-mini-itx-tower-aluminum-computer-case-black-pc-q25b/p/N82E16811112339?srsltid=AfmBOopZj5lUxwUrnKqfw8I3sazkJ-GfzBwvriCTJyyzbcK2N-HZbuRx)
- 20cm x 28cm x 37cm (20.7 liters)
- 22.8 lbs.

Here is the `zpool status` output before I factory-reset the machine:

```
  pool: boot-pool
 state: ONLINE
  scan: scrub repaired 0B in 00:00:16 with 0 errors on Fri Jul  4 03:45:16 2025
config:

	NAME        STATE     READ WRITE CKSUM
	boot-pool   ONLINE       0     0     0
	  ada0p2    ONLINE       0     0     0

errors: No known data errors

  pool: tank
 state: ONLINE
  scan: scrub repaired 0B in 04:04:38 with 0 errors on Sun Jun 29 04:04:39 2025
config:

	NAME                                            STATE     READ WRITE CKSUM
	tank                                            ONLINE       0     0     0
	  raidz2-0                                      ONLINE       0     0     0
	    gptid/bf28cbf7-0d07-11eb-909e-ac1f6b2d1592  ONLINE       0     0     0
	    gptid/793594b6-4ca5-11e4-b3bd-002590f5182a  ONLINE       0     0     0
	    da6                                         ONLINE       0     0     0
	    da1                                         ONLINE       0     0     0
	    gptid/7a96a5d5-4ca5-11e4-b3bd-002590f5182a  ONLINE       0     0     0
	    gptid/7b04562f-4ca5-11e4-b3bd-002590f5182a  ONLINE       0     0     0
	    gptid/7b72feca-4ca5-11e4-b3bd-002590f5182a  ONLINE       0     0     0

errors: No known data errors
```

## ESXi Server, 8-core, 256 GiB ECC, 2 x 10GBase-T, 2TB

- [Intel Xeon D-2141I, 8-Core, 16 Threads, 65W](https://www.intel.com/content/www/us/en/products/sku/136430/intel-xeon-d2141i-processor-11m-cache-2-20-ghz/specifications.html)
- [Supermicro X11SDV-8C+-TLN2F+](https://www.supermicro.com/en/products/motherboard/X11SDV-8C+-TLN2F)
  - 2 x 10GBase-T
  - IPMI / iDRAC / iLO (no monitor, keyboard needed)
- 4 x 64GB (256GB total) DDR4 2133 ECC LRDIMM
- Samsung SSD 870 EVO 2TB SATA SSD
- [Fractal Design Core 500](https://www.fractal-design.com/products/cases/core/core-500/black/)
- 25cm x 22cm x 39cm (21.5 liters)
- 14.5 lbs.

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

## QNAP QSW-1208-8C

- 12-port 10GbE unmanaged switch
- Has 12 SFP+ ports, 8 10Base-T ports
- 10 Gbe ports are paired with SFP+ ports, and you can only use one of the pair at a time
- The 8 10Base-T ports autonegotiate 10G / 5G / 2.5G / 1G
- I've used the 10Base-T ports at 10G / 5G / 2.5G without a hitch.
