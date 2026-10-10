# Tasks

Tasks marked **Owner** run on the live host or in provider and tailnet consoles with owner credentials. Agents guide and review these steps but never execute them. All other tasks are documentation work.

## 1. Roadmap and architecture framing

- [x] 1.1 Update `docs/roadmap.md`:
  - split row L0b into **L0b-1** (`lab-host-bootstrap`, `in progress`) and **L0b-2** (`lab-host-recovery`, `todo`, depends on L0b-1);
  - L0b-1 covers reconnaissance, install, VM networking, overlay and lockdown;
  - L0b-2 covers the edge firewall, disk alerts, recovery drills and the rebuild exam;
  - change L1's dependency to L0b-2, and explain why in one line (practise recovery before automating the firewall; design D1);
  - update "Current Parallelism" and bump "Last updated".

  *Verify:* the ladder table renders, both new rows name their change, and L1 says it depends on L0b-2.
- [x] 1.2 Update `docs/architecture.md` → LAB Host:
  - add the zone address plan from design D5: per-node range, a subnet per zone, and the reserved LAB-2 range;
  - give the *why* behind 10.42/16;
  - state explicitly that VNets do not isolate zones until firewall policy is enforced (L6/L8), and that agent workspaces and ephemeral runners wait for that.

  *Verify:* every requirement of the `lab-network` delta spec that concerns documentation is satisfied by the text.
- [x] 1.3 Update `docs/architecture.md` → Open Questions:
  - mark "Tailscale or Headscale" as provisionally resolved: Tailscale now as a tagged node, Headscale remains an option;
  - add "encryption at rest of LAB disks" with the threat (rented disks are reused by the provider) and the obstacle (no encrypted root in the installer; unlocking needs the console after each reboot);
  - bump "Last updated".

  *Verify:* a provider-name `grep` (see 7.1) on `docs/architecture.md` finds nothing.
- [x] 1.4 Record the forwarding gap (design D7) where later steps will see it:
  - `docs/roadmap.md`, row L1: the firewall role closes forwarding from the public bridge into VNets (only replies to connections the VMs opened) before the first VM that is not a throwaway test VM;
  - `docs/architecture.md`, zone address plan: the same caveat next to "no inbound exposure".

  *Verify:* both documents state the gap and that L1 closes it, and name no provider.

## 2. Reconnaissance (no changes to the host)

- [x] 2.1 **Owner:** in the provider control panel, set server monitoring to "enabled without proactive intervention" for the duration of bootstrap.

  *Why:* reboots and closed ports must not send a technician to the server (design D2).
  *Verify:* the panel shows the new mode.
- [x] 2.2 **Owner:** open the IPMI/KVM console from the control panel. Note its type (HTML5 or Java) and check that the keyboard works.

  *Verify:* the console shows the server's screen, and a keypress reaches it.
- [x] 2.3 **Owner:** boot the rescue system and connect over SSH. Record, without serial numbers, IPs or hostnames:
  - disk models, wear and power-on hours (`lsblk`, `nvme list`, `smartctl -a /dev/nvme0n1`, `smartctl -a /dev/nvme1n1`);
  - the interface name, netmask and gateway pattern (`ip a`, `ip r`);
  - how the panel presents the IPv6 /64.

  *Verify:* both NVMe devices report SMART status PASSED, and the notes contain no deployment values.
- [x] 2.4 **Owner:** in the OS install dialog for the Proxmox VE 9 template, open the partitioning options **without starting the install**. Record which filesystem and RAID level each partition can use, and whether ZFS can be chosen for root.

  *Verify:* the notes answer the D3 question "can the root be ZFS?".
- [x] 2.5 Write the `oob` slot in `providers/ovh/lab-host.md`:
  - how to open the IPMI/KVM console and the rescue system;
  - how to test them before changing anything;
  - what you see in each.

  Replace the unverified notes. Mark the recovery usage "not yet covered: L0b-2". Write bootstrap step 1 in `docs/runbooks/lab-host.md`, in generic form: why the console is tested first, and what "working" means.

  *Verify:* the slot has no "unverified" label left for access, and runbook step 1 names the `oob` slot and no provider.

## 3. Install, audit and base settings

- [x] 3.1 Write bootstrap step 2 in `docs/runbooks/lab-host.md` as **acceptance criteria** (design D3):
  - Proxmox VE 9;
  - system and VM storage redundant on both NVMe devices;
  - a boot loader on each device;
  - ARC capped if ZFS is used;
  - SSH key login only;
  - no-subscription repository on, enterprise repository off;
  - 2FA on `root@pam`.

  Give each criterion the command that checks it, for example `cat /proc/mdstat`, `zpool status`, `proxmox-boot-tool status` or `efibootmgr -v`, and `cat /sys/module/zfs/parameters/zfs_arc_max`.

  *Verify:* every criterion has a check command, and the step points to the `install` slot for *how*.
