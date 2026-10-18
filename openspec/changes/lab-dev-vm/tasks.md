# Tasks

Markers:
- **Owner** tasks run on the live host, in the dev VM, or in the GitHub and tailnet consoles with owner credentials. Agents guide and review these steps but never execute them.
- **Owner writes** tasks produce infrastructure code. While learning, the owner writes it, and agents explain and review it.
- All other tasks are documentation work.

Groups 1–6 are milestone A ("thin client works"). Groups 7–10 are milestone B ("pair agent"). See design D1 for the split option after group 6.

## 1. Framing: roadmap, architecture, runbook skeleton

- [ ] 1.1 Update `docs/roadmap.md`:
  - row L2: status `proposed`, change `lab-dev-vm`, depends on L1;
  - row L6: reword to "agent workspace VMs (ephemeral, per task), attach via ssh + tmux; reuse the GitHub App and token flow from L2" (design D12);
  - bump "Last updated".

  *Verify:* the ladder table renders, L2 names its change and dependency, and L6 no longer introduces the pair agent or the App.
- [ ] 1.2 Update `docs/architecture.md`:
  - Goals → thin client: thin for compute, not for control; the laptop is the control seat for SOPS, OpenTofu and Ansible;
  - Core Principles 3 and 4: the dev VM may host a pair agent because it holds nothing an agent must not have; agent workspaces remain the isolation unit for task-scoped and background work; no access to the Proxmox API or prod;
  - Open Questions → Resolved, IDE: VS Code Remote-SSH primary, SSH + tmux fallback, JetBrains Gateway optional;
  - Secrets by Tier: add the agent App's private key to the bootstrap tier;
  - Zone address plan notes: the laptop reaches VMs by per-VM overlay join, not a subnet router;
  - bump "Last updated".

  *Verify:* a reader of the architecture alone can answer "may an agent run in the dev VM, and why is that safe?", and the provider-name search (see 11.1) finds nothing.
- [ ] 1.3 Create `docs/runbooks/dev-vm.md`:
  - a "Trust model" section first: what the dev VM holds, what it never holds, and where infrastructure operations run;
  - a contents list with step headings 1–9 from design D12, each marked as pending until its group below fills it.

  *Verify:* the runbook has no provider names and no deployment values, and every heading matches a later task.

## 2. Preconditions

- [ ] 2.1 Confirm L1 is done: the Ansible project, its inventory and conventions exist, and forwarding from the public bridge into VNets is restricted to replies.

  *Verify:* the L1 row in the roadmap is `done`, and its forwarding check passes. Record where the `dev_vm` role will live in L1's layout.

## 3. Template and dev VM (by hand)

- [ ] 3.1 **Owner:** on the host, download the Debian 13 `genericcloud` image, verify its checksum, import it as the disk of a new VM on `local-zfs`, add a cloud-init drive, a serial console and the guest agent option, and convert the VM to a template (design D2, D3).

  *Verify:* the template appears in the web UI with a cloud-init drive, and the image checksum matched.
- [ ] 3.2 **Owner:** create an IPAM reservation on `vndev` for the dev VM. Then full-clone the template on `vndev` with the sizing from design D4 and cloud-init settings: hostname, a non-root user, the owner's SSH public key, DHCP.

  *Verify:* the VM boots, gets the reserved address, and the cloud-init configuration holds no secret.
- [ ] 3.3 **Owner:** from the laptop, reach the VM through the host with `ProxyJump` and log in with the key.

  *Verify:* the login works, and a direct connection from the VM to the host's port 22 is refused (host input rules from L0b-1).
- [ ] 3.4 Write runbook steps 1–2 (template; clone and cloud-init), including the "full clone, not linked" and "nothing secret in cloud-init" reasons.

  *Verify:* a reader can repeat 3.1–3.3 from the runbook alone.

## 4. Ansible role `dev_vm`

- [ ] 4.1 **Owner writes:** the role's base:
  - Debian packages (`git`, `tmux`, build essentials) and `unattended-upgrades` for security updates;
  - `sshd`: key-only login, `AllowAgentForwarding no`, TCP forwarding kept;
  - `fs.inotify.max_user_watches` raised;
  - the owner's git author name and noreply address.

  *Verify:* the first run succeeds through `ProxyJump`; a second run reports `changed=0`.
- [ ] 4.2 **Owner writes:** the toolchain from the vendors' Debian repositories: Docker Engine with the owner in the `docker` group, the GitHub CLI, the .NET SDK (current LTS), and the Tailscale package without joining (design D10).

  *Verify:*
  - `docker run --rm hello-world` works without `sudo`;
  - a new console project builds and its test project passes with `dotnet test`;
  - `gh --version` and `tailscale version` print versions;
  - a second run reports `changed=0`.
