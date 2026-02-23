# JHomelab

This repository documents my personal **home lab environment**, built to strengthen my skills in **networking, systems administration, virtualization, and cybersecurity**.  

The lab is designed to simulate **enterprise-level IT and security concepts** within a home setting and is continuously evolving as new technologies and configurations are introduced.

---

## 🧩 System Architecture Overview

### Core Network Layer
**Device:** TP-Link AX6600 Tri-Band Wi-Fi 6 Router  
**Primary Functions:**
- Core routing and DHCP
- firewall management

---

### Distribution / Switch Layer
**Device:** WN2000RPTv2 – Universal Wi-Fi Range Extender  

- SSID broadcasting disabled  
- Functions as a **wireless bridge / access point** to extend network connectivity to lab infrastructure

---

### Access Layer
*(Work in Progress)*

Planned implementation includes additional segmentation and access control for endpoint devices.

---

# 🛸 AlienLab: Master Infrastructure Documentation

## 🏗️ Phase 1: Physical Host (`jhome`)
**Hardware:** Alienware 15 R3  
**Hypervisor:** Proxmox VE 9.1.0 (Kernel: 6.17.9-1-pve)  
**Management IP:** `192.168.0.2`

### 💻 Hardware & Storage
| Component | Specification | Pool / Usage |
| :--- | :--- | :--- |
| **CPU** | Intel i7-7700HQ (4C/8T) | Core Compute |
| **RAM** | 16 GB DDR4 | Over-provisioned Lab Pool |
| **GPU (Host)** | Intel HD Graphics 630 | Console / Host Display |
| **GPU (Passthrough)** | **NVIDIA GTX 1070 Mobile** | Bound to `vfio-pci` (VM 100) |
| **SSD (120GB)** | SK Hynix | `local`, `local-lvm` (OS & ISOs) |
| **HDD (1TB)** | HGST | `vmdata` (Primary VM Store / 864GB Free) |

---

## 🌐 Phase 2: Network & IPAM Logic
The lab uses **Software Defined Networking (SDN)** to isolate production traffic from testing environments.

### 🗺️ Network Zones
| Name | Bridge/VNet | Subnet | IPAM Strategy | Gateway |
| :--- | :--- | :--- | :--- | :--- |
| **Home LAN** | `vmbr0` | `192.168.0.0/24` | Static / External DHCP | `192.168.0.1` |
| **Lab SDN** | `testnet` | `10.10.100.0/24` | **PVE IPAM (Dynamic DHCP)** | `10.10.100.1` |

> [!NOTE]
> **Static Mapping**: All Lab VMs utilize a 1:1 mapping where the last octet of the IP matches the last digit of the VMID (e.g., VM 105 = `.105`).

---

## 🖥️ Phase 3: Virtual Machine Inventory
> **RAM Strategy**: Total assigned (~48GB) > Physical (16GB). Only **VM 109** is set to **Auto-Boot**.



### 🏠 Production & Management (Bridged)
| VMID | Name | IP Address | Status | Role |
| :--- | :--- | :--- | :--- | :--- |
| **109** | `Ubuntu-Server` | **192.168.0.159** | Running | Core Infra (Docker/Monitoring) |
| **100** | `Ubuntu-Desktop`| **10.10.100.100** | Stopped | Workstation (GTX 1070 / Tailscale) |

### 🧪 Cyber Range & Distro Lab (SDN / IPAM)
| VMID | Name | IP Address | OS | RAM |
| :--- | :--- | :--- | :--- | :--- |
| **101** | `Kali` | **10.10.100.101** | Kali Linux | 2GB |
| **102** | `opnsense` | **10.10.100.102** | FreeBSD | 3GB |
| **103** | `OpenSUSE` | **10.10.100.103** | Tumbleweed | 4GB |
| **104** | `Fedora` | **10.10.100.104** | Fedora 40 | 4GB |
| **105** | `ZorinOS` | **10.10.100.105** | Zorin OS | 8GB |
| **106** | `Manjaro` | **10.10.100.106** | Rolling | 4GB |
| **107** | `Linux-Mint` | **10.10.100.107** | Mint 21.x | 4GB |
| **108** | `PopOS` | **10.10.100.108** | Pop!_OS | 8GB |

---

## 📦 Phase 4: Primary Services (VM 109)
Docker stack managed via Portainer at `https://192.168.0.159:9443`.

| Service | Port | Description |
| :--- | :--- | :--- |
| **Homepage** | `3000` | Central Dashboard |
| **Uptime Kuma** | `3001` | Service Health Monitoring |
| **Portainer** | `9443` | Web UI for Container Management |
| **Watchtower** | N/A | Automated Image Updates |

---

## 🎯 Phase 5: Future Roadmap
- [ ] **Tailscale Migration**: Move Tailscale from VM 100 to VM 109.
- [ ] **Subnet Routing**: Advertise `10.10.100.0/24` via Tailscale for remote lab access.
- [ ] **Jellyfin**: Deploy via Docker on VM 109 with GTX 1070 Transcoding.
- [ ] **Networking**: Enable OPNsense (VM 102) as a gateway/firewall for the Lab SDN.

---

## 💾 Maintenance & Configs
- **PCI Isolation**: `pcie_acs_override=downstream,multifunction` active in GRUB.
- **IOMMU Health**: NVIDIA GP104BM verified bound to `vfio-pci`.
- **SDN Configs**: `/etc/pve/sdn/`
## 🧾 Operational Journal (Changelog)

### Foundational Phase — January 2026
- Repurposed TP-Link AX6600 as the core routing device
- Segmented wireless network into two SSIDs:
  - `SSID_ExistingNetwork`
  - `SSID_HomelabNetwork`
- Installed **Proxmox Virtual Environment** on a spare custom-built PC
- Created multiple virtual machines:
  - Ubuntu Server
  - Ubuntu Desktop
  - Kali Linux
- Verified:
  - Static/dynamic IP assignments
  - Gateway routing
  - IP sanitation and connectivity via Linux CLI tools

---

## 🚧 Project Status

This homelab is **actively maintained and expanded**. Future improvements include:
- Dedicated firewall deployment (OPNsense)
- Expanded VLAN architecture
- Improved access layer design
- Additional security monitoring and logging tools
- Pihole

---

## 📌 Goals

- Simulate real-world enterprise networking environments
- Gain hands-on experience with virtualization and network segmentation
- Practice security hardening, monitoring, and access control
- Build a documented, reproducible lab suitable for learning and experimentation
