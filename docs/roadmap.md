# Roadmap

> The backlog for moving toward the [target architecture](architecture.md). It will later move to GitHub Issues/Projects, where agents can pick up tasks too.
>
> Last updated: 2026-10-10.

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
| L0 | **By hand:** measure latency to candidate regions, choose and obtain the LAB host (rented or own), install Proxmox and the overlay network, lock down the Proxmox web UI; learn the out-of-band console (rescue / KVM / IPMI). Write everything down in a runbook | Linux, networking, Proxmox | understanding of what to automate | `todo` | |
| L1 | Ansible for the LAB host: users, SSH, firewall, updates | idempotency, roles, inventory | reproducible host | `todo` | |
| L2 | Dev VM from a template (cloud-init) + Ansible role, connect the IDE | VM templates, cloud-init | **thin client works** | `todo` | |
| L3 | OpenTofu: GitHub org as code (rulesets, runner groups, environments), DNS, SOPS + age, encrypted remote state; repo hygiene (gitleaks, secret scanning, `.gitignore`, license) | state, providers, secrets | GitHub as code | `todo` | |
| L4 | Platform VM: Docker Compose with OTel Collector, Grafana, Prometheus, Loki, Tempo + PG/Redis for dev | Compose, OpenTelemetry | metrics and logs | `todo` | |
| L5 | CI: a single static self-hosted runner for private repos | Actions, labels, runner groups | builds on own hardware | `todo` | |
| L6 | Pair agent in a workspace VM: GitHub App, scoped tokens, attach via ssh + tmux | isolation, agent identity | mode (a) | `todo` | |
| L7 | Runner-manager in .NET: ephemeral JIT runners in VMs on events | Proxmox API, webhooks/polling | foundation for mode (b) | `todo` | |
| L8 | Egress proxy with allowlist, background agents on issue/label, per-PR preview environments, spend limits | network policies | mode (b) | `todo` | |
| L9+ | Optional: LAB-2, k3s in sandbox, OpenBao, Kata/Firecracker, agentic efficiency dashboards | | | `todo` | |

## Current Parallelism

**P0** and **L0** are independent and can run in parallel. Everything else is sequential.