- [ ] 4.3 Write runbook step 3 (the role): what it configures, how to run it before and after the overlay join, and the list of manual steps that remain (design D3).

  *Verify:* the list of manual steps matches the `dev-vm` spec scenario "Manual steps after a rebuild".

## 5. Overlay access and IDE

- [ ] 5.1 **Owner:** in the tailnet policy:
  - add `tag:dev` with the owner as tag owner;
  - allow the owner's devices to reach `tag:dev:22`;
  - add policy tests that the owner can reach `tag:dev:22`, and that `tag:dev` cannot reach `tag:lab` or the owner's devices on any port.

  *Verify:* the policy saves with its tests passing.
- [ ] 5.2 **Owner:** generate a single-use, pre-authorized auth key with `tag:dev` and join the dev VM with it. Do not store the key.

  *Verify:* the admin console shows the node with `tag:dev` and key expiry disabled. From the laptop, `ssh <dev-vm>` by overlay name works. From the dev VM, connecting to the laptop's and the host's overlay addresses fails.
- [ ] 5.3 **Owner:** on the laptop, add an SSH host entry for `<dev-vm>` with `ForwardAgent no`. Connect VS Code with Remote-SSH, open a repository, edit a file, and use the integrated terminal. Start a tmux session, sleep the laptop, wake it, and reattach.

  *Verify:*
  - VS Code works end to end;
  - the tmux session survived;
  - in a session opened with `ssh -A`, `SSH_AUTH_SOCK` is unset.
- [ ] 5.4 Write runbook steps 4–5 (overlay join and policy; IDE and terminal), including why not a subnet router, why not Remote Tunnels, and that other VS Code-based editors and JetBrains Gateway are allowed but not verified (design D5, D6).

  *Verify:* the policy shape uses placeholders only (`<owner-login>`, `<dev-vm>`).

## 6. Rebuild exam (end of milestone A)

- [ ] 6.1 **Owner:** destroy the dev VM, remove its node in the tailnet admin console, and recreate it by following runbook steps 1–5 only.

  *Verify:*
  - the same overlay name works with unchanged laptop SSH configuration;
  - every check from groups 3–5 passes again;
  - the only manual steps were the ones the runbook lists.
- [ ] 6.2 Write runbook step 9's rebuild part: push everything first, the steps in order, and the checks.

  *Verify:* 6.1 was carried out from this text without other notes.
- [ ] 6.3 Decide on the split (design D1): if milestone A used three sessions, move groups 7–10 into a new change (`lab-dev-agent`, L2b) and continue with group 11 here.

  *Verify:* the decision is recorded in this file and, on a split, in the roadmap.

## 7. Dev zone isolation

- [ ] 7.1 **Owner:** create the `dev` security group (design D7):
  - in: policy DROP;
  - out: allow DHCP and DNS to the `vndev` gateway, drop 10.42.0.0/16 and 10.43.0.0/16, policy ACCEPT.

  Enable the firewall on the dev VM's NIC and in its options, and attach the group.

  *Verify:* the web UI shows the group on the VM and the firewall enabled on its NIC.
- [ ] 7.2 **Owner:** with a throwaway VM on `vntest`, run the `lab-network` "Dev zone is isolated from other zones" scenarios:
  - dev → test connection dropped;
  - test → dev connection dropped;
  - dev → host ports 22 and 8006 not accepted;
  - dev → internet works;
  - SSH from the laptop over the overlay still works.

  Destroy the throwaway VM. If the per-VM firewall does not take effect on the SDN VNet, implement the same rules in L1's host forwarding filter and repeat.

  *Verify:* every scenario behaves as specified, and which mechanism was used is recorded in design D7.
- [ ] 7.3 Write runbook step 6 (zone isolation): the rules, the "every VM on `vndev` gets the `dev` group" rule, and the checks. Update the zone note in `docs/architecture.md`: `dev` is isolated from L2; the other zones are not yet.

  *Verify:* the `lab-network` requirement "Zone isolation status is explicit" is satisfied by the text.

## 8. Credential boundary

- [ ] 8.1 Write the credential check into runbook step 9 (design D8): searches for private key headers, age secret keys and SOPS-decrypted files, the `SSH_AUTH_SOCK` check, and the git and `gh` identity check (completed in 9.4).

  *Verify:* each command runs as written in the dev VM.
