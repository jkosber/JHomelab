# JHomelab

My personal homelab, running Proxmox VE on an old Alienware 15 R3. I use it to practice the stuff I don't get to touch at work or in class — running a real hypervisor, carving up an SDN with separate subnets, writing firewall rules, and standing up Linux services in Docker. This repo is the running journal: hardware specs, network layout, VM inventory, what's deployed, what's planned next, and a changelog of what I actually did and when.

## What's in this repo

Right now this is a documentation-only repo. I'll add sanitized configs (SDN exports, firewall rules, Docker compose files, Bash scripts) as separate sub-directories once they settle. The course repos that fed into this build are:

- [SVAD-111-Linux-Virtualization](https://github.com/jkosber/SVAD-111-Linux-Virtualization) — Linux admin and virtualization coursework
- [Networking-109](https://github.com/jkosber/Networking-109) — CCNA Networking I (Cisco IOS, addressing, routing)
- [CyberOps-115](https://github.com/jkosber/CyberOps-115) — Cisco CyberOps Associate (Security Onion, Snort, packet analysis)

Last update: **October 23, 2026**. The [October maintenance notes](docs/maintenance-2026-10-23.md) cover the latest round of patching, monitoring and the dashboard. The [September 22 health check](docs/health-check-2026-09-22.md) is the earlier baseline. Local IPs have the last number replaced with `X`.

## What I get out of running it

- Hypervisor admin on a real Proxmox install (currently PVE 9.2.21, kernel 7.0.14-25-pve).
- Software-defined networking — VNets, managed IPAM/DHCP, predictable per-VM addressing.
- Per-distribution Linux practice (Debian/Ubuntu, RHEL/Fedora, Arch, SUSE, Pop!, Mint, Zorin) without polluting my daily-driver.
- PCI passthrough — the GTX 1070 Mobile is bound to `vfio-pci` for VM use.
- Docker via Portainer for the always-on services on VM 109.
- Monitoring and alerting: Beszel for host and container health, drive SMART and pending updates, Uptime Kuma for service checks, both alerting to Discord.
- Patch management across the host, guests and containers, including a real CA-compromise response in October 2026.
- A documented, reproducible lab I can rebuild from notes if the host gets wiped.

## Network layout

- **Core router** — TP-Link AX6600 Tri-Band Wi-Fi 6. Routing, DHCP, edge firewall.
- **Distribution** — Netgear WN2000RPTv2 range extender, SSID broadcast off, used as a wireless bridge into the lab segment. Planned upgrade: replace with a Cat6 backhaul for gigabit stability.
- **Access** — still flat. Plan is to drop in a managed switch so I can do 802.1Q VLAN tagging between home, lab, and management.

## Physical host — `jhome`

| Component | Specification | Role / Pool |
| :--- | :--- | :--- |
| Platform | Alienware 15 R3 | Hypervisor host |
| Hypervisor | Proxmox VE 9.2.21 / kernel 7.0.14-25-pve | Bare metal |
| Management IP | `192.168.0.X` | Web UI at `https://192.168.0.X:8006` |
| CPU | Intel i7-7700HQ (4C / 8T) | Core compute |
| RAM | 16 GB DDR4 | Over-provisioned across the lab guests |
| GPU (host) | Intel HD Graphics 630 | Console / host display |
| GPU (passthrough) | NVIDIA GTX 1070 Mobile | Bound to `vfio-pci`, configured for VMs 100, 105, 107 and 108 |
| SSD | SK Hynix 128 GB | `local`, `local-lvm` (OS, ISOs); aging, SMART-monitored |
| HDD | HGST 1 TB | `vmdata` (VM store today; planned backup drive) |
| NVMe | Leftover Windows SSD | Hidden by Intel RST RAID mode; planned VM store once the BIOS is switched to AHCI |

## SDN and IPAM

I'm using Proxmox's built-in SDN for the experimental lab subnet. Anything I'm poking at — broken Linux installs, Kali scans, opnsense builds — lives on `testnet`. It's already configured for VMs 100–108. The service guests, VM 109 and CT 110, are still on `vmbr0`; moving them onto the SDN is planned.

| Network | Zone | Bridge / VNet | Subnet | IPAM | Gateway |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Home LAN | — | `vmbr0` | `192.168.0.X/24` | Static / external DHCP | `192.168.0.X` |
| Lab SDN | `test` | `testnet` | `10.10.100.X/24` | PVE IPAM (DHCP) | `10.10.100.X` |

Lab VMs have address mappings in IPAM so they're easy to track.

![SDN IPAM mappings](screenshots/sdn-ipam-2026-09-22.png)

*IPAM mappings for the lab gateway and VMs 100–108. IPs and MAC addresses are masked.*

### Firewall (Datacenter level)

The base input policy is **DROP**: incoming traffic needs an accept rule. Output and forwarding are set to **ACCEPT**.

The seven allow rules cover lab network access, DNS and DHCP, plus management access for Ping, SSH on 22 and the web UI on 8006.

![Datacenter firewall rules](screenshots/datacenter-firewall-2026-09-22.png)

*Datacenter > Firewall — seven enabled allow rules. Per-VNet and per-VM firewalls are the next layer to fill in.*

![Datacenter firewall defaults](screenshots/firewall-options-2026-09-22.png)

*Firewall enabled, with default-drop input.*

## Virtual machines

The VMs have about 47 GB of RAM assigned, plus 4 GB for the GoodMem container and 512 MB for the Caddy pilot container, against 16 GB in the host. VM 109 and CT 110 are set to auto-boot. The other guests stay stopped until I'm working on a lab.

![Datacenter inventory in the Proxmox web UI](screenshots/guest-inventory-2026-09-22.webp)

*September 22 — VM 109 and CT 110 running, with VMs 100–108 stopped.*

### Always-on / infrastructure

| ID | Name | IP | OS | RAM | Disk | State | Role |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| VM 109 | `Ubuntu-Server` | `192.168.0.X` | Ubuntu Server 26.04.1 LTS | 2 GB | 32 GB | Running | Core infra / Docker host |
| CT 110 | `goodmem-vec` | `192.168.0.X` | Debian 13.7 | 4 GB | 24 GB | Running | GoodMem / PostgreSQL |

### Cyber range / distro lab (`testnet`)

All lab guests were stopped at the time of the update.

| VMID | Name | IP | OS | RAM | Disk |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 100 | `Ubuntu-Desktop` | `10.10.100.X` | Ubuntu Desktop | 8 GB | 200 GB |
| 101 | `Kali` | `10.10.100.X` | Kali Linux | 2 GB | 30 GB |
| 102 | `opnsense` | `10.10.100.X` | FreeBSD / OPNsense | 3 GB | 16 GB |
| 103 | `OpenSUSE-Desktop` | `10.10.100.X` | openSUSE | 4 GB | 30 GB |
| 104 | `Fedora-Desktop` | `10.10.100.X` | Fedora | 4 GB | 30 GB |
| 105 | `ZorinOS` | `10.10.100.X` | Zorin OS | 8 GB | 40 GB |
| 106 | `Manjaro-Desktop` | `10.10.100.X` | Manjaro (Arch) | 4 GB | 30 GB |
| 107 | `Linux-Mint` | `10.10.100.X` | Linux Mint | 4 GB | 30 GB |
| 108 | `PopOS` | `10.10.100.X` | Pop!_OS | 8 GB | 30 GB |
| CT 111 | `caddy-pilot` | `10.10.100.X` | Debian 13.6 | 512 MB | 4 GB |

VM 100 also serves as the Tailscale node. The single-NIC OPNsense gateway on VM 102 is still a work in progress. CT 111 is a Caddy reverse-proxy pilot, built in October to test internal HTTPS hostnames in front of the dashboard.

## Services on VM 109

Docker Engine 29.1.3, with the stack managed through Portainer.

| Service | Port | State | What it does |
| :--- | :--- | :--- | :--- |
| Homepage 2.7.0 | 3000 | Running | Central dashboard with live Proxmox, Portainer, Beszel and Uptime Kuma panels |
| Uptime Kuma 2.5.7 | 3001 | Running | Service checks, a LAN status page and Discord alerts |
| Beszel 0.21.1 | 8090 | Running | Host and container health, drive SMART, pending updates, Discord alerts |
| Portainer 2.45.3 | 9443 | Running | Container management UI |
| RustDesk Server 1.1.16 | 21115–21119 | Running | Self-hosted remote support |
| Watchtower (maintained fork) | — | Running | Daily image updates for Homepage, Kuma, Beszel and RustDesk |
| Nginx Proxy Manager | 81 | Planned | Reverse proxy + Let's Encrypt (Caddy is being piloted on CT 111) |

Every service responded on October 23, and Uptime Kuma now checks them each minute.

![Homepage dashboard](screenshots/homepage-dashboard-2026-10-23.png)

*October 23: the Homepage dashboard, configured with live data this time.*

## Services on CT 110

GoodMem runs in a separate Debian LXC container.

| Service | Port | State | What it does |
| :--- | :--- | :--- | :--- |
| GoodMem server v1.0.331 | 8080 (REST), 9090 (gRPC) | Healthy | Memory service |
| PostgreSQL 17.11 / pgvector | 5432 | Running | GoodMem database and vector storage |

## Monitoring and alerting

A Beszel agent runs on the Proxmox host, VM 109 and CT 110 and reports to the hub on VM 109. Uptime Kuma checks the services from the outside. Both send alerts to a private Discord channel.

| Alert | From | Fires when |
| :--- | :--- | :--- |
| Machine down | Beszel | A host or guest stops reporting for 2 minutes |
| Resources | Beszel | CPU or memory above 90% for 10 minutes, or a disk above 85% |
| Drive health | Beszel | Any drive's SMART state gets worse |
| Services and containers | Beszel | A systemd service fails or a Docker health check goes unhealthy |
| Heat | Beszel | The laptop host runs above 85 °C for 5 minutes |
| Service down | Uptime Kuma | Two failed checks in a row on the web UI, GoodMem, Portainer, Beszel, RustDesk or my portfolio site |
| Certificate | Uptime Kuma | The portfolio site's TLS certificate is close to expiring |

Beszel also lists pending package updates per machine and flags the security ones. It's how I found 87 pending updates on CT 110.

## Roadmap

**Infrastructure**

- Switch the BIOS from RAID to AHCI, move guest storage onto the NVMe, and turn the 1 TB HDD into the target for scheduled backups.
- An off-host watchdog that alerts if the whole Proxmox host goes down, since the monitoring runs on it.
- Cat6 backhaul to replace the wireless bridge.
- Managed switch + 802.1Q VLAN tagging.
- Extend the SDN to the service guests.
- Nginx Proxy Manager so I can use clean internal hostnames (`proxmox.home`, `dash.home`, etc.).
- Internal DNS — leaning AdGuard Home over Pi-hole.
- A dedicated NAS for SMB / NFS (the 1 TB HDD is now earmarked for backups).

**Security / cyber lab**

- Move the Tailscale subnet router off VM 100 onto VM 109 so it's always reachable.
- Per-zone firewall rules in PVE for strict segmentation between home, lab, and management.
- Wazuh or Security Onion for SIEM / IDS on inter-zone traffic.
- Jellyfin with the GTX 1070 doing hardware-accelerated transcoding.

## Maintenance notes

- GRUB has `pcie_acs_override=downstream` for PCI passthrough.
- IOMMU verified — NVIDIA GP104BM (GTX 1070 Mobile) is bound to `vfio-pci`.
- SDN config lives in `/etc/pve/sdn/` on the host.
- APT uses only the deb822 trixie sources. The leftover Proxmox 8 (bookworm) entries were removed in October.
- VM 109 runs `qemu-guest-agent`, so snapshots and backups get a consistent filesystem.
- The boot SSD's SMART warnings haven't changed since September, and Beszel alerts if they get worse. There are still no scheduled backups; that's the next project ([details](docs/maintenance-2026-10-23.md#storage-finding)).

## Changelog

### October 2026 — patching, monitoring and a real dashboard

- **Oct 3–5:** Built CT 111 as a Caddy reverse-proxy pilot with internal HTTPS, using Homepage as its first backend. Inventoried and upgraded the VM 109 stack: Uptime Kuma migrated from v1 to v2, Homepage and Portainer upgraded, RustDesk server running. Repaired Watchtower against Docker's newer minimum API.
- **Oct 21:** Responded to the eMudhra (emSign) CA compromise. Removed trust for the compromised root on the host and guests, updated GoodMem, PostgreSQL and Docker on CT 110 and Portainer on VM 109, with snapshots first. Wrote a read-only script that reports patch status per host.
- **Oct 23:** Installed every pending update on the host, VM 109 and CT 110 (88 packages on CT 110 alone, 33 of them security). Cleaned up stale APT sources and an old kernel. Swapped the archived Watchtower for its maintained fork. Installed the guest agent on VM 109 and grew its disk. Set up Beszel and Uptime Kuma with Discord alerts, and brought Homepage back with live panels. Found the hidden NVMe drive and planned the storage and backup move. [Notes](docs/maintenance-2026-10-23.md).

### September 2026 — Proxmox update

- Updated PVE and kernel versions, VM resources and GPU assignments.
- Added CT 110 and its running GoodMem / PostgreSQL services.
- Refreshed the inventory, firewall and IPAM screenshots.
- Documented the firewall defaults and planned SDN rollout to the service guests.
- Kept the IP schemes in the notes, with the last number masked.

### April 2026 — service tier

- Confirmed VM 109 as the only auto-boot VM. The over-committed RAM pool only matters when I'm running a scenario.
- Lined up Nginx Proxy Manager + RustDesk Server as the next services to deploy.

### February 2026 — infrastructure pass

- Audited the host. Kernel 6.17 + PVE 9.1 stable.
- Documented the GTX 1070 passthrough state and the full distro-lab inventory (100–109).
- Locked in the SDN `testnet` config and IPAM mappings.
- Expanded the roadmap with SIEM (Wazuh), NPM, managed switching, and internal DNS.

### January 2026 — foundation

- Set up the TP-Link AX6600 as the core router.
- Split the wireless into two SSIDs (`SSID_ExistingNetwork` and `SSID_HomelabNetwork`).
- Installed Proxmox VE on the Alienware host.
- Brought up the first VMs — Ubuntu Server, Ubuntu Desktop, Kali.
- Verified static and dynamic addressing, gateway routing, and CLI connectivity.
