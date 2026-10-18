# Spec Delta

## Purpose

Defines the owner's long-lived development VM on LAB: how it is built and rebuilt, how the owner reaches it from an IDE or a terminal, what it contains, and what it must never hold, so that a pair agent can work in it safely.

## ADDED Requirements

### Requirement: Dev VM is reproducible from the repository
The dev VM SHALL be created from a VM template with cloud-init and configured by an Ansible role kept in the repository. Rebuilding it from the template and the role SHALL produce an equivalent VM, with no manual configuration inside the VM beyond the steps the runbook lists as manual.

#### Scenario: Rebuild exam
- **WHEN** the dev VM is destroyed and recreated by following the runbook
- **THEN** the new VM passes the same checks as the old one: overlay access, IDE connection, toolchain and credential checks

#### Scenario: Work in progress during a rebuild
- **WHEN** the dev VM is about to be rebuilt
- **THEN** the runbook tells the owner to push all work first, because only what is in git survives a rebuild

#### Scenario: Manual steps after a rebuild
- **WHEN** the role has finished on a fresh VM
- **THEN** the only remaining manual steps are those the runbook lists, such as joining the overlay network and signing in to the agent

### Requirement: Dev VM is reachable only over the overlay network
The dev VM SHALL join the overlay network as its own tagged node and SHALL accept SSH from the owner's overlay devices. It SHALL NOT be reachable from the internet, in line with `lab-network`.

#### Scenario: Owner connects from the laptop
- **WHEN** the owner runs SSH to the dev VM's overlay name from a device on the overlay network
- **THEN** the connection succeeds with key authentication

#### Scenario: Connection attempt from the internet
- **WHEN** someone outside the overlay network tries to reach the dev VM
- **THEN** there is no path to it

#### Scenario: Address changes after a rebuild
- **WHEN** the dev VM is rebuilt
- **THEN** the owner reaches it by the same overlay name, without editing local SSH configuration

### Requirement: Dev VM cannot open connections to overlay devices
The overlay network policy SHALL NOT allow the dev VM to open connections to any other device on the overlay network, including the owner's devices and the LAB host.

#### Scenario: Process in the dev VM probes the overlay network
- **WHEN** a process in the dev VM tries to connect to the laptop or the LAB host over the overlay network
- **THEN** the connection is refused by the overlay policy

#### Scenario: Owner's session over the overlay network
- **WHEN** the owner has an SSH session open to the dev VM
- **THEN** replies within that session work, because the owner opened the connection

### Requirement: IDE and terminal access
The owner SHALL be able to work in the dev VM from VS Code with remote SSH as the primary client and from a plain SSH terminal as the fallback. Terminal sessions SHALL survive a dropped connection.

#### Scenario: VS Code connects
- **WHEN** the owner connects VS Code over remote SSH to the dev VM's overlay name
- **THEN** the remote server installs or starts, and the owner can open a repository, edit, use the integrated terminal and run the agent's extension

#### Scenario: Laptop sleeps and wakes
- **WHEN** the laptop sleeps during a session and wakes up later
- **THEN** a terminal multiplexer session started before sleep is still running and can be reattached, and VS Code reconnects without rebuilding the VM

#### Scenario: High latency makes the IDE laggy
- **WHEN** network latency makes the IDE uncomfortable
- **THEN** the owner can keep working over SSH with a terminal multiplexer, with the same repositories and tools

### Requirement: Toolchain baseline
The dev VM SHALL provide git, a terminal multiplexer, a container engine usable without root, the GitHub CLI, the .NET SDK and the Claude Code CLI. Further languages are added to the role when a project needs them.

#### Scenario: Build a container image
- **WHEN** the owner or the agent runs a container build in the dev VM
- **THEN** it succeeds without `sudo`

#### Scenario: Build a .NET project
- **WHEN** a .NET project is restored, built and tested in the dev VM
- **THEN** it succeeds with the SDK the role installed

### Requirement: No owner infrastructure credentials in the dev VM
The dev VM SHALL NOT hold the age key, decrypted SOPS files, state backend or hypervisor credentials, prod secrets, or the owner's SSH private keys. Infrastructure operations (SOPS, OpenTofu, Ansible) SHALL run from the owner's laptop, not from the dev VM.

#### Scenario: Credential check
- **WHEN** the dev VM's file system is searched for age keys, SSH private keys other than its own host keys, and files decrypted from SOPS
- **THEN** none are found

#### Scenario: Owner needs to apply an infrastructure change
- **WHEN** the owner wants to run OpenTofu or Ansible against LAB or PROD
- **THEN** they run it from the laptop, and the dev VM has neither the credentials nor the network path to do it

#### Scenario: Agent reads its environment
- **WHEN** an agent in the dev VM reads environment variables, shell history and configuration files
- **THEN** it finds no credential beyond its short-lived GitHub token, the agent's own sign-in, and the VM's own machine identities (SSH host keys and overlay node key), which grant nothing the VM does not already have

### Requirement: SSH agent forwarding is refused
The dev VM's SSH server SHALL refuse SSH agent forwarding, so that the owner's keys on the laptop cannot be used from inside the VM, whatever the client requests.

#### Scenario: Client requests agent forwarding
- **WHEN** the owner connects with agent forwarding enabled on the client
- **THEN** no forwarded agent socket is available in the session

### Requirement: Pair agent preconditions
An agent SHALL run in the dev VM only after the `dev` zone is isolated per `lab-network`, the credential check passes, and the agent's GitHub access uses the agent identity per `agent-identity`.

#### Scenario: First agent session in a new dev VM
- **WHEN** the owner is about to start the agent in the dev VM for the first time
- **THEN** the zone isolation checks, the credential check and the GitHub identity check have passed

#### Scenario: Agent runs with the owner's permissions inside the VM
- **WHEN** the agent runs commands in the dev VM
- **THEN** it can do anything the owner's Unix user can do inside the VM, and nothing outside it beyond open internet egress and its GitHub App permissions
