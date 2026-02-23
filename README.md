# 🛸 JHomelab: Master Infrastructure Documentation

This repository documents my personal **home lab environment**, built to strengthen skills in **networking, systems administration, virtualization, and cybersecurity**.  

The lab is designed to simulate **enterprise-level IT and security concepts** within a home setting and is continuously evolving as new technologies and configurations are introduced.

---

## 🧩 System Architecture Overview

### 🌐 Core Network Layer
* **Device:** TP-Link AX6600 Tri-Band Wi-Fi 6 Router  
* **Primary Functions:** Core routing, DHCP, and firewall management.

### 📶 Distribution / Switch Layer
* **Device:** Netgear WN2000RPTv2 – Universal Wi-Fi Range Extender  
* **Status:** SSID broadcasting disabled.  
* **Function:** Wireless bridge / access point to extend network connectivity to lab infrastructure.  
* **🚧 Planned Upgrade:** Transition to **Physical Cat6 Ethernet** backhaul for gigabit stability.

### 🔌 Access Layer
* **Status:** Work in Progress.  
* **Planned:** Additional segmentation and access control for endpoint devices.  
* **🚧 Tier 2 Upgrade:** **Managed Switch Implementation**. Required to enable **Physical VLAN Segmentation** (802.1Q) for hardware-level isolation.

---

## 🏗️ Phase 1: Physical Host (`jhome`)
**Hardware:** Alienware 15 R3 | **Hypervisor:** Proxmox VE 9.1.0 | **Management IP:** `192.168.0.2`

| Component | Specification | Pool / Usage |
| :--- | :--- | :--- |
| **CPU** | Intel i7-7700HQ (4C/8T) | Core Compute |
| **RAM** | 16 GB DDR4 | Over-provisioned Lab Pool |
| **GPU (Host)** | Intel HD Graphics 630 | Console / Host Display |
| **GPU (VM)** | **NVIDIA GTX 1070 Mobile** | Bound to `vfio-pci` (Target: VM 100/109) |
| **SSD (120GB)** | SK Hynix | `local`, `local-lvm` (OS & ISOs) |
| **HDD (1TB)** | HGST | `vmdata` (Primary VM Store / NAS Target) |

---

## 🌐 Phase 2: SDN & IPAM Logic
The lab utilizes **Software Defined Networking (SDN)** to isolate production traffic from experimental zones.



### 🗺️ Network Zones
| Name | Bridge/VNet | Subnet | IPAM Strategy | Gateway |
| :--- | :--- | :--- | :--- | :--- |
| **Home LAN** | `vmbr0` | `192.168.0.0/24` | Static / External DHCP | `192.168.0.1` |
| **Lab SDN** | `testnet` | `10.10.100.0/24` | **PVE IPAM (Dynamic DHCP)** | `10.10.100.1` |

> [!IMPORTANT]
> **Static Mapping Logic**: Lab VMs utilize a 1:1 mapping where the last octet matches the VMID suffix.  
> *Example: VM **105** → IP **10.10.100.105***

---

## 🖥️ Phase 3: Virtual Machine Inventory
> **RAM Strategy**: Total assigned (~48GB) > Physical (16GB). Only **VM 109** is set to **Auto-Boot**.

### 🏠 Infrastructure & High-Performance
| VMID | Name | IP Address | OS | RAM | Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **109** | `Ubuntu-Server` | **192.168.0.159** | Ubuntu 24.04 | 2 GB | **Running** |
| **100** | `Ubuntu-Desktop`| **10.10.100.100** | Ubuntu Desktop | 8 GB | Stopped |
| **102** | `opnsense` | **10.10.100.102** | FreeBSD (OPNsense) | 3 GB | **🚧 WIP** |

> [!NOTE]
> **VM 102 Configuration**: Currently being configured for **Single-NIC topology**. It will use a virtual interface on `vmbr0` as WAN and a virtual interface on `testnet` as LAN to route traffic internally within the host.

