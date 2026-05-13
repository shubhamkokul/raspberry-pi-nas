# Raspberry Pi Personal Cloud Storage

Self-hosted cloud storage built on a Raspberry Pi 5 with NVMe. Auto-sync across all devices — iPhone, Android, Windows, Mac. No subscriptions. Your data, your hardware.

**Stack**: Nextcloud + Docker + Tailscale + Tailscale Serve (HTTPS)

---

## What You Get

### Your own iCloud/Google Drive — for free
- Unlimited storage (limited only by your NVMe size)
- No monthly subscription — ever
- No storage limits, no upsells, no privacy concerns
- Your photos and files never leave your home

### Auto-sync across every device
- iPhone and Android: photos auto-upload in the background the moment they're taken
- Windows and Mac: folder sync just like Dropbox
- If the Pi is offline when you take a photo — no problem. The app queues it and syncs automatically the moment the Pi comes back online
- Works on home WiFi and from anywhere in the world via Tailscale

### Secure remote access — no exposed ports
- Tailscale creates an encrypted tunnel between your devices and the Pi
- No port forwarding on your router
- No dynamic DNS
- No public IP exposure
- Valid HTTPS certificate (Let's Encrypt via Tailscale) — works on iOS without any warnings

### Always on, always ready
- Pi 5 runs on ~5W — less than a phone charger
- Everything auto-starts on boot — Docker, Nextcloud, MariaDB, Tailscale
- Safe shutdown with one command, powers back on by plugging in

### Full control
- You own the hardware and the data
- Web UI to browse files from any browser
- Create separate accounts for family members
- Share folders between family accounts

### Cost breakdown
| Item | One-time cost |
|---|---|
| Raspberry Pi 5 (4GB or 8GB) | ~$60–$80 |
| M.2 NVMe HAT | ~$15–$25 |
| 1TB NVMe SSD | ~$60–$80 |
| microSD card (for flashing) | ~$10 |
| **Total** | **~$145–$195** |

Compare to Google One (2TB): $10/month = $120/year. This pays for itself in under 2 years — then it's free forever.

---

---

## Hardware

| Component | Detail |
|---|---|
| SBC | Raspberry Pi 5 |
| Storage | 1TB M.2 NVMe SSD (via M.2 HAT) |
| Boot drive | microSD card (used only for initial flash — OS cloned to NVMe) |
| Connection | Ethernet recommended |

---

## Architecture

```
Family devices (iPhone / Android / Windows / Mac)
                    │
          [Nextcloud clients]
                    │
        [Tailscale VPN Tunnel]
     (encrypted, works from anywhere)
                    │
           [Raspberry Pi 5]
                    │
      ┌─────────────▼─────────────┐
      │   Tailscale Serve (HTTPS) │
      │   → Nextcloud (port 8080) │
      │   → MariaDB               │
      │   → 1TB NVMe (data)       │
      └───────────────────────────┘
```

---

## Build Steps

### 1 — Flash SD Card

Download **Raspberry Pi Imager** from https://www.raspberrypi.com/software/

On Linux, run as AppImage:
```bash
chmod +x ~/Downloads/imager_*.AppImage
~/Downloads/imager_*.AppImage
```

Imager settings:
- Device: Raspberry Pi 5
- OS: Raspberry Pi OS (64-bit)
- Hostname: `<your-hostname>`
- Username: `<your-username>`
- SSH: enabled (password auth)
- WiFi: optional (ethernet recommended)

### 2 — First Boot & SSH

```bash
ssh <username>@<hostname>.local
uname -a && lsblk   # verify OS + NVMe detected
```

NVMe appears as `nvme0n1`. PCIe is enabled by default on Pi 5.

### 3 — Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# Log out and back in
docker run hello-world   # verify
```

### 4 — Install rpi-clone

```bash
sudo apt install -y git
git clone https://github.com/billw2/rpi-clone.git
cd rpi-clone
sudo cp rpi-clone /usr/local/sbin/
```

### 5 — Clone SD → NVMe

rpi-clone doesn't handle NVMe naming (ends in digit), use `dd` instead:

```bash
sudo dd if=/dev/mmcblk0 of=/dev/nvme0n1 bs=4M status=progress conv=fsync
```

Takes ~20 minutes for a 128GB card. To check progress from a second SSH session:
```bash
sudo kill -USR1 $(pgrep dd)
```

### 6 — Expand NVMe Partition to Full Size

```bash
sudo parted /dev/nvme0n1 resizepart 2 100%
sudo e2fsck -f /dev/nvme0n1p2   # answer 'yes' to any prompts
sudo resize2fs /dev/nvme0n1p2
```

### 7 — Set Boot Order to NVMe

```bash
sudo raspi-config
# Advanced Options → Boot Order → NVMe/USB Boot → reboot
```

Verify after reboot:
```bash
lsblk   # / should be on nvme0n1p2
```

### 8 — Wipe SD Card (optional, from another machine)

```bash
lsblk                        # confirm SD card device name
sudo umount /dev/<sdcard>1
sudo mkfs.ext4 /dev/<sdcard>
```

### 9 — Set a Static IP (Recommended)

Without this, your Pi gets a new IP from DHCP on each boot, which breaks SSH and Nextcloud trusted domains.

```bash
# Check your connection name
nmcli connection show

# Set static IP — adjust to match your router's subnet
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8 1.1.1.1"

sudo nmcli connection up "Wired connection 1"
ip addr show   # verify your static IP is assigned
```

Pick an IP outside your router's DHCP range (usually `.100`–`.254` is safe — check your router settings).

### 10 — Create Nextcloud Data Directory

```bash
sudo mkdir -p /ncdata
sudo chown -R $USER:$USER /ncdata
```

### 11 — Create docker-compose.yml

```bash
mkdir ~/nextcloud && cd ~/nextcloud
vi docker-compose.yml
```

```yaml
services:
  db:
    image: mariadb:10.11
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: <strong-root-password>
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: <strong-db-password>
    volumes:
      - db:/var/lib/mysql

  nextcloud:
    image: nextcloud:latest
    restart: always
    ports:
      - "8080:80"
    depends_on:
      - db
    environment:
      MYSQL_HOST: db
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: <strong-db-password>
    volumes:
      - /ncdata:/var/www/html/data
      - nextcloud:/var/www/html

volumes:
  db:
  nextcloud:
```

Validate:
```bash
docker compose config   # should show /ncdata as bind source
```

### 12 — Start Nextcloud

```bash
docker compose up -d
docker ps   # both containers should show Up
```

Access at `http://<pi-local-ip>:8080` to complete setup. Create admin account.

### 13 — Install Tailscale on Pi

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale ip -4   # note this IP — it's permanent
```

Add Tailscale IP to Nextcloud trusted domains:
```bash
docker exec --user www-data nextcloud-nextcloud-1 php occ config:system:set trusted_domains 2 --value=<tailscale-ip>
```

### 14 — Enable HTTPS via Tailscale Serve

**Why**: iOS blocks plain HTTP. Tailscale Serve adds HTTPS with a valid Let's Encrypt cert — no Caddy or nginx needed.

In Tailscale admin console (tailscale.com/admin/dns):
- Enable **MagicDNS**
- Enable **HTTPS Certificates**

Your Pi gets a domain: `<hostname>.<tailnet>.ts.net`

Generate cert and start Tailscale Serve:
```bash
sudo tailscale cert <hostname>.<tailnet>.ts.net
sudo tailscale serve --bg http://localhost:8080
```

Verify:
```bash
tailscale serve status
# Should show: https://<hostname>.<tailnet>.ts.net/ → proxy http://localhost:8080
```

Add HTTPS domain to Nextcloud trusted domains:
```bash
docker exec --user www-data nextcloud-nextcloud-1 php occ config:system:set trusted_domains 3 --value=<hostname>.<tailnet>.ts.net
```

Nextcloud is now accessible at:
```
https://<hostname>.<tailnet>.ts.net
```

---

## Client Setup

| Platform | App | Server URL |
|---|---|---|
| iPhone | Nextcloud (App Store) | `https://<hostname>.<tailnet>.ts.net` |
| Android | Nextcloud (Play Store) | `https://<hostname>.<tailnet>.ts.net` |
| Windows | Nextcloud Desktop | `https://<hostname>.<tailnet>.ts.net` |
| Mac | Nextcloud Desktop | `https://<hostname>.<tailnet>.ts.net` |

**Auto photo backup**: App → Settings → Auto Upload → enable

**Tailscale**: Install on all devices. Share the Pi with family members via tailscale.com/admin/machines → Share.

---

## Auto-Start Verification

All services start automatically on boot:

```bash
systemctl is-enabled docker tailscaled
# Should return: enabled / enabled

docker inspect --format='{{.Name}} restart={{.HostConfig.RestartPolicy.Name}}' \
  nextcloud-nextcloud-1 nextcloud-db-1
# Should show: restart=always for both

sudo tailscale serve status
# Should show the HTTPS proxy config
```

---

## Maintenance

```bash
# Update Nextcloud
docker compose pull && docker compose up -d

# OS updates
sudo apt update && sudo apt upgrade -y

# Change WiFi
sudo nmtui

# Check storage
df -h /ncdata

# Safe shutdown
sudo shutdown -h now
# Pi powers on automatically when power is reconnected
```

---

## Troubleshooting

**iOS Nextcloud app won't connect**
- Make sure Tailscale is running on the iPhone
- Use HTTPS URL, not HTTP
- Try QR code login instead of manual entry

**Browser hangs on HTTPS**
- Confirm Tailscale is installed and connected on the client device
- Run `tailscale serve status` on Pi to verify proxy is running
- Test with `curl -v https://<hostname>.<tailnet>.ts.net` from client

**NVMe not detected**
- Pi 5 has PCIe enabled by default — no config.txt changes needed
- Check HAT is seated properly on the PCIe FFC connector

**NVMe boots but no network / SSH times out**
- Root cause: NVMe was cloned from SD before WiFi was configured. SD got WiFi config later; NVMe never had it.
- Fix: boot from SD, mount NVMe, copy netplan files:
```bash
sudo mount /dev/nvme0n1p2 /mnt
sudo mkdir -p /mnt/etc/netplan
sudo cp /etc/netplan/*.yaml /mnt/etc/netplan/
sudo chmod 600 /mnt/etc/netplan/*.yaml
sudo umount /mnt
```
- Change boot order back to NVMe, reboot.

**SSH host key warning after reboot ("WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED")**
- Expected after reinstalling OS or regenerating SSH keys. Clear the old key:
```bash
ssh-keygen -R <pi-ip-or-hostname>
ssh <user>@<pi-ip>
```

**SSH corruption after hard shutdown (ssh_host files broken)**
- Boot from SD card, mount NVMe, regenerate SSH keys via chroot:
```bash
sudo mount /dev/nvme0n1p2 /mnt
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo mount --bind /dev /mnt/dev
sudo chroot /mnt
rm /etc/ssh/ssh_host_*
ssh-keygen -A
exit
sudo fsck -f /dev/nvme0n1p2   # fix any filesystem corruption
```
- Reboot from NVMe.

**EEPROM flash stuck / bootloader issues**
- Flash a fresh bootloader image via Raspberry Pi Imager: Misc utility images → Bootloader → SD Card Boot
- Boot from that SD card to reset the EEPROM
- Then re-flash OS SD and start again
