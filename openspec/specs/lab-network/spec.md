# lab-network Specification

## Purpose

Defines how LAB virtual machines are addressed and connected: the per-node zone address plan, egress through NAT on the host, address assignment, no inbound exposure, and the explicit status of isolation between zones.

## Requirements

### Requirement: Zone address plan
Each LAB zone SHALL have a reserved private IPv4 subnet, documented in the architecture before the zone's first VM is created. Each LAB node SHALL have its own non-overlapping private range. Subnets SHALL NOT overlap the overlay network's own address range.

#### Scenario: A new zone is created
- **WHEN** a VNet for a zone such as platform or runners is created at a later step
- **THEN** it uses the subnet already reserved for that zone, and no existing VM has to change its address

#### Scenario: A second LAB node is added
- **WHEN** a second LAB node joins
- **THEN** its zones use that node's own range, which does not overlap the first node's subnets

#### Scenario: Plan is published
- **WHEN** the address plan is written into the public repository
- **THEN** it contains only private addresses and no public address or hostname of any deployment

### Requirement: VM egress through NAT on the host
LAB VMs SHALL reach the internet only through source NAT on the host's public address. A VM's own MAC address SHALL NOT appear on the provider network.

#### Scenario: Test VM reaches the internet
- **WHEN** a throwaway VM on a LAB VNet fetches a page from the internet
- **THEN** the request succeeds, and the provider network sees the host's public address and the host's MAC address as the source

#### Scenario: Provider filters unknown MAC addresses
- **WHEN** the provider blocks or flags traffic from MAC addresses it did not assign
- **THEN** no VM traffic is affected, because only the host's MAC address is ever seen on the provider network

### Requirement: VM addresses are assigned centrally
LAB VMs SHALL get their IPv4 address, gateway and DNS resolver from the host through DHCP, backed by an address registry on the host. Fixed addresses are made as reservations in that registry.

#### Scenario: VM created without network configuration
- **WHEN** a VM is attached to a LAB VNet and boots without static network settings
- **THEN** it receives an address from its zone's subnet and can resolve names and reach the internet

#### Scenario: VM needs a stable address
- **WHEN** a VM such as the dev VM needs the same address across rebuilds
- **THEN** a reservation in the registry gives it that address, without a static configuration inside the VM

### Requirement: No inbound exposure of VMs
LAB VMs SHALL NOT be reachable from the internet. The host SHALL NOT forward any port from its public address to a VM. Inbound access to VMs goes only through the overlay network.

#### Scenario: Port scan of the public address
- **WHEN** the host's public address is scanned from the internet
- **THEN** no port leads to a VM

#### Scenario: Neighbour on the provider network routes into a VNet
- **WHEN** a machine on the provider's network segment sends traffic for a VNet subnet to the host
- **THEN** the host forwards to VMs only replies to connections the VMs opened; this is enforced before the first VM that is not a throwaway test VM is created

### Requirement: Zone isolation status is explicit
Until traffic between VNets is restricted by firewall policy, the documentation SHALL state that zones are not isolated from each other. Workloads that run agent-authored or otherwise untrusted code, such as agent workspaces and the ephemeral runner pool, SHALL NOT be placed on LAB VNets until isolation is enforced.

#### Scenario: Reader assumes VNets isolate zones
- **WHEN** a reader sees separate VNets per zone in the runbook or the architecture
- **THEN** the documentation tells them that traffic between those VNets is routed and not restricted yet

#### Scenario: Placing the first agent workspace
- **WHEN** a step proposes running agent workspaces or ephemeral runners on LAB
- **THEN** that step first enforces isolation between the runners zone and the other zones, or explicitly depends on a step that does
