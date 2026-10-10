# Target Architecture

> Status: vision. This document describes **where** we are heading. What is actually implemented is captured by the specs in `openspec/specs/`. The order of work lives in [roadmap.md](roadmap.md).
>
> The architecture is provider-agnostic: anyone should be able to deploy it on any suitable hardware, whether rented or owned. Concrete providers, regions and machines belong to a deployment's configuration, not to this document.
>
> Last updated: 2026-10-10.

## Goals

1. **Thin client.** The laptop is just a window into powerful hardware available 24/7: IDE, terminal, browser.
2. **Staging and build server.** CI runners, staging and preview environments on own hardware.
3. **Agentic development.** Start with a pair-programming agent (interactive), grow into a background team of agents that pick up tasks and deliver PRs. The architecture targets the background mode from day one.
4. **Separate production** for pet projects, isolated from experiments.
5. **Shared platform:** metrics, logs, traces, databases, queues.
6. **Reproducible by anyone.** Everything is described as code and documentation, and the repository is public. Building the infrastructure is a learning project and a showcase in its own right.

## Big Picture

```
   [laptop, thin client]
        |
        |  overlay network (Tailscale / WireGuard): nothing exposed to the internet
        |
        +-----------------------------------+
        |                                   |
        v                                   v
  +----------------------------------+  +---------------------------+
  | LAB host: bare metal + Proxmox   |  | PROD host: small VPS      |
  | "ok to break"                    |  | "must not break"          |
  |                                  |  |                           |
  |  zone dev       VM dev (owner)   |  |  reverse proxy + TLS      |
  |  zone platform  observability,   |  |  projects in Docker       |
  |                 PG/Redis/queues  |  |  Postgres prod            |
  |                 for dev          |  |  OTel agent --------------+--> platform
  |  zone runners   ephemeral VMs:   |  |                           |    on LAB
  |                 CI + agents      |  +-------------+-------------+
  |  zone sandbox   k3s and other    |                |
  |                 experiments      |                |
  |  zone mgmt      Proxmox, access  |                |
  +----------------+-----------------+                |
                   |                                  |
                   +----------------+-----------------+
                                    v
                 off-site backups (different provider than PROD)
                 external uptime check / dead man's switch

                 +---------------------------------------------+
                 | GitHub (control plane)                      |
                 | org (Team): repos, issues, PRs, Actions,    |
                 | GHCR, rulesets, environments, GitHub App    |
                 +---------------------------------------------+
```

## Core Principles

1. **PROD is independent of LAB.** If LAB is down, prod keeps running and deploying. Images live in GHCR; deploys run on GitHub-hosted runners or via a pull model and are never routed through LAB.
2. **The agent workspace is the isolation unit.** One ephemeral VM per task: its own repo checkout and branch, a short-lived GitHub App token, egress only through an allowlist, resource limits. Pair mode (a) and background mode (b) differ only in who starts the workspace and whether a human is attached.
3. **Agents never get the owner's credentials.** No age key, no prod secrets, no access to the Proxmox API, the dev VM or prod. Agents **propose** changes via PRs; they never **apply** them.
4. **CI runners and agents share one ephemeral runner pool.** Event → clean VM → work → artifact or PR → VM destroyed.
5. **The repository is public.** No plaintext secrets, tfstate, private keys, real IPs or personal details. Public Actions logs are public too, so infra plan/apply never runs there.
6. **Provider-agnostic.** No hard dependency on a specific hosting provider. Provider-specific pieces (provisioning APIs, snapshot features, rescue access) are isolated behind variables or swappable modules and documented as examples.
7. **Industry standards over niche tools:** GitHub, OpenTofu, Ansible, Docker Compose, Proxmox.
8. **Manual first, then code.** You cannot automate what you have never done by hand.
9. **A backup is only a verified restore**, stored off-site and protected from deletion by the source host.

## Components and Decisions

### LAB Host

| Decision | Why |
|---|---|
| A single bare-metal machine running a hypervisor, available 24/7 | Strong isolation between zones, full control, predictable cost. Can be a rented dedicated server or own hardware. |
| Scale horizontally: one node now, a second node for the runner pool when moving to background agents | Lower cost now, no migration later, physical isolation of agents from interactive work. |
| Proxmox | Runs on any x86-64 server. Zone isolation via VMs, snapshots as a safety net while learning, web UI, API for ephemeral VMs. |
| Workspace = VM cloned from a template via cloud-init | Agents and CI need Docker inside (Testcontainers, `docker build`). In a plain container that means `--privileged`, i.e. effectively root on the host. A VM gives strong isolation without extra complexity. Upgrade path: Kata / Firecracker. |
| Proxmox web UI and API reachable only via the overlay network | The first rule after installation. |

**Hardware requirements** (sized for a .NET + React workload; adjust for your stack):

| Resource | Minimum | Comfortable | Notes |
|---|---|---|---|
| RAM | 32 GB | 64 GB | ~54 GB comfortable baseline (host ~4, IDE backend ~8, builds ~8, CI ~12, staging ~8, observability ~8, data ~4), plus 4–8 GB per parallel agent slot |
| CPU | 8 cores | 12–16 cores | Parallel builds from agents compete with interactive work |
| Disk | 1 TB NVMe | 2 TB NVMe, mirrored | HDDs make builds and Docker painful |
| Other | — | ECC RAM | |
| Access | out-of-band console (rescue system / KVM / IPMI) | | Needed to recover from a locked-out firewall or broken SSH |

