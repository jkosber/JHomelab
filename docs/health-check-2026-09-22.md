# Read-only homelab scan — September 22, 2026

## Scope and method

The scan ran from the Windows workstation against the known Proxmox host over SSH with strict host-key checking. It inspected host and guest configuration, runtime status, storage health, backup inventory, SDN/firewall settings, service endpoints, and the running GoodMem LXC workload. Public descriptions omit operational addresses, hostnames, guest identifiers, and raw configuration output.

Checks were read-only. No guests were started or stopped, no services restarted, no SMART tests launched, and no infrastructure configuration changed. This was an inventory and health check of the known Proxmox environment, not a network-wide vulnerability scan. Router administration, other physical hosts, external backup destinations, and stopped-guest operating systems were outside the checked scope.

## Host and runtime

| Check | Observation | Limit |
| :--- | :--- | :--- |
| Version | PVE manager 9.2.20; running kernel 7.0.14-17-pve | Installed/runtime state at scan time |
| Hardware | Alienware 15 R3, i7-7700HQ, 16 GB DDR4 | Firmware inventory and CPU output |
| Resources | About 11 GiB memory available; swap unused; root filesystem 69% used | Point-in-time usage |
| Core services | PVE cluster, daemon, proxy, status, scheduler, legacy firewall, and SMART services active | Does not test every service operation |
| Failed units | None listed on the host or GoodMem LXC | Limited to systemd's reported state |
| Guest inventory | Ten VMs and one LXC; service VM and LXC running | Stopped guests were not exercised |
| GPU | GTX 1070 Mobile bound to `vfio-pci`; four stopped desktops have assignments | Guest passthrough and acceleration untested |

