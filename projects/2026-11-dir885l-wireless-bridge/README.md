# DIR-885L wireless bridge: November 2026

On November 11, 2026 I replaced the Wi-Fi link between my lab and the home router. A Netgear WN2000RPTv2 range extender had been doing that job over 2.4 GHz. Now a spare D-Link DIR-885L running OpenWrt 25.12.6 does it over 5 GHz, as a `relayd` client bridge. Lab traffic to the rest of the house went from about 47 Mbit/s to between 239 and 272 Mbit/s in the same 40 MB test, and up to 369 Mbit/s on 400 MB transfers. Every host kept its address.

The cutover was at 01:20 EST. I planned a 24-hour soak test and ended it at the 19-hour mark, 20:20, because every check had held since the cutover. It passed its written condition, no "deauth" lines in the bridge's log, and all nine Uptime Kuma monitors stayed at 100% with no bridge reboot. That condition missed three Wi-Fi drops of 5 to 7 seconds each, right after the router asked the bridge to move to its other 5 GHz radio. The soak didn't cover the evening, when the house uses Wi-Fi most. I read the bridge's log for those hours afterwards, and it shows no drops from 20:20 to 01:27 on November 12. Right after that I turned the radio firmware's roaming off as a test against the drops. Whether it stops them isn't known yet.

Local IPs keep the first three octets and swap the last one for `X`. The [topology page](../../docs/topology.md) shows how everything connects now.

## What changed and why

The lab has no cable to the home router, a TP-Link Archer AX90. Until this project the 24-port lab switch plugged into the Netgear extender, which joined the router's Wi-Fi on 2.4 GHz 802.11n and bridged the switch onto the home LAN. Before touching anything I measured that link: 46 to 48 Mbit/s from the lab to the home LAN and 77 to 79 Mbit/s the other way.

A Cat6 run to the lab is still the real fix and stays on the roadmap. The DIR-885L was sitting unused, and it has a 5 GHz 802.11ac radio and gigabit ports, so I set it up as the step before the cable. I had three requirements:

- No lab host changes address.
- The AX90 stays the only DHCP server and gateway.
- The extender stays available as the rollback until the bridge has proven itself.

## Hardware and its limits

The unit is hardware revision A1, the one OpenWrt's image targets. When I first connected to it, it was already running OpenWrt 24.10.4 with factory defaults, so there was no flash from stock firmware to do.

