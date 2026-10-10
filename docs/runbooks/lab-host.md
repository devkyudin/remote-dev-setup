# Runbook: LAB Host

> How to choose, order and bootstrap the LAB host: the bare-metal machine that runs Proxmox for dev, CI, agents and the platform (see [architecture → LAB Host](../architecture.md#lab-host)).
>
> This runbook is **provider-agnostic**. Steps that differ by hosting provider are declared as named **slots**. Each provider adapter in [`providers/`](../../providers/README.md) answers them in `providers/<name>/lab-host.md`.
>
> Last updated: 2026-10-10.

## Contents

- [Selection](#selection): set a budget, filter, rank, measure latency, choose, order and accept the host.
- [Bootstrap](#bootstrap): verify out-of-band access, install Proxmox VE and check it against acceptance criteria, set up VM networking with NAT, join the overlay network, lock down the management plane. Steps 6–7 follow in L0b-2.

## Selection

You will end this section with a host ordered and checked, and the decision written down. Plan for one session of research plus the provider's delivery time.

### 1. Set a budget

Write down:

- the monthly range you are willing to pay (for example "50–100 per month");
- whether that range **includes VAT**. Providers show prices with or without VAT, depending on the page and your account type. Mixing the two makes offers look cheaper or more expensive than they are;
- whether you would pay a **setup fee** or **prepay** a longer term to get a lower price.

*Why:* without a fixed budget, every comparison drifts towards "a bit more for a bit better". A budget that states its VAT basis lets you compare offers directly.

To compare offers with different fees, use the cost of the first year:

```
first-year cost = setup fee + 12 x monthly price
```

### 2. Apply the hard filters

A hard filter is pass/fail. A candidate that fails **any** of them is dropped, however cheap it is. Collect the candidates from the providers you are considering, then drop every offer that fails:

| # | Filter | Passes when | Why |
|---|---|---|---|
| F1 | Hardware baseline | x86-64, at least 32 GB RAM, at least 8 CPU cores, at least 512 GB usable on **two NVMe devices** that you can mirror | One disk failure must not lose the host; HDD-only storage makes builds and Docker disk-bound. |
| F2 | Out-of-band access | A rescue system, KVM or IPMI console that works without the host's own OS or network | You will lock yourself out with a firewall or SSH mistake at least once; the console is how you get back in. |
| F3 | Hypervisor installable | Proxmox VE can be installed, from a provider image or from the official installer | Offers limited to the provider's own managed OS images cannot run the LAB. |
| F4 | VM networking with one public IP | VMs can reach the internet through **NAT** behind the host's single public IP without breaking the provider's network policy | Extra public IPs cost money and are not needed. Some providers block or flag traffic from MAC addresses they did not assign, so VMs must stay behind NAT and never appear on the provider network with their own MAC. |
| F5 | Traffic | Unmetered, or an included volume large enough for CI image pulls, platform telemetry and restore drills | Metered traffic turns every `docker pull` into a cost. |
| F6 | Independence from PROD | A different provider from the PROD host, or at least a separate account | See [step 5](#5-check-independence-from-the-prod-provider). |

You can **raise** a filter for your own workload, but not lower it. For example, if you plan to run an IDE backend and CI from day one, 32 GB RAM is tight and you might filter at 64 GB. Write down any raised filter together with the decision.

*Tip:* the hardware filters (F1) are on the offer page. F2–F5 are usually in the provider's documentation or FAQ; search it for "rescue", "IPMI", "KVM", "Proxmox", "MAC address" and "traffic".

### 3. Rank the remaining candidates

Sort the candidates that passed every filter by these criteria, **in this order**. A later criterion only matters when the earlier ones are close.

1. **Price**, using the first-year cost from step 1.
2. **Single-thread CPU performance.** Compare the CPU models on a public benchmark that publishes single-thread scores. *Why:* a remote IDE backend and incremental builds mostly wait on one core, so a newer CPU feels faster than more older cores.
3. **ECC memory.** True ECC protects against silent memory corruption. *Why it is not higher:* LAB is "ok to break", and important data (PROD backups) does not depend on it.
4. **RAM headroom** above your filter. More agent slots and staging environments later.
5. **Latency** from where you work, measured as in step 4. It is a tie-breaker, not a filter.

### 4. Measure latency

Latency decides how responsive a remote IDE feels. Measure it rather than guessing from a map.

1. For each candidate datacenter, find the provider's **test host**: a looking-glass or speed-test address that the provider publishes for this purpose. Do not measure against a random customer server.
2. From the machine and network you usually work on, run:

   ```bash
   ping -c 100 <test-host>
   ```

   ```bash
   mtr --report --report-cycles 100 <test-host>
   ```

   `ping` prints `min/avg/max/stddev` (on Linux, `mdev`) at the end. `avg` is the latency, and `stddev`/`mdev` is a good proxy for jitter. `mtr` shows where along the route loss or delay appears.
3. Repeat at **several times of day**: at least a working-hours run, an evening peak run and a quiet run (night or early morning).
4. Record the results in a table:

   | Datacenter | Time of day | Avg (ms) | Jitter (ms) | Loss |
   |---|---|---|---|---|
   | `<dc-1>` | morning | | | |
   | `<dc-1>` | evening | | | |
   | `<dc-2>` | morning | | | |

**Rules of thumb** for round-trip latency to a remote IDE. They are rough guides; jitter and packet loss hurt more than a stable average.

| Average RTT | Full remote IDE (backend on the host, thin client locally) | Editor over SSH (local UI, remote files and terminal) |
|---|---|---|
| under 30 ms | feels local | feels local |
| 30–60 ms | comfortable | comfortable |
| 60–100 ms | usable, typing lag is noticeable | comfortable |
| over 100 ms | frustrating | usable |

If all candidates land in the same band, latency cannot separate them; go back to price and hardware.

### 5. Check independence from the PROD provider

Look at where the PROD host runs.

- **Preferred:** the LAB host is at a **different provider**. An account suspension, a billing dispute or a provider-wide outage then cannot take down both.
- **Acceptable:** if the best candidate is at PROD's provider, take it only under a **separate account**, and write down why in the decision.

*Why:* the [architecture](../architecture.md#core-principles) keeps PROD independent of LAB. A shared account is a shared point of failure, even when the two hosts never talk to each other.

### 6. Choose a billing term

Providers usually offer some of:

- **month-to-month**, often with a setup fee;
- **prepaid terms** (for example 12 months), often without a setup fee or with a discount.

For your **first** LAB host, prefer a short term even if it costs a little more. *Why:* the bootstrap (L0b) may reveal a problem (an installer that will not mirror the disks, a console that does not work as documented) that makes switching worthwhile. Prepay once the host has proven itself.

### 7. Record the decision in the provider adapter

Write the decision into the `order` slot of `providers/<name>/lab-host.md` (create the adapter if it does not exist; see [`providers/README.md`](../../providers/README.md)):

- the offers at this provider that passed the hard filters, with their trade-offs;
- the billing options;
- what to re-check in the order flow;
- the offer you chose and why, including any raised filter from step 2.

Every price and stock statement gets the **date it was checked** and a **source link**. Do **not** record deployment values: no IPs, hostnames, account IDs or order numbers. They belong in the password manager.

### 8. Order and accept the host

1. Open the provider's order flow and re-check price, CPU, RAM, disk configuration (two NVMe devices), port speed and stock in the datacenter you want. If anything differs from the adapter, update the adapter (with the new date) first.
2. Place the order. Store the provider account credentials and any server access details in the password manager (bootstrap tier, see [Secrets by Tier](../architecture.md#secrets-by-tier)).
3. When the host is delivered, check it against **every hard filter** in the provider's control panel: CPU, RAM, two NVMe devices, traffic, and the out-of-band options.
4. If something does not match, raise it with the provider **before** any bootstrap work. Bootstrap does not start until it is resolved.

## Bootstrap

You will end steps 1–5 with a Proxmox VE host that is installed on mirrored disks, gives VMs internet access through NAT, and accepts management connections only over the overlay network. Plan for two or three sessions. Steps 6–7 (edge firewall and recovery drills) follow in L0b-2.

Two rules apply to every step:

- **Verify the new path before closing the old one.** Never close or restrict a way into the host (public SSH, the web UI on the public address) until the way that replaces it has been tested in the same session. Keep the out-of-band console open in another window during risky steps. *Why:* a firewall rule or a typo in a network file can lock you out in one keystroke, and the console is the only way back that does not depend on the host's own network.
- **No VM ever appears on the provider network with its own MAC address.** VMs sit on private subnets behind NAT on the host. *Why:* many providers block or flag traffic from MAC addresses they did not assign, and one bridged VM can get the whole server disconnected.

Write down everything you do by hand, including commands and what you saw. L1 automates exactly these steps, and your notes are its specification. Keep notes that contain deployment values (IPs, hostnames, serial numbers) in the password manager, not in the repository.

### Slots

Each slot has a stable name and the question it must answer. The answers live in `providers/<name>/lab-host.md` under a heading with the slot's exact name. To find the adapter for your provider, see the index in [`providers/README.md`](../../providers/README.md).

| Slot | Question the adapter must answer | Used in |
|---|---|---|
| `order` | Which offers at this provider pass the hard filters, what do they cost, which billing terms exist, and what must be re-checked in the order flow? | [Selection, steps 7–8](#7-record-the-decision-in-the-provider-adapter) |
| `oob` | How do I reach a console or rescue system when the host is unreachable over the network, and how do I test that access before I need it? | [Bootstrap step 1](#1-verify-out-of-band-access) |
| `install` | How do I install Proxmox VE on this host with the two NVMe devices mirrored: a provider template, or the official installer through the console? | [Bootstrap step 2](#2-install-proxmox-ve-with-mirrored-nvme) |
| `network` | How is the host's public interface configured (IPv4 and IPv6), and how do VMs reach the internet through NAT without violating the provider's network policy, such as MAC filtering? | [Bootstrap step 3](#3-configure-vm-networking-with-nat) |
| `edge-firewall` | Does the provider offer a network firewall in front of the host, what can it filter, and how should it be configured alongside the host firewall? | Bootstrap step 6 |
| `recovery` | How do I boot a rescue system, reach the host's disks, and repair a broken boot, network or firewall configuration? How is a failed disk replaced? | Bootstrap step 7 |
| `private-network` | Can a second node join this host over a provider private network, or must it connect over the overlay network? | Bootstrap step 8 (L9+) |

### Steps

#### 1. Verify out-of-band access

Provider-specific: slot **`oob`** (how to open the console and the rescue system at your provider).

Do this **before you change anything** on the host. *Why:* every later step can cut you off from the host's network, and the console is the way back. You want to find out that it does not work now, while the host holds nothing, not at the moment you need it.

1. If the provider reacts to a server that stops answering (for example, monitoring that sends a technician), make sure the reboots and closed ports of the next steps will not trigger it. The `network` slot says how.
2. Open the **out-of-band console** (KVM, IPMI or similar). It works when you see the server's screen (firmware, boot loader or a login prompt) and a keypress you type reaches it, for example by opening a firmware menu or typing at a prompt.
3. Boot the **rescue system** and connect to it. It works when you get a root shell and can see both NVMe devices. While you are there, take an inventory:

   ```bash
   lsblk
   ```

   ```bash
   nvme list
   ```

   ```bash
   smartctl -a /dev/nvme0n1
   ```

   ```bash
   smartctl -a /dev/nvme1n1
   ```

   ```bash
   ip a
   ```

   ```bash
   ip r
   ```

   Record the disk models, `SMART overall-health` (must be `PASSED` on both), `Percentage Used` (wear) and `Power On Hours`. Rented disks may be used, and you want a baseline before you depend on them. From the network output, record the interface name, the netmask and how the gateway is reached.
4. Leave the rescue system as the `oob` slot describes, so that the host boots from its disks again.

#### 2. Install Proxmox VE with mirrored NVMe

Provider-specific: slot **`install`** (which installation paths your provider offers, which one to pick, and what a provider template changes).

This step does not prescribe *how* to install. It defines what a correct install looks like, so you can check yours the same way whether it came from a provider template or the official installer.

##### Acceptance criteria

Check every criterion before any other bootstrap step. An install that fails one is fixed or redone. *Why:* a mirror with a boot loader on only one disk looks healthy until that disk dies, and then the host does not boot.

| # | Criterion | Check | Passes when |
|---|---|---|---|
| A1 | Proxmox VE 9 | `pveversion` | it prints `pve-manager/9.` |
| A2 | System and VM storage redundant across both NVMe devices | `lsblk -f`, then `zpool status` (ZFS) and `cat /proc/mdstat` (Linux software RAID) | every pool shows a `mirror` with a partition from each NVMe device, both `ONLINE`; every md array shows `[UU]` with a partition from each device. Nothing that holds the system or VM disks lives on one device only |
| A3 | A boot loader on each device | `proxmox-boot-tool status` (if the install uses it), otherwise `efibootmgr -v`, `lsblk -o NAME,PARTTYPENAME,MOUNTPOINT` and `cat /proc/mdstat` | each NVMe device has an EFI system partition with a boot loader and a boot entry pointing to it. Either `proxmox-boot-tool` lists both partitions, or the two partitions form a software RAID 1 with metadata at the end (version 0.90 or 1.0), so the firmware sees a plain FAT file system on each device. If the boot entries use the fallback path `\EFI\BOOT\BOOTX64.EFI`, `debconf-show grub-efi-amd64` must show `grub2/force_efi_extra_removable: true`, or GRUB updates leave that copy stale. On a legacy BIOS system: `debconf-show grub-pc` lists both devices under `install_devices` |
| A4 | ZFS ARC capped, if ZFS is used | `cat /sys/module/zfs/parameters/zfs_arc_max` | it shows the cap you chose (not `0`, which means "half of RAM") |
| A5 | SSH key login only | `sshd -T \| grep -Ei '^(passwordauthentication\|kbdinteractiveauthentication\|permitrootlogin)'` | `passwordauthentication no`, `kbdinteractiveauthentication no`, `permitrootlogin prohibit-password` (or `without-password`) |
| A6 | No-subscription repository on, enterprise repository off | `apt-get update`, then `apt-cache policy \| grep -i proxmox` | no `401 Unauthorized` errors; `pve-no-subscription` is listed and no `enterprise.proxmox.com` source is |
| A7 | Second factor on `root@pam` | `pveum user tfa list root@pam` | a TOTP entry is listed |

A5 and A7 are completed in [step 5](#5-lock-down-the-management-plane). Check A1–A4 and A6 at the end of this step.

##### If you installed from a provider template: audit it first

A provider template is a black box. Right after the first login, **before changing anything**, record what it produced:

```bash
lsblk -f; cat /proc/mdstat; zpool status
```

```bash
proxmox-boot-tool status; efibootmgr -v
```

```bash
cat /etc/network/interfaces
```

```bash
ls -l /etc/apt/sources.list /etc/apt/sources.list.d/; cat /etc/apt/sources.list.d/*
```

```bash
apt-mark showmanual
```

```bash
systemctl list-unit-files --state=enabled; ls /etc/cron.d
```

```bash
cat /root/.ssh/authorized_keys
```

Save the output privately: it contains deployment values. Then compare it with a stock Proxmox VE install and write the differences, without deployment values, into the `install` slot: partitioning, boot entries, network configuration, repositories, extra packages, agents or services, and authorized keys. Any key or agent you did not add is either removed or kept with a written reason. *Why:* L1 automates from this state, and its author should not have to inspect a live host to learn it.

##### Base settings

1. **Root password.** Set a strong one with `passwd` and store it in the password manager. SSH never accepts it (see A5), but the web UI login as `root@pam` and a login on the out-of-band console both need it.
2. **Repositories.** In the web UI, open *your node → Updates → Repositories*: add `No-Subscription`, and disable every `enterprise` entry (Proxmox VE, and Ceph if present). In Proxmox VE 9 these are deb822 files under `/etc/apt/sources.list.d/`; a disabled source carries `Enabled: false`. Then run:

   ```bash
   apt update && apt full-upgrade
   ```

   *Why:* the enterprise repository needs a paid subscription and returns `401` without one. The no-subscription repository gets the same packages slightly later.
3. **ARC cap (ZFS only).** ZFS uses up to half of RAM as a read cache by default, and that memory is not available to VMs. This runbook caps it at 8 GiB on a 64 GB host and revisits the value once real workloads exist:

   ```bash
   echo "options zfs zfs_arc_max=8589934592" > /etc/modprobe.d/zfs.conf
   ```

   ```bash
   update-initramfs -u -k all
   ```

   The second command matters when the root itself is on ZFS: the value must be in the initramfs to apply at boot.
4. **Security updates only, unattended.** Install `unattended-upgrades` and, in `/etc/apt/apt.conf.d/50unattended-upgrades`, keep only the `Debian-Security` lines under `Unattended-Upgrade::Origins-Pattern`. Leave `Unattended-Upgrade::Automatic-Reboot` at `false`.

   ```bash
   apt install unattended-upgrades
   ```

   ```bash
   unattended-upgrades --dry-run --debug 2>&1 | grep -i 'allowed origins'
   ```

   The allowed origins must list only security origins. *Why:* Debian security fixes are low-risk and should not wait for you. Proxmox packages (kernel, `pve-*`) can change behavior or need a reboot, so they are upgraded by hand, when you are watching.
5. **Storage for VM disks.** If Proxmox has no storage of type `zfspool` (check with `pvesm status`), add one on the ZFS mirror that holds VM data, and keep the directory storage for files only:

   ```bash
   zfs create <data-pool>/vm
   ```

   ```bash
   pvesm add zfspool local-zfs --pool <data-pool>/vm --content images,rootdir --sparse 1
   ```

   ```bash
   pvesm set local --content iso,vztmpl,backup,snippets
   ```

   *Why:* on a `zfspool` storage each VM disk is a ZFS volume, so snapshots and clones from a template are instant and take no space until they diverge. L2 clones the dev VM from a template, and L7 clones a VM for every CI job. On a directory storage, disks are files and get none of that.
6. **Reboot once**, then run the checks A1–A4 and A6.

#### 3. Configure VM networking with NAT

Provider-specific: slot **`network`** (how the provider configured the public interface, IPv6 delivery, and anything that must stay reachable for the provider's own monitoring).

##### The model

```
  internet
     |
  public interface / bridge   <- only the host's own IP and MAC are ever seen here
     |
  host: routes and source-NATs (SNAT) VM traffic
     |
     +-- vndev   10.42.20.0/24   gateway 10.42.20.1   dev zone
     +-- vntest  10.42.90.0/24   gateway 10.42.90.1   test zone
     +-- (later) vnplat, vnrun, vnsbx
```

- Each zone gets its own private subnet and virtual network (VNet). The subnets come from the [zone address plan](../architecture.md#lab-host) in the architecture. *Why plan every zone now:* once VMs live on a network, renumbering it means touching every VM, firewall rule and DNS record. Writing the plan down is free; creating a VNet only when its zone is needed keeps the host simple.
- VMs reach the internet through **source NAT** on the host: outgoing traffic leaves with the host's address and MAC. Nothing is forwarded inbound: no port on the public address leads to a VM. You reach VMs over the overlay network (decided in L2).
- VMs get their address, gateway and DNS resolver over **DHCP from the host**, backed by an **address registry (IPAM)** on the host. A VM that needs a stable address gets a **reservation** in the registry, not a static configuration inside the VM.
- VMs use IPv4 only. IPv6 stays on the host.
- **Never attach a VM to the host's public bridge** (often `vmbr0`). It carries the host's public address, so a VM on it would appear on the provider network with its own MAC address. VMs attach only to the VNets below.

> **Zones are not isolated yet.** Separate VNets look like separate security zones, but they are not: the host routes between them, and nothing restricts that traffic until firewall policy between VNets is enforced (L6/L8). Until then, do not place agent workspaces, ephemeral runners or anything else that runs untrusted code on LAB VNets.

##### Walkthrough: a Proxmox SDN Simple zone

Proxmox's Software-Defined Networking (SDN) does NAT, DHCP and IPAM without hand-written firewall rules, shows them in the web UI, and exposes them through the API that later automation uses. A **Simple zone** is local to one node.

1. Install the DHCP server that SDN drives, and disable its default instance (SDN starts its own instance per zone):

   ```bash
   apt install dnsmasq
   ```

   ```bash
   systemctl disable --now dnsmasq
   ```

2. Check that `/etc/network/interfaces` ends with `source /etc/network/interfaces.d/*`. SDN writes its bridges there; without that line they are never brought up.
3. *Datacenter → SDN → Zones → Add → Simple*: ID `lab`, IPAM `pve`, **automatic DHCP** on.
4. *Datacenter → SDN → VNets → Create*: `vndev` in zone `lab`. Then, with `vndev` selected, *Subnets → Create*:
   - subnet `10.42.20.0/24`, gateway `10.42.20.1`, **SNAT** on;
   - a DHCP range, for example `10.42.20.100`–`10.42.20.199`, which leaves the lower addresses for reservations;
   - leave the DHCP DNS server empty. The zone's DHCP server also answers DNS on each VNet's gateway address and forwards queries to the host's own resolvers, so VMs get the gateway as their resolver.
5. Repeat for `vntest` with `10.42.90.0/24`.
6. *Datacenter → SDN → Apply*. The zone and both VNets show as `available`, and `ip a` on the host shows the `vndev` and `vntest` bridges with their gateway addresses.

VNet IDs are limited to 8 characters; that is why the names are short.

After *Apply*, check that every subnet got its NAT rule and that the DHCP/DNS server listens only on the VNet gateways, never on the public address:

```bash
grep -E '^iface|SNAT' /etc/network/interfaces.d/sdn
```

```bash
ss -lunp | grep ':53 '
```

Each VNet needs a `post-up iptables -t nat ... SNAT` line; a missing one means the SNAT box was not ticked on that subnet.

##### Verify with a throwaway VM

1. Create a small VM from any cloud image in `vntest`, with cloud-init networking set to DHCP and no static configuration.
2. In the VM, check that it got an address from `10.42.90.0/24`, that it resolves a name, and that it fetches a page:

   ```bash
   ip -4 a
   ```

   ```bash
   getent hosts deb.debian.org
   ```

   ```bash
   curl -sI https://deb.debian.org
   ```

3. On the host, while the VM repeats the `curl`, capture its traffic on the public interface (its name is in the `network` slot):

   ```bash
   tcpdump -e -n -c 20 -i <public-interface> 'tcp port 443'
   ```

   Compare the source MAC of every outgoing frame with `ip link show <public-interface>`. It must be the host's MAC, and the source IP must be the host's public address, never a `10.42.x.x` address.
4. Destroy the VM.

If DHCP or DNS does not work, give the VM a reservation in *Datacenter → SDN → IPAM* plus a static cloud-init configuration (L2 uses cloud-init anyway), and record what failed.

#### 4. Join the overlay network

Not provider-specific. This step uses Tailscale, the [architecture's](../architecture.md#open-questions) provisional choice. Self-hosted Headscale works the same way, with `--login-server` pointing at your server.

1. **Tag and policy.** In the overlay's admin console, define a tag for LAB hosts and allow only your own devices to reach it, on SSH and the Proxmox web UI. **Remove the default allow-all rule** first, or the rule below changes nothing. The shape of the policy:

   ```json
   {
     "tagOwners": {
       "tag:lab": ["<owner-login>"]
     },
     "acls": [
       { "action": "accept", "src": ["<owner-login>"], "dst": ["tag:lab:22,8006"] }
     ],
     "tests": [
       { "src": "<owner-login>", "accept": ["tag:lab:22", "tag:lab:8006"], "deny": ["tag:lab:80"] }
     ]
   }
   ```

   A user in `src` means "that user's own devices"; tagged machines, such as future runners, are not included. Keep the policy in the admin console for now; moving it into the repository as code is an L3 topic.

2. **Auth key.** Generate a **single-use, pre-authorized** auth key with `tag:lab`. Do not store it: use it right away. *Why a tagged node:* a tagged node has no key expiry, so the host does not silently drop off the overlay after the default expiry period, and it is not tied to your personal identity.
3. **Install and join.** Install Tailscale on the host from its official package repository (the install instructions for Debian on its website), then:

   ```bash
   tailscale up --auth-key=<auth-key>
   ```

   The key is single-use, so it is worthless once the host has joined.
4. **Verify.** The admin console shows the host with `tag:lab` and key expiry disabled. From your laptop, using the host's overlay name (`<lab-host>`):

   ```bash
   ssh root@<lab-host>
   ```

   and `https://<lab-host>:8006` opens the web UI.

**OpenSSH, not Tailscale SSH.** SSH stays the standard OpenSSH server, reached over the overlay address. It is what Ansible (L1) uses as is. Both options fail the same way when the overlay is down, and the out-of-band console is the way in either way.

**Not a subnet router yet.** The host joins as a plain node. How your laptop reaches VMs (the host routing LAB subnets onto the overlay, or each VM joining it) is decided in L2.

#### 5. Lock down the management plane

Not provider-specific: after this step, the Proxmox web UI, its API and SSH are reachable only over the overlay network.

##### Order of work

Follow this order; it is the "verify the new path before closing the old one" rule applied:

1. Verify the out-of-band console (step 1), and keep it open in another window.
2. Verify SSH and the web UI over the overlay network (step 4).
3. Turn on key-only SSH and the second factor (below), and verify both.
4. Add the firewall rules, then enable the firewall.
5. Verify SSH and the web UI over the overlay network again, and that the console still shows a login prompt.
6. Check from outside that the public ports are closed.

##### Key-only SSH and a second factor

1. Create `/etc/ssh/sshd_config.d/10-lockdown.conf`:

   ```
   PasswordAuthentication no
   KbdInteractiveAuthentication no
   PermitRootLogin prohibit-password
   ```

   ```bash
   sshd -t && systemctl reload ssh
   ```

   `sshd -t` checks the syntax first, so a typo does not take SSH down. Keep your current session open and test from a new one. A password attempt is refused without a prompt:

   ```bash
   ssh -o PubkeyAuthentication=no root@<lab-host>
   ```

2. In the web UI, open *Datacenter → Permissions → Two Factor* (or the user menu at the top right → *TFA*) and add **TOTP** for `root@pam`. Make sure the tree on the left has *Datacenter* selected, not the node. Keep your current session open and test in a private window: a login with the correct password but without the code fails.

   **Recovery keys are optional.** If you lose the device that holds the second factor, remove it over SSH or the out-of-band console, then enroll a new one:

   ```bash
   pveum user tfa delete root@pam
   ```

The second factor protects the web UI and API logins. SSH is protected by the key, and the out-of-band console by the root password, which is why that password lives in the password manager.

##### Host firewall

Use the Proxmox VE firewall with its default backend, at datacenter and node level. Leave the `nftables` option off: in Proxmox VE 9 the nftables backend is a tech preview. *Why this rather than binding `sshd` and the web UI to the overlay address only:* if the overlay client starts late at boot, those services fail to bind and the host looks broken in a confusing way. The firewall is one visible control, in the web UI, and it is what L1 automates. Binding can be added later as defence in depth.

Rules, at *your node → Firewall* (stored in `/etc/pve/nodes/<node>/host.fw`):

| Direction | Interface | Protocol / port | Why |
|---|---|---|---|
| in | any | ICMP and ICMPv6 | diagnostics; the provider's monitoring may ping the host (see the `network` slot) |
| in | any | UDP 41641 | the overlay's WireGuard port, for direct connections; without it the overlay still works through relays, only slower |
| in | `tailscale0` | TCP 22 | SSH over the overlay; the overlay policy decides *who* |
| in | `tailscale0` | TCP 8006 | web UI and API over the overlay |
| in | `vndev`, `vntest` | UDP 67 | DHCP requests from VMs to the host |
| in | `vndev`, `vntest` | UDP and TCP 53 | DNS queries from VMs to the gateway address |

In file form:

```
[RULES]
IN ACCEPT -p icmp
IN ACCEPT -p ipv6-icmp
IN ACCEPT -p udp -dport 41641
IN ACCEPT -i tailscale0 -p tcp -dport 22
IN ACCEPT -i tailscale0 -p tcp -dport 8006
IN ACCEPT -i vndev -p udp -dport 67
IN ACCEPT -i vntest -p udp -dport 67
IN ACCEPT -i vndev -p udp -dport 53
IN ACCEPT -i vndev -p tcp -dport 53
IN ACCEPT -i vntest -p udp -dport 53
IN ACCEPT -i vntest -p tcp -dport 53
```

**Close the `local_network` gap.** The Proxmox firewall automatically allows SSH, the web UI and the console ports from the "local network", which it detects as the network of the host's own address. Check it:

```bash
pve-firewall localnet
```

On a rented server, the detected network is usually the provider's public subnet, which means every other server in that subnet could reach your management ports. Override it at *Datacenter → Firewall → Alias*: add an alias named `local_network` with the host's own address only (`<lab-host-ip>`), and run `pve-firewall localnet` again to confirm. Do not add anything to an IP set called `management`.

Before enabling anything, check that the rules compile; the output should contain your `tailscale0` rules:

```bash
pve-firewall compile | grep -E 'tailscale0|dport (22|8006)'
```

Finally, at *Datacenter → Firewall → Options*, set **Input Policy** to `DROP` and **Firewall** to `Yes` (in `/etc/pve/firewall/cluster.fw`: `policy_in: DROP` and `enable: 1`). Enabling the datacenter firewall is the switch that applies everything, which is why it comes last. `pve-firewall status` then shows `enabled/running`.

**If you lock yourself out:** log in on the out-of-band console and run `pve-firewall stop`. It removes the rules until the firewall service is started again, so you can fix the rules over SSH.

Then verify:

- SSH and the web UI still work over the overlay network, and the console still shows a login prompt;
- a VM in a VNet still gets a DHCP lease and resolves names (boot a throwaway VM).

##### External check

From a machine outside the overlay network (for example your laptop with the overlay client stopped), scan the public address:

```bash
nmap -Pn -p 22,8006 <lab-host-ip>
```

```bash
nmap -Pn -p- <lab-host-ip>
```

Ports 22 and 8006 must be `filtered` or `closed`, and no open port may lead to a VM.

> **Known gap until L1: forwarded traffic is not filtered.** NAT needs the host to forward packets that arrive on the public bridge, so a machine in the provider's network segment that routes your VNet subnets through the host could reach VMs. The default firewall backend cannot filter forwarded traffic. Until L1 closes this (only replies to connections the VMs opened may pass from the public bridge into VNets), run nothing but throwaway test VMs. The external scan cannot see the `local_network` gap above, because you are not scanning from inside the provider's subnet; that is why `pve-firewall localnet` is checked separately.

#### 6. Configure the edge firewall

Provider-specific: slot **`edge-firewall`** (see [`providers/README.md`](../../providers/README.md)). *To be written in L0b-2 (`lab-host-recovery`).*

#### 7. Practice recovery

Provider-specific: slot **`recovery`** (see [`providers/README.md`](../../providers/README.md)). Break something on purpose and repair it through the console, boot from either disk, and rebuild the host from these notes. *To be written in L0b-2 (`lab-host-recovery`).*

#### 8. Private network for a second node

Provider-specific: slot **`private-network`** (see [`providers/README.md`](../../providers/README.md)). *To be written when a second node is added (L9+).*
