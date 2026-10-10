# Tasks

## 1. Provider adapter convention

- [x] 1.1 Update the provider rule in the `context` of `openspec/config.yaml` as described in design D8:
  - provider names appear only under `providers/` and in change artifacts;
  - architecture and generic runbooks stay provider-free;
  - real IPs, hostnames and account IDs are forbidden everywhere.

  *Why:* future artifacts must not treat the new folder as a rule violation.
  *Verify:* `openspec context --json` still resolves the root, and the new wording is visible in `openspec instructions proposal --change lab-host-selection --json`.
- [x] 1.2 Create `providers/README.md`. It explains:
  - generic runbooks in `docs/runbooks/` declare named slots;
  - adapters live in `providers/<name>/<runbook>.md`, address every slot by name, and mark uncovered slots with the change expected to fill them;
  - placeholders such as `<lab-host>` replace deployment values;
  - volatile facts carry a check date and a source.

  It also lists the reference deployment's providers by role:
  - LAB → `ovh`;
  - PROD → Hetzner (adapter pending, P0/P2);
  - backups → TBD in `prod-offsite-backups`.

  *Verify:* every requirement of the `provider-adapters` spec is reflected in the README.

## 2. Generic LAB host runbook: selection

- [x] 2.1 Create `docs/runbooks/lab-host.md` with a "Selection" section that walks a newcomer through:
  1. Setting a budget, and stating whether it includes VAT.
  2. Applying the hard filters from the `lab-host` spec, each with a one-line "why".
  3. Ranking the remaining candidates by price, single-thread CPU, ECC, RAM headroom, then latency.
  4. Measuring latency with `ping`/`mtr` to provider test hosts, at several times of day, recording average and jitter, with the IDE comfort thresholds as rules of thumb.
  5. Checking independence from the PROD provider.
  6. Choosing a billing term.
  7. Recording the decision in the provider adapter.

  *Verify:* every `lab-host` spec requirement maps to a step.
- [x] 2.2 Add a "Bootstrap" skeleton to the same runbook. It declares the slots `order`, `oob`, `install`, `network`, `edge-firewall`, `recovery` and `private-network`. Each slot gets the question it must answer and a pointer to `providers/README.md`. Steps that do not depend on the provider (overlay network, locking down the web UI) are listed as headings to be written in L0b.

  *Verify:* every slot has a stable name and a question.
- [x] 2.3 Check the runbook for vendor neutrality and deployment values. Search it for the names of all providers mentioned in `design.md` and for IPv4-like patterns, e.g. `grep -niE 'ovh|hetzner|ionos|scaleway|contabo|netcup|[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' docs/runbooks/lab-host.md`.

  *Verify:* the search finds nothing.

## 3. OVH adapter

- [x] 3.1 Create `providers/ovh/lab-host.md` and fill the `order` slot:
  - offer lines that pass the hard filters (Rise-S, RISE-1, SYS-GAME-2) with their trade-offs;
  - billing options (month-to-month with setup fee vs 12-month prepaid);
  - what to re-check in the order flow (price, stock per datacenter, disk configuration);
  - the reference deployment's choice and why.

  Every price and stock statement carries the date it was checked (2026-10-10) and a source link.

  *Verify:* no undated price appears in the file.
- [x] 3.2 In the same adapter, list the remaining slots (`oob`, `install`, `network`, `edge-firewall`, `recovery`, `private-network`) as "not yet covered: L0b" (`private-network`: "L9+"). Each includes the unverified research notes from `design.md` → "Notes for L0b", clearly labelled as unverified.

  *Verify:* every slot from `docs/runbooks/lab-host.md` appears in the adapter by its exact name.

## 4. Architecture and roadmap

- [x] 4.1 Update `docs/architecture.md`:
  - Hardware table: the disk minimum becomes "512 GB usable on mirrored NVMe"; 1–2 TB stays comfortable.
  - New core principle: LAB and PROD are hosted at different providers, or at least under different accounts.
  - LAB host table: VMs use NAT behind the host's single public IP, with a note on provider MAC filtering.
  - "Choosing where to host LAB": link to the runbook's hard filters, and note that latency can be a tie-breaker.
  - Open Questions: resolve the IDE question (Gateway primary, VS Code Remote-SSH fallback) and the deployment-specific configuration question (the provider adapter convention).
  - Bump "Last updated".

  *Verify:* the `grep` from 2.3 run against `docs/architecture.md` finds nothing.
- [x] 4.2 Update `docs/roadmap.md`:
  - split L0 into L0a (`lab-host-selection`, `in progress`) and L0b (`lab-host-bootstrap`, `todo`, depends on L0a);
  - keep L1's dependency pointing at L0b;
  - mention under "Current Parallelism" that L0a and L0b run in sequence because of the WIP limit;
  - bump "Last updated".

  *Verify:* the ladder table renders, and both rows reference their change names.

## 5. Order and accept the host (owner only)

- [x] 5.1 **Owner:** open the provider's order flow and re-check the chosen offer's price, CPU, RAM, disk configuration (2 NVMe), port speed and stock in an EU datacenter. Choose the billing term. If anything differs materially from the adapter, update the adapter's `order` slot with the new date before ordering.

  *Verify:* the adapter's facts match the order flow on the order date.
- [x] 5.2 **Owner:** place the order. Store the provider account credentials and any server access details in the password manager (bootstrap tier). Commit nothing that identifies the server.

  *Verify:* order confirmation received, and the password manager entry exists.
- [x] 5.3 **Owner:** when the server is delivered, check it against the hard filters in the provider's control panel: CPU, RAM, two NVMe devices, traffic, and IPMI/KVM and rescue options present. If something does not match, raise it with the provider before L0b.

  *Verify:* every hard filter in the `lab-host` spec is confirmed on the delivered host. Write a short note (no deployment values) in the adapter's `order` slot about anything surprising in the order or delivery process.

## 6. Final checks

- [x] 6.1 Run the neutrality and secrets search from 2.3 across `docs/` and `providers/`.

  *Verify:* provider names appear only under `providers/`, and no IPs or hostnames of the deployment appear anywhere.
- [x] 6.2 Run `openspec validate lab-host-selection --strict`.

  *Verify:* it passes.
- [x] 6.3 **Owner:** read `docs/runbooks/lab-host.md` top to bottom as a newcomer would.

  *Verify:* every step is understandable without this change's design notes. Any gaps are fixed before archiving.
