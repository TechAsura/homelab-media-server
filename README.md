# Homelab Media Server

Self-hosted media server running on Proxmox VE. Jellyfin serves media stored on a Windows 11 machine over SMB, with automated downloading via Sonarr, Radarr, and qBittorrent — all routed through NordVPN.

## Environment

| Component | Details |
|-----------|---------|
| Hypervisor | Proxmox VE on HP ProDesk 300 |
| Jellyfin Host | LXC Container — Ubuntu 22.04 (10.0.0.134) |
| Media Storage | Windows 11 PC via SMB share |
| Media Player | Jellyfin |
| Request Management | Jellyseerr |
| Download Privacy | NordVPN (kill switch enabled) |

---

## Architecture

```
Windows 11 PC (SMB share)
        ↓
Proxmox host mounts share at /mnt/media
        ↓
Bind-mounted into Jellyfin LXC at /mnt/media
        ↓
Jellyfin serves media over local network + Tailscale
```

Requests flow through Jellyseerr → Sonarr/Radarr → Prowlarr → qBittorrent. Downloaded files land in `/mnt/media` and are picked up automatically by Jellyfin.

![Architecture Diagram](architecture/media_server_diagram.png)

---

## Setup

### SMB Share (Windows 11)
- Created a dedicated local user with read-only access
- Shared media folder over SMB
- Access restricted to the jellyfin user only

### Proxmox LXC Container
- Ubuntu 22.04, unprivileged, nesting enabled
- 2 cores, 2GB RAM, 18GB disk
- Config: `features: keyctl=1,nesting=1`

### Mounting the SMB Share

```bash
# Install cifs-utils on Proxmox host
apt install cifs-utils

# Mount the share
mount -t cifs //WINDOWS-PC/media /mnt/media -o username=jellyfin,password=XXX

# Add to /etc/fstab for persistence
//WINDOWS-PC/media /mnt/media cifs username=jellyfin,password=XXX,_netdev 0 0
```

Bind-mount into LXC via `/etc/pve/lxc/102.conf`:
```
mp0: /mnt/media,mp=/mnt/media
```

### Jellyfin
- Installed via official APT repository
- Media library pointed to `/mnt/media`
- Metadata pulled from TheMovieDB

---

## Media Automation

| Service | Role | Port |
|---------|------|------|
| Jellyseerr | Request UI | 5055 |
| Sonarr | TV show automation | 8989 |
| Radarr | Movie automation | 7878 |
| Prowlarr | Indexer manager | 9696 |
| qBittorrent | Torrent client | 8080 |

### Media Paths
| Type | Path |
|------|------|
| Movies | `/mnt/media/Movies` |
| Shows | `/mnt/media/Shows` |
| Anime | `/mnt/media/Anime` |

---

## Download Privacy

qBittorrent traffic is routed through NordVPN for privacy.

- NordVPN installed as a system service inside the jellyfin LXC
- qBittorrent bound to the `nordlynx` network interface
- Kill switch enabled — downloads stop if VPN disconnects
- LAN subnet (10.0.0.0/24) whitelisted so local services can always reach qBittorrent
- Auto-connect enabled so VPN reconnects on reboot

---

## Remote Access

Remote access is via **Tailscale** (WireGuard-based). No ports are exposed to the internet directly.

| Service | URL |
|---------|-----|
| Jellyfin | `http://100.92.204.101:8096` |
| Jellyseerr | `http://100.92.204.101:5055` |
| Sonarr | `http://100.92.204.101:8989` |
| Radarr | `http://100.92.204.101:7878` |
| Prowlarr | `http://100.92.204.101:9696` |
| qBittorrent | `http://100.92.204.101:8080` |

All services are also available on the local network via `https://*.idiotproductions.net` through Nginx Proxy Manager with wildcard SSL.

---

## Troubleshooting

### qBittorrent not downloading — NordVPN kill switch

**Symptom:** Jellyseerr requests reach Sonarr/Radarr, items appear in the queue as "Queued", and torrents show up in qBittorrent but sit at 0% with 0 seeds, 0 peers, and DHT showing 0 nodes.

**Cause:** NordVPN's kill switch blocks all traffic when the VPN is disconnected. If the LXC reboots or nordvpnd crashes without auto-connect enabled, qBittorrent loses all connectivity.

**Fix:**
```bash
# Check status
nordvpn status

# Reconnect
nordvpn connect

# Whitelist LAN so Sonarr/Radarr (on a separate LXC) can reach qBittorrent
nordvpn whitelist add subnet 10.0.0.0/24

# Auto-connect on reboot
nordvpn set autoconnect on
```

---

### NordVPN overwrites /etc/resolv.conf

**Symptom:** DNS failures inside the LXC after NordVPN connects.

**Cause:** NordVPN replaces `/etc/resolv.conf` with its own servers.

**Fix:**
```bash
# /etc/nordvpn/dns-fix.sh
echo "nameserver 8.8.8.8" > /etc/resolv.conf
echo "nameserver 1.1.1.1" >> /etc/resolv.conf
```

```bash
# crontab -e
* * * * * /etc/nordvpn/dns-fix.sh
```

---

### LXC disk full

**Symptom:** Services failing, no space left on device errors.

**Cause:** Docker images and Jellyfin metadata filled the original 8GB disk.

**Fix:** Resize the LXC disk from the Proxmox host:
```bash
pct stop 102
e2fsck -f /dev/mapper/pve-vm--102--disk--0
lvresize -L +10G /dev/pve/vm-102-disk-0
resize2fs /dev/mapper/pve-vm--102--disk--0
pct start 102
```

---

## Future Improvements

- Hardware transcoding in Jellyfin
- Migrate media storage to a dedicated NAS
- Second Proxmox node for redundancy
- Automated backups
