# lab-host Specification

## Purpose

Defines the properties a LAB host must have before it is used for dev, CI, agents and platform workloads, and how a candidate host is selected and accepted against them.

## Requirements

### Requirement: Hardware baseline
The LAB host SHALL be an x86-64 machine with at least 32 GB RAM, at least 8 CPU cores, and at least 512 GB of usable storage on two NVMe devices in a mirror.

#### Scenario: Candidate with a single NVMe device
- **WHEN** a candidate offer has only one NVMe device
- **THEN** it is rejected, because a single disk failure would lose the host

#### Scenario: Candidate with HDD-only storage
- **WHEN** a candidate offer has only HDDs
- **THEN** it is rejected, because builds and Docker workloads become disk-bound

#### Scenario: Candidate with two 512 GB NVMe devices
- **WHEN** a candidate offer has two NVMe devices of about 512 GB each and otherwise meets the baseline
- **THEN** it passes the storage filter

### Requirement: Out-of-band access
The LAB host SHALL provide console access that does not depend on the host's own OS or network configuration, such as a rescue system, KVM or IPMI.

#### Scenario: Locked out by a firewall or broken SSH
- **WHEN** the host's firewall or SSH configuration makes it unreachable over the network
- **THEN** the owner can still reach a console or rescue system through the provider and repair the host

#### Scenario: Provider without out-of-band access
- **WHEN** a candidate offer has neither a rescue system nor a KVM/IPMI console
- **THEN** it is rejected

### Requirement: Hypervisor installable
The LAB host SHALL allow the owner to install Proxmox VE, either from a provider image or from the official installer.

#### Scenario: Provider allows only its own managed OS images
- **WHEN** a candidate offer allows neither a Proxmox image nor a custom installer
- **THEN** it is rejected

### Requirement: VM networking with one public IP
The LAB host SHALL let VMs reach the internet through NAT behind the host's single public IP without violating the provider's network policy, so that no additional public IPs are required.

#### Scenario: Provider filters unknown MAC addresses
- **WHEN** the provider blocks or flags traffic from MAC addresses it did not assign
- **THEN** VMs stay behind NAT on the host and never send traffic with their own MAC to the provider network

### Requirement: Traffic allowance
The LAB host SHALL come with unmetered traffic, or with an included traffic volume large enough for CI image pulls, platform telemetry and restore drills.

#### Scenario: Offer with low metered traffic
- **WHEN** a candidate offer bills outgoing traffic beyond a small included volume
- **THEN** it is rejected

### Requirement: Independence from the PROD provider
The LAB host SHALL be rented from a different provider than the PROD host, or at minimum under a different account, so that an account or billing problem cannot take down both.

#### Scenario: PROD provider account is suspended
- **WHEN** the account that holds the PROD host is suspended or locked
- **THEN** the LAB host remains reachable and unaffected

#### Scenario: Candidate at the PROD provider
- **WHEN** the best-ranked candidate is offered by the PROD host's provider
- **THEN** it is chosen only if it runs under a separate account, and the decision is recorded

### Requirement: Selection is documented
The LAB host SHALL be chosen by applying the documented hard filters first and then ranking the remaining candidates. The ranking criteria are price, single-thread CPU performance, ECC memory, RAM headroom and latency, in that order.

#### Scenario: Latency as a tie-breaker
- **WHEN** two candidates pass all hard filters and are close on price and hardware
- **THEN** the one with lower measured latency from where the owner works is preferred

#### Scenario: Reader repeats the selection
- **WHEN** someone with a different budget or location follows the selection runbook
- **THEN** they can apply the same filters and ranking without knowing which provider the reference deployment uses

### Requirement: Delivered host is verified
The delivered LAB host SHALL be checked against the hard filters before bootstrap starts.

#### Scenario: Delivered hardware differs from the order
- **WHEN** the delivered host has a different CPU, less RAM, or different disks than ordered
- **THEN** the issue is raised with the provider before any bootstrap work, and bootstrap does not start until it is resolved

### Requirement: Installation meets the acceptance criteria
The LAB host SHALL run Proxmox VE with both the system and the VM storage kept redundantly on both NVMe devices, and with a boot loader installed on each of them. Whichever installation path is used, the result SHALL be checked against the acceptance criteria in the generic runbook before any other bootstrap step.

#### Scenario: Install from a provider template
- **WHEN** the host is installed from a provider's template instead of the official installer
- **THEN** the same acceptance criteria are checked, and a template that leaves the system or VM storage on a single device is rejected or corrected before continuing

#### Scenario: Boot loader only on the first disk
- **WHEN** the check shows that only one NVMe device carries a boot loader
- **THEN** the installation does not pass, because losing that one device would make the host unbootable despite the mirror

#### Scenario: Reader at another provider
- **WHEN** someone installs Proxmox VE at a different provider or from the official installer
- **THEN** they can verify their install with the same criteria without reading any provider adapter

### Requirement: Installed baseline is recorded
The changes a provider's template makes compared with a stock Proxmox VE install SHALL be recorded in the provider adapter. This covers partitioning, network configuration, package repositories, extra packages or agents, and authorized keys, so that automation knows the state it starts from.

#### Scenario: Automation starts from the template
- **WHEN** host automation (L1) is written for a host installed from the template
- **THEN** its author can see in the adapter every file and setting the template changed, without inspecting a live host

#### Scenario: Template adds an unknown access key or agent
- **WHEN** the audit finds an authorized key or an agent the owner did not add
- **THEN** it is recorded, and it is either removed or kept with a written reason

### Requirement: Management plane reachable only over the overlay network
The Proxmox web UI, its API and SSH on the LAB host SHALL be reachable only over the overlay network. The host's public address SHALL NOT accept connections to them. The out-of-band console is the break-glass path when the overlay network is unavailable.

#### Scenario: Connection attempt from the internet
- **WHEN** someone outside the overlay network connects to the host's public address on the web UI or SSH port
- **THEN** the connection is not accepted

#### Scenario: Connection from the provider's neighbouring subnet
- **WHEN** a machine in the same provider subnet as the host's public address connects to the web UI or SSH port
- **THEN** the connection is not accepted, because no automatic firewall rule trusts that subnet

#### Scenario: Owner connects from an overlay device
- **WHEN** the owner connects from a device on the overlay network
- **THEN** the web UI, the API and SSH are reachable

#### Scenario: Overlay network is down
- **WHEN** the overlay network or the host's overlay client is unavailable
- **THEN** the owner reaches the host through the out-of-band console, which does not depend on the host's network configuration

### Requirement: Access paths are verified before others are closed
A new access path to the LAB host SHALL be verified to work before an existing access path is closed or restricted.

#### Scenario: Closing public SSH
- **WHEN** public SSH is about to be closed
- **THEN** SSH over the overlay network and the out-of-band console have both been verified first in the same session

### Requirement: Administrative access uses keys and a second factor
SSH access to the LAB host SHALL use keys only, with password login disabled. Web UI login for administrative accounts SHALL require a second factor. The SSH keys and the root password live in the password manager, the second factor lives in an authenticator, and the repository holds none of them. A lost second factor is removed over SSH or the out-of-band console, so stored recovery codes are optional.

#### Scenario: Password login attempt over SSH
- **WHEN** someone attempts SSH login with a password
- **THEN** the server does not offer password authentication

#### Scenario: Web UI login without the second factor
- **WHEN** someone signs in to the web UI as an administrator with a correct password but no second factor
- **THEN** access is denied

#### Scenario: Second factor lost
- **WHEN** the owner loses the device that holds the second factor
- **THEN** they remove it over SSH or the out-of-band console and enroll a new one, without needing the web UI
