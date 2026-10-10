# Spec Delta

## Purpose

Defines the properties a LAB host must have before it is used for dev, CI, agents and platform workloads, and how a candidate host is selected and accepted against them.

## ADDED Requirements

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
