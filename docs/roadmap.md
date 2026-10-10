# Roadmap

> The backlog for moving toward the [target architecture](architecture.md). It will later move to GitHub Issues/Projects, where agents can pick up tasks too.
>
> Last updated: 2026-10-11 (L0b-1 done; L0b-2 next).

## How We Work

Every item goes through the OpenSpec cycle:

```
  roadmap item --> /opsx:explore --> /opsx:propose --> /opsx:apply --> /opsx:archive
                   (open questions   (proposal,        (implement)    (delta -> specs,
                    of the item)      specs, design,                   update status here)
                                      tasks)
```

- One item may split into several changes. A good change size: the result is verifiable within 1–3 sessions.
- Manual steps (a runbook rather than system behavior) become a change without specs, or just a document.
- **WIP limit: no more than 2 active changes.**
- If we learn something that changes the picture, update [architecture.md](architecture.md).

Statuses: `todo` · `explore` · `proposed` · `in progress` · `done`

## PROD Track

Applies what was first practiced on LAB. Always one step behind the LAB ladder.

| # | Item | Depends on | Status | Change |
|---|---|---|---|---|
| P0 | Prod inventory + off-site backups (2 layers), dead man's switch, first restore drill | — | `proposed` (proposal only) | `prod-offsite-backups` |
| P1 | Ansible against prod in `--check --diff` mode: describe reality without changing it | L1 | `todo` | |
| P2 | `tofu import` of the existing VPS and DNS | L3 | `todo` | |
| P3 | Prod telemetry → platform on LAB | L4 | `todo` | |
| P4 | Deploys via GHCR + GitHub Environment `production`, dedicated deploy user | L3, L5 | `todo` | |
| P5 | Exam: rebuild prod from the repo on a clean VM in LAB | P1–P4 | `todo` | |
| P6 | Automated regular backup restore verification on LAB | P0, L4 | `todo` | |

## LAB Ladder

| # | Item | Learn | Immediate value | Status | Change |
|---|---|---|---|---|---|
| L0a | **By hand:** set a budget, apply the hard filters, rank candidates, measure latency to candidate regions; choose, order and accept the LAB host (rented or own). Write the method down in the [LAB host runbook](runbooks/lab-host.md) and the decision in a [provider adapter](../providers/README.md) | provider evaluation, hardware sizing | a host ready for bootstrap | `done` | `lab-host-selection` |
| L0b-1 | **By hand** (depends on L0a): reconnaissance of the out-of-band console, rescue system, disks and installer options; install Proxmox VE on mirrored NVMe and check it against the acceptance criteria; VM networking with NAT and a zone address plan; join the overlay network; lock down the web UI and SSH. Fill the runbook's bootstrap steps 1–5 | Linux, networking, Proxmox | a host that is installed and safe to leave on the internet | `done` | `lab-host-bootstrap` |
| L0b-2 | **By hand** (depends on L0b-1): the provider edge firewall, disk and array alerts, recovery drills through the console, and a rebuild exam. Fill bootstrap steps 6–7 | recovery, monitoring | confidence to break and repair the host | `todo` | `lab-host-recovery` |
| L1 | Ansible for the LAB host (depends on L0b-2: practise recovery before automating the firewall, since an Ansible mistake can lock you out as easily as a manual one): users, SSH, firewall, updates. The firewall role also closes forwarding from the public bridge into VNets (only replies to connections VMs opened), which must be done before the first VM that is not a throwaway test VM | idempotency, roles, inventory | reproducible host | `todo` | |
| L2 | Dev VM from a template (cloud-init) + Ansible role, connect the IDE | VM templates, cloud-init | **thin client works** | `todo` | |
| L3 | OpenTofu: GitHub org as code (rulesets, runner groups, environments), DNS, SOPS + age, encrypted remote state; repo hygiene (gitleaks, secret scanning, `.gitignore`, license) | state, providers, secrets | GitHub as code | `todo` | |
| L4 | Platform VM: Docker Compose with OTel Collector, Grafana, Prometheus, Loki, Tempo + PG/Redis for dev | Compose, OpenTelemetry | metrics and logs | `todo` | |
| L5 | CI: a single static self-hosted runner for private repos | Actions, labels, runner groups | builds on own hardware | `todo` | |
| L6 | Pair agent in a workspace VM: GitHub App, scoped tokens, attach via ssh + tmux | isolation, agent identity | mode (a) | `todo` | |
| L7 | Runner-manager in .NET: ephemeral JIT runners in VMs on events | Proxmox API, webhooks/polling | foundation for mode (b) | `todo` | |
| L8 | Egress proxy with allowlist, background agents on issue/label, per-PR preview environments, spend limits | network policies | mode (b) | `todo` | |
| L9+ | Optional: LAB-2, k3s in sandbox, OpenBao, Kata/Firecracker, agentic efficiency dashboards | | | `todo` | |

## Current Parallelism

**P0** and the LAB ladder are independent and can run in parallel. Everything else is sequential.

L0 is split into **L0a** (`lab-host-selection`, done), **L0b-1** (`lab-host-bootstrap`, done) and **L0b-2** (`lab-host-recovery`, next), which run in sequence: with P0 active, the WIP limit of 2 leaves one slot for the LAB ladder.
