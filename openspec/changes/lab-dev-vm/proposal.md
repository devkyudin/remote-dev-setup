# Proposal

## Why

The architecture's first goal is a thin client: the laptop is only a window into the LAB host. After L0b and L1 the host is installed, locked down and automated, but nothing runs on it that the owner can develop in. L2 delivers the dev VM, the first long-lived VM on LAB, and with it the "thin client works" milestone.

Exploration also changed two decisions the architecture had recorded, and L2 is where both take effect:

- **IDE.** Development is agent-driven and not limited to C#, so a C#-specific IDE as the primary client no longer fits. VS Code with Remote-SSH becomes the primary IDE, and a plain terminal (SSH + tmux) is the fallback. JetBrains Gateway (Rider) moves from primary to optional.
- **Where the pair agent runs.** The architecture keeps agents out of the dev VM entirely, and plans the pair agent for an agent workspace VM (L6). In practice the agent works inside the IDE, which runs in the dev VM. Instead of banning that, the dev VM becomes a place that holds **nothing an agent must not have**: no infrastructure or prod secrets, no route to the management plane or prod, and no GitHub identity that can merge on the owner's behalf. Infrastructure operations move to the owner's laptop. The agent pushes as a dedicated GitHub App with short-lived tokens, and its changes reach the default branch only through a pull request the owner approves.

## What Changes

- **Dev VM from a template.** By hand first: a cloud image becomes a Proxmox VM template with cloud-init, and the dev VM is cloned from it on the `dev` VNet with an address reservation. An Ansible role then configures everything inside the VM. A rebuild exam proves that the VM can be destroyed and recreated from the repository.
- **Access over the overlay network only.** The dev VM joins the tailnet as its own tagged node. The tailnet policy lets the owner's devices reach it over SSH and does not let the dev VM open connections to any other tailnet device. This settles how the laptop reaches VMs, which L0b-1 left to L2.
- **IDE.** VS Code Remote-SSH by overlay name is the primary client, and SSH + tmux is the fallback. Other VS Code-based editors work the same way but are not verified.
- **Toolchain baseline:** git, tmux, Docker Engine, the GitHub CLI, the .NET SDK, and Claude Code (CLI, plus the VS Code extension on the remote side).
- **Dev zone isolation.** Before an agent runs in the dev VM, traffic between the `dev` zone and every other LAB zone is restricted by the host, outside the VM's control. Per `lab-network`, a workload that runs agent-authored code is not placed on a LAB VNet until its zone is isolated, and the dev VM with an agent is such a workload.
- **No owner credentials in the dev VM.** No age key, no decrypted SOPS files, no tfstate or Proxmox credentials, no prod secrets, and no owner SSH private keys. The SSH server refuses agent forwarding. SOPS, OpenTofu and Ansible run from the owner's laptop.
- **Agent identity on GitHub.** A GitHub App for agents, introduced here and reused later by L6–L8. The laptop holds the App's private key and mints installation tokens of at most one hour, which it pushes to the dev VM while the owner is connected. git and the GitHub CLI in the dev VM use only that token.
- **Review gate.** A ruleset on the default branch of the repositories the agent works on requires a pull request with one approval. The App is not on the bypass list. The owner approves and merges from the laptop.
- **Documentation:**
  - a new generic runbook `docs/runbooks/dev-vm.md`;
  - `docs/architecture.md`: the IDE decision, rewording of principles 3 and 4 and the thin-client goal, the dev VM trust model, the App private key as a bootstrap-tier secret, and per-VM tailnet join instead of a subnet router;
  - `docs/roadmap.md`: L2 status and dependencies, and L6 reworded so that it introduces agent workspace VMs instead of the pair agent and the App.

Out of scope:
- dev containers (`devcontainer.json`), reconsidered at L6 together with workspace VMs;
- describing the dev VM in OpenTofu, which is L3 ("what exists"); here it is created by hand and documented;
- the GitHub org and rulesets as code (L3); here the ruleset is created by hand;
- an egress allowlist for the dev VM (L8 covers runners; the dev VM keeps open egress, see Impact);
- backups of the dev VM: git is its backup, and the rebuild exam proves the rest is reproducible;
- the platform zone and dev databases (L4), and the allow rules from `dev` to them;
- prod deploy approvals (P4), beyond noting that the App cannot approve them.

## Capabilities

### New Capabilities
- `dev-vm`: the owner's long-lived development VM on LAB — built from a template and an Ansible role, reachable only over the overlay network, reached from an IDE or a terminal, with a known toolchain, holding no owner infrastructure credentials, and allowed to host a pair agent.
- `agent-identity`: how agents act on GitHub — a dedicated GitHub App, short-lived tokens minted outside the agent's machine, and a default-branch review gate the agent cannot bypass. Reused by agent workspaces (L6) and background agents (L8).

### Modified Capabilities
- `lab-network`: the zone isolation requirement now covers a dev VM where an agent runs, and a new requirement isolates the `dev` zone from the other LAB zones.

## Impact

- **Order: depends on L1.** L2 needs L1's Ansible project, and L1 closes forwarding from the public bridge into VNets before the first VM that is not a throwaway test VM. The change is proposed now so that the decisions above are recorded; implementation starts after L1, within the WIP limit.
- **LAB host:** a VM template, the dev VM, and forwarding rules between zones. Host-level firewall input rules from L0b-1 are unchanged.
- **Repository:** the runbook, architecture, roadmap and specs, plus code: an Ansible role for the dev VM and a token-minting script for the laptop. While learning, the owner writes infrastructure code and agents explain and review it.
- **External services:** a GitHub App, a ruleset on the agent's repositories, a tailnet tag and policy rules. A Claude subscription login lives in the dev VM, because the agent needs it.
- **Secrets:** the App's private key is bootstrap-tier and lives in the password manager. Installation tokens are ephemeral tier. The tailnet auth key for the dev VM is single-use and never stored. No real IPs, hostnames or tailnet names enter the repository.
- **Accepted risks:**
  - the dev VM has open internet egress, so an agent could send the code it sees anywhere. The code is in repositories the agent may already read, and the VM holds no secrets;
  - the owner's Claude login in the dev VM could be read by the agent. It only grants model usage and can be revoked;
  - the agent runs as the owner's Unix user, with sudo and the Docker group. The VM is the boundary, so root inside the VM gives no more than the VM itself holds.
