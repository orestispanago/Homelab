# Issues

## jed drops off the network (Wi-Fi driver gets stuck)

Status: **testing fix** (Ethernet connected 2026-11-07)

**Symptom**
- jed disappears from LAN and Tailscale, the Cloudflare tunnel goes down
- Doesn't recover by itself, only after a power cycle
- Happened 2026-10-29, 2026-10-30, 2026-11-07

**Cause** (found 2026-11-07 in the logs)
- Pi kept running, only Wi-Fi was lost
- The router disconnected the Pi and moved 5 GHz to another channel. The Wi-Fi driver (brcmfmac) got stuck: `Association request to the driver failed`, then failed to reconnect for 2 h
- Known Pi 4 driver issue with fast roaming (802.11r) / WPA3
- Not confirmed for 10-29 and 10-30 (no logs saved then), but same symptoms

**Ruled out**
- Power: no under-voltage warnings in the kernel log of the boot that included the 11-07 drop. `throttled=0x0` after the 2026-11-07 13:51 reboot, recheck after a few days of uptime
- Freeze: the watchdog (on by default, 1 min) never rebooted it. It only reacts to a frozen system, not to lost network
- Wi-Fi power saving: already off

**Fix**
- Ethernet cable to the router. Wi-Fi stays on as a backup
- If it happens again on Ethernet: driver option `options brcmfmac roamoff=1 feature_disable=0x82000` in `/etc/modprobe.d/brcmfmac.conf`, or a script that reboots jed when it can't reach the router

## Apps down after a power cut (SSD "dirty")

Status: **workaround**

**Symptom**
- After unplugging the Pi, all apps are down (Cloudflare returns 502)
- Happened after every power cycle: 2026-10-30, 2026-10-31, 2026-11-07

**Cause**
- Power cut leaves the NTFS SSD dirty → `/mnt/ssd` doesn't mount → Docker doesn't start (it requires `/mnt/ssd`)

**Fix**
``` bash
sudo ntfsfix -d /dev/sda2 && sudo mount /mnt/ssd
sudo systemctl reset-failed docker.socket docker.service && sudo systemctl start docker
```
- Permanent: run `chkdsk /f` from Windows, or move the SSD to ext4