The [README inventory](../README.md#guest-inventory) records the observed resource allocations. The Kali VM has two sockets with two cores each, so its configured total is four vCPUs. Exact distribution releases in stopped VMs were not inferred from their display names.

## Storage and recovery

All three configured storage pools were active. PVE pool usage was 65.33% for local directory storage, 0.01% for the boot-disk LVM-thin pool, and 23.96% for the guest LVM-thin pool. The guest HDD is separate from the host boot SSD; this layout does not by itself provide a backup.

| Device | SMART evidence | Interpretation |
| :--- | :--- | :--- |
| Boot SSD | Overall PASSED; `Offline_Uncorrectable=24`, `Reported_Uncorrect=3`, `End-to-End_Error=3`, `Reallocated_Event_Count=0`, `UDMA_CRC_Error_Count=0` | Persistent warning despite the overall result |
| Guest HDD | Overall PASSED; reported reallocated-sector, offline-uncorrectable, and CRC counts all zero; no SMART errors logged | No warning in these inspected fields; not a recovery test |

The SSD's three nonzero error counters match the September 10 observation. Repeated `smartd` messages still report 24 offline-uncorrectable sectors. Its recorded self-test history includes aborted tests; no fresh self-test was run. A targeted search of the current boot's kernel journal found no matching block I/O, ATA failure, or ext4 error messages. That does not clear the SMART warning or establish future drive reliability.

The Proxmox scheduled-backup inventory was empty, and configured local backup storage listed no backup volumes. No external backup storage was configured in the inspected PVE storage inventory. Manual/off-host copies may exist and were not searched. No restore test was performed. One experimental VM's primary disk is explicitly excluded from PVE backup with `backup=0`.

Follow-up: verify recoverable guest and host-configuration backups, review the SSD warning, and assess disk maintenance separately. This scan made no backup or disk changes.

## Application observations

| Target | Observed evidence | Unchecked |
| :--- | :--- | :--- |
| Proxmox web UI | HTTP 200 over HTTPS | Authentication and administrative workflows |
| Homepage | HTTP 200 | Dashboard integrations |
| Uptime Kuma | HTTP 200 after redirect | Monitor execution and notification delivery |
| Portainer | HTTP 200 over HTTPS | Container-management operations |
| GoodMem server | Docker health status healthy; API responded, advertising `server-v1.0.285` | Recovery and full application workflows |
| PostgreSQL/pgvector | Docker container running with image tag `pgvector/pgvector:pg17` | Database integrity and restoration |

HTTP checks were issued from the workstation with timeouts. HTTPS checks permitted the existing self-signed certificates, so they establish an HTTP response, not certificate trust. No public exposure claim is made from these local checks.

The GoodMem LXC runs Debian 13.1 with two vCPUs, 4096 MiB RAM, 512 MiB swap, and a 24 GiB root disk. Its root filesystem was 18% used, memory availability about 3.4 GiB, and swap unused. The server and database containers had been up for four days. The server image is tagged `latest`; its advertised version is recorded separately because image tags can change.

The service VM has guest-agent support enabled in PVE, but the ping command reported that the QEMU guest agent is not running. A noninteractive SSH attempt from the trusted hypervisor used an existing known host key and was denied authentication. Its container inventory and installed OS version therefore remain unverified. No credentials or host-key checks were bypassed.

## SDN and firewall evidence

Jadon clarified that the SDN is already configured on selected non-service VMs; extending it to service VMs is planned. The hypervisor configuration shows the experimental VMs attached to the SDN VNet, while the running service VM and GoodMem LXC use the existing service network. Configuration was verified; traffic through the stopped experimental guests was not tested.

- A Simple zone, a separate VNet, PVE IPAM, a DHCP range, a gateway, and source NAT are configured. Zone and VNet must not be treated as the same object.
- dnsmasq was listening for DNS and DHCP on the lab interface, and a source-NAT rule was present.
- Global IPv4 forwarding and `conf.all.forwarding` were `0`; forwarding on both relevant bridge interfaces was `1`. This mixed configuration was recorded without declaring routing broken or working.
- The legacy and nftables firewall services were active, and an nftables firewall table existed. Legacy `pve-firewall status` reported `enabled/running (pending changes)`; the scan did not establish why.
- Seven enabled datacenter allow rules were listed. One rule commented as internet access actually names the home LAN as its destination. Comments alone do not describe an effective security boundary.
- The running guests' firewall option queries returned no explicit option values. This does not prove guest firewall enforcement.
- The host management bridge had zero reported RX/TX errors or drops at scan time.

All experimental guests were stopped. There was no DHCP lease acquisition test, guest-to-internet test, or denied-traffic test across the intended boundary. The scan does not establish isolation or successful lab routing. Review policy intent and runtime state before the planned extension to service VMs; these findings do not establish an outage of the services currently in use.

## Repeating the checks

Use your own trusted SSH alias and verified guest IDs. The placeholders below are not this environment's connection details. Review output privately before publishing it: inventories, firewall rules, and guest configuration can expose internal information.

From a workstation with OpenSSH:

```text
ssh -o BatchMode=yes -o StrictHostKeyChecking=yes YOUR_HYPERVISOR_ALIAS "pveversion; qm list; pct list; pvesm status"
```

On the verified Proxmox host, useful read-only checks include:

```text
systemctl --failed --no-legend --no-pager
free -h
df -hT /
pvesh get /cluster/backup --output-format json
pvesh get /storage --output-format json
pvesh get /cluster/sdn/zones --output-format json
pvesh get /cluster/sdn/vnets --output-format json
pve-firewall status
sysctl net.ipv4.ip_forward
```

Then inspect the selected guest's configuration and, when authorized access is available, its runtime. Include both `qm list` and `pct list`: scanning only VMs omits LXC workloads. Read SMART health/attributes from verified device paths without initiating self-tests. Check service endpoints from the workstation and distinguish HTTP response, process state, Docker health status, and tested application behavior.

## Documentation changes

This refresh replaces stale versions, incomplete guest inventory, outdated GPU targets, and unsupported isolation claims. Current public tables replace screenshots of an older PVE release that also revealed operational details. Original screenshots and addressing remain in earlier Git history; this update does not rewrite that history.

The existing roadmap remains a set of ideas. No infrastructure remediation, new service deployment, or recovery exercise is represented as completed by this documentation update.
