# Design

## Context

- The host is delivered bare: nothing has been installed. It passed every hard filter at delivery (see `providers/ovh/lab-host.md` → `order`). Reinstalling it costs minutes and loses nothing.
- The OVH adapter holds only *unverified* notes for `oob`, `install`, `network` and `edge-firewall` (from the `lab-host-selection` design, "Notes for L0b").
- Research during exploration (2026-10-10, not yet verified on the host):
  - OVHcloud announced that Proxmox VE 9 is the first version where its template enables ZFS by default for the **data** partition ([announcement](https://community.ovhcloud.com/t/new-os-available-proxmox-virtual-environment-ve-9/1254)). It says nothing about the root partition.
  - An older OVHcloud forum answer says its default partitioning puts the system on `md` RAID ([thread](https://community.ovhcloud.com/t/conseil-proxmox-sur-advance-1/5969)). Proxmox does not officially support that layout.
  - OVHcloud monitoring pings dedicated servers. With proactive intervention enabled, a server that stops answering triggers a datacenter technician. A firewall that drops ICMP must allow OVHcloud's monitoring addresses ([docs](https://docs.ovhcloud.com/en/guides/bare-metal-cloud/dedicated-servers/network-ip-monitoring)).
- WIP: `prod-offsite-backups` is active, and this change takes the second slot.
- The architecture lists "Tailscale or self-hosted Headscale" as an open question. This change cannot finish without picking one.

## Goals / Non-Goals

**Goals:**
- A host that meets the `lab-host` and `lab-network` requirements in this change, reached in an order that never leaves the owner without a working access path.
- A generic runbook that a reader at any provider can follow for steps 1–5, with the provider-specific parts in the OVH adapter's slots.
- A written record of everything done by hand, precise enough for L1 to automate.

**Non-Goals:**
- Choosing the final overlay product. Tailscale is a provisional choice; moving to Headscale later must not require changing this change's specs.
- Proving resilience: the boot-from-either-disk test, recovery drills and the rebuild exam belong to L0b-2. This change only *inspects* that both disks carry a boot loader.
- Tuning: ZFS recordsize, CPU governor, KSM and similar settings wait until real workloads exist.

## Decisions

### D1. Split L0b and move L1's dependency to L0b-2
- **L0b-1** (this change) ends when the host is installed and safe to leave on the internet.
- **L0b-2** (`lab-host-recovery`) takes the edge firewall, disk alerts, recovery drills and the rebuild exam.
- L1 depends on **L0b-2**, not L0b-1. An Ansible firewall role can lock the owner out as easily as a manual mistake, so recovery should be practised before it is automated. The WIP limit would block running L0b-2 and L1 in parallel anyway.

*Alternative:* one L0b change. With reconnaissance and the exam included, it would likely exceed three sessions and has no natural stopping point.

### D2. Reconnaissance first, with no changes to the host
Before installing anything, one session answers the unverified notes. Each check below says what it decides:
- Control panel:
  - the monitoring mode, set to **"enabled without proactive intervention"** for the duration of bootstrap so that reboots and closed ports do not trigger a technician;
  - the installer's partitioning options for the Proxmox VE 9 template: can the root be ZFS, and is RAID1 offered for every partition?
- IPMI/KVM console: does it open (HTML5 or Java), and is the keyboard usable? This is the break-glass path for the rest of the change.
- Rescue system: boot it, connect over SSH, then:
  - inventory the disks (`lsblk`, `nvme list`, `smartctl -a`; record wear and power-on hours, since rented disks may be used);
  - inventory the network (`ip a`, `ip r`: interface name, netmask, gateway);
  - note how the IPv6 /64 is delivered, as shown in the panel.

Results go straight into the adapter's `oob`, `install` and `network` slots, replacing the unverified notes.

### D3. Install from the provider template; the runbook defines acceptance, not procedure
The owner chose the template path over installing the official ISO through the console. The template is fast, and the time saved goes into networking, the overlay and lockdown.

To keep the generic runbook provider-neutral, step 2 becomes a list of **acceptance criteria** that any install must pass:
- Proxmox VE 9;
- system and VM storage redundant across both NVMe devices;
- a boot loader on each device: either separate ESPs kept in sync by `proxmox-boot-tool`, or one ESP mirrored by Linux software RAID 1 with its metadata at the end, so the firmware sees a plain FAT partition on each device. If the boot entries use the fallback path, GRUB must be set to refresh that copy on every update;
- the ZFS ARC capped if ZFS is used;
- SSH key login only;
- the no-subscription repository enabled and the enterprise repository disabled;
- 2FA on `root@pam`.

The `install` slot explains how to get there with the template.

Storage preference, in order:
1. ZFS mirror for root and data, if the installer offers it.
2. The template default: root on `md` RAID1, data on a ZFS mirror. This is accepted only if both disks carry a boot loader. Either it already does, or the boot entries are mirrored by hand and documented.
3. If neither works, the fallback is the official ISO through the IPMI console. It is documented in the adapter as an alternative, not walked through here.

*Outcome (2026-10-11):* the template offered ZFS for root and used it by default: root and VM data on ZFS mirrors, `/boot` on ext4 over `md` RAID1, and the ESP as an `md` RAID1 with metadata at the end, with a boot entry per device. That is option 1 in substance. `/boot` stays on `md` because GRUB reads it more reliably than a ZFS pool, whose features it does not all support.

Settings:
- **ARC cap: 8 GiB** of the 64 GB (about 12%). That leaves room for VMs while keeping a useful read cache. It is revisited when the platform VM exists.
- **SSH key:** the template installer accepts an SSH key, so no root password is sent by email. Password login over SSH is disabled from the first boot onward.
- **Root password:** set locally with `passwd` right after the first login and stored in the password manager. SSH never accepts it, but the web UI login as `root@pam` (needed for 2FA in D7) and a login on the out-of-band console both need it. Without it, the break-glass path ends at a login prompt.
- **Storage for VM disks:** a `zfspool` storage (`local-zfs`) on a dataset of the data pool, with the directory storage `local` kept for ISO images, templates, backups and snippets. VM disks then become ZFS volumes with instant snapshots and clones from templates, which L2 (dev VM from a template) and L7 (a VM per job) rely on. A stock Proxmox VE install with ZFS creates the same split; the template did not.

*Alternatives:*
- *Official ISO through IPMI:* teaches more and gives a ZFS root for sure, but the ISO streams from the laptop over about 70 ms, and the template is faster.
- *Rescue system plus QEMU running the ISO against the physical disks:* a workaround that is hard to explain to a beginner.

### D4. Audit the template immediately after install
The template is a black box. Right after the first login, before changing anything, capture what it produced:
- `lsblk -f`, `/proc/mdstat`, `zpool status`, `proxmox-boot-tool status` or `efibootmgr -v`;
- `/etc/network/interfaces`;
- `/etc/apt/sources.list*`;
- packages beyond a stock Proxmox VE (compared with the ISO's package list);
- enabled services;
- `/root/.ssh/authorized_keys`.

The findings go into the `install` slot as "what the template changes", with no deployment values. Anything unexpected, such as an extra key or agent, is removed or kept with a written reason.

Anything that would re-apply the provider's configuration at boot, such as cloud-init reading a config drive, is disabled, so manual changes stay put.

### D5. VM networking: address plan for every zone, Proxmox SDN Simple zone, two VNets now
The architecture names five zones. Renumbering a network that VMs already live on is the most painful change to defer, so the address plan for every zone is written now, and only what L0b-1 and L2 need is created.

| Zone | VNet | Subnet | Created in | First user |
|---|---|---|---|---|
| mgmt | — (the host itself) | — | — | Proxmox, overlay client |
| dev | `vndev` | 10.42.20.0/24 | L0b-1 | dev VM (L2) |
| platform | `vnplat` | 10.42.30.0/24 | L4 | observability, dev databases |
| runners | `vnrun` | 10.42.40.0/23 | L5/L7 | ephemeral VMs (many, short-lived) |
| sandbox | `vnsbx` | 10.42.50.0/24 | L9+ | k3s and experiments |
| test | `vntest` | 10.42.90.0/24 | L0b-1 | throwaway VMs, the P5 exam |

- **Range per node:** 10.42.0.0/16 for LAB-1, with 10.43.0.0/16 reserved for LAB-2. A Simple zone is local to one node, so each node routes its own range.
- **Why 10.42:** it is unlikely to collide with home networks (192.168.x, 10.0.x), which matters once the host becomes a subnet router on the tailnet (L2). It also stays clear of the tailnet's 100.64.0.0/10. VNet IDs are at most 8 characters, which is an SDN limit.
- **Mechanism:**
  - a Proxmox SDN **Simple zone** with SNAT enabled per subnet;
  - DHCP from the SDN integration (dnsmasq), with PVE IPAM as the address registry;
  - fixed addresses (for example the dev VM) as IPAM reservations;
  - DNS for VMs from the same dnsmasq instance, which answers on each VNet's gateway address and forwards to the host's resolvers. The subnet's DHCP DNS server field stays empty. dnsmasq listens only on the VNet gateways, never on the public address.

  This matches the `lab-network` requirements without hand-written `iptables` rules. It is visible in the GUI, and it is reachable from the API that L7's runner-manager will use.
- **No port forwarding to VMs**, and IPv6 stays on the host only. VMs are IPv4 behind NAT.
- **Isolation is not enforced:** a Simple zone routes between its VNets. Per `lab-network`, the runbook and the architecture say so explicitly. Enforcing isolation is a prerequisite of the step that first places agent or ephemeral runner workloads (L6/L7).
- **Verification with a throwaway VM in `vntest`:**
  - it gets an address over DHCP;
  - it resolves names;
  - it fetches a page from the internet;
  - `tcpdump -e` on the host's public interface shows only the host's MAC address as the source of its traffic.

  The VM is then destroyed.

*Alternatives:*
- *One flat bridge with masquerade in `/etc/network/interfaces`:* simplest to read, but it means renumbering later and has no IPAM.
- *A VLAN-aware bridge with hand-written NAT:* similar result, but more hand-written configuration to keep in sync, and no API.

### D6. Overlay: Tailscale as a tagged node, OpenSSH over the tailnet
- **Tailscale SaaS (free plan)** is the provisional choice. It needs no server of its own to run or secure, which LAB cannot provide yet. Headscale stays possible later without changing the specs: the specs say "overlay network", not a product.
- The host joins with a **pre-authorized, tagged auth key** (`tag:lab`):
  - a tagged node has **no key expiry**, so the host does not silently drop off the tailnet after the default 180 days;
  - the node is not tied to the owner's personal identity.
- The tailnet policy (ACL) allows only the owner's devices to reach `tag:lab` on ports 22 and 8006. The policy file is kept in the tailnet admin console for now; moving it to the repository is an L3 topic.
- **SSH stays OpenSSH**, reached over the tailnet address, rather than Tailscale SSH. It is the industry standard, and Ansible (L1) uses it as is. Both options fail in the same way if the tailnet is down, and the out-of-band console is the break-glass path either way.
- Host only: **no subnet router** in this change. How the laptop reaches VMs is decided in L2, together with the ACLs that keep future runners off the owner's devices.

### D7. Lockdown with the host firewall, in "rope" order
- **Order of steps:**
  1. Verify the out-of-band console.
  2. Verify SSH and the web UI over the tailnet.
  3. With the IPMI console open in another window, enable the firewall.
  4. Verify the tailnet paths again, and that a VM on a VNet still gets a lease and resolves names.
  5. Check from outside that the public ports are closed.

  The runbook states the rule behind this order: *verify the new path before closing the old one*.
- **Tool:** the Proxmox VE firewall with its default iptables backend (`pve-firewall`), enabled at datacenter and node level, with input policy DROP. *Why:* it is what L1 will manage, and it is visible in the GUI. The nftables backend (`proxmox-firewall`) is installed but off: in PVE 9 it is still a tech preview that its documentation calls not suited for production use.
- **Allowed in on the public interface:**
  - ICMP (diagnostics, and provider monitoring);
  - Tailscale's WireGuard UDP port, 41641. This allows direct connections; otherwise the overlay still works through relays, only slower.
- **Allowed on `tailscale0`:** 22 and 8006. The tailnet ACL decides *who* may connect.
- **Allowed on the VNet bridges (`vndev`, `vntest`):** UDP 67 (DHCP) and UDP/TCP 53 (DNS) to the host. Input policy DROP applies to every host interface, including VNet bridges, so without these rules VMs stop getting leases and resolving names once the firewall is on. VMs reach nothing else on the host, including 22 and 8006 on the gateway addresses.
- **`local_network` override:** the Proxmox firewall automatically allows SSH, the web UI and the console ports from the alias `local_network`, which it detects as the network of the host's address. On a rented server that is the provider's public subnet, shared with other customers. The alias is overridden at datacenter level with the host's own address, as the Proxmox documentation recommends for a single host on a public network, and checked with `pve-firewall localnet`. The external scan cannot reveal this gap, because it runs from outside that subnet.
- **Forwarded traffic is not filtered yet (accepted until L1).** NAT needs forwarding on the public bridge, so a machine in the provider's network segment that routes 10.42.0.0/16 through the host could reach VMs. The default backend cannot filter forwarded traffic, and the nftables backend that can is a tech preview. Until L1 no VM exists except throwaway test VMs; L1's firewall role closes it (from the public bridge into VNets, only replies to connections the VMs opened) before the first VM that is not a throwaway test VM. The same mechanism later enforces zone isolation (L6/L8).
- **External check:** scan the public address from outside the tailnet, for example from the laptop with Tailscale stopped. 22 and 8006 must be closed, and no port may lead to a VM.
- **Also in this step:**
  - `PasswordAuthentication no` in `sshd`;
  - TOTP on `root@pam`. Recovery keys are optional: a lost second factor is removed over SSH or the out-of-band console with `pveum user tfa delete root@pam`, both of which work without the web UI;
  - `unattended-upgrades` for Debian security updates only. Proxmox package upgrades stay manual, so that a kernel or `pve-*` upgrade never reboots or breaks the host unattended.

*Alternative:* bind `pveproxy` and `sshd` to the tailnet address only. That is defence in depth, but if `tailscaled` starts late, the services fail to bind at boot, and the result is easy to misread. The firewall is the single, visible control. Binding can be reconsidered in L1.

### D8. Runbook and adapter layout
- `docs/runbooks/lab-host.md`, "Bootstrap":
  - a short preamble with the rope rule and the "never on the provider network with a VM MAC" rule;
  - steps 1–5 fully written, with step 2 as acceptance criteria and step 3 including the address plan's *why* and the isolation caveat;
  - steps 6–7 stay as skeletons pointing to L0b-2. Step 8 stays at L9+.
- `providers/ovh/lab-host.md`:
  - `oob`: how to open the console and rescue mode, and how to test them. L0b-2 adds the recovery usage;
  - `install`: template options, the path chosen, and the template audit;
  - `network`: interface layout, gateway particulars and IPv6 delivery. The monitoring mode and the monitoring IP caveat are noted here and moved to `edge-firewall` in L0b-2.

## Risks / Trade-offs

- [Locked out while enabling the firewall or editing the network] → The IPMI console is verified first and kept open during risky steps. In the worst case, reinstalling from the template takes minutes, since nothing of value is on the host.
- [The template leaves the boot loader on only one disk] → The acceptance criteria catch it. Either fix it by hand and document how, or switch to the ISO fallback (D3). The boot test itself is in L0b-2.
- [The template root is on `md` RAID] → Accepted if the acceptance criteria pass. The host OS then has no ZFS snapshots: risky host changes are tried on a throwaway VM first, and reinstalling is cheap.
- [OVHcloud monitoring sends a technician during bootstrap] → Set monitoring to "without proactive intervention" during reconnaissance (D2). L0b-2 decides the long-term mode together with the edge firewall.
- [SDN DHCP/IPAM integration or its DNS handling behaves differently than documented] → The throwaway VM checks it in this change. Fallback: an IPAM reservation plus static configuration through cloud-init, which L2 uses anyway.
- [Tailscale SaaS outage or account problem] → The out-of-band console is the break-glass path. The overlay sits behind an open question (Headscale), not a hard dependency in the specs.
- [Tagged auth key leaks] → Single-use and short-lived. It is generated at join time and never stored. Revoking the node in the admin console cuts access.
- [No disk alerts until L0b-2] → Accepted and recorded in the proposal. Nothing on the host needs protecting yet.
- [The firewall's automatic rules trust the provider's public subnet] → `local_network` override, checked with `pve-firewall localnet` (D7).
- [A neighbour in the provider's network routes traffic into a VNet] → Accepted until L1 and recorded in the proposal; L1 closes it before the first real VM (D7).
- [A provider template re-applies its configuration at boot] → Anything that does so, such as cloud-init reading a config drive, is disabled during the audit (D4).
- [Private addresses (10.42.x) in the repository trip the IP-pattern secrets check] → The check excludes RFC 1918 ranges explicitly instead of dropping the IP check altogether.

## Migration Plan

This change acts on a bare host and on documents.
- **Rollback for the host:** reinstall from the template.
- **Rollback for the documents:** revert the commit.
- **Order of work across sessions:**
  - session 1: reconnaissance, install, audit, base settings;
  - session 2: SDN and the test VM, Tailscale, lockdown and the external check;
  - session 3, if needed: documents, validation, and the owner's read-through.

## Open Questions

- ~~Does the template's installer offer a ZFS root?~~ Resolved 2026-10-11: yes, and it is the default (see D3, outcome).
- Is the IPv6 /64 on-link or routed to the host? It only matters when VMs get IPv6, which is out of scope.
- Encryption at rest of LAB disks: the Proxmox installer does not support an encrypted root, and unlocking after each reboot would go through the console. It is recorded as an open question in the architecture, not decided here.
- L5 (a static runner for private repositories) comes before isolation is enforced. Whether the owner's own CI code counts as "trusted" under `lab-network` is decided when L5 is explored.
