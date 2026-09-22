# Homelab health check — September 22, 2026

A read-only check of the Proxmox host, guest inventory and running services. Nothing was restarted or reconfigured, and the stopped lab VMs stayed stopped.

The [README](../README.md) has the network layout, current inventory and fresh Proxmox screenshots. Local IPs have their last number replaced with `X`.

## Proxmox and guests

- Proxmox VE 9.2.20, running kernel 7.0.14-17-pve.
- Alienware 15 R3, Intel i7-7700HQ and 16 GB DDR4.
- VM 109 and CT 110 running, both set to auto-boot. VMs 100–108 stopped.
- Core Proxmox services active, with no failed systemd units listed.
- GTX 1070 Mobile bound to `vfio-pci` and assigned in the configs for VMs 100, 105, 107 and 108. VM 109 has no GPU assignment. Passthrough wasn't tested inside a guest.

The [guest tables](../README.md#virtual-machines) use the current Proxmox resource settings. RAM is rounded to GB there; VM 108's exact allocation is 8196 MiB. Installed releases in the stopped VMs weren't checked.

## Running services

| Host | Service | Check |
| :--- | :--- | :--- |
| Proxmox | Web UI | Responded over HTTPS; inventory, firewall and IPAM pages inspected |
| VM 109 | Homepage | HTTP 200 |
| VM 109 | Uptime Kuma | HTTP 200 after redirect |
| VM 109 | Portainer | HTTPS 200 |
| CT 110 | GoodMem | Docker health check healthy; API responded with version `server-v1.0.285` |
| CT 110 | PostgreSQL / pgvector | Container running, image `pgvector/pgvector:pg17` |

CT 110 is an unprivileged Debian 13.1 container with 2 cores, 4 GB RAM, 512 MiB swap and a 24 GB disk. GoodMem publishes REST on 8080 and gRPC on 9090; PostgreSQL uses 5432. Reachability wasn't checked separately for every port.

VM 109's guest agent wasn't running, and the SSH login attempt was denied. Its Docker inventory and installed Ubuntu version couldn't be checked directly. The web checks above show the services responding; individual app workflows weren't tested. HTTPS probes allowed the local self-signed certificates.

## Storage and backups

All three storage pools were active.

| Pool | Use |
| :--- | :--- |
| `local` | 65.33% |
| `local-lvm` | 0.01% |
| `vmdata` | 23.96% |

The boot SSD's overall SMART result was PASSED, but the warnings from September 10 remain: 24 offline-uncorrectable sectors, 3 reported-uncorrectable errors and 3 end-to-end errors. The counts haven't increased. The HDD passed, with zero reported reallocated sectors, uncorrectable sectors or CRC errors.

No scheduled Proxmox backup jobs or local backup files were listed. Manual or off-host copies weren't checked, and no restore test was run. VM 106's primary disk has `backup=0`.

Backup coverage and the SSD warning need a separate follow-up before disk work.

## SDN and firewall

The SDN is already configured for non-service VMs 100–108. They use VNet `testnet` in zone `test`, with the `10.10.100.X/24` address scheme. IPAM has mappings for the gateway and those VMs. dnsmasq and a source-NAT rule were present.

VM 109 and CT 110 use `vmbr0` on `192.168.0.X/24`. Moving the service guests onto the SDN is planned.

The datacenter firewall is enabled. **Input defaults to DROP**, with seven enabled allow rules. **Output and forwarding default to ACCEPT**. The [firewall screenshots](../README.md#firewall-datacenter-level) show the rules and settings.

A few details to check before extending the SDN:

- One rule is labelled as internet access but has the home LAN as its destination.
- The legacy firewall status shows pending changes, while the nftables firewall is active.
- Global IPv4 forwarding is `0`, while forwarding on `vmbr0` and `testnet` is `1`.

The lab VMs were stopped, so DHCP, internet access and isolation weren't tested from inside them.

## Screenshots

The four screenshots in the README were taken from Proxmox on September 22: guest inventory, firewall rules, firewall options and SDN IPAM. IPs and MAC addresses are masked where shown, and account details are cropped out. The display edits used for the captures didn't change the Proxmox configuration.
