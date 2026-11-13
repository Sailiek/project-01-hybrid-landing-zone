# ADR-001 — Hypervisor topology and workload placement

**Status:** Accepted for design — implementation pending
**Project:** P1 — Automated Hybrid Landing Zone
**Phase:** 0 — Architecture and documentation

## Context

The homelab uses two personal computers with different roles and hardware capacities.

- **PVE-01:** HP ProBook 430 G5, physical Proxmox VE host, approximately 8 GiB RAM.
- **ZBook:** Linux Mint workstation with 32 GB RAM, KVM/QEMU and virt-manager.
- **PVE-02:** Planned nested Proxmox VE instance hosted on the ZBook.

PVE-01 currently depends on the ZBook for Internet access through an Ethernet connection and Linux-based NAT.

The architecture must support infrastructure automation, network segmentation and workloads for future DevSecOps and detection engineering projects without compromising the primary workstation.

## Decision

Adopt a two-hypervisor architecture with separated responsibilities.

### PVE-01 — Core infrastructure

- Host OPNsense as the primary lab firewall and router.
- Host the initial VLAN segmentation: DMZ, ADMIN, APP and MANAGEMENT.
- Host lightweight infrastructure services and validation workloads.
- Preserve direct management access through `192.168.137.10/24`.

### PVE-02 — Complementary compute

- Run as a nested Proxmox VE instance under KVM/QEMU.
- Host additional experimental and application workloads.
- Use separate management and workload networking.
- Connect to the PVE-01 environment through controlled Layer 3 connectivity, with exact routes and transport to be defined before implementation.

### Linux Mint — Protected workstation

- Remain the administration workstation, KVM host and upstream Internet gateway.
- Preserve the existing Ethernet management path.
- Deny workload-initiated access to the Mint host by default, with narrowly scoped exceptions where necessary.
- Avoid exposing personal directories, privileged host services or sensitive devices to lab workloads.

### Network security

- Use OPNsense to enforce policies for traffic passing through its interfaces.
- Use host-level and virtualization-level controls for traffic that does not traverse OPNsense.
- Permit controlled outbound Internet access for workloads.
- Avoid extending VLANs between hypervisors in the initial architecture.

## Alternatives considered

1. **Single-hypervisor architecture:** simpler, but severely constrained by PVE-01 RAM.
2. **Layer 2 VLAN extension:** provides shared VLANs across hypervisors, but increases networking complexity and troubleshooting overhead.
3. **Layer 3 interconnection:** separates network domains and avoids a stretched Layer 2 design.

The third option is selected for the initial architecture.

## Consequences and trade-offs

### Benefits

- Functional network segmentation.
- Better use of available computing resources.
- Clear security boundaries.
- Support for progressively adding P2 and P3 workloads.
- Architecture decisions that can be explained and defended.

### Limitations

- Nested virtualization introduces resource overhead.
- Communication between hypervisors requires deliberate routing and firewall policies.
- PVE-02 is dependent on the availability of the ZBook.
- The environment does not provide production-grade high availability.
- Host isolation and virtualization cannot guarantee absolute security.

## Implementation constraints

- No immediate modification of the existing Mint-to-PVE-01 management network.
- No direct connection of untrusted workloads to the physical management bridge.
- Network and firewall changes require validation and a rollback procedure.
- PVE-02 deployment is deferred to a dedicated session.
- Exact Layer 3 network design remains to be documented and tested.

## Validation criteria

This decision will be considered implemented when:

1. PVE-01 management access remains operational.
2. PVE-02 can be administered through an explicitly authorized path.
3. Workloads communicate only through approved network paths.
4. Workloads have controlled outbound Internet access.
5. Unauthorized workload-initiated connections to Mint are blocked.
6. Network isolation and routing behavior are demonstrated with reproducible tests.
