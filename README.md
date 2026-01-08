# JHomelab
This repository documents my Home lab. The Goal of it is to strenghten my skills in networking and cybersec concepts. This homelab is designed to simulate enterprise-level IT, networking, and cybersecurity concepts in a home environment.

Homelab is managed by me still a work in progress.

🧩 System Architecture Overview
------------------------------------------
Device: TP-Link AX6600 Tri-Band Wi-Fi 6 Router
Functions: Core routing, DHCP, VLAN trunking, firewall management, VPN endpoint
Features Enabled:
IP/MAC binding for static endpoints (Proxmox, NAS, main PC)
Port forwarding for key services (Caddy HTTP/HTTPS, qBittorrent)
Dynamic DNS: Custom No-IP domain ensuring VPN resilience even under ISP IP changes
VPN: OpenVPN
Pi-hole: Network wide adblocking


Switch Layer
------------------------------------------
Device: WN2000RPTv2 - Universal WiFi Range Extender 
SSID broadcasting turned off to act as a wireless access point/switch


Access Layer
------------------------------------------
(WIP)



🧱 Compute & Virtualization Layer
Host	        Role	                     Key Workloads	                                                                                                 Notes
Proxmox 1 ||	Network & Security Core	|| Pi-hole VM, (future OPNsense)                                                                                || Higher RAM allocation, dedicated for infrastructure tasks
Proxmox 2 ||  Application Layer	      || Proxmox VM running Ubuntu Server + Ubuntu + Kali Linux, Docker VM running Portainer Agent + Omada Controller	|| 24/7 uptime, reduced dependency on main PC




🧾 Operational Journal (Full Changelog)
Foundational Phase (January 2026)
Using existing TP-Link AX6600 as core router.
Segmented Wifi into two SSIDs:
  -SSID_ExistingNetwork
  -SSID_HomelabNetwork
Installed Proxmox Virtual Enviroment on a spare custom pc.
created three VM partitions
  -Ubuntu Server (80 GB)
  -Ubuntu Desktop
  -Kali Linux
Verified static IP configuration, AdGuard DNS, gateway routing, and IP sanitation via Linux CLI.




🌐 Network Segmentation & VLAN Policy
VLAN	Subnet	        Purpose	Notes
1	    192.168.0.0/24	Core Infrastructure	Router, Proxmox, NAS, Controller, key services
2	    192.168.30.0/24	Stable “Untouched” Home Network	Partner and household devices — isolated from homelab VLANs
