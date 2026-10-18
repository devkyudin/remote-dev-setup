# Design

## Context

- The LAB host runs Proxmox VE 9 with a ZFS mirror and a `local-zfs` storage for VM disks. A Proxmox SDN Simple zone provides the `vndev` (10.42.20.0/24) and `vntest` VNets with SNAT, DHCP and IPAM (see the `lab-host-bootstrap` design, D5).
- The host firewall (`pve-firewall`, iptables backend, input policy DROP) already lets VMs reach the host only for DHCP and DNS, so 22 and 8006 on the gateway addresses are closed to VMs. Forwarded traffic is not filtered at host level. L1 closes forwarding from the public bridge into VNets, and its notes say the nftables backend that could filter forwarding is a tech preview.
- The host is a tagged Tailscale node (`tag:lab`). The tailnet policy allows only the owner's devices to reach it on 22 and 8006, and the default allow-all rule is removed. L0b-1 left open how the laptop reaches VMs.
- L1 (Ansible for the host) is not done yet. Its project layout, inventory and conventions are inputs to this change, not decisions made here.
- The repositories currently live under the owner's personal GitHub account. On a free personal account, rulesets are enforced for public repositories only. The Team-plan organization from the architecture arrives with L3.
- WIP: `prod-offsite-backups` is active, and `lab-host-recovery` (L0b-2) and L1 come before this change.

## Goals / Non-Goals

**Goals:**
- Meet the `dev-vm`, `agent-identity` and `lab-network` deltas in an order where the agent never runs before its boundaries are in place.
- Keep every step manual-first and explainable: the template and the VM by hand, then the role; the token flow as a short script the owner can read.
- Leave L3, L6 and L8 a reusable base: the App, the token flow and the zone isolation pattern.

**Non-Goals:**
- An egress allowlist for the dev VM.
- High availability or backups of the dev VM.
- Supporting every IDE: only VS Code Remote-SSH and the terminal are verified.
- Making the agent safe against a malicious owner session: the owner's own session has the same rights as the agent inside the VM.

## Decisions

### D1. One change, two milestones, with a split option
- **Milestone A, "thin client works":** template, dev VM, Ansible role, overlay access, IDE, toolchain without the agent, rebuild exam.
- **Milestone B, "pair agent":** dev zone isolation, credential boundary checks, GitHub App and token flow, ruleset, Claude Code.

The agent is not installed or started until milestone B's checks pass, which is what the `dev-vm` "Pair agent preconditions" requirement enforces. A is useful without B.

If A alone takes three sessions, B is split off as its own change (`lab-dev-agent`, L2b) and this change is archived after A. That keeps to the 1–3 session rule without deciding the split now.

*Alternative:* two changes from the start. The owner chose one L2 change. Milestones keep the option to split open at no cost.

### D2. Guest OS: Debian 13 cloud image
The template is built from the official Debian 13 `genericcloud` image.
- It is the same family as the Proxmox host (Debian 13), so L1 and L2 share one set of Ansible knowledge, package tooling and security update habits.
- The `genericcloud` image is minimal and cloud-init ready, with no desktop and no hardware drivers.
- Docker, the GitHub CLI, the .NET SDK and Tailscale all publish official Debian 13 packages.

*Alternative:* Ubuntu LTS. It has more desktop-developer documentation, but it would be a second distribution to learn and patch. If a toolchain later turns out to be Ubuntu-only, the template can be swapped without changing the specs.

### D3. Template by hand, VM by hand, configuration by Ansible
- **Template (manual, runbook):**
  - download the cloud image to the host and import it as the disk of a new VM on `local-zfs`;
  - add a cloud-init drive, a serial console and the QEMU guest agent option;
  - convert the VM to a template.

  *Why by hand:* the owner should see every piece once before L3 describes VMs in OpenTofu.
- **Dev VM (manual, runbook):**
  - a full clone (not linked) of the template on `vndev`, with an IPAM reservation for a stable address;
  - cloud-init sets the hostname, a non-root user with the owner's SSH **public** key, and DHCP.

  *Why a full clone:* a linked clone depends on the template forever, and a rebuilt template would break it.