### 🧪 Cyber Range & Distro Lab (SDN: `testnet`)
| VMID | Name | IP Address | OS Distro | RAM | Disk |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **101** | `Kali` | **10.10.100.101** | Kali Linux | 2 GB | 30 GB |
| **103** | `OpenSUSE-Desktop`| **10.10.100.103** | OpenSUSE Tumbleweed | 4 GB | 30 GB |
| **104** | `Fedora-Desktop` | **10.10.100.104** | Fedora 40 | 4 GB | 30 GB |
| **105** | `ZorinOS` | **10.10.100.105** | Zorin OS 17 | 8 GB | 40 GB |
| **106** | `Manjaro-Desktop` | **10.10.100.106** | Manjaro (Arch) | 4 GB | 30 GB |
| **107** | `Linux-Mint` | **10.10.100.107** | Linux Mint 21 | 4 GB | 30 GB |
| **108** | `PopOS` | **10.10.100.108** | Pop!_OS | 8 GB | 30 GB |

---

## 📦 Phase 4: Primary Services (VM 109)
Docker stack managed via **Portainer**.

| Service | Port | Description |
| :--- | :--- | :--- |
| **Homepage** | `3000` | Central Dashboard |
| **Uptime Kuma** | `3001` | Service Health Monitoring |
| **Portainer** | `9443` | Container Orchestration |
| **Nginx Proxy Manager**| `81` | (Planned) Reverse Proxy/SSL Management |
| **RustDesk Server** | `21115+` | (Planned) Self-hosted Remote Support |

---

## 🎯 Phase 5: Future Roadmap & Goals

### 🛠️ Core Infrastructure & Services
- [ ] **Ethernet Overhaul**: Replace wireless bridge with physical Cat6 cabling.
- [ ] **Managed Switch**: Procure and configure for **Physical VLAN tagging (802.1Q)**.
- [ ] **Nginx Proxy Manager**: Implement clean internal domains (e.g., `proxmox.home`).
- [ ] **Internal DNS**: Deploy **AdGuard Home** or **Pi-hole**.
- [ ] **NAS VM**: Dedicated storage VM for SMB/NFS using the 1TB HDD.

### 🛡️ Cybersecurity Lab Expansion
- [ ] **Tailscale Subnet Router**: Centralize remote access on VM 109 for the `10.10.100.x` range.
- [ ] **VLAN Firewall Rules**: Strict hardware/software segmentation between zones.
- [ ] **Wazuh / Security Onion**: Deploy enterprise SIEM and IDS for traffic monitoring.
- [ ] **Jellyfin**: Hardware-accelerated media server using the GTX 1070.

---

## 💾 Maintenance & Configs
* **PCI Isolation:** `pcie_acs_override=downstream,multifunction` active in GRUB.
* **IOMMU Health:** NVIDIA GP104BM verified bound to `vfio-pci`.
* **SDN Configs:** `/etc/pve/sdn/`

---

## 🧾 Operational Journal (Changelog)

### February 2026 — Infrastructure Deep-Dive
- **Audited Proxmox Host:** Verified Kernel 6.17 and PVE 9.1 stability.
- **Hardware Inventory:** Documented GTX 1070 status and mapped out Distro Lab inventory (VMIDs 100-109).
- **Network Mapping:** Finalized SDN `testnet` logic and implemented the `VMID-to-IP` mapping system (10.10.100.1xx).
- **Strategic Roadmap:** Expanded goals to include SIEM (Wazuh), Proxy (NPM), Managed Switching, and DNS.

### January 2026 — Foundational Phase
- Repurposed TP-Link AX6600 as the core routing device.
- Segmented wireless network into two SSIDs: `SSID_ExistingNetwork` and `SSID_HomelabNetwork`.
- Installed **Proxmox Virtual Environment** on Alienware host.
- Created initial VM stack (Ubuntu Server, Ubuntu Desktop, Kali Linux).
- **Verified:**
  - Static/dynamic IP assignments.
  - Gateway routing.
  - IP sanitation and connectivity via Linux CLI tools.

---

## 📌 Goals

- Simulate real-world enterprise networking environments.
- Gain hands-on experience with virtualization and network segmentation.
- Practice security hardening, monitoring, and access control.
- Build a documented, reproducible lab suitable for learning and experimentation.