- [x] 3.2 **Owner:** install the Proxmox VE 9 template with the best storage option found in 2.4 (D3: a ZFS root if offered, otherwise the template default). Provide the SSH public key in the installer; set no password there, so none is sent by email.

  *Verify:* SSH login with the key works on the host's public address. This is temporarily allowed until group 6.
- [x] 3.3 **Owner:** before changing anything, run the template audit from design D4 and save the output privately.

  *Verify:* the audit covers storage layout, boot entries, network config, repositories, extra packages, services and `authorized_keys`.
- [x] 3.4 Write the `install` slot in `providers/ovh/lab-host.md`:
  - the template options;
  - the option chosen and why;
  - "what the template changes compared with stock Proxmox", from 3.3 with deployment values removed;
  - the ISO-through-console fallback as an alternative.

  Any unexpected key or agent is listed with its decision (removed, or kept with a reason).

  *Verify:* the `lab-host` requirement "Installed baseline is recorded" is satisfied by the slot alone.
- [x] 3.5 **Owner:** apply the base settings:
  - set a strong root password with `passwd` and store it in the password manager (D3);
  - disable anything from the template that re-applies configuration at boot, such as cloud-init (D4);
  - switch to the no-subscription repository and disable the enterprise repository (the Ceph one too, if present), then run `apt update && apt full-upgrade`;
  - cap the ARC at 8 GiB if ZFS is used (D3);
  - install `unattended-upgrades`, limited to Debian security updates;
  - add the `zfspool` storage `local-zfs` for VM disks and limit `local` to files (D3);
  - reboot once.

  *Verify:* after the reboot, `apt update` shows no 401 errors, `zfs_arc_max` is 8589934592 (if ZFS), `unattended-upgrades --dry-run` lists only security origins, `pvesm status` shows `local-zfs` active, and the out-of-band console accepts the root password.
- [x] 3.6 **Owner:** run every acceptance check from 3.1. If the boot loader is on only one disk, fix it and document the fix in the `install` slot, or fall back to the ISO path (D3).

  *Verify:* every acceptance criterion passes, except SSH-key-only and 2FA, which are completed in group 6.

## 4. VM networking

- [x] 4.1 Write bootstrap step 3 in `docs/runbooks/lab-host.md`:
  - the NAT model and the "no VM MAC on the provider network" rule;
  - a link to the zone address plan in the architecture;
  - a generic walkthrough of a Simple zone with SNAT, DHCP and IPAM;
  - the isolation caveat;
  - the throwaway-VM verification.

  The public interface details point to the `network` slot.

  *Verify:* the step names no provider, and it explains *why* every zone's subnet is planned before it is created.
- [x] 4.2 **Owner:** install the SDN DHCP dependency (`dnsmasq`, with its default service disabled, as the Proxmox SDN docs require). Create a Simple zone with IPAM `pve` and DHCP `dnsmasq`. Create the VNets `vndev` and `vntest` with the subnets from D5, gateway `.1`, SNAT on, a DHCP range, and the DHCP DNS server left empty (D5). Apply the SDN configuration.

  *Verify:* `Datacenter → SDN` shows the zone and both VNets as `available`, `ip a` on the host shows the VNet bridges with their gateway addresses, the SDN network file has a SNAT line for every subnet, and `ss -lunp` shows dnsmasq on port 53 only on the VNet gateways.
- [x] 4.3 **Owner:** create a throwaway VM in `vntest` (any small cloud image) with no static network configuration. Check that:
  - it gets an address over DHCP;
  - it resolves a name;
  - it fetches a page with `curl`.

  While it runs, capture `tcpdump -e -n -i <public-interface>` on the host and confirm that every frame from the VM's traffic carries the host's MAC address. Destroy the VM afterwards.

  *Verify:* the `lab-network` scenarios "Test VM reaches the internet" and "VM created without network configuration" both hold. If DHCP or DNS fails, apply the fallback from design Risks and record it.
- [x] 4.4 Write the `network` slot in `providers/ovh/lab-host.md`:
  - how the template configured the public interface (bridge, netmask and gateway pattern);
  - how the IPv6 /64 is delivered;
  - that NAT needs neither an additional IP nor a virtual MAC;
  - a note on the monitoring mode and the monitoring addresses that must stay allowed if ICMP is ever filtered. Link the provider's page; do not copy the address list.

  *Verify:* no "unverified" label remains for facts checked on the host, and the slot contains no deployment values.

