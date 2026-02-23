# 🛸 JHomelab: Master Infrastructure Documentation

This repository documents my personal **home lab environment**, built to strengthen skills in **networking, systems administration, virtualization, and cybersecurity**.

---

## 🧩 Network Architecture

### 🌐 Layer 3: Core Routing
* **Device:** TP-Link AX6600 Tri-Band Wi-Fi 6 Router
* **Functions:** Core routing, DHCP, and Firewall management.
* **Primary SSIDs:** * `SSID_ExistingNetwork` (General Use)
    * `SSID_HomelabNetwork` (Dedicated Lab Access)

### 📶 Layer 2: Distribution (Current)
* **Device:** WN2000RPTv2 (Universal Wi-Fi Range Extender)
* **Mode:** Wireless Bridge / Access Point (SSID broadcasting disabled).
* **🚧 Planned Upgrade:** Transitioning to **Physical Cat6 Ethernet** backhaul to the TP-Link AX6600 for gigabit stability.

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
| **HDD (1TB)** | HGST | `vmdata` (Primary VM Store / 864GB Free) |

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

### 🏠 Infrastructure & Workstations
| VMID | Name | IP Address | Status | Role |
| :--- | :--- | :--- | :--- | :--- |
| **109** | `Ubuntu-Server` | **192.168.0.159** | Running | **Core Infra** (Docker/Monitoring) |
| **100** | `Ubuntu-Desktop`| **10.10.100.100** | Stopped | Workstation (GTX 1070 / Tailscale) |

### 🧪 Cyber Range & Distro Lab (SDN)
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
Docker stack managed via **Portainer** at `https://192.168.0.159:9443`.

| Service | Port | Description |
| :--- | :--- | :--- |
| **Homepage** | `3000` | Central Dashboard |
| **Uptime Kuma** | `3001` | Service Health Monitoring |
| **Portainer** | `9443` | Container Orchestration |
| **Watchtower** | N/A | Automated Image Updates |
| **RustDesk** | N/A | (Planned) Self-hosted Remote Desktop |

---

## 🎯 Phase 5: Future Roadmap & Goals

### 🛠️ Short-Term Projects
- [ ] **Tailscale Migration**: Move Tailscale from VM 100 to VM 109 to act as a **Subnet Router**.
- [ ] **Subnet Routing**: Advertise `10.10.100.0/24` via Tailscale for remote lab access.
- [ ] **Jellyfin/Anime Server**: Deploy on VM 109 using 1TB HDD and GTX 1070 hardware transcoding.
- [ ] **Ethernet Overhaul**: Run physical cabling to replace wireless bridging.

### 🛡️ Cybersecurity & Networking
- [ ] **OPNsense Firewall**: Configure VM 102 as a gateway/firewall for the Lab SDN.
- [ ] **Pi-hole**: Network-wide ad-blocking and DNS sinkhole.
- [ ] **VLAN Expansion**: Increased segmentation for IoT and Guest devices.
- [ ] **SIEM/Logging**: Implement monitoring tools to track "attack" traffic in the Cyber Range.

---

## 💾 Maintenance & Configs
* **PCI Isolation:** `pcie_acs_override=downstream,multifunction` active in GRUB.
* **IOMMU Health:** NVIDIA GP104BM verified bound to `vfio-pci`.
* **SDN State:** Configs managed in `/etc/pve/sdn/`.

---

## 🧾 Operational Journal (Changelog)

### February 2026 — Infrastructure Deep-Dive
- **Audited Proxmox Host:** Verified Kernel 6.17 and PVE 9.1 stability.
- **Hardware Inventory:** Documented GTX 1070 passthrough status and storage distribution (SSD vs HDD).
- **Network Mapping:** Finalized SDN `testnet` logic and implemented the `VMID-to-IP` mapping system (10.10.100.1xx).
- **Docker Audit:** Verified health of Homepage, Uptime Kuma, and Portainer.
- **Strategic Planning:** Defined roadmap for Tailscale centralization and media server deployment.

### January 2026 — Foundational Phase
- Repurposed TP-Link AX6600 as the core routing device.
- Segmented wireless network into dual SSIDs.
- Installed Proxmox VE and provisioned initial VM stack (Ubuntu/Kali).
