# Provider Adapters

This repository is provider-agnostic: the [architecture](../docs/architecture.md) and the generic runbooks in [`docs/runbooks/`](../docs/runbooks/) never name a hosting provider. But some steps are different at every provider: how you order a server, how you reach its console, how you install an OS on it. That knowledge lives here, in **provider adapters**.

## How it works

### Generic runbooks declare slots

A generic runbook describes *what* to do and *why*. When it reaches a step that depends on the provider, it does not explain it. Instead it declares a **slot**: a stable name (for example `oob`) and the question that slot must answer (for example "How do I reach a console when the host is unreachable over the network?").

Slot names are stable, so adapters can refer to them. Renaming a slot means updating every adapter that fills it.

### Adapters fill slots

Each provider gets a folder, `providers/<name>/`. Inside it, one adapter file mirrors each generic runbook it supports:

```
docs/runbooks/lab-host.md         <- generic: steps + slot declarations
providers/<name>/lab-host.md      <- provider-specific: answers to those slots
```

Rules for an adapter file:

- It addresses **every** slot of its runbook by its exact name, as a heading.
- A slot that has no content yet is listed as **not yet covered**, together with the change or roadmap item expected to fill it (for example "not yet covered: L0b").
- Notes that have not been checked on a real host are labelled **unverified**.

To add another provider, create `providers/<name>/` and fill the same slots. The generic runbook does not change.

### No deployment values

The repository is public. Adapters and runbooks never contain the real values of any deployment:

- no IP addresses or hostnames;
- no account, customer or project IDs;
- no order, invoice or contract numbers.

Use placeholders instead, such as `<lab-host>`, `<lab-host-ip>` or `<provider-account>`. The real values live in the password manager (bootstrap tier, see [Secrets by Tier](../docs/architecture.md#secrets-by-tier)) or are reached by overlay network names.

### Volatile facts are dated

Prices, offer names, stock and similar facts change over time. Every such fact in an adapter states **the date it was checked** and **links to its source**, so a reader knows when to re-check it. Example:

> 64.99 EUR/month excl. VAT (checked 2026-10-10, [source](https://example.com/offer))

## Adapters

| Provider | Adapter files | Notes |
|---|---|---|
| `ovh` | [`lab-host.md`](ovh/lab-host.md) | `order`, `oob` (access), `install` and `network` filled; `edge-firewall` and `recovery` pending L0b-2; `private-network` pending L9+ |

## Reference deployment

Which providers the reference deployment (the owner's own setup) uses, by role. You do not have to use the same ones: pick yours by following the generic runbooks.

| Role | Provider | Adapter |
|---|---|---|
| LAB | `ovh` | [`ovh/lab-host.md`](ovh/lab-host.md) |
| PROD | Hetzner | pending, added by P0/P2 (see [roadmap](../docs/roadmap.md)) |
| Off-site backups | TBD | decided in the `prod-offsite-backups` change |

LAB and PROD are deliberately at different providers, so that one account or billing problem cannot take down both (see [lab-host runbook → Independence from PROD](../docs/runbooks/lab-host.md#5-check-independence-from-the-prod-provider)).