## 5. Overlay network

- [x] 5.1 **Owner:** in the Tailscale admin console:
  - define `tag:lab` with the owner as tag owner;
  - write an ACL rule that lets only the owner's devices reach `tag:lab` on ports 22 and 8006;
  - generate a single-use, pre-authorized auth key with `tag:lab`.

  Do not store the key; use it immediately in 5.2. Store the tailnet's admin login in the password manager.

  *Verify:* the ACL preview shows owner devices → `tag:lab:22,8006` and nothing else into `tag:lab`.
- [x] 5.2 **Owner:** install Tailscale on the host from its official repository and join with the auth key.

  *Verify:* the admin console shows the host tagged `tag:lab` with key expiry disabled. From the laptop, SSH and `https://<lab-host>:8006` work over the tailnet name.
- [x] 5.3 Write bootstrap step 4 in `docs/runbooks/lab-host.md`:
  - why a tagged node (no key expiry, not tied to a person);
  - the ACL shape;
  - why OpenSSH rather than Tailscale SSH;
  - that the host is not a subnet router yet (L2).

  Use `<lab-host>` for the tailnet name.

  *Verify:* the step contains no real tailnet or host name and names no hosting provider.

## 6. Lockdown

- [x] 6.1 Write bootstrap step 5 in `docs/runbooks/lab-host.md`:
  - the "verify the new path before closing the old one" rule, with the order from design D7;
  - the firewall rules: public ICMP and UDP 41641 in; 22 and 8006 only on the overlay interface; DHCP and DNS in on the VNet bridges; input policy DROP;
  - the `local_network` override and its check;
  - that forwarded traffic is not filtered until L1 (D7);
  - `sshd` password login off;
  - TOTP on `root@pam`;
  - the external check.

  Explain why the firewall is preferred over binding services to the tailnet address.

  *Verify:* each `lab-host` requirement added by this change maps to a sentence in steps 2–5.
- [x] 6.2 **Owner:** enable TOTP for `root@pam` in the web UI over the tailnet. Recovery keys are optional (D7). Set `PasswordAuthentication no` in `sshd` and reload it.

  *Verify:* a web UI login without the TOTP code fails, and `ssh -o PubkeyAuthentication=no root@<lab-host>` is refused without a password prompt.
- [x] 6.3 **Owner:** with the IPMI console open in another window and SSH over the tailnet verified, add the firewall rules at datacenter and node level (including DHCP and DNS on `vndev` and `vntest`), override the `local_network` alias with the host's own address, and then enable the firewall.

  *Verify:* SSH and the web UI still work over the tailnet, the IPMI console still shows a login prompt, `pve-firewall localnet` shows only the host's own address, and a throwaway VM in `vntest` gets a lease and resolves a name with the firewall on.
- [x] 6.4 **Owner:** from outside the tailnet (for example the laptop with Tailscale stopped), scan the host's public address: `nmap -Pn -p 22,8006 <lab-host-ip>`, then a full TCP port scan.

  *Verify:* 22 and 8006 are filtered or closed, and no open port leads to a VM. This satisfies the scenarios "Connection attempt from the internet" and "Port scan of the public address".

## 7. Integration checks

- [x] 7.1 Run the neutrality and secrets search across `docs/` and `providers/`:
  - `grep -rniE 'ovh|hetzner|ionos|scaleway|contabo|netcup|tailscale\.(com|net)|ts\.net' docs/` must find only allowed mentions (a product name like Tailscale is fine; tailnet domains are not);
  - IPv4 literals outside private ranges: `grep -rnoE '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' docs providers | grep -vE ':(10\.|192\.168\.|172\.(1[6-9]|2[0-9]|3[01])\.|100\.(6[4-9]|[7-9][0-9]|1[01][0-9]|12[0-7])\.)'` must find nothing.

  *Verify:* provider names appear only under `providers/`, and no public IP, hostname or tailnet name of the deployment appears anywhere.
- [x] 7.2 Update the "Adapters" table in `providers/README.md`: `ovh` covers `order`, `oob` (access), `install` and `network`; `edge-firewall` and `recovery` are pending L0b-2.

  *Verify:* the table matches the adapter's slot headings.
- [x] 7.3 Run `openspec validate lab-host-bootstrap --strict`.

  *Verify:* it passes.
- [x] 7.4 **Owner:** read `docs/runbooks/lab-host.md` bootstrap steps 1–5 top to bottom as a newcomer at a different provider would.

  *Verify:* every step is understandable without this change's design notes, and every provider-specific step points to a slot. Fix any gaps before archiving.
