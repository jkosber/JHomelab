# Homelab health check: September 22, 2026

A read-only look at the Proxmox host, the guest inventory and the running services. I didn't restart or reconfigure anything, and the stopped lab VMs stayed stopped. This is the baseline the [October maintenance](../projects/2026-10-patching-and-monitoring/README.md) builds on.

The [README](../README.md) has the network layout, the current inventory and the Proxmox screenshots. Local IPs keep the first three octets and swap the last one for `X`.

## Proxmox and guests

- Proxmox VE 9.2.20 on kernel 7.0.14-17-pve.
- Alienware 15 R3 with an Intel i7-7700HQ and 16 GB DDR4.
- VM 109 and CT 110 running, both set to start with the host. VMs 100–108 stopped.
- Core Proxmox services active, and no failed systemd units.
- The GTX 1070 Mobile is bound to `vfio-pci` and assigned in the configs for VMs 100, 105, 107 and 108. VM 109 has no GPU. I didn't test passthrough inside a guest.

The guest tables in the README use the Proxmox resource settings. RAM is rounded to GB there; VM 108 is actually 8196 MiB. I didn't check which releases are installed in the stopped VMs.

## Running services

| Host | Service | Check |
| :--- | :--- | :--- |
| Proxmox | Web UI | Answered over HTTPS. I went through the inventory, firewall and IPAM pages |
| VM 109 | Homepage | HTTP 200 |
| VM 109 | Uptime Kuma | HTTP 200 after a redirect |
| VM 109 | Portainer | HTTPS 200 |
| CT 110 | GoodMem | Docker health check healthy, and the API reported `server-v1.0.285` |
| CT 110 | PostgreSQL / pgvector | Container running, image `pgvector/pgvector:pg17` |

CT 110 is an unprivileged Debian 13.1 container with 2 cores, 4 GB RAM, 512 MiB swap and a 24 GB disk. GoodMem serves REST on 8080 and gRPC on 9090, and PostgreSQL uses 5432. I didn't test every port on its own.

I couldn't get a direct look inside VM 109. Its guest agent wasn't running and my SSH login was denied, so I couldn't confirm its Docker inventory or Ubuntu version from inside. The web checks above show the services answering, but I didn't test what each app does. The HTTPS checks accepted the self-signed local certificates. (Both gaps got fixed in October: SSH works, and the guest agent is installed.)

## Storage and backups

All three storage pools were active.

| Pool | Use |
| :--- | :--- |
| `local` | 65.33% |
| `local-lvm` | 0.01% |
| `vmdata` | 23.96% |

The boot SSD's overall SMART result was PASSED, but the warnings from September 10 were still there: 24 offline-uncorrectable sectors, 3 reported-uncorrectable errors and 3 end-to-end errors. None of those numbers went up. The HDD passed cleanly, with zero reallocated sectors, uncorrectable sectors or CRC errors.

There were no scheduled Proxmox backup jobs and no local backup files. I didn't check for manual or off-host copies, and I didn't run a restore test. VM 106's main disk is set to `backup=0`.

The backups and the SSD warning need their own follow-up before I do any disk work.

## SDN and firewall

VMs 100–108 are on the SDN: VNet `testnet` in zone `test`, using `10.10.100.X/24`. IPAM has mappings for the gateway and those VMs, and dnsmasq and a source-NAT rule are in place.

VM 109 and CT 110 are on `vmbr0` (`192.168.0.X/24`). Moving them onto the SDN is planned.

The datacenter firewall is on. **Input defaults to DROP** with seven allow rules, and **output and forwarding default to ACCEPT**. The [firewall screenshots](../README.md#firewall-datacenter-level) show the rules and settings.

Three things to sort out before extending the SDN:

- One rule is labeled as internet access, but its destination is the home LAN.
- The legacy firewall status shows pending changes, while the nftables firewall is the one running.
- Global IPv4 forwarding is `0`, while forwarding on `vmbr0` and `testnet` is `1`.

The lab VMs were off, so I couldn't test DHCP, internet access or isolation from inside them.

## Screenshots

I took the four screenshots in the README from Proxmox on September 22: guest inventory, firewall rules, firewall options and SDN IPAM. IPs and MAC addresses are masked where they show up, and account details are cropped out. The display edits I made for the captures didn't change any Proxmox settings.
