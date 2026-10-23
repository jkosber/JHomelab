# Homelab maintenance: October 2026

Most of October went to patching and to finally being able to see what the lab is doing. A certificate authority compromise on the 21st kicked it off. By the 23rd I had every host caught up, swapped out an abandoned container updater, and set up monitoring with Discord alerts and a dashboard that shows live data instead of a page of links.

Local IPs keep the first three octets and swap the last one for `X`. The [README](../README.md) has the current inventory, and the [September 22 health check](health-check-2026-09-22.md) is the baseline I compared against.

## October 21: the CA compromise

The eMudhra (emSign) certificate authority had a key compromise, and Node.js, Python and uv all put out fixes. Here's what I did in the lab:

- Made sure `emSign ECC Root CA - G3` was out of the system trust store on the Proxmox host and every guest.
- CT 110: took a snapshot, then updated ca-certificates and OpenSSL, Docker (29.6.0 to 29.9.2), the GoodMem server (v1.0.285 to v1.0.331) and PostgreSQL/pgvector (17.10 to 17.11).
- VM 109: took a snapshot, updated Portainer from 2.45.1 to 2.45.2, then pulled and redeployed every stack.
- Wrote a small read-only script that checks each host and prints OK, WAIT, ACT or FAIL for every fix. Some vendors take a while to ship patched images, and I didn't want to keep checking by hand.

## October 23: updates

| Host | What I did |
| :--- | :--- |
| Proxmox host | Removed leftover Proxmox 8 (bookworm) APT entries, plus a duplicate security list that made apt print warnings on every run. Installed the pending updates and dropped an old 6.17.2 kernel, keeping the running kernel and two fallbacks |
| VM 109 | Installed the updates Ubuntu would give me. Its phased rollout is still holding back systemd 259.9 and containerd, which will come later. Updated Portainer to 2.45.3, Homepage to 2.7.0 and Uptime Kuma to 2.5.7, the last two rebuilt on patched Node. Installed `qemu-guest-agent`, grew the root volume from 14.5 GB to the whole disk while it was running, and cleared about 2.3 GB of old rollback images |
| CT 110 | Took a snapshot, installed all 88 pending updates (33 security) and rebooted so every service loaded the new libc and systemd. GoodMem answered again 26 seconds later |
| Mint laptop | Installed updates and turned on Mint's automatic daily updates |

Watchtower was its own problem. The original `containrrr/watchtower` project was archived in December 2025, and it only worked on Docker 29 because I'd forced an older API version. I replaced it with the maintained fork (`nickfedor/watchtower`) using the same settings. The fork negotiates the Docker API on its own, so the override is gone.

I took a snapshot or backup before each change and deleted them once everything checked out.

## Monitoring and alerting

Before this I had Uptime Kuma installed and nothing configured in it, and nothing at all watching the hardware. Now:

| Tool | Where it runs | What it watches |
| :--- | :--- | :--- |
| Beszel 0.21.1 | Hub on VM 109, agents on the Proxmox host, VM 109 and CT 110 | CPU, memory, disk, temperature, container stats, drive SMART health, pending package updates (with security ones flagged) |
| Uptime Kuma 2.5.7 | VM 109 | The Proxmox web UI, GoodMem, Portainer, Beszel, both RustDesk ports and my portfolio site, including when its certificate expires |

Both send alerts to a private Discord channel. Beszel alerts if:
- a machine is down for two minutes
- CPU or memory stays above 90%
- a disk passes 85%
- a service fails or a container goes unhealthy
- the laptop runs hot
- any drive's SMART state gets worse

Kuma alerts after two failed checks in a row and again when the service comes back. It also has a status page anyone on my LAN can open without logging in.

The first time Beszel ran, it showed 87 pending updates on CT 110. That one finding paid for the setup.

## Dashboard

Homepage was already on VM 109, but I'd never configured it, so it was just a page of links I didn't open. Before setting it up again I compared it with Homarr, Dashy, Glance, Homer, Heimdall and some Proxmox-specific tools (ProxMenux, Pulse, PegaProx, ProxCenter). Homepage won for four reasons:

1. It can pull live data from all four things I run (Proxmox, Portainer, Beszel and Uptime Kuma).
2. A lot of people use it.
3. It releases often.
4. Its current image is on a patched Node version.

Homarr came close, but it can't show Portainer.

![Homepage dashboard](../screenshots/homepage-dashboard-2026-10-23.png)

*Live Proxmox, Portainer, Beszel and Uptime Kuma data on one page.*

Each panel has its own credential instead of my admin logins:

- **Proxmox:** an API token on a separate user that only has the `PVEAuditor` role, with privilege separation on. It can read everything, and when I tried a write it got a 403.
- **Portainer:** an access token made just for the dashboard.
- **Beszel:** a separate superuser with a random password. Homepage's Beszel panel won't work with anything less.

None of those are in the dashboard config. Homepage reads them from root-only files mounted read-only into the container, and the config itself lives in its own repo with a deploy script.

## Storage finding

While working out where backups should go, I found a third drive: an NVMe SSD left over from when the laptop ran Windows. Proxmox had never seen it. The SATA controller is in Intel RST RAID mode, and the kernel log says so: "Found 1 remapped NVMe devices. Switch your BIOS from RAID to AHCI mode to use them."

The plan:
1. Switch the BIOS to AHCI.
2. Wipe the NVMe and move the guest disks onto it.
3. Turn the 1 TB HDD into the drive scheduled backups go to.

I still have no scheduled backups, so this is next. The boot SSD's SMART warnings haven't changed since September (24 offline-uncorrectable sectors), and Beszel will alert me if they get worse.

## What I didn't check

- Portainer actions and the Kuma UI beyond the monitors themselves.
- The phased Ubuntu updates on VM 109, which show up on Ubuntu's schedule.
- GoodMem's image still trusts the compromised root inside its own Java trust store. That fix has to come from the vendor.
