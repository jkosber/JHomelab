# JHomelab

This repository documents my personal **home lab environment**, built to strengthen my skills in **networking, systems administration, virtualization, and cybersecurity**.  

The lab is designed to simulate **enterprise-level IT and security concepts** within a home setting and is continuously evolving as new technologies and configurations are introduced.

---

## 🧩 System Architecture Overview

### Core Network Layer
**Device:** TP-Link AX6600 Tri-Band Wi-Fi 6 Router  
**Primary Functions:**
- Core routing and DHCP
- VLAN trunking and firewall management
- VPN endpoint

**Enabled Features:**
- **IP/MAC binding** for static infrastructure endpoints (Proxmox hosts, NAS, main PC)
- **Port forwarding** for internal services (Caddy HTTP/HTTPS, qBittorrent)
- **Dynamic DNS** using a custom No-IP domain to maintain VPN access despite ISP IP changes
- **VPN:** OpenVPN
- **Network-wide ad blocking:** Pi-hole

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

## 🧱 Compute & Virtualization Layer

| Host       | Role                     | Key Workloads                                                                 | Notes |
|------------|--------------------------|--------------------------------------------------------------------------------|-------|
| **Proxmox 1** | Network & Security Core | Pi-hole VM, future OPNsense firewall                                           | Higher RAM allocation, dedicated to infrastructure services |
| **Proxmox 2** | Application Layer       | Ubuntu Server, Ubuntu Desktop, Kali Linux, Docker VM (Portainer Agent, Omada Controller) | 24/7 uptime, minimal dependency on main workstation |

---

## 🌐 Network Segmentation & VLAN Policy

| VLAN | Subnet              | Purpose                              | Notes |
|------|---------------------|--------------------------------------|-------|
| 1    | 192.168.0.0/24      | Core Infrastructure                  | Router, Proxmox, NAS, controllers, critical services |
| 2    | 192.168.30.0/24     | Stable Home Network                  | Household and partner devices, isolated from homelab |

---

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
  - Static IP assignments
  - AdGuard DNS configuration
  - Gateway routing
  - IP sanitation and connectivity via Linux CLI tools

---

## 🚧 Project Status

This homelab is **actively maintained and expanded**. Future improvements include:
- Dedicated firewall deployment (OPNsense)
- Expanded VLAN architecture
- Improved access layer design
- Additional security monitoring and logging tools

---

## 📌 Goals

- Simulate real-world enterprise networking environments
- Gain hands-on experience with virtualization and network segmentation
- Practice security hardening, monitoring, and access control
- Build a documented, reproducible lab suitable for learning and experimentation
