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

### The ~~Obstacles~~ Requirements

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
  file (`dhcpd.conf`) vastly outweighs the challenge of logging into dozens of
  different devices and re-configuring manually.
- vnet_default_interface: **ix1**. We chose our second 10Gbe ethernet
  interface, which we weren't using for anything. Note, we wanted to place this
  jail on the guest network, but there were awful performance problems (98%
  decline in bandwidth). See [References](#not-able-to-use-the-guest-network)
  for more info.
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
pkg install bash zsh sudo git neovim go htop gmake tmux python # agree to install pkg management
adduser -G wheel -s /usr/local/bin/zsh -w yes
 # follow the prompts to add user "cunnie"
visudo
 # uncomment the line "wheel ALL=(ALL:ALL) ALL"
```

We should be able to ssh into our new jail:

```
ssh -A cunnie@validator.nono.io
```

#### Installing the Execution Client ~~Erigon~~ Geth

From <https://goerli.launchpad.ethereum.org/en/select-client>:

> To process incoming validator deposits from the execution layer (formerly
> 'Eth1' chain), you'll need to run an execution client

I choose ~~[Erigon](https://github.com/ledgerwatch/erigon/)~~
[Geth](https://geth.ethereum.org/docs/install-and-build/installing-geth) as the
client because it's written in Golang, and I've had an easier time getting
Golang-based programs to run under FreeBSD. I originally wanted to use Erigon to
support "client diversity" (over 66% of the clients are Geth-based), but Erigon
would `panic()` ("crash") at leasst once a day.

```bash
mkdir -p ~/workspace
cd ~/workspace
git clone https://github.com/ethereum/go-ethereum.git
cd go-ethereum
git checkout v1.10.25
make geth
sudo install build/bin/geth /usr/local/bin
 # sudo pkg install go-ethereum # <-- installs older v1.10.21
tmux # <-- not necessary
geth \
  --goerli \
  --http \
  --nat extip:73.189.219.4
geth attach --datadir .ethereum/goerli/ # <-- to troubleshoot
  net.peerCount # 2
  net.listening # true
  admin.peers
  debug.verbosity(5)
  net
  admin
  debug
  eth
```

#### The Concensus Client: Lighthouse

```bash
sudo pkg install llvm13 cmake protobuf
 # blindly running shell-scripts is a fool's errand
 # and we are nothing if not fools
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
exit
```
Log back in so that Rust-compiled executables are in our PATH (`$HOME/.cargo/env`).

```bash
cd ~/workspace
git clone https://github.com/sigp/lighthouse.git
cd lighthouse
git checkout stable
gmake
```

_We were gonna install Pryzm because it was Golang, but they didn't have a
pre-built FreeBSD binary, and their build tool, Bazel, doesn't run on FreeBSD
according to their [website](https://bazel.build/) (only Windows, Linux, and
macOS)._

Let's start the client:

```bash
lighthouse \
  beacon_node \
  --network goerli \
  --execution-endpoint http://127.0.0.1:8545
```

### The Network

{{< figure src="https://docs.google.com/drawings/d/e/2PACX-1vR4aa7YvXRx6cpwwRsH5INVZFEU5CbFr706GACb1b9eKFw-6oIxFi3pHUt2N-trZ8brV_-C6cjRICOg/pub?w=1999&amp;h=662" width="100%" height="100%" alt="Network Diagram" caption="Network Diagram: The firewall port-forwards eth/66 & 67 (port 30303) traffic to validator.nono.io's internal IP, and, on IPv6, passes port 30303 directly through">}}

### Addendum

Disturbing issue at initial sync:

> panic: mdbx_txn_begin: MDBX_PANIC: Maybe free space is over on disk. Otherwise it's hardware failure. Before creating issue please use tools like https://www.memtest86.com to test RAM and tools like https://www.smartmontools.org to test Disk. To handle hardware risks: use ECC RAM, use RAID of disks, run multiple application instances (or do backups). If hardware checks passed - check FS settings - 'fsync' and 'flock' must be enabled.  Otherwise - please create issue in Application repo. On default DURABLE mode, power outage can't cause this error. On other modes - power outage may break last transaction and mdbx_chk can recover db in this case, see '-t' and '-0|1|2' options., label: downloader, trace: [kv_mdbx.go:454 kv_mdbx.go:640 mdbx_piece_completion.go:29 mmap.go:95 torrent.go:276 torrent.go:1331 torrent.go:2084 torrent.go:2214 asm_amd64.s:1571]

Another one, days later:

> WARN[09-23|04:06:11.741] nodeDB.QuerySeeds failed                 err="mdbx_txn_begin: MDBX_PANIC: Maybe free space is over on disk. Otherwise it's hardware failure. Before creating issue please use tools like https://www.memtest86.com to test RAM and tools like https://www.smartmontools.org to test Disk. To handle hardware risks: use ECC RAM, use RAID of disks, run multiple application instances (or do backups). If hardware checks passed - check FS settings - 'fsync' and 'flock' must be enabled.  Otherwise - please create issue in Application repo. On default DURABLE mode, power outage can't cause this error. On other modes - power outage may break last transaction and mdbx_chk can recover db in this case, see '-t' and '-0|1|2' options., label: sentry, trace: [kv_mdbx.go:454 kv_mdbx.go:640 nodedb.go:548 table.go:318 table.go:301 asm_amd64.s:1571]"

To fix (watch out! There's a thirteen-minute delay between hitting ^C and getting a prompt—be patient):

```
^C
  INFO[09-23|08:58:05.696] Got interrupt, shutting down...
  ...
  INFO[09-23|09:11:04.877] [] etl: temp files removed               total size=484.1MB
rm -rf ~/.local/share/erigon/chaindata
./build/bin/erigon --chain=goerli --torrent.download.rate=20mb
```

Another panic:

> panic: mdbx_txn_begin: MDBX_PANIC: Maybe free space is over on disk. Otherwise it's hardware failure. Before creating issue please use tools like https://www.memtest86.com to test RAM and tools like https://www.smartmontools.org to test Disk. To handle hardware risks: use ECC RAM, use RAID of disks, run multiple application instances (or do backups). If hardware checks passed - check FS settings - 'fsync' and 'flock' must be enabled.  Otherwise - please create issue in Application repo. On default DURABLE mode, power outage can't cause this error. On other modes - power outage may break last transaction and mdbx_chk can recover db in this case, see '-t' and '-0|1|2' options., label: downloader, trace: [kv_mdbx.go:454 kv_mdbx.go:640 mdbx_piece_completion.go:29 mmap.go:95 torrent.go:276 torrent.go:1331 torrent.go:2084 torrent.go:2214 asm_amd64.s:1571]

```
goroutine 6355434 [running]:
github.com/anacrolix/torrent/storage.mmapStoragePiece.Completion({{0x82a6b6178, 0xc0017261d0}, {0xc0001cd100, 0x14ee}, {0xd0, 0x3f, 0x88, 0xc8, 0x77, 0x49, ...}, ...})
        github.com/anacrolix/torrent@v1.46.1-0.20220808053819-61302332cfc5/storage/mmap.go:97 +0xcc
github.com/anacrolix/torrent.(*Torrent).pieceCompleteUncached(0xc00012a010?, 0xc0001e20b0?)
        github.com/anacrolix/torrent@v1.46.1-0.20220808053819-61302332cfc5/torrent.go:276 +0x4e
github.com/anacrolix/torrent.(*Torrent).updatePieceCompletion(0xc002b7f500, 0x14ee)
        github.com/anacrolix/torrent@v1.46.1-0.20220808053819-61302332cfc5/torrent.go:1331 +0x66
github.com/anacrolix/torrent.(*Torrent).pieceHashed(0xc002b7f500, 0x14ee, 0x1, {0x0, 0x0?})
        github.com/anacrolix/torrent@v1.46.1-0.20220808053819-61302332cfc5/torrent.go:2084 +0x6f0
github.com/anacrolix/torrent.(*Torrent).pieceHasher(0xc002b7f500, 0x14ee)
        github.com/anacrolix/torrent@v1.46.1-0.20220808053819-61302332cfc5/torrent.go:2214 +0x4aa
created by github.com/anacrolix/torrent.(*Torrent).tryCreatePieceHasher
        github.com/anacrolix/torrent@v1.46.1-0.20220808053819-61302332cfc5/torrent.go:2152 +0x129
```

#### Not able to use the Guest Network

- <https://www.truenas.com/community/threads/freenas-mini-xl-vnet-jail-bridged-to-vlan-extremely-slow.80878/>
- <https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=230996>

#### Don't `pkg install rust`

`package `lighthouse v3.1.0 (/usr/home/cunnie/workspace/lighthouse/lighthouse)` cannot be built because it requires rustc 1.62 or newer, while the currently active rustc version is 1.61.0`

#### Helpful Howtos

- <https://medium.com/simplystaking/setting-up-an-eth-2-0-validator-node-simply-staking-40b5f96a9e8d>
- <https://medium.com/coinmonks/how-to-setup-ethereum-2-0-validator-node-lighthouse-meddala-goerli-4f0b85d5c8f>
