# Proposal

## Why

The LAB ladder cannot start without a LAB host, and the architecture describes only generic options for choosing one. Exploring step L0 surfaced decisions and gaps that the documents do not capture yet:
- a provider must pass hard filters such as out-of-band access, a way to install Proxmox, and VM networking with one public IP;
- 2026 hardware prices make the "1 TB minimum disk" requirement unrealistic in a typical budget;
- LAB should not share a provider account with PROD.

There is also no convention for keeping the repository vendor-agnostic while still documenting provider-specific steps. This change records the selection method and the decision, and gets the host ordered, so that L0b (bootstrap) can start on real hardware.

## What Changes

- Split roadmap item L0 into two sequential changes, which respects the WIP limit:
  - **L0a**, `lab-host-selection` (this change): choose and order the host.
  - **L0b**, `lab-host-bootstrap`: OOB access, Proxmox, networking, overlay and lockdown.
- New convention, **provider adapters**:
  - Generic runbooks stay provider-free and define named *slots* for provider-specific steps.
  - Each provider gets a public folder `providers/<name>/` that fills those slots.
  - Real deployment values (IPs, hostnames, account IDs) never enter the repository.
- New generic runbook `docs/runbooks/lab-host.md`:
  - a complete "Selection" section: budget, hard filters, ranking, latency measurement method, provider independence from PROD, billing term;
  - a skeleton of the bootstrap slots, to be filled by L0b.
- New provider adapter `providers/ovh/lab-host.md`. It fills the `order` slot: which offer lines pass the filters, the billing options, and what to check before ordering. It records that the reference deployment uses an OVH Rise-S class server for LAB.
- `docs/architecture.md` updates:
  - Hardware table: minimum disk becomes 512 GB usable on mirrored NVMe; 1–2 TB stays comfortable.
  - New principle: LAB and PROD live at different providers, or at least on different accounts, to avoid a shared point of failure.
  - VM networking defaults to NAT behind the host's single public IP, with a warning about provider MAC filtering.
  - The IDE open question is resolved: JetBrains Gateway is primary, VS Code Remote-SSH is the latency-tolerant fallback.
  - "Choosing where to host LAB" links to the runbook's hard filters.
  - The open question on deployment-specific configuration is resolved by the adapter convention.
- `openspec/config.yaml` context: clarify that provider names are allowed only under `providers/`. Architecture docs and generic runbooks stay provider-free, and real IPs and hostnames stay forbidden everywhere.
- **Manual (owner only):** order the LAB host and confirm the delivered hardware matches the hard filters.

Out of scope:
- installing Proxmox, overlay network, firewall, and any other bootstrap slot (L0b);
- where provider-specific *code* (OpenTofu modules, Ansible vars) will live (decided at L1/L3);
- adapters for providers not used by the reference deployment, including Hetzner for PROD, which P0/P2 will add;
- a second LAB node.

## Capabilities

### New Capabilities
- `lab-host`: properties the LAB host must have — hardware baseline, out-of-band access, mirrored NVMe storage, independence from the PROD provider — and how a host is selected against them.
- `provider-adapters`: how provider-specific knowledge is isolated from the vendor-agnostic core — slots in generic runbooks, `providers/<name>/` folders, and no deployment values in the repository.

### Modified Capabilities
<!-- None: the project has no specs yet. -->

## Impact

- **Repository:**
  - new `docs/runbooks/`, `providers/README.md` and `providers/ovh/`;
  - edits to `docs/architecture.md`, `docs/roadmap.md` and `openspec/config.yaml`.
  - The change is documentation only.
- **Cost:** a LAB host at about 65 EUR/month excluding VAT. Month-to-month billing adds a setup fee of one month's rent; a 12-month prepay removes it. The budget is 50–100 EUR/month.
- **External services:** an account at the LAB provider, which is a different provider from PROD's.
- **Secrets:** provider account credentials are bootstrap-tier and live in the password manager. Nothing is committed.
- **Follow-ups:**
  - L0b starts after the host is delivered and this change is archived.
  - P0 and P2 will add a provider adapter for PROD's provider following the same convention.