| Limit | What it means here |
| :--- | :--- |
| The Wi-Fi driver, `brcmfmac`, has no WDS (4-address) or 802.11s mode | No true layer-2 wireless bridge is possible. The kernel source returns "not supported" for both interface types, and `iw list` on this unit lists neither |
| Running an access point and a client on the same radio is reported to make the uplink flap | Not needed here, so the bridge runs no access point at all |
| The 5 GHz radio can't see channels 149 and up | The AX90 has two 5 GHz radios, on channels 44 and 157. The bridge only ever saw the channel-44 one, with every firmware file I tried. The driver logs `no clm_blob available, device may have limited channels` |
| The 5 GHz radio firmware OpenWrt ships is broken in client mode on this unit | See [the radio firmware problem](#the-radio-firmware-problem-and-the-fix) |
| Left alone, the LAN bridge interface gets a random MAC on every boot | I saw four different values, and lab hosts saw their neighbour at a new MAC after each bridge reboot. On November 12 I fixed it in the config, to the value it had at the time |

## Why relayd

OpenWrt's own documentation says to use WDS when both ends run OpenWrt, and `relayd` when the main router doesn't run OpenWrt or doesn't support WDS or 802.11s. The AX90 runs stock TP-Link firmware, and the DIR-885L can't do WDS or 802.11s anyway, so `relayd` was the standard answer for this pair.

| Option | Verdict |
| :--- | :--- |
| WDS client | Not available on the DIR-885L, and the stock AX90 doesn't offer it either |
| 802.11s mesh | Not available on either end (TP-Link's mesh is its own OneMesh) |
| `relayd` pseudo-bridge | Chosen. The uplink is an ordinary Wi-Fi client, so it works with any access point. A DIR-885L owner on the OpenWrt forum reported the same setup working in 2024 |
| Routed client on its own subnet | Rejected. Every lab address, monitor and URL would change, and the home LAN would need a static route to reach the lab |

`relayd` has costs, and I accepted them:

- It relays IPv4 only. IPv6 link-local traffic stops crossing between the lab and the house. The home LAN has no global IPv6, only link-local addresses.
- Multicast such as mDNS isn't relayed.
- The home LAN sees every lab host at the bridge's MAC, and the lab sees every home host at the bridge's MAC. The extender worked the same way, so this wasn't new.

One forum thread I only saw as a search summary had a user reporting that DHCP didn't pass through `relayd`. I tested DHCP directly on the bench because of it.

## Upgrade and hardening

I did all of this on the bench with a single cable to a PC, inside an isolated network namespace. The reason: the router's defaults run a DHCP server, and its bootloader recovery mode answers on the same address as my home router. Neither could be allowed near the home LAN.

- **Image check.** I verified the OpenWrt 25.12.6 image's signature against the OpenWrt build key fingerprints published on openwrt.org, and its SHA-256 against the signed checksum list.
- **Upgrade.** A dry run with `sysupgrade -T` passed, then `sysupgrade -n` went from 24.10.4 to 25.12.6 without keeping settings. 25.12 replaces `opkg` with `apk`, so a clean start seemed safer. Recovery mode wasn't needed.
- **Access.** SSH works with keys only, and password login is refused. I proved key login in a second connection before turning passwords off.
- **Web UI.** LuCI listens only on a separate management address on the LAN ports, so it's closed on the bridge's home-LAN address. I reach it through an SSH tunnel.
- **Radios.** I deleted the default open access points before enabling any radio, and the 2.4 GHz radio stays off.
- **DHCP and router advertisements** are off on the LAN side, so the AX90 stays the only DHCP server.

## The radio firmware problem and the fix

With the radio firmware OpenWrt ships for this chip (`brcmfmac4366b-pcie.bin`, version 10.28.2), the 5 GHz link looked fine and passed nothing. The bridge associated and completed the WPA2 handshake, then 39 of 44 transmits failed. No DHCP lease came back, and no frames reached the home LAN. Turning transmit power down to 5 dBm changed nothing. On 2.4 GHz with the same file, everything worked: a lease, no failed transmits, 180 of 180 pings.

The OpenWrt device page and OpenWrt issue #16114 describe 5 GHz failures with this file, but only with the DIR-885L as an access point. Nothing I found said whether client mode was affected. On this unit it is.

I decided to replace the file and looked at three candidates:

| File | Where it came from | Result |
| :--- | :--- | :--- |
| 10.10.69.3309 (2016) | OpenWrt's signed 18.06.9 package. I verified the package index signature | Passed the 15-minute 5 GHz gate. DHCP and `iperf3` through the relay worked |
| 10.10.69.69042 (built August 2021), **in use** | An attachment on issue #16114. Its MD5 and SHA-256 match the values two other commenters posted, and it is said to match Asus's RT-AC88U firmware | Performed the same as the 2016 build. I kept it because it's newer |
| 10.10.122.45 | A third-party repository the device page links to | Downloaded and hash-checked, not installed. No DIR-885L owner has reported using it |

I have not checked the 2021 file against Asus's own download, and I decided not to. Its origin rests on matching hashes that other users posted.

All three files stay on the router under versioned names, and they're on the upgrade keep-list. An upgrade or an update of the firmware package can still put 10.28.2 back if the keep-list isn't honoured, so after either I check the loaded version in the kernel log.

## Configuration summary

| Item | Setting |
| :--- | :--- |
| Firmware | OpenWrt 25.12.6, kernel 6.12.112, 147 packages including `relayd` and `luci-proto-relay` |
| Radio firmware | 10.10.69.69042, with 10.28.2 and 10.10.69.3309 kept beside it |
| Uplink | 5 GHz client, locked to the router's channel-44 radio (80 MHz), WPA2-PSK, country US |
| Bridge | `relayd` between the LAN ports and the Wi-Fi client interface. The bridge's own address is a static `192.168.0.X/24`, outside the router's DHCP pool, with the AX90 as gateway and DNS |
| Firewall | The Wi-Fi interface and the relay sit in the LAN zone. The default WAN interfaces are removed |
| Management | LAN ports on `192.168.2.X/24`, no DHCP server, no router advertisements. LuCI only listens there |
| SSH | Keys only, on every address |
| Since November 12 | Firmware roaming off (`roamoff=1` for the `brcmfmac` driver), as a test against the steering drops. A fixed MAC on the LAN bridge interface |
| CA store | OpenWrt 25.12.6 still ships the emSign ECC Root CA - G3, the root I removed from my other hosts in October. I removed it from the bridge's bundle on November 11. A reflash or a `ca-bundle` update brings it back |

The Wi-Fi name and key, the passwords and the config backups stay off this repo.

## Bench tests

All bench numbers come from next to the AX90, at about -20 to -28 dBm, so they say nothing about range.

| Test | Firmware | Result |
| :--- | :--- | :--- |
| 5 GHz as shipped | 10.28.2 | Failed: 39 of 44 transmits failed, no lease, no traffic |
| 2.4 GHz, same position | 10.28.2 | Worked: lease, 180 of 180 pings |
| Gate 1: 15-minute 5 GHz soak, bridge to router | 10.10.69.3309 | 900 of 900 pings, avg 2.018 ms, 0 disconnects |
| `iperf3`, 30 s across the relay | 10.10.69.3309 | 222 Mbit/s home LAN to lab side, 392 Mbit/s lab side to home LAN |
| `iperf3`, 30 s across the relay | 10.10.69.69042 | 228 and 388 Mbit/s |
| Gate 2: 30-minute soak across the relay | 10.10.69.69042 | 1800 of 1800 pings, avg 2.521 ms, max 17.5 ms, 0 disconnects, 0 failed of 842,709 transmits |
| DHCP through the relay | 10.10.69.3309 | Full discover, offer, request and acknowledge from the AX90. The bench client got a lease |
| 2.4 GHz fallback | 10.10.69.69042 | The relay moved to the 2.4 GHz interface; 30 of 30 pings, 102 Mbit/s each way |
| Simulated 23-second uplink outage | 10.10.69.69042 | Reconnected within 3 s; the relay stayed up and traffic resumed on its own |
| Reboots | all three | The uplink and relay came back on their own, three times |

Two things went wrong on the bench that I now know to watch for. After switching the uplink to the other radio at runtime with `wifi up`, the Wi-Fi interface stayed down and `relayd` never started; `ifup wwan` or a reboot fixes it, and a plain link drop doesn't cause it. And `ifstatus repeater_bridge` reports the relay as down even while it passes traffic, so I check the `relayd` process and actual traffic instead.

## Cutover

Before the cutover I put the bridge at the extender's spot with its LAN ports empty. At that range the 5 GHz signal was -54 to -56 dBm with link rates of 867 to 1083 Mbit/s, and 1100 of 1100 pings came back.

The cutover itself was one cable: the lab switch's uplink came out of the extender and went into DIR-885L LAN port 1. The lab was offline from 01:20:05 to 01:20:32, 27 seconds. After that:

- jhome, VM 109 and CT 110 answered from the home LAN behind the bridge's MAC. One home-LAN machine needed its neighbour cache flushed once. An ARP check showed only the bridge answering for the lab addresses.
- The lab hosts reached the router, the internet and DNS.
- Proxmox, Homepage, Uptime Kuma, Beszel, Portainer and GoodMem all answered. Uptime Kuma recorded no downtime across the move.
- I added a DHCP reservation on the AX90 for jhome. Its address was inside the DHCP pool with no reservation, which was a risk that predated this project.

A few minutes later I moved jhome's own cable from the switch straight into LAN port 2 (8 seconds offline) to see whether the switch was costing anything. It wasn't. The Wi-Fi hop is the limit either way. jhome stayed on port 2 until I simplified the cabling during the soak (see [Soak result](#soak-result)).

## Results

The same commands I ran against the extender before the change, rerun after the cutover:

| Measurement | Netgear extender | DIR-885L |
| :--- | :--- | :--- |
| jhome to router, 200 pings at 0.2 s | 0% loss, avg 2.124 ms, max 31.25 ms | 0% loss, avg 2.241 ms, max 6.6 ms |
| Lab to home LAN, 40 MB over SSH | 46.2 and 47.7 Mbit/s | 239 to 272 Mbit/s |
| Home LAN to lab, 40 MB over SSH | 76.6 and 78.7 Mbit/s | 219 to 226 Mbit/s |
| Lab to home LAN, 400 MB | not measured | 305 to 369 Mbit/s |
| Home LAN to lab, 400 MB | not measured | 247 to 248 Mbit/s |

Average latency is about the same, and the worst-case ping dropped from 31 ms to under 7 ms. On the like-for-like 40 MB runs, throughput rose five to six times from the lab to the house and about three times the other way.

## Soak result

The soak ran from the cutover, 01:20:32, to 20:20:32 on November 11. The state at the finish:

| Check at 20:20 | Result |
| :--- | :--- |
| Uptime Kuma | 100% over its 24-hour window on all nine monitors, 19 hours of it on the bridge, no down beats |
| Bridge uptime | 19 h 11 min, no reboot since the cutover |
| Uplink | Still on the AX90's channel-44 radio, -56 dBm, 1083 Mbit/s, 0 failed transmits |
| Bridge to home router | 10 of 10 pings, avg 1.908 ms |
| `relayd` | Running |
| Radio firmware | 10.10.69.69042, still loaded |
| OpenWrt | 25.12.6 |
| "deauth" lines in the bridge's log | 0 |
| Wi-Fi drops | 3, all from band steering, none after 16:56. See [Band steering during the soak](#band-steering-during-the-soak) |

The soak didn't cover the evening, so I read the bridge's log for it at 01:27 on November 12. This count comes from the log after the soak ended. It isn't part of the 19-hour soak.

| Check at 01:27, for 20:20:32 to 01:27 | Result |
| :--- | :--- |
| Wi-Fi disconnects in the bridge's log | 0 |
| Steering requests from the AX90 | 0 |
| Log coverage | The log reached back to 16:20, so the window has no gap |
| Driver's `Unknown Frame` lines in the kernel log since boot | 3, the same three as during the soak |
| Uplink | Channel-44 radio, -56 dBm, 0 failed transmits, connected since the reconnect at 16:56:15 |
| Bridge uptime | 1 day 18 min, no reboot |
| Uptime Kuma | 100% over its 24-hour window on all nine monitors |

That makes three drops in the bridge's first 24 hours and 18 minutes, all between 13:27 and 16:56.

I changed the cabling during the soak. At 19:22:49 I unplugged the 24-port switch from the bridge. The lab has too few wired devices to need it, and nothing was behind it. It's unplugged, not gone. At 19:30:52 I moved jhome's cable from LAN port 2 to LAN port 1, 3 seconds offline. All four LAN ports sit in one bridge interface, so no configuration changed. Afterwards port 1 linked at 1000 Mb/s full duplex, the three lab hosts and six services answered, and Uptime Kuma recorded no downtime. LAN ports 2 to 4 are empty now.

Right after the soak, at 20:20, I took a fresh config backup with a package list. The list has 147 packages, the same count as the one from before the cutover. I haven't tried restoring from it. I also removed the bench network namespace and NetworkManager drop-in from the PC I used for flashing.

## Monitoring

I added two Uptime Kuma monitors, which brings it to nine. Both alert to Discord and sit on the status page.

- A ping to the home router, sent from inside the lab, so it has to cross the bridge. Pinging the bridge itself would prove nothing, because the bridge answers its own address locally even when its Wi-Fi link is down.
- A TCP check on the bridge's SSH port.

## What I checked and what I didn't

Checked:

- Every bench test and cutover step above.
- DHCP with a real lab guest: VM 109 took a new lease from the AX90 through the bridge at 01:30:10, ten minutes after the cutover. I won't see a renewal, because the AX90 gives reserved hosts an infinite lease.
- At 16:00 on November 11, 14 hours and 40 minutes in: no bridge reboot, link on channel 44 at -57 dBm, `relayd` running, 0% loss at 2.3 ms from jhome to the router, 100% 24-hour uptime on all nine Kuma monitors, and the replacement radio firmware still loaded.
- The finish of the soak at 20:20, 19 hours in (see [Soak result](#soak-result)).
- The evening after the soak, 20:20 to 01:27, from the bridge's log: no drops. It isn't part of the soak.
- The bridge after the November 12 reboot with `roamoff=1` and the fixed MAC: the setting is active, the link and the relay came back, and the lab hosts and services answer.
- jhome, the other lab hosts and the services after I moved jhome to LAN port 1.
- The first config backup's key files match the live router by hash.

Not checked:

- More than one day. The drop count comes from the bridge's first 24 hours: the 19-hour soak, plus the evening after it, which I read from the log.
- Whether `roamoff=1` stops the steering drops. No steering request had come in when I wrote this.
- The 2021 radio firmware against Asus's own download (my decision; see above).
- An actual restore from either config backup, failsafe mode, and bootloader recovery. None has been needed.
- The bridge's own log from 03:12 to 05:30 on November 11. It had already rolled out of the log buffer by the time I looked, so anything other than a steering drop in that window went unseen; the kernel log, which covers it, shows no steering drop there.
- Which channel the router's other 5 GHz radio is on now. I ran no scans during the soak.

VM 109 stopped answering about three minutes after the cutover. It looked like a bridge problem but wasn't. The stall reproduced from inside the lab, and ARP on the home LAN was clean. The cause was a log query I ran on the VM as part of the checks. With a 1.5 GB journal on the hard disk of a 2 GB VM, it turned into a storm of random reads that starved every service on the VM. I restarted the VM, and afterwards capped its journal at 200 MB.

## Band steering during the soak

The soak turned up three short Wi-Fi drops on November 11, at 13:27:43, 16:04:41 and 16:56:08 EST, lasting about 5, 5 and 7 seconds. There were none from 16:56 to the end of the soak at 20:20, and none in the log for the evening after it. That count covers the whole soak: the bridge's normal log only holds about 9 hours, but its kernel log reaches back to boot at about 01:09 and has exactly three of the driver's `Unknown Frame: category 0xa, action 0x8` lines, one per drop. Uptime Kuma registered none of them.

Each one ran the same way:

1. The AX90 sent the bridge an 802.11v steering request pointing it at its other 5 GHz radio.
2. The bridge's radio tried to move there.
3. The key handshake on the other radio failed four times, and the link dropped with reason code 2. The log can't show which side ended it.
4. About a second later the bridge was back on the channel-44 radio.

A read-only research pass through the wpa_supplicant and `brcmfmac` source, plus checks on the bridge, found the likely cause. Firmware roaming is on in the radio. The BSSID lock I set lives only in wpa_supplicant, so it doesn't stop the firmware from roaming on its own. What the lock does do is make wpa_supplicant refuse the key handshake on the other radio, which is why the link drops instead of moving. That the firmware acts on the steering request is an inference from the timing and from a missing "Trying to associate" line in the log. I didn't read it from the firmware.

A second pass through the source added one detail: the firmware does associate to the other radio and receives handshake frames there, even though no scan from the bridge has ever listed that radio.

My soak check also missed these. It counted log lines containing "deauth", and none of these lines contain that word, so it still read 0 while the check's real intent, no disconnects, wasn't met.

The options I had:

- Leave it and accept drops of a few seconds.
- Turn firmware roaming off on the bridge (`options brcmfmac roamoff=1`, a reboot, and the file on the upgrade keep-list). The first research pass recommended trying this first. I haven't confirmed that it covers roams the access point asks for.
- Turn Smart Connect off on the AX90. That changes band selection for every device in the house.
- Remove the BSSID lock so the roam can complete. This is the riskiest, because I never tested the other radio for data.

Turning off 802.11v handling in wpa_supplicant (`disable_btm`) is ruled out. The firmware does the roam, and the `brcmfmac` driver passes only vendor-specific elements from wpa_supplicant into the association request.

I chose to test `roamoff=1`. I applied it on November 12 at 01:42, after the evening drop count, together with a fixed MAC for the LAN interface, and rebooted the bridge. The lab was offline for 34 seconds. After the reboot:

| Check | Result |
| :--- | :--- |
| Driver | Reports `roamoff` as 1, and the radio no longer advertises roaming support |
| Uplink | Back on the channel-44 radio at -56 dBm, 0 failed transmits, 10 of 10 pings to the router |
| LAN interface MAC | The same value as before the reboot, now from the config |
| `relayd`, radio firmware, OpenWrt | Running, 10.10.69.69042, 25.12.6 |
| Lab | jhome, VM 109, CT 110 and the six services answer |
| Uptime Kuma | No down beat on any of the nine monitors across the reboot |

Whether it stops the drops isn't known yet. The evening before had no steering requests at all, so a quiet day proves nothing. The test needs a steering request in the bridge's log with no disconnect after it. If the drops continue with firmware roaming off, I'll accept them: each one lasts a few seconds and nothing in the lab noticed any of the three.

## Rollback

Through the soak the extender stayed powered at its old spot with no cable in it. Until 19:22, rolling back meant moving the switch's uplink from DIR-885L LAN port 1 back to the extender and putting jhome back on the switch. After the recabling at 19:30, it meant taking jhome off LAN port 1 and rebuilding that layout. The two bridges could never both be cabled to the lab, since each answers for the whole lab under its own MAC.

I retired the extender on the evening of November 11, after the soak passed, and unplugged it. That rollback is no longer standing.

The bridge has two fallbacks of its own. The 2.4 GHz radio worked on the bench with the same relay. If an update breaks 5 GHz, the replacement firmware file is still on the router and in my backups.

## Lessons

- **Test the mode you'll actually use.** The known 5 GHz firmware failure was documented for access-point mode only. Client mode failed too, and only the bench test showed it.
- **A pin in one layer doesn't bind another.** The BSSID lock in wpa_supplicant didn't stop the radio firmware from roaming on its own, and the mismatch is what turned a steering request into a drop.
- **A check that greps for one word can pass while its intent fails.** The soak check counted "deauth" and read 0 through three disconnects.
- **Validation can cause the outage it's looking for.** The VM 109 stall came from my own log query, not from the bridge.
- **Measure before you change anything.** Running the same commands against the extender first is what made the before-and-after table possible.
- **Isolate a device with a recovery mode on your gateway's address.** The DIR-885L's bootloader answers on the same address as my home router, so the bench work happened on a single cable in a separate network namespace.

## Next steps

- Watch the bridge's log for the next steering request, to see whether `roamoff=1` stops the drops. If it doesn't, I accept them.

The Cat6 run to the lab is still the long-term plan. It would remove the Wi-Fi hop, which the direct-cable test during the cutover showed is the limit.
