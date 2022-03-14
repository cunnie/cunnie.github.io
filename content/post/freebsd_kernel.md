---
title: "How to Customize NIC Drivers in the FreeBSD Kernel"
date: 2022-03-13T14:38:00-07:00
draft: true
images:
- https://freebsdfoundation.org/wp-content/uploads/2020/03/event-slider-freebsd-logo-cropped.jpg
---

{{< figure src="https://freebsdfoundation.org/wp-content/uploads/2020/03/event-slider-freebsd-logo-cropped.jpg" alt="FreeBSD logo" >}}

In this blog post we build a custom FreeBSD kernel which uses an updated Intel
X550 NIC (Network Interface Controller) driver (i.e.
[`ixgbe`](https://www.freebsd.org/cgi/man.cgi?query=ixgbe&sektion=4)) which
enables NBASE-T speeds for our Intel X550-based motherboard.

#### Background: Hoping to Break the 1Gbe Barrier

We subscribe to Xfinity's [Gigabit internet](https://www.xfinity.com/gig) plan,
which offers "Download speeds up to 1200 Mbps". Finally we can break the 1Gbe
barrier!

We use an [Arris SURFboard
S33](https://www.surfboard.com/products/cable-modems/s33/), the second piece of
the puzzle, which features a 2.5 Gigabit ethernet port. Now we can break the
1Gbe barrier all the way to our firewall.

Our firewall has a [Supermicro
A2SDi-H-TF](https://www.supermicro.com/en/products/motherboard/A2SDi-H-TF?locale=en),
which features 2 x 10GBaseT network interfaces. We run FreeBSD 13 on our
firewall.

#### Disappointment!

We quickly discovered that our firewall was negotiating the wrong speed with our
cable modem: it negotiated 1Gbe, not 2.5Gbe. What was amiss? We checked our
interface.

```
ifconfig -m ix0
ix0: flags=8863<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
	options=4e53fbb<RXCSUM,TXCSUM,VLAN_MTU,VLAN_HWTAGGING,JUMBO_MTU,VLAN_HWCSUM,TSO4,TSO6,LRO,WOL_UCAST,WOL_MCAST,WOL_MAGIC,VLAN_HWFILTER,VLAN_HWTSO,RXCSUM_IPV6,TXCSUM_IPV6,NOMAP>
	capabilities=4f53fbb<RXCSUM,TXCSUM,VLAN_MTU,VLAN_HWTAGGING,JUMBO_MTU,VLAN_HWCSUM,TSO4,TSO6,LRO,WOL_UCAST,WOL_MCAST,WOL_MAGIC,VLAN_HWFILTER,VLAN_HWTSO,NETMAP,RXCSUM_IPV6,TXCSUM_IPV6,NOMAP>
	ether 00:0d:b9:48:92:48
	hwaddr 3c:ec:ef:5f:f3:90
	inet6 fe80::20d:b9ff:fe48:9248%ix0 prefixlen 64 scopeid 0x1
	inet6 2001:558:6045:109:892f:2df3:15e3:3184 prefixlen 128
	inet 73.189.219.4 netmask 0xfffffe00 broadcast 255.255.255.255
	media: Ethernet autoselect (1000baseT <full-duplex,rxpause,txpause>)
	status: active
	supported media:
		media autoselect
		media 1000baseT
		media 10Gbase-T
	nd6 options=23<PERFORMNUD,ACCEPT_RTADV,AUTO_LINKLOCAL>
```

Aha! We see the problem: our interface has only two supported media: `1000baseT`
(1Gbe) and `10Gbase-T` (10Gbe). The NBASE-T speeds (2.5Gbe and 5Gbe) are
missing.

By the way, we weren't the only ones disappointed that the NBASE-T speeds were
missing: the Netgate forums featured wails of unhappiness stretching back two
years.

To double-check that we're really, truly, limited to 1Gbe, we run [Speedtest
CLI](https://www.speedtest.net/apps/cli):

```text
   Speedtest by Ookla

     Server: Razzolink Inc - San Jose, CA (id = 24934)
        ISP: Comcast Cable
    Latency:     9.22 ms   (2.34 ms jitter)
   Download:   940.76 Mbps (data used: 1.0 GB )
     Upload:    42.65 Mbps (data used: 76.0 MB )
Packet Loss:     0.0%
 Result URL: https://www.speedtest.net/result/c/1d4ea5ae-0ec4-4d0e-ab84-e72ba4b545d0
```

The results confirmed our worst fears: the download speed of 940.76 Mbps means
we're definitely limited to 1Gbe.

#### Intel to the Rescue!

Nine days ago Intel patched their `ixgbe` driver to include the NBASE-T speeds,
but that patch will take a while to work its way to the stable release that
we're using, and we don't want to wait. So what to do? We're gonna build a
custom kernel with the updated driver.

#### Building the Kernel

The FreeBSD Handbook has great instructions on _[Configuring the FreeBSD
Kernel](https://docs.freebsd.org/en/books/handbook/kernelconfig/)_ and _[Using
Git](https://docs.freebsd.org/en/books/handbook/mirrors/index.html#git)_ to
check out the kernel source. We're shamelessly copying their instructions.

First, let's check out the FreeBSD sources. This took almost ten minutes:

```shell
sudo git clone -o freebsd https://git.FreeBSD.org/src.git /usr/src
```

Great, we've checked out the kernel's main branch, which has the code for
FreeBSD
[CURRENT](https://docs.freebsd.org/en/books/handbook/cutting-edge/#current).
But that's not what we want—we want the stable/13 branch.

```shell
cd /usr/src
sudo git switch stable/13
cd /usr/src/sys/amd64/conf
sudo cp GENERIC NBASE-T
cd /usr/src
```

Before we compile, we make the necessary changes to
`/usr/src/sys/dev/ixgbe/ixgbe_x550.c`:

```diff
--- a/sys/dev/ixgbe/ixgbe_x550.c
+++ b/sys/dev/ixgbe/ixgbe_x550.c
@@ -1945,6 +1945,8 @@ s32 ixgbe_get_link_capabilities_X550em(struct ixgbe_hw *hw,
                        /* fall through */
                default:
                        *speed = IXGBE_LINK_SPEED_10GB_FULL |
+                                IXGBE_LINK_SPEED_2_5GB_FULL |
+                                IXGBE_LINK_SPEED_5GB_FULL |
                                 IXGBE_LINK_SPEED_1GB_FULL;
                        break;
                }
```

Let's compile:

```
sudo make -j 8 buildkernel KERNCONF=NBASE-T
sudo make installkernel    KERNCONF=NBASE-T
sudo shutdown -r now
```

#### Finding the precise hardware

Let's use `dmesg` to find the precise hardware we're using:

```
ix0: <Intel(R) X553/X557-AT (10GBASE-T)> mem 0xdfc00000-0xdfdfffff,0xdfe04000-0xdfe07fff at device 0.0 on pci5
```

Now let's use `dmidecode`:

```
Reference Designation: Intel Ethernet X557-AT2 #1
```

And finally, let's use `sudo pciconf -lvc`:

```
ix0@pci0:5:0:0:	class=0x020000 rev=0x11 hdr=0x00 vendor=0x8086 device=0x15c8 subvendor=0x15d9 subdevice=0x15c8
    vendor     = 'Intel Corporation'
    device     = 'Ethernet Connection X553/X557-AT 10GBASE-T'
```

The device id, `0x15c8`, is important when browsing the device code.

### References

- Piotr Pietruszewski's commit which adds "[control of 2.5/5G autonegotiation
  speeds](https://cgit.freebsd.org/src/commit/sys/dev/ixgbe/ixgbe.h?id=d381c807510de2ebb453a563540bd17e344a2aab)" to FreeBSD's ixgbe NIC driver
- netgate forum post [bemoaning the lack of NBASE-T Support for Intel
  X550](https://forum.netgate.com/topic/146913/nbase-t-support-for-intel-x550/27)
