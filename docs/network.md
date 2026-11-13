# Network Architecture — Automated Hybrid Landing Zone

**Project:** P1 — Automated Hybrid Landing Zone

**Status:** Design phase — implementation pending

**Architecture baseline:** S02

## 1. Purpose

This document defines the network addressing, segmentation, routing principles and connectivity requirements of the homelab.

It distinguishes existing infrastructure from planned components and identifies networking decisions that require implementation and testing.

Related documents:

- [Architecture overview](architecture.md)
- [Security architecture](security.md)
- [ADR-001 — Hypervisor topology](decisions/ADR-001-hypervisor-topology.md)

## 2. Existing Physical Network — AS-IS

| Device | Interface | Address | Role |
|---|---|---|---|
| Linux Mint | `enp0s31f6` | `192.168.137.1/24` | Ethernet gateway |
| PVE-01 | `vmbr0` | `192.168.137.10/24` | Physical management |
| Linux Mint | `wlp0s20f3` | Dynamic | Internet uplink |

The ZBook and PVE-01 are connected directly through Ethernet.

PVE-01 currently uses:

- Default gateway: `192.168.137.1`
- Management network: `192.168.137.0/24`
- Internet access: routing/NAT through Linux Mint using `lanshare`

The external Wi-Fi IP address and subnet are deliberately excluded from the project addressing plan.

### Existing Proxmox bridges

| Bridge | Physical attachment | Current state | Intended use |
|---|---|---|---|
| `vmbr0` | `nic0` | Operational | Physical management and upstream connectivity |
| `vmbr1` | None | Internal, no IPv4 | Virtual laboratory network |

The current configuration does not enable VLAN-aware mode on `vmbr1`.

## 3. Network Segmentation — TO-BE

The following VLANs are defined by the Project 1 requirements.

| VLAN ID | Zone | Subnet | Planned gateway | Purpose |
|---|---|---|---|---|
| 10 | DMZ | `10.10.10.0/24` | `10.10.10.1` | Exposed services and reverse proxies |
| 20 | ADMIN | `10.10.20.0/24` | `10.10.20.1` | Bastion and administration services |
| 30 | APP | `10.10.30.0/24` | `10.10.30.1` | Internal applications |
| 99 | MANAGEMENT | `10.10.99.0/24` | `10.10.99.1` | Internal workload management |

All gateways are planned to be hosted by OPNsense on PVE-01.

The physical Proxmox management network remains separate from VLAN 99.

### Segmentation principles

- Workloads are assigned to security zones according to their function and trust requirements.
- Inter-VLAN traffic must pass through an appropriate Layer 3 routing and filtering point.
- Traffic within the same VLAN may be switched locally without traversing OPNsense.
- Host-based or hypervisor-based security controls may therefore be necessary for same-VLAN isolation.
- Connectivity must be explicitly authorized according to documented service requirements.

## 4. OPNsense Network Design

OPNsense is hosted on PVE-01 as VM 101.

Current virtual interfaces:

| Interface | Proxmox bridge | Planned role |
|---|---|---|
| `net0` | `vmbr0` | WAN |
| `net1` | `vmbr1` | Internal VLAN trunk |

The internal interface will provide connectivity to the four VLAN zones.

### Planned routing behavior

OPNsense will:

1. Route traffic between internal VLANs.
2. Apply stateful firewall policies.
3. Route authorized outbound traffic towards its WAN interface.
4. Perform outbound NAT where necessary.

The WAN addressing and gateway configuration must be verified during implementation.

The lab must preserve PVE-01 management access independently of OPNsense.

## 5. VLAN Transport

The initial architecture uses PVE-01 as the main location for VLAN segmentation.

A VLAN-aware bridge is planned on `vmbr1`.

The OPNsense-facing internal interface will transport the required tagged VLANs.

Workload VM interfaces will be assigned to appropriate VLANs through Proxmox network configuration.

No physical managed switch is required for this initial virtual segmentation.

Extending the VLANs to PVE-02 is not part of the initial design.

## 6. PVE-02 Networking — Planned