- **Nothing secret in cloud-init.** User data sits in plaintext in the hypervisor's configuration and on the VM's cloud-init drive, so it carries only public keys and names.
- **Inside the VM: one Ansible role** (`dev_vm`) in L1's Ansible project, run from the laptop. It covers:
  - packages, the toolchain (D10), `sshd` settings (D8) and kernel limits for file watchers;
  - the Tailscale package, without joining;
  - the dev VM side of the token flow (D9).

  Before the VM joins the overlay, Ansible reaches it through the host (SSH `ProxyJump` to the VM's private address). Afterwards it uses the overlay name. The host may open connections to VMs; VMs may not open them to the host.
- **Manual steps after the role,** listed in the runbook as the only ones allowed:
  - joining the overlay with a single-use key;
  - signing in to Claude Code;
  - running the token script on the laptop.

*Alternatives:*
- *Cloud-init doing all the configuration:* one file, but no idempotent re-runs, and it would duplicate what Ansible is in the project for.
- *Packer-built golden images:* faster rebuilds, but one more tool before the basics are learned.

### D4. Sizing
Initial values, revisited after two weeks of real use:

| Resource | Value | Why |
|---|---|---|
| vCPU | 8, CPU type `host` | builds and the agent's test runs are the main load; `host` exposes the real CPU features to compilers |
| RAM | 16 GiB, ballooning off | covers the architecture's estimates of about 8 GiB for the IDE backend and 8 GiB for builds; ballooning hides memory pressure, which is what we want to measure |
| Disk | 128 GiB on `local-zfs`, discard on | repositories, Docker images and build caches; ZFS snapshots before risky changes; can be grown later |

After two weeks, record peak memory use. Update the architecture's hardware table, whose "IDE backend ~8 GB" line was sized for the JetBrains backend.

### D5. Overlay access: the dev VM joins the tailnet as `tag:dev`
- The dev VM runs its own Tailscale client and joins with a **single-use, pre-authorized** auth key carrying `tag:dev`. The key is typed in once and never stored. A tagged node has no key expiry and is not tied to the owner's identity, as for the host.
- **Policy:**
  - the owner's devices may reach `tag:dev` on port 22;
  - nothing grants `tag:dev` access to anything, so the dev VM cannot open connections to the laptop, the host or future nodes;
  - the policy test section asserts both directions.
- **OpenSSH over the overlay,** not Tailscale SSH, for the same reasons as on the host (L0b-1 D6).
- The overlay name stays the same across rebuilds once the old node is removed in the admin console. That is a runbook step, so the laptop's SSH configuration never changes.

*Alternative:* the host as a subnet router for `vndev`. One fewer client, but:
- the whole subnet becomes reachable from the tailnet, and the policy has to work with IP addresses instead of tags;
- tailnet traffic would enter VNets through the host, a new path for the zone isolation rules to cover;
- the per-node identity is lost.

Ephemeral runners (L7) will not join the tailnet at all, so this choice does not commit them to anything.

### D6. IDE: VS Code Remote-SSH primary, terminal fallback
- VS Code on the laptop connects with the Remote-SSH extension to the overlay name. It installs its server in the owner's home directory on first connect. The dev VM only needs `sshd`, TCP port forwarding for that server, internet egress to download it, and a raised `fs.inotify.max_user_watches`.
- The fallback is `ssh` + `tmux` with the same repositories and tools. tmux sessions survive dropped connections, and L6 attaches to agent workspaces the same way.
- **Not used:** VS Code Remote Tunnels. They route through a third-party relay and need a separate sign-in, while the overlay network already provides the path.
- Other editors based on VS Code (for example Cursor) connect the same way with their own remote extension. They are allowed but not verified. JetBrains Gateway also connects over SSH and is optional.

### D7. Dev zone isolation: the Proxmox per-VM firewall on the dev VM
- The dev VM's network interface has the Proxmox firewall enabled. The VM's rules come from a security group (`dev`):
  - **in:** policy DROP. Replies to connections the VM opened are allowed by connection tracking, which covers its internet traffic and Tailscale's NAT traversal. The overlay path is encrypted inside those flows, so SSH from the owner arrives through it.
  - **out:** allow DHCP and DNS to the `vndev` gateway; drop 10.42.0.0/16 and 10.43.0.0/16 (every LAB zone on both nodes); policy ACCEPT for everything else (internet egress).
- Traffic from the dev VM to any host address already hits the host's input rules from L0b-1, which allow only DHCP and DNS.
- **Why the per-VM firewall:**
  - the host enforces it on the VM's own bridge port, outside the VM, so root inside the VM cannot change it;
  - it is visible in the web UI next to the VM;
  - it works with the default `pve-firewall` backend, so it does not wait for the nftables backend to leave tech preview.
- **Every VM placed on `vndev` gets the `dev` security group.** The rule goes into the runbook, and the check is part of the rebuild exam.
- The first implementation session verifies that the per-VM firewall applies to NICs on SDN VNets in this Proxmox version. Checks:
  - a dev VM → `vntest` connection is dropped;
  - a `vntest` → dev VM connection is dropped;
  - internet access and the overlay still work.

  If it does not apply, the same rules go into the host-level forwarding filter that L1 introduces. The specs stay the same, because they require the behavior, not the mechanism.

*Alternative:* zone-level forwarding rules on the host from the start. They are the better shape for many VMs per zone (runners in L7), but they depend on L1's filtering mechanism and are harder to inspect for a single VM. L7 can lift the `dev` rules to zone level when it builds the runners' isolation.

### D8. Credential boundary
- **In the dev VM, never:**
  - the age key or decrypted SOPS files;
  - state backend credentials, Proxmox tokens or passwords;
  - prod secrets;
  - the owner's SSH private keys;
  - a GitHub token of the owner.

  SOPS, OpenTofu and Ansible run on the laptop, which becomes the **control seat**. The architecture's thin-client goal is reworded: the laptop is thin for compute, not for control.
- **`sshd`:**
  - `AllowAgentForwarding no`, enforced on the server so that a client misconfiguration cannot override it;
  - password login off.

  TCP forwarding stays on, because the VS Code server needs it. The runbook also sets `ForwardAgent no` for this host in the laptop's SSH configuration, as a second layer.
- **Git:** the owner's name and noreply address as commit author; pushes over HTTPS with the agent token (D9). No SSH key for GitHub in the VM. Commits made in the VM therefore show the owner as author and the App as pusher. The agent adds its co-author trailer.
- **Claude Code sign-in:** the owner signs in with their Claude subscription inside the VM. The credential grants only model usage, can be revoked from the account, and is listed as an accepted risk in the proposal.
- **Credential check:** a short command list in the runbook. It is run at the end of the rebuild exam and before the first agent session:
  - search for private key headers and age secret keys;
  - search for SOPS-decrypted files;
  - check that `SSH_AUTH_SOCK` is unset in a session opened with agent forwarding requested;
  - check that `git` and `gh` authenticate as the App.

### D9. Agent identity: GitHub App, tokens minted on the laptop
- **The App:**
  - one GitHub App for agents, owned by the account that owns the repositories (the personal account now; transferred to the organization when L3 creates it);
  - installed on selected repositories only;
  - permissions: *Contents* read/write, *Pull requests* read/write, *Issues* read/write, *Metadata* read. **No** *Administration*, *Secrets*, *Environments* or *Workflows*;
  - without *Workflows*, a push that changes `.github/workflows/` is rejected, so the agent cannot change what CI runs with. The owner edits workflows.
- **Private key:** generated in the App settings and stored in the password manager (bootstrap tier). It is never copied to the dev VM.
- **Minting, on the laptop:** a short script in the repository (`scripts/agent-token/`), written by the owner. It:
  1. reads the private key from the password manager's CLI or an owner-only file;
  2. signs a JWT valid for under ten minutes;
  3. requests an installation token for the selected repositories with the permissions above;
  4. writes it atomically over SSH to a mode-600 file in the dev VM.

  With `--loop` it repeats every 50 minutes while the laptop is awake and connected. If the laptop sleeps, the last token expires within the hour, which `agent-identity` relies on.
- **Using the token, in the dev VM:** the role installs two small helpers that read the token file at use time:
  - a git credential helper for `https://github.com`, user `x-access-token`;
  - a wrapper that runs the GitHub CLI with `GH_TOKEN` taken from the file.

  Long-lived shells never hold a stale token.
- **Direction:** the laptop pushes to the VM, and the VM never connects to the laptop. That is consistent with D5.

*Alternatives:*
- *Fine-grained personal access token:* acts as the owner, so the owner cannot approve the agent's pull requests, and admin bypass may apply. Rejected.
- *App user-to-server token* (the App acting on behalf of the owner): carries the owner's identity and bypass. Rejected.
- *Repository deploy key:* never expires and cannot open pull requests. Rejected.
- *The App's private key in the dev VM:* the agent could mint tokens forever. Rejected.
- *A token broker service on LAB:* the right shape for background agents (L7/L8), and premature for one dev VM.

### D10. Toolchain installation
- From the vendors' official Debian repositories, pinned by the role and updated by `unattended-upgrades` for security fixes:
  - Docker Engine with the owner in the `docker` group;
  - the GitHub CLI;
  - the .NET SDK (current LTS);
  - Tailscale.
- Also: `git`, `tmux` and build essentials from Debian.
- **Claude Code:** its official native installer, run by the role for the owner's user. It updates itself.
- **Docker group membership is root-equivalent inside the VM.** That is accepted, because the VM is the boundary (proposal, accepted risks). Rootless Docker is a possible later hardening, not needed for the specs.

### D11. Ruleset on the agent's repositories
- **Created by hand now** (L3 codifies it). It targets the default branch and:
  - requires a pull request with one approval;
  - dismisses stale approvals on new pushes;
  - blocks force pushes and deletion.
- **Bypass:** only the repository admin role, in "for pull requests only" mode. The owner can merge their own pull requests from the laptop without a second person, but nobody pushes directly. The App is not on the list.
- **Repository scope:** while the repositories live under a free personal account, rulesets are enforced only on public repositories. **Until the organization exists (L3), the App is installed only on public repositories.** A private repository joins the installation after it moves to the Team organization.
- **Prod deploys (P4):** environment reviewers are users or teams, so the App cannot approve a deployment. P4 records this when it creates the `production` environment.

### D12. Documentation layout
- **`docs/runbooks/dev-vm.md`, a new generic runbook** with no provider slots, since nothing here depends on the hosting provider:
  1. template;
  2. clone and cloud-init;
  3. role;
  4. overlay join;
  5. IDE and terminal;
  6. zone isolation;
  7. GitHub App, token flow and ruleset;
  8. agent;
  9. rebuild exam and credential check.

  It states the dev VM trust model at the top.
- **`docs/architecture.md`:**
  - the thin-client goal;
  - principles 3 and 4: the dev VM may host a pair agent because it holds nothing an agent must not have, and agent workspaces stay the isolation unit for task-scoped and background work;
  - the IDE item in Resolved;
  - the App private key in the bootstrap tier;
  - per-VM tailnet join;
  - the zone isolation note: `dev` is isolated from L2;
  - the hardware table note after D4's measurement.
- **`docs/roadmap.md`:**
  - L2 → `proposed`, change `lab-dev-vm`, depends on L1;
  - L6 reworded to "agent workspace VMs (ephemeral, per task), attach via ssh + tmux; reuse the App and token flow from L2";
  - "Last updated".

## Risks / Trade-offs

- [The per-VM firewall does not apply to NICs on SDN VNets] → Verified in the first session of milestone B. The fallback is rules in L1's host forwarding filter; the specs are unchanged (D7).
- [A new VM is put on `vndev` without the `dev` security group] → The dev VM's inbound policy still protects it. The runbook rule and the rebuild exam check cover new VMs.
- [The token loop on the laptop stops silently] → The agent gets 401 from GitHub within the hour. That is the intended failure mode, and the fix is to restart the loop. The script prints each refresh time.
- [The agent logs the dev VM out of the tailnet or changes its settings] → The owner reaches the VM through the host (`ProxyJump`) and rejoins it. The tag is fixed by the auth key, and subnet routes or exit nodes need approval in the admin console.
- [Convenience pulls infrastructure work back into the dev VM] → The credential check is part of the rebuild exam, and the runbook explains why the boundary exists.
- [The laptop becomes more valuable as the control seat] → Disk encryption, the password manager and the overlay device approval already protect it. On loss, revoke its overlay node and rotate the App private key. The age key rotation procedure is already planned in the architecture.
- [Rulesets are not enforced on private personal repositories] → The App is installed only on public repositories until L3 (D11).
- [The agent needs a workflow change] → The push is rejected for lack of the *Workflows* permission, and the owner makes the change. This is intended friction.
- [Docker images and caches fill the disk] → 128 GiB with periodic `docker system prune`. The ZFS volume can be grown.
- [Milestone A takes longer than planned] → Split B off as L2b (D1).
- [VS Code's remote server or the Claude Code extension changes behavior on update] → The terminal fallback keeps work going. Both are user-space installs and can be reinstalled without rebuilding the VM.

## Migration Plan

This change creates new resources only; nothing existing is migrated.
- **Order of work across sessions:**
  - session 1 (A): template, clone, cloud-init, first role run through `ProxyJump`;
  - session 2 (A): overlay join and policy, VS Code and the terminal, rebuild exam;
  - session 3 (B): per-VM firewall and its checks, credential check, GitHub App, token script, ruleset, Claude Code, first agent pull request reviewed and merged from the laptop;
  - session 4, if needed: documents, validation, the owner's read-through.
- **Rollback:**
  - destroy the dev VM and the template;
  - remove the `tag:dev` node and policy rules;
  - suspend or delete the App installation;
  - delete the ruleset;
  - revert the documentation commit.

## Open Questions

- Peak memory use of the dev VM in real work, which updates the architecture's hardware table (D4).
- Whether rootless Docker is worth the friction later (D10).
- The App's display name. It must not contain personal details beyond what the GitHub account already shows.
