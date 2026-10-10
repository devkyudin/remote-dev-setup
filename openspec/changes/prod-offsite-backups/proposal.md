# Proposal

## Why

Production projects live on a single VPS, and their "backups" are stored on that same VPS. A disk failure, a compromise, an accidental `rm -rf` or an account issue would destroy the data together with its copies, so in practice there are no backups. This is the biggest risk to the existing projects. It must be fixed now, before and independently of building LAB.

## What Changes

- Prod inventory: which projects exist, where their data lives (databases, volumes, bind mounts), what runs outside Docker. The result is a runbook in the repository, without IPs, hostnames or secrets, because the repository is public.
- Layer 1, snapshots: enable the hosting provider's server snapshots/backups (where available) for fast whole-server rollback.
- Layer 2, off-site: daily encrypted backups of data (logical DB dumps + file volumes) via restic to S3-compatible storage at **a provider other than the prod host's provider**.
- Deletion protection: the credentials stored on prod cannot delete or overwrite existing backups.
- Retention policy and regular pruning, executed from a trusted machine, not from prod.
- Execution monitoring: an external dead man's switch alerts when a backup fails or does not run.
- Verified restore: a documented procedure and a first restore drill on an isolated machine.
- Existing local copies on the VPS remain as "fast copies" but no longer count as backups.

Out of scope:
- automation via Ansible (step L1 of the LAB ladder);
- SOPS and secrets in the repository;
- automated regular restore drills on LAB;
- hardening GitHub Actions deploy access to prod;
- migrating prod to a new host.

## Capabilities

### New Capabilities
- `prod-backups`: backups of production data — what is backed up, where, how often, how it is protected from deletion and corruption, how execution is monitored and how restores are verified.

### Modified Capabilities
<!-- None: the project has no specs yet. -->

## Impact

- **Prod host:** new systemd services and timers for backups, restic installation, a root-only file with backup secrets. Small CPU, disk and network load during the backup window. Short DB dumps should not require stopping projects.
- **Cost:** provider snapshots (typically a percentage of the server price) + off-site storage (small at pet-project volumes). The host itself is not recreated or resized.
- **External services:** an account at an S3-compatible provider (Backblaze B2 or similar), Healthchecks.io or similar.
- **Repository:** backup and restore scripts, systemd unit files, inventory and restore runbooks. Code and documentation only; secrets live in the password manager.
- **Secrets:** the restic repository password and storage keys are bootstrap-tier and live in the password manager. Without the restic password the backups are unrecoverable.
