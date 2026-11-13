# Architecture — Automated Hybrid Landing Zone

**Project:** P1 — Automated Hybrid Landing Zone
**Status:** Design phase — partially deployed
**Architecture baseline:** S02
**Related decision:** [ADR-001 — Hypervisor topology and workload placement](decisions/ADR-001-hypervisor-topology.md)

## 1. Overview

This project aims to build a small, automated and security-oriented landing zone using Proxmox VE, OPNsense, Terraform and Ansible.

The infrastructure provides the foundation for two complementary portfolio projects:

- **P2 — DevSecOps & GitOps Software Factory:** application build, security testing and automated delivery.
- **P3 — Detection-as-Code & SOAR:** security event collection, detection engineering and controlled incident response.

The architecture prioritizes network segmentation, automation, reproducibility, least privilege and measurable validation.

The homelab is an educational environment. It is not intended to provide production-grade high availability or continuous service availability.

## 2. Physical Infrastructure — AS-IS

### 2.1. Administration workstation — HP ZBook 14u G6

| Component | Current configuration |
|---|---|
| Operating system | Linux Mint 22.3 |
| RAM | 32 GB installed, approximately 30 GiB usable |
| Storage | 512 GB SSD |
| Virtualization | KVM/QEMU with virt-manager |
| Upstream connectivity | Wi-Fi, dynamically assigned network configuration |
| Wi-Fi interface | `wlp0s20f3` |
| Ethernet interface | `enp0s31f6` |
| Ethernet IP | `192.168.137.1/24` |
| Current responsibilities | Administration workstation, upstream router/NAT, future nested virtualization host |

The ZBook connects to an external network over Wi-Fi and to PVE-01 through a direct Ethernet cable.

The external Wi-Fi network is considered an **environment-dependent upstream connection**. Its IP address, subnet and gateway may change without affecting the internal homelab addressing plan.

Internet sharing is managed through a local script named `lanshare`, which enables routing and NAT between the direct Ethernet network and the available upstream connection.

The ZBook is also the primary workstation used for SSH administration, infrastructure development and Git operations.

### 2.2. Physical hypervisor — PVE-01

| Component | Current configuration |
|---|---|
| Hardware | HP ProBook 430 G5 |
| CPU | Intel Core i5-8250U, 4 cores / 8 threads |
| RAM | Approximately 7.6 GiB usable |
| Disk | 238.5 GiB SSD |
| Operating system | Proxmox VE on Debian 13 |
| Kernel observed | `7.0.14-5-pve` |
| Management IP | `192.168.137.10/24` |
| Default gateway | `192.168.137.1` |

The physical hypervisor currently has two Linux bridges:

| Bridge | Current configuration | Purpose |
|---|---|---|
| `vmbr0` | Physical interface `nic0`, static IP `192.168.137.10/24` | Physical management and upstream connectivity |
| `vmbr1` | No physical interface, no IPv4 address | Internal virtual networking |

The two existing virtual machines are:

| VMID | Name | Allocated RAM | Virtual disk | Current state |
|---|---|---|---|---|
| 101 | OPNsense | Up to 3 GiB | 50 GB | Stopped |
| 9000 | Debian template | 2 GiB configured | 40 GB | Stopped |

OPNsense VM 101 currently has two virtual network interfaces:

- `net0` connected to `vmbr0`.
- `net1` connected to `vmbr1`.

The target OPNsense WAN, VLAN interfaces, routing and firewall configuration have not yet been validated.

### 2.3. Virtualization environment on the ZBook

KVM/QEMU and libvirt are installed.

The default libvirt network uses `192.168.122.0/24`, with `virbr0` configured as `192.168.122.1/24`.

No system-level libvirt virtual machine was present during the S02 inventory.

**PVE-02 has not been deployed.**

The existing libvirt default network does not automatically define the final management and workload topology of PVE-02.

## 3. Existing Network Architecture — AS-IS

```text
               External network / Internet
                           |
                      Wi-Fi uplink
                  (dynamic addressing)
                           |
                 HP ZBook / Linux Mint
                           |
                     Routing + NAT
                       (lanshare)
                           |
                   Ethernet interface
                     192.168.137.1
                           |
                    Direct Ethernet
                           |
                     PVE-01 / vmbr0
                     192.168.137.10
                           |
                +----------+----------+
                |                     |
           OPNsense net0       Internal vmbr1
                |                     |
                +------ OPNsense -----+
                          net1

                 OPNsense currently stopped
```

This diagram documents the observed infrastructure, not an operational segmented landing zone.

The upstream Wi-Fi configuration is intentionally excluded from the addressing plan because it is external, dynamic and not controlled by the project.

The direct Ethernet management subnet is part of the documented local infrastructure and remains stable independently of the upstream network.

## 4. Target Architecture — TO-BE

The target environment consists of two complementary Proxmox hypervisors.

PVE-01 provides the core network infrastructure, while PVE-02 provides additional compute capacity for workloads and future experiments.

### 4.1. PVE-01 — Core infrastructure

PVE-01 will host:

