# Homelab

Raspberry Pi 4 (2 GB RAM) at home, Debian 13 Trixie, root on SD card (29G). Hostname `jed`.
Serves Plex, Navidrome, Transmission and the Fitness app to the internet through a **Cloudflare tunnel**.
## Connect
``` bash
ssh pi@jed.local   # home LAN (wifi)
ssh pi@jed         # Tailscale (anywhere)
```
Key login only, user `pi`. sudo asks for a password, so run sudo commands in a real terminal (not with Claude's `!`).
## Services
Docker compose, one folder each in `~/`: `plex`, `navidrome`, `transmission`, `Fitness-webapp`.
cloudflared runs as a systemd service:

| URL                      | Port                        |
| ------------------------ | --------------------------- |
| music.example.com        | 4533 Navidrome              |
| plex.example.com         | 32400 Plex                  |
| transmission.example.com | 9091 Transmission           |
| fitness.example.com      | 5000 Flask (127.0.0.1 only) |

Data is on the USB SSD **"red"** (NTFS, RTL9201 enclosure) mounted at `/mnt/ssd`: Music, Movies, torrents, plex/config, navidrome/data.
**Docker needs `/mnt/ssd`** (`/etc/systemd/system/docker.service.d/override.conf` → `RequiresMountsFor=/mnt/ssd`). No SSD = no apps.
## Health check
``` bash
tailscale status | grep jed
ssh pi@jed.local 'findmnt /mnt/ssd; docker ps; systemctl --failed; vcgencmd get_throttled'
```
`throttled=0x0` is good, anything else = undervoltage/overheating.
## Watchdog
Reboots the Pi automatically if it freezes.
``` bash
sudo nano /etc/systemd/system.conf
```
Uncomment and edit:
```
RuntimeWatchdogSec=15
```
Apply with `sudo systemctl daemon-reexec` (or reboot). Check: `systemctl show -p RuntimeWatchdogUSec`
## Persistent logs
Raspberry Pi OS keeps logs in RAM only (`Storage=volatile` in `/usr/lib/systemd/journald.conf.d/40-rpi-volatile-storage.conf`), so they are lost on reboot. Override it:
``` bash
sudo mkdir -p /etc/systemd/journald.conf.d
echo -e '[Journal]\nStorage=persistent' | sudo tee /etc/systemd/journald.conf.d/99-persistent.conf
sudo systemctl restart systemd-journald
sudo journalctl --flush
```
After a crash, read the previous boot: `journalctl -b -1 -e`
## Log
#### 2026-10-30: everything down
- Pi vanished about 15h earlier (not on LAN or Tailscale). Cause unknown, logs weren't saved yet
- Rebooted → SSD NTFS **dirty** → `/mnt/ssd` didn't mount → Docker didn't start
- Fix:
``` bash
sudo ntfsfix -d /dev/sda2 && sudo mount /mnt/ssd
sudo systemctl reset-failed docker.socket docker.service && sudo systemctl start docker
```
- Suspect: the SSD enclosure, it also dropped off USB on the PC on 2026-10-03
- Added watchdog and persistent logs (see above)
## TODO
- [x] Watchdog
- [x] Persistent logs
- [ ] `chkdsk /f` on the SSD from Windows (ntfsfix found `$MFTMirr` damage), or move it to ext4
- [ ] Check `vcgencmd get_throttled` in a few days
