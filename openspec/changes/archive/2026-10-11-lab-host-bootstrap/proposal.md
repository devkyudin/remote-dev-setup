# Proposal

## Why

The LAB host has been ordered and delivered (L0a), but it is bare: no OS, no hypervisor, no network design, and no safe way to manage it. Every later LAB step (L1 Ansible, L2 dev VM, L4 platform) needs a Proxmox host that is installed on mirrored disks, gives VMs internet access through NAT, and exposes nothing to the internet except what is strictly needed. Doing this by hand first is the "manual first, then code" step: it shows what L1 must automate and which invariants automation must keep.

Exploration split the original L0b into two sequential changes so each stays verifiable within 1–3 sessions:
- **L0b-1, `lab-host-bootstrap` (this change):** the host is installed and safe to leave on the internet.
- **L0b-2, `lab-host-recovery` (next):** the provider edge firewall, disk alerts, recovery drills, and a rebuild exam.

## What Changes

- **Reconnaissance before any change:** test the out-of-band console and rescue system, inventory the delivered disks and network, check the installer's storage options, and set the provider's monitoring so that bootstrap reboots do not trigger a technician visit.
- **Install Proxmox VE 9 from the provider's template**, with an SSH key instead of a password. The generic runbook defines *acceptance criteria* for a correct install (both NVMe devices mirrored, boot loader on both disks, ARC capped if ZFS is used, key-only login, no-subscription repository), so a reader at any provider can check their own install the same way.
- **Audit the template:** record what the provider's template changed compared with a stock Proxmox install (partitioning, network config, repositories, extra packages, authorized keys). This is the starting point L1 automates from.
- **VM networking with Proxmox SDN:** a documented address plan for all LAB zones (dev, platform, runners, sandbox, test) from a per-node private range, a Simple zone with SNAT and DHCP/IPAM, and only the `dev` and `test` VNets created now. A throwaway VM proves NAT works and that no VM MAC address appears on the provider network.
- **Overlay network:** the host joins a Tailscale tailnet as a tagged node. This provisionally resolves the "Tailscale or Headscale" open question.
- **Lockdown:** the Proxmox web UI/API and SSH become reachable only over the overlay network, with 2FA on the web UI and password login disabled. Break-glass access is the out-of-band console. Every access path is verified before the previous one is closed.
- **Generic runbook** `docs/runbooks/lab-host.md`: write bootstrap steps 1–5 and the "verify the new path before closing the old one" rule. Steps 6–7 stay as skeletons for L0b-2.
- **OVH adapter** `providers/ovh/lab-host.md`: fill the `install` and `network` slots and the access-testing part of `oob` with verified content.
- **Architecture and roadmap:**
  - `docs/architecture.md`: add the LAB zone address plan, record Tailscale as the provisional overlay choice, and add encryption at rest of LAB disks as an open question.
  - `docs/roadmap.md`: split L0b into L0b-1 and L0b-2. L1 now depends on L0b-2, because automating a firewall before recovery has been practised turns every Ansible mistake into an emergency.

Out of scope (L0b-2 or later):
- the provider edge firewall, disk and array alerting, recovery drills, boot-from-either-disk test and the rebuild exam (L0b-2);
- zone isolation between VNets (L6/L8);
- the host as a Tailscale subnet router, and how the laptop reaches VMs (L2);
- IPv6 for VMs;
- encryption at rest;
- any automation (L1), including filtering forwarded traffic from the public bridge into VNets, which L1 does before the first real VM.

## Capabilities

### New Capabilities
- `lab-network`: how LAB VMs are addressed and connected — the per-node zone address plan, egress through NAT on the host, address assignment, no inbound exposure to the internet, and the explicit status of isolation between zones.

### Modified Capabilities
- `lab-host`: adds requirements for the bootstrapped host — an install that meets the acceptance criteria, a recorded template baseline, a management plane reachable only over the overlay network, and key-only administrative access.

## Impact

- **LAB host:** installed with Proxmox VE 9, SDN zone and VNets, Tailscale, and host firewall rules. No workloads yet besides a throwaway test VM.
- **Repository:** documentation only — the generic runbook, the OVH adapter, the architecture, the roadmap, and specs. No code.
- **External services:** a Tailscale account (free plan). The tailnet's identity login is bootstrap-tier and lives in the password manager.
- **Secrets:** the root SSH key, the root password and the IPMI access path are bootstrap-tier and live in the password manager; the TOTP secret lives in an authenticator. Nothing is committed. Real IPs, hostnames and tailnet names never enter the repository.
- **Accepted risks:**
  - between this change and L0b-2, a failed disk raises no alert. The host holds nothing of value yet, so this is recorded rather than fixed here;
  - until L1, forwarded traffic from the public bridge into VNets is not filtered, so a machine in the provider's network segment could route traffic to VMs. No VM except throwaway test VMs exists before L1, which closes it (design D7).
- **Follow-ups:**
  - L0b-2 (`lab-host-recovery`) is proposed when this change is close to done, because reconnaissance may change its plan.
  - L2 decides how the laptop reaches VMs: either the host acts as a subnet router, or each VM joins the tailnet.
