# OVHcloud Adapter: LAB Host

> Fills the slots of the generic runbook [`docs/runbooks/lab-host.md`](../../docs/runbooks/lab-host.md) for OVHcloud dedicated servers. For the slot convention, see [`providers/README.md`](../README.md).
>
> The reference deployment uses OVHcloud for **LAB**.
>
> Last updated: 2026-10-11.

Prices below are in EUR per month **excluding VAT**, as shown on the Irish (`en-ie`) site. Prices and stock change often: every figure carries the date it was checked, and you should re-check it in the order flow before paying.

## `order`

*Which offers at this provider pass the hard filters, what do they cost, which billing terms exist, and what must be re-checked in the order flow?*

### Offers that pass the hard filters

The reference deployment raised the RAM filter (F1) to **64 GB**, because 32 GB is tight from day one with an IDE backend and CI. The offers below pass that raised filter and the rest of F1–F6. All three include an out-of-band console and rescue mode, and allow a Proxmox VE install (see [`oob`](#oob) and [`install`](#install)).

| Offer | CPU | RAM | Disks | Port | Price | Trade-offs |
|---|---|---|---|---|---|---|
| **Rise-S** ([source](https://www.ovhcloud.com/en-ie/bare-metal/rise/rise-s/)) | AMD Ryzen 7 9700X (Zen 5) | 64 GB DDR5, on-die ECC only | 2x512 GB NVMe | 1 Gbps, unmetered | 64.99 (checked 2026-10-10) | Best single-thread CPU in budget. No true ECC. No vRack (private network) on this model. |
| **RISE-1** ([source](https://www.ovhcloud.com/en-ie/bare-metal/rise/rise-1/)) | Intel Xeon-E 2386G | 64 GB ECC | 2x512 GB NVMe | 1 Gbps, unmetered | about 64.99 (checked 2026-10-10) | True ECC and vRack, but a noticeably slower CPU. With 2x960 GB NVMe (or RISE-2) it costs 85–93 (checked 2026-10-10), at or over budget including VAT. |
| **SYS-GAME-2** ([source](https://eco.ovhcloud.com/en-ie/)) | AMD Ryzen 7 3800X (Zen 2, 2019) | 64 GB ECC | 2x960 GB NVMe | 500 Mbps | 64.99 (checked 2026-10-10) | Same price, more disk and true ECC, but a CPU three generations older and half the port speed. |

All prices: OVHcloud catalog and product pages, checked 2026-10-10 (overview: [Rise range](https://www.ovhcloud.com/en-ie/bare-metal/rise/), [dedicated server prices](https://www.ovhcloud.com/en-ie/bare-metal/prices/)). Several providers raised dedicated server prices in 2026 because of RAM and NVMe costs. During research (2026-10-10) OVHcloud stated that the Rise range was not affected by its H2 2026 increase; re-check on the [prices page](https://www.ovhcloud.com/en-ie/bare-metal/prices/).

### Billing options

Checked 2026-10-10 for Rise-S, on the [Rise-S page](https://www.ovhcloud.com/en-ie/bare-metal/rise/rise-s/) and in the order flow:

| Term | Setup fee | Monthly | First-year cost |
|---|---|---|---|
| Month-to-month | one month's rent (64.99) | 64.99 | about 845 |
| 12 months prepaid | none | about 5% less | about 741 |

At 2026-10-10 prices, the prepay saves about 100 EUR in the first year but locks in the choice. See runbook step 6 for why the first host should prefer month-to-month.

### What to re-check in the order flow

- **Price** for the term you pick, and whether the page shows it with or without VAT (at 20% VAT, Rise-S was about 78 on 2026-10-10).
- **Stock per datacenter.** On 2026-10-10 Rise-S was in stock only in two datacenters in France ([source](https://www.ovhcloud.com/en-ie/bare-metal/rise/rise-s/), availability selector). Any EU datacenter is fine for the reference deployment, since latency differences between them were within noise.
- **Disk configuration:** two NVMe devices, not a single larger one. Some configurators offer storage upgrades that change the layout.
- **Port speed** and traffic: 1 Gbps unmetered on Rise.
- **Fallback** if Rise-S is out of stock: RISE-1 with 2x512 GB NVMe at about the same price.

### Reference deployment's choice

**Rise-S**, month-to-month (decided 2026-10-10). Why:

- the best single-thread CPU within budget, which matters most for a remote IDE backend and .NET builds;
- IPMI/KVM included, and rescue mode available;
- a Proxmox VE 9 install template exists ([OS list](https://www.ovhcloud.com/en-ie/bare-metal/os/));
- 1 Gbps unmetered traffic;
- a different provider from PROD, which removes the shared failure domain entirely.

Accepted trade-offs:

- no true ECC (DDR5 on-die ECC only): acceptable for a LAB that is "ok to break";
- no vRack: a second node (L9+) joins over the overlay network, or a different model is chosen for it;
- 512 GB usable: enough for the dev VM, CI caches, the platform with bounded retention, and VM templates.

Latency from the owner's usual location to all candidate EU datacenters (this provider and others) averaged 64–76 ms with similar jitter (measured 2026-10-10), so latency did not separate the candidates.

Order and delivery notes (2026-10-10): ordered month-to-month; the delivered host was checked in the control panel against every hard filter (CPU, RAM, two NVMe devices, traffic, IPMI/KVM and rescue mode) and passed.

## `oob`

*How do I reach a console or rescue system when the host is unreachable over the network, and how do I test that access before I need it?*

Checked on the reference deployment's Rise-S on 2026-10-10. Menu names in the control panel change from time to time; look for the nearest equivalent.

### IPMI/KVM console

- **Where:** Control Panel → *Bare Metal Cloud* → *Dedicated servers* → your server → the *IPMI/KVM* tab.
- **Type:** an HTML5 console that runs in the browser. No Java is needed.
- **What you see:** the server's screen, with a toolbar above it. Three parts of the toolbar matter later:
  - *Power*: power the server off, on or reset it, independently of the OS;
  - *Send Keys* / *Hot Keys*: key combinations the browser would otherwise catch, such as `Ctrl+Alt+Del`;
  - *CD Image → Browse File → Start Media*: mounts an ISO file from your computer as a virtual CD. This is what the ISO fallback in [`install`](#install) uses.
- **Test:** open the console and check that you see the screen and that a keypress reaches it (for example, Enter on a blank screen or a login prompt).

### Rescue system

- **Where:** your server → *General information* → *Boot* → change it to **Boot in rescue mode** (the Linux rescue, `rescue-customer`), then *Restart*.
- **Access:** the dialog can take an SSH public key. If it asks you to pick a key instead of pasting one, add the key first under your account's *SSH keys*, with type *Dedicated*. Without a key, a temporary root password is sent by email.
- **What you see:**
  - on the console, a banner with the rescue version, its OS (Debian at 2026-10-10), the CPU, the RAM and the disks. `Alt+F2` changes the keyboard layout, which matters when you type a password on the console;
  - over SSH (`ssh root@<lab-host-ip>`), a root shell. The rescue system has its own SSH host key, so expect a host key warning; remove the old entry with `ssh-keygen -R <lab-host-ip>`.
- **Test:** log in over SSH and check that `lsblk` lists both NVMe devices. The many `nbd*` devices of size 0 are unused network block devices of the rescue system; ignore them.
- **Leaving rescue:** set *Boot* back to the hard disk, then restart. Otherwise every reboot lands in rescue again.

### Recovery usage

**Not yet covered: L0b-2.** How to repair a broken boot, network or firewall configuration from rescue or the console is written in L0b-2, under [`recovery`](#recovery).

## `install`

*How do I install Proxmox VE on this host with the two NVMe devices mirrored: a provider template, or the official installer through the console?*

Checked on the reference deployment's Rise-S on 2026-10-10, with the template "Proxmox Virtual Environment 9" (it installed Proxmox VE 9.2).

### Installation paths

- **Template from the control panel (chosen).** Your server → *Install* / *Reinstall* → *Installation from an OVHcloud template* → Proxmox VE 9. Four steps; step 3 is the partition scheme and step 4 takes, among other things, the hostname and an SSH public key. Give it the key; the template sets no root password. Installation takes 10–20 minutes, and you can watch it on the IPMI/KVM console.
- **Fallback: the official Proxmox VE ISO through the console.** Mount the ISO with the console's virtual media (see [`oob`](#oob)) and run the official installer. It always offers a ZFS mirror for everything, but the ISO streams from your computer, so it is slow on a high-latency link. Not walked through here.

### Template options

Every partition can use `ext4`, `xfs`, `zfs` or `swap`, with RAID 0 or 1. The default scheme for two NVMe devices:

| Partition | File system | RAID | Size |
|---|---|---|---|
| `/boot` | ext4 | 1 | 1 GiB |
| `/` | zfs | 1 | 20 GiB |
| swap | swap | — | 1 GiB on each device |
| `/var/lib/vz` | zfs, pool `data` | 1 | the rest |

The EFI system partition is not shown in the dialog; the template creates it anyway (see below).

### Option chosen and why

The **default scheme**: the root and the VM data on ZFS mirrors, `/boot` on ext4 over Linux software RAID 1. ZFS for `/boot` is offered too, but then GRUB has to read a ZFS pool, and GRUB does not support every ZFS pool feature; ext4 on a RAID 1 mirror is what it reads most reliably.

### What the template changes compared with stock Proxmox VE

Recorded from an audit right after the first login, before any change. Commands: [runbook step 2](../../docs/runbooks/lab-host.md#if-you-installed-from-a-provider-template-audit-it-first).

**Storage and boot**

- Each NVMe device carries the same partitions: an EFI system partition (ESP), `/boot`, the root pool member, swap and the data pool member.
- **ESP:** both ESPs form a Linux software RAID 1 with metadata version 0.90, mounted at `/boot/efi`. The metadata sits at the end of the partition, so the firmware sees a plain FAT file system on each device, while every write goes to both. Stock Proxmox VE with a ZFS root instead keeps separate ESPs in sync with `proxmox-boot-tool`, which this layout does not use (`proxmox-boot-tool status` reports an error; that is expected).
- **Boot loader:** GRUB, in `\EFI\proxmox\` and as a copy at the fallback path `\EFI\BOOT\BOOTX64.EFI`. The debconf setting `grub2/force_efi_extra_removable` is `true`, which tells Debian's GRUB packages to refresh that copy on every GRUB update. At 2026-10-11 no GRUB update had happened on the host yet, so this is the documented behavior, not an observed one.
- **Boot entries:** one entry per device, each pointing to the fallback path on that device's ESP. The firmware boot order starts with network boot: the provider's netboot service decides whether the server boots from disk or into rescue mode. An EFI shell entry comes before the disk entries; what that means when netboot is unavailable is covered in L0b-2.
- **`/boot`:** ext4 on a second software RAID 1.
- **Root:** a ZFS mirror pool named `zp0`, dataset `zp0/zd0` mounted at `/` (stock Proxmox VE uses `rpool/ROOT/pve-1`).
- **VM data:** a ZFS mirror pool named `data`, dataset `data/zd1` mounted at `/var/lib/vz`.
- **Swap:** one partition per device, both active, not mirrored.
- **Config drive:** an extra small partition labelled `config-2`, on the **first device only**, which cloud-init reads (see below).
- **Proxmox storage:** no `/etc/pve/storage.cfg`, so only the built-in `local` directory storage exists. Stock Proxmox VE with ZFS also creates a `local-zfs` storage for VM disks.
- **ZFS ARC:** not capped.

**Network**

- `/etc/network/interfaces` uses the stock Proxmox layout: the physical NIC is a port of the bridge `vmbr0`, and the bridge holds the host's public addresses. Details in [`network`](#network).
- It ends with `source /etc/network/interfaces.d/*`, which Proxmox SDN needs.

**Package repositories**

- deb822 files in `/etc/apt/sources.list.d/`.
- Debian: `trixie`, `trixie-updates` and `trixie-security`, through mirror lists under `/etc/apt/mirrors/` (pointing at `deb.debian.org`), with `deb-src` enabled.
- Proxmox: `pve-no-subscription` already enabled; the enterprise source is present but commented out. No Ceph source.

**Packages, services and agents**

- **cloud-init**, enabled and run at every boot, with the config drive as its data source and Debian's cloud defaults. It created no extra user.
- **unattended-upgrades**, installed and enabled, allowing Debian point-release updates as well as security updates.
- Extra admin tools that a stock install does not have, such as `sudo`, `screen`, `vim`, `tcpdump`, `traceroute`, `bind9-host`, `ssh-import-id`, `reportbug` and `apt-listchanges`.
- `skopeo` with `golang-github-containers-common`, `netavark` and `aardvark-dns` (container image tooling, with two enabled `netavark` units). Assumed to come with Proxmox VE's OCI container image support; not compared against a stock install.
- `pveversion -v` reports `pve-edk2-firmware: not correctly installed`. That is a leftover package name; the VM UEFI firmware is in `pve-edk2-firmware-ovmf` and `pve-edk2-firmware-legacy`, which are installed.

**Access**

- `/root/.ssh/authorized_keys` holds the key you gave the installer and the node's own RSA key (`root@<hostname>`, the same as `/root/.ssh/id_rsa.pub`). Proxmox VE adds the latter itself for operations between nodes. No other keys.
- No root password is set.

### Decisions on the template's additions

| Finding | Decision | Why |
|---|---|---|
| Node's own RSA key in `authorized_keys` | kept | stock Proxmox VE behavior; its private key never leaves the host |
| cloud-init | disabled with `touch /etc/cloud/cloud-init.disabled`; package and config drive kept | it would re-apply the provider's configuration at every boot and could undo manual changes. Removing the package or the partition gains nothing before L1 |
| unattended-upgrades | kept, limited to `Debian-Security` | see runbook step 2, base settings |
| No VM disk storage | added `local-zfs` (type `zfspool`) on a new dataset `data/vm`; `local` limited to ISO images, templates, backups and snippets | VM disks become ZFS volumes with instant snapshots and clones, as on a stock install |
| ARC not capped | capped at 8 GiB | see runbook step 2, base settings |
| No root password | set one locally, stored in the password manager | the web UI login as `root@pam` and the console login need it; SSH never accepts it |
| Extra admin tools, container tooling | kept | harmless; L1 decides what the host's package set should be |

## `network`

*How is the host's public interface configured (IPv4 and IPv6), and how do VMs reach the internet through NAT without violating the provider's network policy, such as MAC filtering?*

Checked on the reference deployment's Rise-S on 2026-10-11.

### Public interface, as the template configures it

- **Ports:** one port with a link. A second port is present, without a link, and is not configured.
- **Layout:** the stock Proxmox VE one. The physical port is the only member of the bridge `vmbr0`, the bridge holds the host's public addresses, and its MAC address is pinned to the port's (`hwaddress`). This bridge is the provider network: never attach a VM to it.
- **IPv4:** the main address with a **/24** netmask, and the gateway at the `.254` address of the same /24. That /24 is shared with other customers' servers, which is why runbook step 5 overrides the firewall's `local_network`.
- **IPv6:** a /64 is included; the server's page in the control panel shows the block and its gateway.
  - The template gives the host the /64's all-zero address (`<prefix>::`). Formally that is the subnet-router anycast address, but TCP and ICMP from it worked (checked 2026-10-11). Moving the host to `<prefix>::1` is left to L1 as tidying, not a fix.
  - The gateway lies **outside** the /64: it keeps the first three groups of the prefix, sets the last byte of the fourth group to `ff`, and ends in `ff:ff:ff:ff`. The template's plain `gateway` line still works, because `ifupdown2` installs the default route as `onlink`. If you configure IPv6 by other means, add a route to the gateway on the bridge first, then the default route through it.
  - VMs get no IPv6 (see runbook step 3).
- **Upstream routers:** outgoing frames go to a virtual router MAC address, and replies come from a physical router's MAC. This only matters when you read a packet capture.

### The out-of-band controller shares the port

The IPMI/KVM controller uses the same physical port as the host, in a separate VLAN with a private address. While the KVM console is open, a capture on `vmbr0` shows its frames: VLAN-tagged, with a MAC address next to the port's own, and port 443 traffic to a provider address. They are not VM traffic.

### NAT for VMs

- NAT behind the main IPv4 needs **neither an additional IP nor a virtual MAC**. Verified on 2026-10-11 with a throwaway VM on a VNet (runbook step 3): its traffic on `vmbr0` carried the host's MAC and public address, and a capture filtered on the VM's MAC saw nothing.
- Not used here: an additional IPv4 costs 1.99 EUR/month (checked 2026-10-10) and needs a virtual MAC per IP if a VM is bridged onto the provider network with it.

### Provider monitoring

- OVHcloud monitoring pings the server. With **proactive intervention**, a server that stops answering triggers a datacenter technician.
- The mode is set on your server → *General information* → *Monitoring*. During bootstrap use **enabled without proactive intervention**, so that reboots and closed ports do not send anyone. The long-term mode is decided in L0b-2, together with the edge firewall.
- If ICMP is ever filtered, OVHcloud's monitoring addresses must stay allowed ([monitoring IP addresses](https://docs.ovhcloud.com/en/guides/bare-metal-cloud/dedicated-servers/network-ip-monitoring)). The host firewall in runbook step 5 allows ICMP from everywhere, so it needs no exception. This note moves to [`edge-firewall`](#edge-firewall) in L0b-2.

## `edge-firewall`

**Not yet covered: L0b-2.**

*Unverified research notes (2026-10-10, [OVHcloud dedicated server docs](https://help.ovhcloud.com/csm/en-ie-documentation-bare-metal-cloud-dedicated-servers?id=kb_browse_cat)):*

- A stateless network firewall, IPv4 only, up to 20 rules per IP.
- Anti-DDoS is always on.

## `recovery`

**Not yet covered: L0b-2.**

Access to rescue mode and the IPMI/KVM console is verified, see [`oob`](#oob). How to use them for repairs is not written yet.

## `private-network`

**Not yet covered: L9+.**

*Unverified research notes (2026-10-10, [vRack](https://www.ovhcloud.com/en-ie/network/vrack/)):*

- vRack (private network) is available on most Rise models, but not on Rise-S.
