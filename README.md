# 🛸 JHomelab: Master Infrastructure Documentation

This repository documents my personal **home lab environment**, built to strengthen skills in **networking, systems administration, virtualization, and cybersecurity**.

---

## 🧩 Network Architecture

### 🌐 Layer 3: Core Routing
* **Device:** TP-Link AX6600 Tri-Band Wi-Fi 6 Router
* **Functions:** Core routing, DHCP, and Firewall management.

### 📶 Layer 2: Distribution & Access (Evolution Path)
* **Current State:** WN2000RPTv2 (Wireless Bridge mode).
* **🚧 Tier 1 Upgrade:** Physical **Cat6 Ethernet** backhaul to replace wireless bridging.
* **🚧 Tier 2 Upgrade:** **Managed Switch Implementation**. Required to enable **Physical VLAN Segmentation** (802.1Q) for hardware-level isolation.

---

## 🏗️ Phase 1: Physical Host (`jhome`)
**Hardware:** Alienware 15 R3 | **Hypervisor:** Proxmox VE 9.1.0 | **Management IP:** `192.168.0.2`

| Component | Specification | Pool / Usage |
| :--- | :--- | :--- |
| **CPU** | Intel i7-7700HQ (4C/8T) | Core Compute |
| **RAM** | 16 GB DDR4 | Over-provisioned Lab Pool |
| **GPU (VM)** | **NVIDIA GTX 1070 Mobile** | Bound to `vfio-pci` (Target: VM 100/109) |
| **SSD (120GB)** | SK Hynix | `local`, `local-lvm` (OS & ISOs) |
| **HDD (1TB)** | HGST | `vmdata` (Primary VM Store / NAS Target) |

---

## 🌐 Phase 2: SDN & IPAM Logic
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

### 🏠 Infrastructure & Workstations
| VMID | Name | IP Address | Status | Role |
| :--- | :--- | :--- | :--- | :--- |
| **109** | `Ubuntu-Server` | **192.168.0.159** | Running | **Core Infra** (Docker/Monitoring) |
| **100** | `Ubuntu-Desktop`| **10.10.100.100** | Stopped | Workstation (GTX 1070 / Tailscale) |
| **TBD** | `NAS-VM` | **TBD** | Planned | Dedicated Storage / Network Shares |

### 🧪 Cyber Range & Distro Lab (SDN)
| VMID | Name | IP Address | OS | RAM |
| :--- | :--- | :--- | :--- | :--- |
| **101** | `Kali` | **10.10.100.101** | Kali Linux | 2GB |
| **102** | `opnsense` | **10.10.100.102** | FreeBSD | 3GB |
| **105** | `ZorinOS` | **10.10.100.105** | Zorin OS | 8GB |
| **108** | `PopOS` | **10.10.100.108** | Pop!_OS | 8GB |
| **103-107**| `Distro-Hop` | **10.10.100.10x** | Various | 4GB |

---

## 📦 Phase 4: Primary Services (VM 109)
Docker stack managed via **Portainer**.

| Service | Port | Description |
| :--- | :--- | :--- |
| **Homepage** | `3000` | Central Dashboard |
| **Uptime Kuma** | `3001` | Service Health Monitoring |
| **Portainer** | `9443` | Container Orchestration |
| **Nginx Proxy Manager**| `81` | (Planned) Reverse Proxy/SSL |
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
- [ ] **Tailscale Subnet Router**: Centralize remote access on VM 109.
- [ ] **VLAN Firewall Rules**: Strict hardware/software segmentation.
- [ ] **Wazuh / Security Onion**: Deploy enterprise SIEM and IDS for traffic monitoring.
- [ ] **Jellyfin**: Hardware-accelerated media server using the GTX 1070.

---

## 💾 Maintenance & Configs
* **PCI Isolation:** `pcie_acs_override=downstream,multifunction` active in GRUB.
* **IOMMU Health:** NVIDIA GP104BM verified bound to `vfio-pci`.

---

## 🧾 Operational Journal (Changelog)

### February 2026 — Infrastructure Deep-Dive
- **Audited Proxmox Host:** Verified Kernel 6.17 and PVE 9.1 stability.
- **Hardware Inventory:** Documented GTX 1070 passthrough status and storage distribution.
- **Network Mapping:** Finalized SDN `testnet` logic and implemented the `VMID-to-IP` mapping system.
- **Roadmap Expanded:** Integrated SIEM (Wazuh), Proxy (NPM), Managed Switching, and DNS goals.

### January 2026 — Foundational Phase
- Repurposed TP-Link AX6600 as the core routing device.
- Segmented wireless network into two SSIDs: `SSID_ExistingNetwork` and `SSID_HomelabNetwork`.
- Installed **Proxmox Virtual Environment** on Alienware host.
- Created initial VM stack (Ubuntu Server, Ubuntu Desktop, Kali Linux).
- **Verified:**
  - Static/dynamic IP assignments.
  - Gateway routing.
  - IP sanitation and connectivity via Linux CLI tools.
