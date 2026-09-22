# JHomelab

My personal homelab, running Proxmox VE on a repurposed Alienware 15 R3. I use it for Linux administration, virtualization, software-defined networking, firewall configuration, and services in Docker.

This repository documents the hardware, guest inventory, observed service state, and future ideas. **Last read-only scan: September 22, 2026.** The [dated scan report](docs/health-check-2026-09-22.md) records the checks, warnings, and limits behind this inventory. Runtime states are a snapshot, not a continuous health guarantee.

## What's in this repo

This is a documentation-only repository. Reproducible configuration exports and recovery instructions are still future work. Public tables use descriptive roles rather than internal hostnames, addresses, or guest identifiers; operational connection details are maintained privately.

Related coursework:

- [SVAD-111-Linux-Virtualization](https://github.com/jkosber/SVAD-111-Linux-Virtualization) — Linux administration and virtualization
- [Networking-109](https://github.com/jkosber/Networking-109) — networking fundamentals, addressing, and routing
- [CyberOps-115](https://github.com/jkosber/CyberOps-115) — security monitoring and packet analysis

## Current snapshot

| Area | Observed on September 22, 2026 |
| :--- | :--- |
| Hypervisor | Proxmox VE manager **9.2.20**, running kernel **7.0.14-17-pve** |
| Guests | Ten configured VMs: one running, nine stopped; one running LXC container |
| Startup | The service VM and GoodMem LXC container both have automatic startup enabled |
| Host services | Core PVE services active; no failed systemd units listed |
| Web endpoints | Proxmox, Homepage, Uptime Kuma, and Portainer returned HTTP 200 |
| GoodMem | Server container healthy; PostgreSQL/pgvector container running; API responded |
| Storage | All three configured pools active; boot SSD warning persists |
| Recovery | No scheduled PVE backup jobs or backups in configured local backup storage were listed; external copies and restores unverified |

## Physical host

| Component | Verified specification | Role |
| :--- | :--- | :--- |
| Platform | Alienware 15 R3 | Hypervisor |
| CPU | Intel Core i7-7700HQ, 4 cores / 8 threads | Compute |
| RAM | 16 GB DDR4, two 8 GB modules at 2400 MT/s | Shared guest and host memory |
| GPU | NVIDIA GeForce GTX 1070 Mobile, bound to `vfio-pci` | Configured for selected desktop guests; guest use untested |
| Boot SSD | SK hynix SC311 SATA 128 GB; about 119.2 GiB visible | Host OS and local storage |
| Guest HDD | HGST 1 TB; about 931.5 GiB visible | Guest thin-provisioned storage |

The boot SSD's overall SMART result is **PASSED**, but it continues to report 24 offline-uncorrectable sectors, three reported-uncorrectable errors, and three end-to-end errors. Those counters match the September 10 check; recurring SMART alerts remain. See the [storage findings](docs/health-check-2026-09-22.md#storage-and-recovery) before planning disk work.

The host root filesystem was 69% used. PVE reported pool usage of 65.33% for local directory storage, 0.01% for the boot-disk thin pool, and 23.96% for the guest thin pool. Filesystem and PVE storage percentages use different accounting.

## Guest inventory

Names below describe intended guest roles. Stopped guests were not booted, so their installed distribution releases were not reverified. CPU and memory values come from hypervisor configuration; memory is in MiB and primary virtual-disk capacity is in GiB.

| Guest role | Type | vCPU | RAM (MiB) | Primary disk (GiB) | State | Auto-start | GPU configured |
| :--- | :--- | ---: | ---: | ---: | :--- | :--- | :--- |
| Ubuntu desktop | VM | 8 | 8192 | 200 | Stopped | No | Yes |
| Kali lab | VM | 4 | 2048 | 30 | Stopped | No | No |
| OPNsense experiment | VM | 2 | 3072 | 16 | Stopped | No | No |
| openSUSE desktop | VM | 2 | 4096 | 30 | Stopped | No | No |
| Fedora desktop | VM | 4 | 4096 | 30 | Stopped | No | No |
| Zorin OS desktop | VM | 8 | 8192 | 40 | Stopped | No | Yes |
| Manjaro desktop | VM | 4 | 4096 | 30 | Stopped | No | No |
| Linux Mint desktop | VM | 4 | 4096 | 30 | Stopped | No | Yes |
| Pop!_OS desktop | VM | 4 | 8196 | 30 | Stopped | No | Yes |
| Ubuntu service host | VM | 2 | 2048 | 32 | Running | Yes | No |
| GoodMem service host | Unprivileged LXC | 2 | 4096 | 24 | Running | Yes | No |

The VMs total about 47 GiB of configured RAM, plus 4 GiB for LXC, against 16 GB of physical memory. This is configured overcommit, not evidence that all guests can run simultaneously. Only the service VM and LXC container were running during the scan. The Pop!_OS allocation of 8196 MiB is the observed value.

Four stopped desktops reference the same physical GPU. Their configurations do not demonstrate simultaneous GPU use or successful passthrough inside a guest. The service VM has no GPU assignment, correcting the older inventory.

The GoodMem container runs Debian 13.1 and has 512 MiB of configured swap. The service VM's previously documented Ubuntu release was not reverified: its QEMU guest agent is unavailable, and the attempted noninteractive SSH login was denied.

## Services

| Service | Role | Evidence from this scan |
| :--- | :--- | :--- |
| Homepage | Dashboard | HTTP 200 from the workstation |
| Uptime Kuma | Availability monitoring | HTTP 200 after following the redirect |
| Portainer | Container management | HTTPS 200 from the workstation |
| GoodMem | Memory service | Docker health check healthy; API responded and advertised `server-v1.0.285` |
| PostgreSQL with pgvector | GoodMem data store | Docker container running, using image tag `pgvector/pgvector:pg17` |

Homepage, Uptime Kuma, and Portainer were checked at their existing endpoints. Their guest-level Docker inventory, image versions, and application workflows were not inspected because service-VM access was unavailable. HTTP responses do not establish application correctness or recovery readiness.

GoodMem's running server uses an image tagged `latest`; that tag is mutable. The advertised API version above is the observed runtime identifier. Database restoration and end-to-end application recovery were not tested.

Nginx Proxy Manager and RustDesk remain previously recorded deployment ideas; this scan did not establish whether they are installed.

## Planned SDN implementation

The SDN is configured for selected experimental VMs that do not provide the services I currently use. Its implementation remains planned; it is not a dependency of the running service workloads. This scope was clarified on September 22, 2026.

The preparatory Proxmox configuration includes a Simple SDN zone and a distinct VNet object, with PVE IPAM, dnsmasq DHCP, and source NAT. Zone and VNet names are different objects; older documentation conflated them. A DNS/DHCP listener and the NAT rule were present during this scan.

**Isolation, address assignment, and internet access from an experimental guest were not tested.** All experimental VMs were stopped, and none were started for the scan. The previously described per-guest address convention remains an intention rather than verified lease state.

Both firewall services were active, and an nftables firewall table was present. Seven datacenter allow rules were configured. The legacy firewall status command reported pending changes, and one rule's descriptive comment did not match its destination scope. These observations warrant review; they do not prove that the intended policy is enforced.

Global IPv4 forwarding was `0`, while forwarding on both relevant bridge interfaces was `1`. Record both values: the global setting alone is insufficient evidence of a routing outage. An intentional guest traffic test is still needed.

Previously recorded physical-network equipment includes a TP-Link AX6600 router and Netgear WN2000RPTv2 wireless bridge. Router configuration, cabling, wireless behavior, and switch capabilities were outside this host scan and were not reverified.

## Maintenance and follow-ups

- Review the persistent boot SSD alerts and verify recoverable backups before disk maintenance.
- Establish backup coverage and test restoration. One experimental guest's primary disk explicitly has `backup=0`.
- Restore service-VM observability by checking guest-agent installation/service state and approved SSH access.
- As part of the planned SDN implementation, review firewall rule intent and runtime state, then test allowed and denied traffic, DHCP, and egress from an intentionally started lab guest.
- Verify GPU use inside a selected guest before claiming working hardware acceleration.

The running kernel command line includes `intel_iommu=on`, `iommu=pt`, and `pcie_acs_override=downstream`. The earlier claim that `multifunction` was also enabled is not supported by this scan. The NVIDIA device is bound to `vfio-pci`; no passthrough or boot settings were changed.

## Roadmap

These are existing ideas, not newly selected work or verified deployments:

- Replace the wireless bridge backhaul with Cat6.
- Complete the planned SDN implementation for selected non-service lab VMs.
- Add managed switching and VLAN practice.
- Evaluate Nginx Proxy Manager, internal DNS, and RustDesk.
- Explore a dedicated NAS guest for SMB/NFS.
- Revisit the placement of the Tailscale subnet router after verifying its current deployment.
- Explore Wazuh or Security Onion and more detailed firewall segmentation.
- Explore Jellyfin with GPU transcoding after validating passthrough.
- Add sanitized SDN/firewall exports, Compose files, maintenance scripts, and tested recovery instructions.

## Changelog

### September 22, 2026 — read-only inventory and documentation refresh

- Updated PVE/kernel versions, hardware, guest resources, startup flags, and GPU assignments from the live host.
- Added the running GoodMem LXC service and its observed container state.
- Recorded persistent SSD warnings, backup inventory gaps, and service-VM access limits.
- Replaced untested isolation and deployment claims with dated evidence and explicit limits.
- Clarified that SDN implementation is planned for selected non-service VMs; running services do not depend on it.
- Replaced historical screenshots with current tables and removed internal connection details from this revision. Older Git history still contains the original material; history was not rewritten.
- Made no infrastructure changes. See the [full scan report](docs/health-check-2026-09-22.md).

### April 2026 — service tier (historical record)

- Recorded the service VM as the only auto-start VM; the current inventory also includes an auto-start LXC container.
- Listed Nginx Proxy Manager and RustDesk as deployment ideas.

### February 2026 — infrastructure pass (historical record)

- Documented PVE 9.1-era software, the desktop guest inventory, GPU configuration, and SDN addressing intentions.
- Expanded the roadmap with monitoring, reverse proxying, switching, and DNS.

### January 2026 — foundation (historical record)

- Recorded the router setup, Proxmox installation, initial Ubuntu/Kali guests, and initial connectivity checks.

Historical entries retain the earlier project's record; they are not fresh validation of those outcomes.