- The main OPNsense firewall/router.
- Virtual network segmentation.
- Lightweight administration and infrastructure services.
- Initial validation workloads.

The planned internal zones are:

| VLAN | Zone | Planned subnet |
|---|---|---|
| 10 | DMZ | `10.10.10.0/24` |
| 20 | ADMIN | `10.10.20.0/24` |
| 30 | APP | `10.10.30.0/24` |
| 99 | MANAGEMENT | `10.10.99.0/24` |

These VLANs will be implemented and tested during a later networking phase.

The physical management network `192.168.137.0/24` remains separate from the internal MANAGEMENT VLAN.

This separation preserves an administration path to PVE-01 that does not depend on OPNsense routing or firewall configuration.

### 4.2. PVE-02 — Complementary compute

PVE-02 will be deployed as a nested Proxmox VE virtual machine using KVM/QEMU on the ZBook.

Initial resource allocation under consideration:

- 4 vCPU.
- 8 GiB RAM.
- Approximately 80 GB virtual disk.

PVE-02 will provide additional compute capacity for:

- Application workloads.
- DevSecOps and CI/CD experiments.
- Lightweight security monitoring and detection components.
- Infrastructure validation scenarios.

Its management network and workload networks must be separated.

The initial design favors controlled Layer 3 interconnection rather than extending VLANs between hypervisors.

The exact network transport, routing and IP addressing for PVE-02 remain implementation design items.

PVE-02 is not required to be continuously operational, and its deployment must not compromise the stability or security of the ZBook.

### 4.3. Network and security responsibilities

**Linux Mint**

- Provides administration access and upstream Internet connectivity.
- Hosts PVE-02 through KVM/QEMU.
- Enforces host-level protection for the workstation.
- Does not expose personal resources to lab workloads.
- Keeps the upstream network separate from the internal homelab addressing plan.

**PVE-01**

- Provides the main physical virtualization environment.
- Hosts internal virtual networking and the primary OPNsense instance.
- Preserves a direct management path independent of OPNsense.

**OPNsense**

- Routes between configured internal lab networks.
- Enforces firewall policies for traffic traversing its interfaces.
- Provides controlled outbound connectivity for internal workloads.
- Does not replace security controls on Linux Mint or individual hypervisors.

**PVE-02**

- Hosts complementary virtual workloads.
- Uses isolated workload networking.
- Does not provide unrestricted connectivity to Mint or the physical management network.

## 5. Trust Boundaries

The architecture distinguishes three security domains.

### Domain 1 — Personal workstation

Linux Mint hosts personal files, development tools, administrative credentials and the virtualization platform.

Protecting its confidentiality, integrity and availability is a priority.

The upstream Wi-Fi network is treated as an external, potentially untrusted network.

### Domain 2 — Infrastructure management

Includes Proxmox administration interfaces and authorized management services.

Administrative access must be explicitly controlled and documented.

The physical management path to PVE-01 is maintained independently of the virtual lab networks.

### Domain 3 — Laboratory workloads

Includes application VMs, services, testing environments and future P2/P3 components.

Workloads are considered less trusted than the workstation and management infrastructure.

Workload-initiated access to Mint is denied by default, except for explicitly approved services required by the architecture.

Traffic bypassing OPNsense must be controlled by other appropriate network or host security mechanisms.

## 6. Infrastructure Automation

The planned responsibilities are separated as follows:

| Component | Responsibility |
|---|---|
| Terraform | Provisioning VMs, virtual hardware and infrastructure resources |
| Cloud-init | Initial guest configuration, users, SSH keys and networking |
| Ansible | Operating system configuration, hardening and software installation |
| OPNsense | Network routing, firewalling and segmentation |
| Git/GitHub | Version control, architecture documentation and change history |

The implementation must remain modular and reproducible.

## 7. Constraints and Trade-offs

The primary constraints are:

- Approximately 8 GiB RAM on PVE-01.
- Nested virtualization overhead on the ZBook.
- No dedicated managed physical switch.
- Direct Ethernet connectivity between the two computers.
- Dependency on Linux Mint for upstream Internet access.
- Dynamic external network configuration.
- No production-grade high availability.
- Limited storage and compute resources.

The architecture deliberately avoids introducing unnecessary infrastructure components such as Ceph or a mandatory Proxmox cluster.

Services may be activated on demand to reduce resource usage.

## 8. Pending Implementation and Validation

The following items are **not yet operational**:

- VLAN-aware bridge configuration.
- OPNsense VLAN interfaces and firewall policies.
- Inter-VLAN routing.
- PVE-02 deployment.
- PVE-01/PVE-02 Layer 3 workload interconnection.
- Terraform-based VM provisioning.
- Ansible configuration management.
- Network isolation and firewall validation tests.

No architecture diagram or documentation statement should imply that these components have already been successfully deployed.

## 9. Related Documentation

- [Network architecture](network.md)
- [Security architecture](security.md)
- [ADR-001 — Hypervisor topology and workload placement](decisions/ADR-001-hypervisor-topology.md)
- [Project 1 requirements](CDC-P1.md)