- [ ] 8.2 **Owner:** run the credential check in the dev VM.

  *Verify:* nothing is found apart from the VM's own host keys and overlay node state.

## 9. Agent identity on GitHub

- [ ] 9.1 **Owner:** create the agents' GitHub App with the permissions from design D9 (no *Administration*, *Secrets*, *Environments* or *Workflows*) and no webhook. Generate a private key and store it in the password manager. Install the App on selected **public** repositories only (design D11).

  *Verify:* the installation lists only the chosen repositories, and the private key exists only in the password manager.
- [ ] 9.2 **Owner:** create the default-branch ruleset on each of those repositories:
  - require a pull request with one approval;
  - dismiss stale approvals;
  - block force pushes and deletion;
  - bypass only for the repository admin role, "for pull requests only".

  *Verify:* the ruleset shows as active, and the App is not on the bypass list.
- [ ] 9.3 **Owner writes:** `scripts/agent-token/`: mint an installation token for the selected repositories (JWT under ten minutes, then the installation token request), and write it atomically over SSH to a mode-600 file in the dev VM. `--loop` refreshes every 50 minutes and prints each refresh time. Include a short README.

  *Verify:*
  - after one run, the token file exists in the VM with mode 600;
  - the token's expiry reported by GitHub is under one hour away;
  - the script never writes the private key to disk outside the password manager's own handling.
- [ ] 9.4 **Owner writes:** extend the role with the git credential helper for `https://github.com` and the `gh` wrapper, both reading the token file at use time (design D9).

  *Verify:*
  - in the dev VM, `gh api /installation/repositories` lists the installed repositories;
  - a branch push succeeds and shows the App's bot account as pusher;
  - a second role run reports `changed=0`.
- [ ] 9.5 **Owner:** run the `agent-identity` scenarios:
  - a direct push to the default branch is rejected;
  - a pull request opened by the App cannot be merged before approval and can be merged after the owner approves it from the laptop;
  - a push that changes `.github/workflows/` is rejected;
  - after stopping the loop, the token stops working within the hour;
  - suspending the installation makes a fresh token fail immediately (resume afterwards).

  *Verify:* every scenario behaves as specified.
- [ ] 9.6 Write runbook step 7 (GitHub App, token flow, ruleset), with the reasons for rejecting PATs, user-to-server tokens, deploy keys and a key in the VM, and the "public repositories only until L3" rule.

  *Verify:* a reader can create the App, mint a token and set up the ruleset from the runbook and the script's README alone.

## 10. Pair agent

- [ ] 10.1 **Owner writes:** extend the role to install Claude Code with its official native installer for the owner's user.

  *Verify:* `claude --version` works in a fresh shell, and a second role run reports `changed=0`.
- [ ] 10.2 **Owner:** confirm the `dev-vm` "Pair agent preconditions": 7.2, 8.2 and 9.4 passed on this VM. Then sign in to Claude Code in the dev VM and open the Claude Code extension in VS Code over Remote-SSH.

  *Verify:* both the CLI and the extension start a session in the dev VM.
- [ ] 10.3 **Owner:** end-to-end pair session. The agent makes a small documentation change on a branch, pushes it, and opens a pull request with `gh`. The owner reviews, approves and merges it from the laptop.

  *Verify:* the pull request shows the App as author, the owner as approver, and the merge passed the ruleset.
- [ ] 10.4 Write runbook step 8 (agent): the preconditions, signing in, the accepted risks (open egress, the Claude sign-in in the VM, the owner's Unix rights), and what to do on suspected misuse (suspend the App installation, remove the tailnet node).

  *Verify:* the accepted risks match the proposal's Impact section.

## 11. Integration checks

- [ ] 11.1 Run the neutrality and secrets search across `docs/`, `providers/` and `scripts/`:
  - provider names appear only under `providers/`;
  - no public IPv4 literal, hostname, tailnet name, App ID or installation ID of the deployment appears anywhere (use the same `grep` patterns as the `lab-host-bootstrap` change, extended to `scripts/`).

  *Verify:* both searches find nothing that is not allowed.
- [ ] 11.2 Record the dev VM's peak memory use so far in design D4's open question. If two weeks of real use have not passed yet, add a roadmap note to update the architecture's hardware table later.

  *Verify:* the number or the follow-up note exists.
- [ ] 11.3 Run `openspec validate lab-dev-vm --strict`.

  *Verify:* it passes.
- [ ] 11.4 **Owner:** read `docs/runbooks/dev-vm.md` top to bottom as a newcomer would.

  *Verify:* every step is understandable without this change's design notes. Fix any gaps before archiving.
