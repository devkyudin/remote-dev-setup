# Design

## Context

- The repository has only target architecture and roadmap documents, and no specs. The architecture already lists options for hosting LAB and a hardware table, but it does not define hard filters for providers.
- PROD runs on a small VPS at an existing provider (Hetzner).
- Roadmap WIP limit is 2 active changes. `prod-offsite-backups` already takes one slot.
- Market snapshot (2026-10-10): RAM and NVMe prices pushed several providers to raise dedicated server prices in 2026. With 64 GB RAM at 50–100 EUR/month, two mirrored NVMe devices usually come as 2x512 GB, not 1 TB+.
- Constraints from the owner, agreed during exploration:
  - LAB is rented; own hardware at home is out (uplink, PROD telemetry going to a home connection, upfront cost);
  - budget is 50–100 EUR/month;
  - IDE is JetBrains Gateway (Rider), with VS Code Remote-SSH as fallback;
  - price and hardware come first, latency second;
  - the repository stays vendor-agnostic, but provider-specific knowledge may be public in separate folders.

## Goals / Non-Goals

**Goals:**
- A selection method anyone can repeat with their own budget, location and provider.
- A concrete, justified choice for the reference deployment, and the host ordered.
- A slot-based adapter convention that L0b, P0 and P2 can extend without restructuring.

**Non-Goals:**
- Writing the content of bootstrap slots (`oob`, `install`, `network`, `edge-firewall`, `recovery`, `private-network`). L0b does that on the real host.
- Comparing providers for PROD or backup storage.
- Automating any part of selection or ordering.

## Decisions

### D1. Split L0 into L0a (selection) and L0b (bootstrap), run sequentially
Selection is documentation work that fits in one session. Bootstrap is hands-on work on a live host that takes 1–3 sessions. Splitting keeps each change verifiable, and running them in sequence respects the WIP limit while P0 is active.
*Alternative:* one L0 change. It would stay open across the ordering wait and mix document review with hands-on work.

### D2. Hard filters first, then ranking
The hard filters are the `lab-host` spec requirements:
- hardware baseline;
- out-of-band access;
- Proxmox installable;
- NAT with one public IP;
- traffic;
- independence from the PROD provider.

Candidates that pass are ranked by price, then single-thread CPU performance, ECC, RAM headroom and latency. Single-thread performance ranks above ECC and RAM because Gateway backend responsiveness and .NET builds depend on it most.

For its own search, the reference deployment raised the RAM filter to 64 GB. The architecture minimum stays at 32 GB, but 32 GB is tight from day one with an IDE backend and CI.

### D3. Latency is a tie-breaker, not a filter
All candidate datacenters measured from the owner's usual location averaged 64–76 ms, with similar jitter. The differences between them were within noise, so latency cannot separate the candidates.

At that latency Gateway works, but with noticeable lag. The fallback is VS Code Remote-SSH, which tolerates latency better.

The runbook keeps the measurement method: `ping`/`mtr` to each provider's test hosts, at several times of day, recording average and jitter. Distance matters more for other readers than it did here.

### D4. Accept 512 GB usable storage and lower the architecture minimum
At 64 GB RAM within budget, 2x1 TB NVMe costs either about 20 EUR/month more or a much older CPU. 512 GB covers:
- the dev VM;
- CI caches;
- the platform with bounded retention;
- VM templates.

The architecture table changes to "minimum 512 GB usable on mirrored NVMe; comfortable 1–2 TB".
*Alternative:* keep 1 TB as the minimum. That would force SYS-GAME-2 (Zen 2, 500 Mbps) or go over budget.

### D5. Reference LAB host: OVH Rise-S class (Ryzen 7 9700X, 64 GB DDR5, 2x512 GB NVMe)
The price is 64.99 EUR/month excluding VAT; the snapshot was taken on 2026-10-10. The deciding factors:
- it has the best single-thread CPU in budget;
- it includes IPMI/KVM;
- a Proxmox VE 9 install template exists;
- traffic is unmetered at 1 Gbps;
- OVH said its Rise range is not affected by its H2 2026 price increase.

Trade-offs accepted:
- no true ECC (DDR5 on-die ECC only);
- no vRack (private network) on this model, so a second node will connect over the overlay network instead.

Alternatives considered (prices excluding VAT, 2026-10-10):

