---
title: How I Became an Ethereum Validator
date: 2022-09-18T15:38:28-07:00
draft: true
images:
- https://ethereum.org/static/6b935ac0e6194247347855dc3d328e83/13c43/eth-diamond-black.png
---

### Summary

<!--
{{< figure src="https://ethereum.org/static/6b935ac0e6194247347855dc3d328e83/13c43/eth-diamond-black.png" width="50%" height="50%" alt="Ethereum logo" >}}
-->

This blog posts describes the steps I took to become an ETH "solo home staker"
(an Ethereum validator). I wanted to participate in Ethereum's proof-of-stake
validation of its blockchain for no other reason than it would be "cool". Sure,
there are rewards, ("[Earn fresh ETH!](https://ethereum.org/en/staking/solo/)"),
but that's not what drove my interest.

"[Ethereum is a decentralized, open-source blockchain with smart contract
functionality](https://en.wikipedia.org/wiki/Ethereum)". Ether (ETH) is its
[cryptocurrency](https://www.coinbase.com/price/ethereum).

### The <s>Obstacles</s> Requirements

Erigon's GitHub's README gives a detailed set of
[requirements](https://github.com/ledgerwatch/erigon/#system-requirements):

- 2 TB disk. Technically, the Mainnet Full node only [requires
  400GB](https://github.com/ledgerwatch/erigon/#system-requirements) of disk,
  but the Erigon docs suggest 2 TB.
- &gt;= 16GiB RAM

The Ethereum site adds [important info](https://ethereum.org/en/staking/):

- "a dedicated computer connected to the internet ~24/7."
- "at least 32 ETH" (currently valued at ~$43k according to
  [Coinbase](https://www.coinbase.com/price/ethereum))
- "[You should anticipate your ETH being locked for _at least_ one-to-two
  years](https://ethereum.org/en/staking/solo/)"

### FreeBSD Jail

When I saw the hardware requirements, I realized that my initial thought of a
Raspberry Pi 4 was woefully inadequate, and even running it on my vSphere ESXi
server would would exact a heavy RAM & disk toll. I was loath to purchase an
additional computer solely for the purpose of ETH validation—when it comes to
hardware, my motto is "less is more", and maintaining my existing flotilla of seven
bare-metal machines is enough work without adding an eighth.

My final solution? Run it in a FreeBSD jail on my TrueNAS fileserver. That's
right, I have a 12 TB TrueNAS server with 128 GiB RAM, which is more than enough
RAM and disk.

FreeBSD's Jails are analogous to Linux's containers (Docker): a separate
filesystem sharing the host VM's kernel (yes, I know that's a simplified
explanation, glossing over nuances such as namespaces and cgroups, but this post
is about becoming a validator, not a deep-dive into Linux's container internals)

True confession: I've never set up a jail in my life, which means I'm going to
use the TrueNAS web interface rather than CLI; I'm not inclined to prove my
manhood by wrestling with the CLI just yet.

#### Creating the Jail

Creating the jail was easy: From the TrueNAS sidebar menu, **Jails** → **Add**

- Name: **validator.nono.io**. I like using the fully-qualified domain name
  (FQDN) for my machines. When you have more than ten devices on your network,
  there's no better way to organize than with the FQDN.
- Release **13.1-RELEASE**, the most recent available release.

{{< figure src="https://user-images.githubusercontent.com/1020675/191574258-5b0ab325-db19-436a-a989-46876441a8c3.png" width="100%" height="100%" alt="Adding a FreeBSD Jail" caption="Adding a FreeBSD Jail from the TrueNAS web interface">}}

Clicking **Next** brings you to the networking configuration:

- **Checked** DHCP Autoconfigure IPv4. I'm a big fan of using DHCP over
  statically-configured IPv4 addresses; if you've ever had to re-IP a large
  network, and I've had to do that at least twice, the ease of editing a single
  file vastly outweighs the challenge of logging into dozens of different
  devices and re-configuring manually.
- vnet_default_interface: **vlan2**. We chose VLAN 2, which is our guest
  network. Our validator has an open inbound port (30303) from the internet,
  which increases the chance of it being compromised. If it becomes
  compromised, we want to limit the blast radius, so we put it on the guest
  network so that it can't easily be used as a launchpad to attach my machines
  on my main network.
- **Checked** Autoconfigure IPv6. I'm an IPv6 fanboy. And the guest network has
  one of the 8 IPv6 /64 networks that Comcast has delegated to me
  (`2601:646:100:69f3::/64`).

{{< figure src="https://user-images.githubusercontent.com/1020675/191611408-899a1436-bd6e-4746-8d8b-666105fc3980.png" width="100%" height="100%" alt="Adding a FreeBSD Jail" caption="Adding a FreeBSD Jail from the TrueNAS web interface, networking">}}

Click **Submit**; the jail will be created. Next we need to start the jail;
**check** the box next to validator.nono.io, and click **Start** (below the ▶
icon).

Curiously, the jail had trouble getting its IP address, and the TrueNAS server
spontaneously rebooted. This was an unfortunate development. However, after
rebooting, I was able to start the jail, and it was able to acquire its IP
address.

#### Tangent: DHCP

I had created a DNS entry for validator.nono.io (10.9.2.11), but I needed to
wire that into my DHCP configuration because I can see from the TrueNAS web UI
that my jail is getting the wrong IP address (10.9.2.103). I need to find the
MAC address of the jail's ethernet and plug that into my DHCP configuration:

```bash
ssh atom.nono.io # my firewall
arp -an | grep 10.9.2.103 # the [wrong IP address] given to the jail
  ? (10.9.2.103) at ae:1f:6b:2b:7c:f2 on ix1.2 expires in 1192 seconds [vlan]
sudo -E nvim /usr/local/etc/dhcpd.conf
  host validator { hardware ethernet ae:1f:6b:2b:7c:f2; fixed-address validator.nono.io;}
sudo /usr/local/etc/rc.d/dhcpd restart
```

By the way, you can use the 48-bit MAC address to generate the bottom 64 bits of
the IPv6 address: `ae:1f:6b:2b:7c:f2` → `ac1f:6bff:fe2b:7cf2`; i.e. logical-and
the top byte with `0xfd` and then insert `0xfffe` in the middle.

I created the proper DNS entries (A & AAAA records) for "validator.nono.io",
and modified my DHCP server to hand out the correct IP address to my jail.
Once that was done, I clicked **Restart** on my jail to confirm it got the
correct IP address.

- The git
  [commit](https://github.com/cunnie/shay.nono.io-usr-local-etc/commit/0b430dfd1ead547a96b423472e3e10a450ba6bb0)
  adding the DNS A & AAAA records for "validator.nono.io"
- The git [commit](https://github.com/cunnie/freebsd-firewall/commit/bd3866d714fe47eb3d23e1a4647f23ca1a8a7694) for adding the DHCP entry for "validator.nono.io"

#### Setting Up the Jail

Let's click the **Shell** button in the TrueNAS UI.

```bash
/etc/rc.d/sshd onestart
pkg install zsh sudo git neovim go htop gmake tmux # agree to install pkg management
adduser -G wheel -s /usr/local/bin/zsh -w yes
 # follow the prompts
visudo
 # uncomment the line "wheel ALL=(ALL:ALL) ALL"
```

We should be able to ssh into our new jail:

```
ssh -A cunnie@validator.nono.io
mkdir workspace
cd workspace
git clone git@github.com:ledgerwatch/erigon.git
cd erigon
git switch alpha
gmake erigon
tmux
./build/bin/erigon --chain=goerli --torrent.download.rate=20mb
```

### The Network

{{< figure src="https://docs.google.com/drawings/d/e/2PACX-1vR4aa7YvXRx6cpwwRsH5INVZFEU5CbFr706GACb1b9eKFw-6oIxFi3pHUt2N-trZ8brV_-C6cjRICOg/pub?w=1999&amp;h=662" width="100%" height="100%" alt="Network Diagram" caption="Network Diagram: The firewall port-forwards eth/66 & 67 (port 30303) traffic to validator.nono.io's internal IP, and, on IPv6, passes port 30303 directly through">}}
