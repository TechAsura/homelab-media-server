# Homelab Media Server

## Overview
This project documents the setup of a self-hosted media server using Jellyfin
on a Proxmox homelab. Media is stored on a Windows 11 machine and accessed
by Jellyfin over a network share.

## Environment
- **Hypervisor:** Proxmox VE
- **Hardware:** HP ProDesk 300
- **Jellyfin Host:** LXC Container (Ubuntu 22.04)
- **Media Storage:** Windows 11 PC (SMB share)
- **Media Player:** Jellyfin
- **Request Management:** Jellyseerr

## Architecture
- Windows 11 PC hosts media files via SMB share
- Proxmox mounts the SMB share on the host
- Share is bind-mounted into the Jellyfin LXC container
- Jellyfin serves media over the local network

![Media Server Architecture](architecture/media_server_diagram.png)

## Setup Summary

### SMB Share (Windows 11)
- Created a dedicated local user with read-only access
- Shared media folder over SMB
- Restricted access to the jellyfin user only

### Proxmox LXC Container
- Ubuntu 22.04 LXC container (2 cores, 2GB RAM, 8GB disk)
- Unprivileged container with nesting enabled

### Mounting the Share
- Installed cifs-utils on Proxmox host
- Mounted SMB share at /mnt/media
- Added to /etc/fstab for persistence
- Bind-mounted into LXC container via /etc/pve/lxc/102.conf

### Jellyfin
- Installed via official repository
- Media library pointed to /mnt/media
- Metadata pulled from TheMovieDb

## Remote Access

Remote access is provided via **Tailscale** (WireGuard-based VPN).
No ports are exposed to the internet directly.

### Services accessible via Tailscale

| Service | URL |
|---------|-----|
| Jellyfin | http://YOUR-TAILSCALE-IP:8096 |
| Sonarr | http://YOUR-TAILSCALE-IP:8989 |
| Radarr | http://YOUR-TAILSCALE-IP:7878 |
| Prowlarr | http://YOUR-TAILSCALE-IP:9696 |
| qBittorrent | http://YOUR-TAILSCALE-IP:8080 |

### Setup Summary
- Installed Tailscale on Proxmox host
- Configured iptables to forward Tailscale traffic to LXC container
- Rules saved with netfilter-persistent for persistence on reboot

### Jellyseerr
- Installed via Docker
- Connected to Jellyfin, Radarr and Sonarr
- Shows automatically sorted to `/mnt/media/Shows`
- Anime automatically sorted to `/mnt/media/Anime`
- Movies sorted to `/mnt/media/Movies`
- Accessible via Tailscale at `http://YOUR-TAILSCALE-IP:5055`

## Download Privacy

qBittorrent traffic is routed through NordVPN for privacy.

- NordVPN installed inside Jellyfin LXC
- qBittorrent bound to `nordlynx` network interface
- Kill switch enabled — downloads stop if VPN disconnects
- LAN discovery enabled to maintain local network access

## Troubleshooting

### Disk Space
The LXC disk filled up due to Docker images and Jellyfin metadata.
Resolved by resizing the LXC disk from 8GB to 18GB:
```bash
pct stop 102
e2fsck -f /dev/mapper/pve-vm--102--disk--0
lvresize -L +10G /dev/pve/vm-102-disk-0
resize2fs /dev/mapper/pve-vm--102--disk--0
pct start 102
```

### NordVPN DNS Issue
NordVPN overwrites `/etc/resolv.conf` causing DNS failures.
Resolved by creating a cron job to restore DNS every minute:
```bash
# /etc/nordvpn/dns-fix.sh
echo "nameserver 8.8.8.8" > /etc/resolv.conf
echo "nameserver 1.1.1.1" >> /etc/resolv.conf
```

Added to crontab:
```
* * * * * /etc/nordvpn/dns-fix.sh
```

## Future Improvements
- Add hardware transcoding
- Migrate media storage to a NAS
- Set up automatic backups
