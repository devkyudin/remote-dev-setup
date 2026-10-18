# Spec Delta

## MODIFIED Requirements

### Requirement: Zone isolation status is explicit
Until traffic between VNets is restricted by firewall policy, the documentation SHALL state which zones are not isolated from each other. Workloads that run agent-authored or otherwise untrusted code, such as agent workspaces, the ephemeral runner pool and a dev VM where an agent runs, SHALL NOT be placed on LAB VNets until isolation of their zone is enforced.

#### Scenario: Reader assumes VNets isolate zones
- **WHEN** a reader sees separate VNets per zone in the runbook or the architecture
- **THEN** the documentation tells them which zones are isolated and that traffic between the others is routed and not restricted yet

#### Scenario: Placing the first agent workspace
- **WHEN** a step proposes running agent workspaces or ephemeral runners on LAB
- **THEN** that step first enforces isolation between the runners zone and the other zones, or explicitly depends on a step that does

#### Scenario: Running an agent in the dev VM
- **WHEN** a step lets an agent run in the dev VM
- **THEN** that step first enforces isolation of the dev zone, or explicitly depends on a step that does

## ADDED Requirements

### Requirement: Dev zone is isolated from other zones
Traffic from the dev zone to other LAB zones SHALL be dropped, except for specific services a later step explicitly allows. Traffic from other zones into the dev zone SHALL be dropped, except replies to connections that dev VMs opened. Dev VMs SHALL reach the host only for DHCP and DNS.

#### Scenario: Dev VM connects to a VM in another zone
- **WHEN** a process in the dev VM opens a connection to a VM on the test VNet
- **THEN** the connection is dropped

#### Scenario: VM in another zone connects to the dev VM
- **WHEN** a VM on the test VNet opens a connection to the dev VM
- **THEN** the connection is dropped

#### Scenario: Dev VM connects to the host's management ports
- **WHEN** a process in the dev VM connects to the host's SSH or web UI port on any host address
- **THEN** the connection is not accepted

#### Scenario: Dev VM reaches the internet
- **WHEN** the dev VM fetches packages or talks to GitHub and the model provider
- **THEN** the traffic leaves through NAT on the host as before

#### Scenario: A later step needs a service in another zone
- **WHEN** a later step, such as the platform zone with dev databases, needs the dev VM to reach a service in another zone
- **THEN** that step adds an explicit rule for that service only, and documents it