**Choosing where to host LAB:**

| Option | Pros | Cons |
|---|---|---|
| Own hardware at home | Lowest latency on the local network, one-time cost | Upfront cost, power and cooling, uplink and uptime depend on home infrastructure |
| Rented dedicated server | 24/7 uptime, datacenter network, no hardware to maintain | Monthly cost, latency depends on region |
| Rented older-generation server (auction/second-hand offers) | Often the best price per GB of RAM, especially when memory prices are high | Older hardware, inventory varies |
| Cloud VM running 24/7 | Flexible | Usually the most expensive option for an always-on large machine |

Rule of thumb: compare the upfront cost of buying hardware plus power against monthly rent; with volatile RAM prices, renting can win for years. Measure latency from where you work to candidate regions before choosing, since it affects how responsive a remote IDE feels.

### PROD Host

- A small VPS (around 4 vCPU / 8 GB is enough for several pet projects) running projects in Docker, deployed via GitHub Actions. Any provider works.
- An existing prod host is adopted gradually (brownfield): inventory and backups first, then Ansible in `--check` mode, then `tofu import`, telemetry to LAB, deploys via a GitHub Environment.
- Do not recreate or resize an existing host without an explicit decision: beyond the risk, it may be on legacy pricing.
- The "rebuild prod from the repo" exam runs on a test VM in LAB.

### GitHub

| Decision | Why |
|---|---|
| GitHub as the control plane, not self-hosted Forgejo | Industry standard. Ready-made integrations (`claude-code-action`, self-hosted runners). Prod does not depend on LAB. |
| Organization on the Team plan | On Free, rulesets and branch protection are not enforced for private repos, and they are the security boundary for agents. Org-level self-hosted runners and runner groups exist only for organizations. |
| Mix of public and private repositories | Public repos are the showcase: CI on GitHub-hosted runners, agents only on owner-triggered events. Private repos use LAB runners and the full background mode. LAB runners are restricted to private repos via a runner group. |
| Dedicated GitHub App for agents | Tokens live 1 hour and are scoped to specific repos. Does not take a seat. Easy to revoke. |
| Ephemeral JIT runners + a custom runner-manager in .NET | One job, one VM. The runner-manager is a pet project and a showcase in its own right. |

Risk: GitHub postponed, but did not cancel, the per-minute charge for self-hosted runners (announced in December 2025).

### Secrets by Tier

| Tier | Examples | Where it lives |
|---|---|---|
| 0. Bootstrap | age key, hosting provider and Proxmox root access, overlay network auth keys, backup repository passwords | password manager (+ a paper copy of the age key) |
| 1. Infrastructure | LAB service passwords, API keys | SOPS + age, encrypted in the repo |
| 2. Prod runtime | connection strings, application keys | GitHub Environment `production` |
| 3. Ephemeral | agent installation tokens, JIT configs | nowhere: issued for ~1 hour |

Related rules: address machines by overlay network names (e.g. Tailscale MagicDNS) instead of IPs; tfstate in a remote backend with client-side encryption (OpenTofu); gitleaks in pre-commit and CI; secret scanning + push protection on GitHub; an age key rotation procedure written down in advance. OpenBao is a possible future addition for runtime secrets.

### IaC Tooling

```
  "what exists"          OpenTofu         VPS, DNS, GitHub org, overlay ACLs, Proxmox VMs
  "how it's configured"  Ansible          users, SSH, firewall, packages, services
  "what runs"            Docker Compose   platform and apps (later k8s/GitOps in sandbox)
```

Provider-specific resources (creating a VPS, DNS records) live in separate OpenTofu modules, so that switching providers means swapping a module, not rewriting the stack.

Infrastructure code is written so a newcomer can follow it: while learning, agents explain and review infrastructure code rather than write it, and every line must be understood before `apply`.

### Observability

OpenTelemetry Collector as the single entry point, then Grafana + Prometheus + Loki + Tempo on LAB. The stack may be revisited. Prod telemetry and agent metrics land here as well (Claude Code exports metrics via OpenTelemetry). Agentic work metrics (PRs per week, lead time, cost per PR) are part of the public showcase.

### PROD Backups

Two layers: provider-level snapshots for fast whole-server rollback (where the provider offers them), and off-site data backups (logical DB dumps + volumes) at a provider different from the prod host's, protected from deletion by prod. Later LAB will verify restores regularly. Details are in the `prod-offsite-backups` change.

## Cost Structure

Prices vary by provider, region and time, so this document lists cost items rather than numbers:

- LAB host (rent or hardware + power), later a second node for the runner pool;
- PROD host;
- provider snapshots + off-site backup storage;
- GitHub Team plan;
- model usage for agents — a separate item; in background mode it may exceed the hardware cost.

## Open Questions

- Tailscale or self-hosted Headscale.
- IDE: Rider via Gateway or VS Code Remote-SSH (latency-sensitive).
- Runner-manager: webhook (via a tunnel) or GitHub API polling.
- Final observability stack.
- Model budget and billing mode for background agents.
- Off-site provider and append-only mechanism for prod backups.
- How the current GitHub Actions deploy authenticates to prod; a dedicated deploy user.
- How deployment-specific configuration (provider, region, machine names) is kept out of the public repo: private overlay repo, encrypted variables, or local-only files.