| Candidate | Why not chosen |
|---|---|
| OVH SYS-GAME-2 (Ryzen 7 3800X, 64 GB ECC, 2x960 NVMe, 64.99) | Same price, more disk and real ECC, but a Zen 2 CPU (2019) and a 500 Mbps port. CPU matters more here. |
| OVH RISE-1/RISE-2 with 2x960 NVMe (85–93) | ECC, vRack and 1 TB, but at or over budget including VAT, with a slower CPU. |
| Hetzner AX42-1-LTD (Zen 4, 64 GB, 2x512, 77.30 + 39 setup) | PROD's provider. Stock-limited "-LTD" tier. KVM console now appears to be paid. |
| Hetzner server auction (i7-7700 / Xeon E3, 64–71) | PROD's provider. CPUs from 2016–17. |
| IONOS AR8-64 NVMe (Zen 2, ECC, 2x960, 87.39) | 12-month term to avoid the setup fee. Over budget including VAT. Proxmox install path not verified. |
| Scaleway Dedibox Pro (Xeon E3/E5, SATA SSD, 56–66) | SATA SSDs and older CPUs. |
| netcup root server | Virtual machine, not bare metal. Nested virtualization reportedly unavailable. |
| Contabo, Cherry Servers, Leaseweb | Over budget, or 2026 EUR prices could not be verified. |

A Hetzner host under a separate account would satisfy the spec, but a different provider removes the shared failure domain entirely, and the owner chose that.

### D6. Adapter layout: top-level `providers/<name>/<runbook>.md` + `providers/README.md`
Generic runbooks live in `docs/runbooks/<runbook>.md` and declare slots. Each adapter file mirrors a runbook name: `providers/ovh/lab-host.md` fills the slots of `docs/runbooks/lab-host.md`. The index lists adapters and the reference deployment's provider for each role.

Alternatives considered:
- *Private overlay repo:* rejected. The owner does not mind the provider being public, and a private repo hides the most useful part from readers.
- *Placeholders only:* rejected. Readers would get no worked example.
- *`docs/providers/`:* possible, but a top-level folder leaves room for provider-specific code to sit next to the docs later. Where that code lives is decided at L1/L3.

### D7. Slot set for `lab-host`
The slots are `order`, `oob`, `install`, `network`, `edge-firewall`, `recovery` and `private-network`. This change fills `order` only. Each other slot states its question and is marked "not yet covered: L0b" (or "L9+" for `private-network`).

### D8. Clarify the provider rule in `openspec/config.yaml`
Today the rule says provider names must not appear in architecture docs. It becomes:
- provider names appear only under `providers/` and in change artifacts;
- architecture and generic runbooks stay provider-free;
- real IPs, hostnames and account IDs are forbidden everywhere.

This keeps future artifacts from tripping over the new folder.

### D9. Billing term is chosen by the owner at order time
- Month-to-month adds a setup fee equal to one month's rent.
- A 12-month prepay has no setup fee and is 5% cheaper, about 100 EUR less in the first year, but it locks the choice in.

Recommendation: month-to-month for the first host, because L0b may reveal problems (for example, ZFS install) that make switching worthwhile. The adapter documents both options.

## Risks / Trade-offs

- [Stock: Rise-S was in stock only in two French datacenters on 2026-10-10] → Order from any EU datacenter that has it; latency differences are within noise. Fallback: RISE-1 (2x512 NVMe, ECC, vRack) at the same price with a slower CPU.
- [Prices and stock change; the snapshot came from web research and catalog APIs] → The order task re-checks price, configuration and stock in the provider's order flow before paying.
- [Budget basis is unclear: prices above exclude VAT] → Rise-S stays within budget either way (about 78 EUR including 20% VAT).
- [512 GB fills up] → L4 sets bounded retention for observability. Disk usage becomes one of the first metrics. Moving to a larger host is a documented exit.
- [No true ECC with ZFS] → Accepted for a LAB that is "ok to break". Important data (PROD backups) does not depend on LAB.
- [ZFS mirror through the provider's Proxmox template is unverified] → L0b falls back to installing the official Proxmox ISO through IPMI.
- [Gateway lag at 60–75 ms] → VS Code Remote-SSH fallback. The dev VM supports both from L2.
- [No vRack on Rise-S] → A second node (L9+) joins over the overlay network, or a different model is chosen for it.

## Migration Plan

This change edits documents only. The one real-world action is ordering the host; its rollback is cancelling the server. With month-to-month billing, that costs at most the setup fee plus one month.

## Open Questions

- Does the 50–100 EUR budget include VAT? This does not change the choice and only affects how much headroom remains.

## Notes for L0b (research, not yet verified on the host)

Collected 2026-10-10 from OVH documentation and catalog. L0b verifies them and moves them into the adapter's slots:
- `oob`: rescue mode on all ranges, IPMI/KVM included on Rise.
- `install`: an official Proxmox VE 9 template exists. Reports conflict on whether its RAID1 option gives a ZFS mirror or md-RAID with LVM. The fallback is installing the ISO through IPMI.
- `network`: an additional IPv4 costs 1.99 EUR/month and needs a virtual MAC per IP when bridged. NAT behind the main IP needs neither. A /64 IPv6 block is included.
- `edge-firewall`: a stateless network firewall, IPv4 only, up to 20 rules per IP. Anti-DDoS is always on.
- `private-network`: vRack exists on most Rise models but not on Rise-S.