PVE-02 will run as a nested Proxmox VE virtual machine under KVM/QEMU on Linux Mint.

Its networking will separate:

- Hypervisor management.
- Laboratory workloads.
- Authorized upstream connectivity.

The existing libvirt `default` network uses `192.168.122.0/24`, but it is not yet approved as the final PVE-02 management network.

A candidate workload network is:

| Parameter | Proposed value |
|---|---|
| Network | `10.20.30.0/24` |
| Role | PVE-02 application workloads |
| Gateway | To be defined |
| Routing | Controlled Layer 3 interconnection |
| Status | Candidate — not implemented |

The final design must define the exact routing path between PVE-01 and PVE-02 without exposing the Mint host to untrusted workload traffic.

### Interconnection requirements

- Avoid extending Layer 2 VLANs between hypervisors in the initial implementation.
- Prevent unrestricted workload access to the Mint host.
- Allow only explicitly authorized inter-network flows.
- Preserve administration of both hypervisors.
- Ensure workload Internet access is controlled.
- Document any required NAT or static routes.

## 7. Internet Connectivity

The target traffic flow for PVE-01 workloads is:

```text
Internal VM
   |
Internal VLAN
   |
OPNsense
   |
OPNsense WAN
   |
192.168.137.0/24
   |
Linux Mint
   |
Upstream Wi-Fi
   |
Internet
```

Outbound NAT may occur at OPNsense and again at Linux Mint.

This creates a possible double-NAT configuration.

Double NAT is acceptable for the initial outbound-only laboratory use case, but may complicate inbound connections and troubleshooting.

For PVE-02 workloads, the exact outbound route and firewall enforcement points remain to be designed.

Internet access must not imply unrestricted access to the Mint host or management services.

## 8. Initial Traffic Policy

The following table describes intended traffic permissions, not implemented firewall rules.

| Source | Destination | Initial policy | Reason |
|---|---|---|---|
| ADMIN | APP | Allow selected management ports | Administration |
| ADMIN | DMZ | Allow selected management ports | Administration |
| DMZ | APP | Allow explicitly required application ports | Reverse proxy to backend |
| DMZ | ADMIN | Deny by default | Prevent lateral movement |
| APP | ADMIN | Deny by default | Management isolation |
| Workloads | Physical management | Deny by default | Protect hypervisors |
| Workloads | Linux Mint host | Deny by default, minimal exceptions | Protect personal workstation |
| Internal VLANs | Internet | Allow selected outbound traffic | Updates and dependencies |
| Internet | Internal networks | Deny unsolicited inbound traffic | Reduce exposure |

Responses to authorized connections are permitted according to the behavior of the applicable stateful firewall.

The detailed rule matrix, including source addresses, destination addresses, protocols and ports, will be defined in `security.md`.

## 9. Network Validation Plan

Network implementation will be considered successful only after verifying:

1. PVE-01 remains reachable over `192.168.137.10/24`.
2. A test VM can communicate with its configured VLAN gateway.
3. Inter-VLAN traffic is routed correctly.
4. Explicitly prohibited inter-VLAN flows are blocked.
5. Approved outbound Internet traffic works.
6. Unauthorized workload-initiated access to Mint is blocked.
7. PVE-02 management remains accessible through an authorized path.
8. PVE-02 workloads can communicate with permitted PVE-01 services.
9. The actual path of packets matches the documented architecture.

Tests must record expected results, observed results and relevant evidence.

## 10. Open Design Items

The following items remain unresolved and must be validated before implementation:

- Exact PVE-02 management network and access method.
- Exact PVE-01/PVE-02 Layer 3 interconnection mechanism.
- Final PVE-02 workload addressing.
- Routing ownership and return routes.
- NAT requirements between internal networks.
- Mint firewall and libvirt isolation policies.
- OPNsense WAN addressing and gateway.
- VLAN-aware bridge configuration and permitted VLAN tags.
- Specific application traffic requirements.
- Rollback procedures for network configuration changes.

These open items are intentional: the architecture distinguishes approved principles from implementation details that have not yet been demonstrated.
