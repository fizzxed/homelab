# Homelab Servarr Stack on Synology with Remote Seedbox

A production-ready Docker stack for Synology NAS, managing media with Sonarr, Radarr, and other Servarr applications, integrated with a remote seedbox via Tailscale and NFS.

## Architecture Overview

This setup follows the [TRaSH Guides](https://trash-guides.info/) best practices while adapting for Synology NAS and remote seedbox architecture:

- **Synology NAS**: Hosts Servarr stack with DSM's native Tailscale installation
- **tsbridge**: Exposes all Docker containers via Tailscale DNS names for secure remote access
- **Remote Seedbox**: Handles all torrent downloading via qBittorrent
- **Connectivity**: Secure Tailscale VPN mesh network
- **File Access**: Native Docker NFS volumes for seamless remote file access

## Directory Structure

```
homelab/
├── appdata/                 # Application configurations
│   ├── docker-compose.yml   # Main compose file with YAML anchors
│   ├── .env                 # Environment configuration
│   ├── tailscale-auth.env   # Tailscale auth key (gitignored)
│   └── [service configs]/   # Individual service configs
├── data/
│   └── media/               # Local media library
│       ├── tv/
│       ├── movies/
│       ├── music/
│       └── books/
├── torrents-hbd-nvme/       # NFS mount: remote active downloads
└── torrents-hbd-hdd/        # NFS mount: remote completed/seeding
```

## Remote Server Structure

```
remote-server (100.64.0.2)/
├── torrents/
│   └── qbittorrent/         # Active downloads on NVMe
│       ├── race-anime/
│       └── race-music/
└── storage/
    └── qbittorrent/         # Completed/seeding on HDD
        └── music/
```

## Future: Migration to Traefik

Once [Traefik PR #11733](https://github.com/traefik/traefik/pull/11733) (native tsnet support) is merged and released, we plan to migrate from tsbridge to Traefik for the following benefits:

- **Native Tailscale integration**: Built-in tsnet support without separate OAuth setup
- **Advanced routing**: Path-based and host-based routing capabilities
- **Middleware support**: Authentication, rate limiting, headers manipulation
- **Better observability**: Prometheus metrics, access logs, and tracing
- **Single reverse proxy**: Unified solution for both local and Tailscale access

The migration will be straightforward:
1. Replace tsbridge container with Traefik
2. Update service labels from `tsbridge.*` to `traefik.*`
3. Configure Traefik's tsnet entrypoint
4. Benefit from Traefik's enterprise-grade features

## Prerequisites

- Synology NAS with Docker package installed
- Tailscale installed on Synology DSM
- Tailscale OAuth credentials
- Remote server with:
  - qBittorrent installed and configured
  - NFS server configured and running
  - Tailscale installed and connected

## Setup Instructions

### 1. Remote Server Configuration

Configure NFS exports on your remote server:

```bash
# Edit /etc/exports
sudo vim /etc/exports

# Add these lines (replace 100.64.0.0/10 with your Tailscale network if its different)
/home/fizzy/torrents/qbittorrent 100.64.0.0/10(rw,async,no_subtree_check,no_root_squash)
/home/fizzy/storage/qbittorrent 100.64.0.0/10(rw,async,no_subtree_check,no_root_squash)

# Apply changes
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

### 2. Tailscale OAuth Setup for tsbridge

1. Go to https://login.tailscale.com/admin/settings/oauth
2. Create a new OAuth client:
   - Add a descriptive name (e.g., "homelab-tsbridge")
   - Select scopes: `devices:write` (required for tsbridge)
   - Save the client ID and secret
3. Update `appdata/.env` with your OAuth credentials:
   ```bash
   TS_OAUTH_CLIENT_ID=your-client-id
   TS_OAUTH_CLIENT_SECRET=your-client-secret
   ```

### 3. Synology Setup

1. SSH into your Synology and clone this repository:
   ```bash
   ssh admin@synology-ip
   cd /volume1/docker
   git clone <your-repo-url> homelab
   cd homelab
   ```

2. Configure environment:
   ```bash
   # Edit the .env file
   mv appdata/example.env appdata/.env
   vim appdata/.env

   # Update these values:
   # - DOCKERCONFDIR=/volume1/docker/appdata
   # - DOCKERSTORAGEDIR=/volume1/data
   # - REMOTE_HOST (Tailscale IP of remote server)
   # - TS_OAUTH_CLIENT_ID and TS_OAUTH_CLIENT_SECRET
   # - PUID/PGID (typically 1026/100 for docker user)
   # - TZ (your timezone)
   # - PLEX_CLAIM (from plex.tv/claim)
   ```

3. Create data directories:
   ```bash
   sudo mkdir -p /volume1/data/media/{tv,movies,music,books}
   sudo chown -R 1026:100 /volume1/data/media
   ```

4. Start the stack:
   ```bash
   cd /volume1/docker/homelab/appdata

   # Start tsbridge first to establish Tailscale connections
   docker-compose up -d tsbridge

   # Verify tsbridge is running
   docker logs tsbridge

   # Start all services
   docker-compose up -d
   ```

## Service Configuration

### Download Client (qBittorrent)

In Sonarr/Radarr, add qBittorrent as download client:
- **Host**: `100.64.0.2` (Tailscale IP of remote server)
- **Port**: `8080`
- **Username/Password**: Your qBittorrent credentials
- **Category**: Configure per service (e.g., `tv` for Sonarr, `movies` for Radarr)

### Remote Path Mappings

Configure in Sonarr/Radarr settings:

For active downloads:
- **Remote Path**: `/home/fizzy/torrents/qbittorrent/`
- **Local Path**: `/torrents-nvme/`

For completed/seeding:
- **Remote Path**: `/home/fizzy/storage/qbittorrent/`
- **Local Path**: `/torrents-hdd/`

### Root Folders

- **Sonarr**: `/data/media/tv`
- **Radarr**: `/data/media/movies`
- **4K Radarr**: `/data/media/movies-4k`

## Service Access

### Local Access (within your network)
Services are accessible via your Synology's IP address:

| Service | Local URL | Default Credentials |
|---------|-----------|-------------------|
| Sonarr | http://synology-ip:8989 | No auth by default |
| Radarr | http://synology-ip:7878 | No auth by default |
| 4K Radarr | http://synology-ip:7877 | No auth by default |
| Prowlarr | http://synology-ip:9696 | No auth by default |
| Bazarr | http://synology-ip:6767 | No auth by default |
| Overseerr | http://synology-ip:5055 | Setup on first visit |
| Plex | http://synology-ip:32400 | Plex account |
| Tautulli | http://synology-ip:8181 | No auth by default |
| Autobrr | http://synology-ip:7474 | Setup on first visit |
| Notifiarr | http://synology-ip:5454 | Setup on first visit |

### Tailscale Access (from anywhere)
With tsbridge, services are accessible via Tailscale DNS names:

| Service | Tailscale URL |
|---------|---------------|
| Sonarr | https://sonarr.&lt;tailnet&gt;.ts.net |
| Radarr | https://radarr.&lt;tailnet&gt;.ts.net |
| 4K Radarr | https://4kradarr.&lt;tailnet&gt;.ts.net |
| Prowlarr | https://prowlarr.&lt;tailnet&gt;.ts.net |
| Bazarr | https://bazarr.&lt;tailnet&gt;.ts.net |
| Overseerr | https://overseerr.&lt;tailnet&gt;.ts.net |
| Plex | https://plex.&lt;tailnet&gt;.ts.net |
| Tautulli | https://tautulli.&lt;tailnet&gt;.ts.net |
| Autobrr | https://autobrr.&lt;tailnet&gt;.ts.net |
| Notifiarr | https://notifiarr.&lt;tailnet&gt;.ts.net |

Replace `<tailnet>` with your actual Tailnet name (found in Tailscale admin console).

## Troubleshooting

### Check NFS Mounts

```bash
# Verify volumes are created
docker volume ls | grep torrents

# Check if mounted in container
docker exec radarr df -h | grep torrents
```

### Tailscale Connectivity

```bash
# Check Tailscale status
docker exec tailscale tailscale status

# Ping remote server
docker exec tailscale ping 100.64.0.2
```

### NFS Mount Issues

```bash
# Test NFS connectivity
showmount -e 100.64.0.2

# Manual mount test
sudo mount -t nfs4 100.64.0.2:/home/fizzy/torrents/qbittorrent /tmp/test
ls /tmp/test
sudo umount /tmp/test
```

### View Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f radarr
```

## Maintenance

### Updating Containers

With Pullio configured:
```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock hotio/pullio
```

Or manually:
```bash
docker-compose pull
docker-compose up -d
```

### Backup Configuration

```bash
# Backup all configs
tar -czf backup-$(date +%Y%m%d).tar.gz appdata/
```

## Key Features

1. **Advanced YAML Anchors**:
   - `x-base-service`: Common configuration for all services
   - `x-servarr-service`: Extended base for Servarr applications
   - Clean label merging for pullio and tsbridge
   - Minimal repetition, maximum maintainability
2. **tsbridge Integration**:
   - Automatic Tailscale DNS names for all services
   - HTTPS by default on Tailscale connections
   - Works alongside Synology's native Tailscale
3. **Native NFS Volumes**: Docker handles NFS mounting automatically
4. **Secure Networking**: All remote access via Tailscale VPN
5. **Performance Optimized**:
   - NVMe for active downloads (torrents-hbd-nvme)
   - HDD for long-term seeding (torrents-hbd-hdd)
   - 1MB read/write buffers for NFS
6. **Synology Optimized**: Proper paths and permissions for Synology Docker
7. **Future-Ready**: Clear migration path to Traefik when native tsnet lands

## Security Notes

- Never commit `.env` file with OAuth credentials and sensitive data
- All service configs are gitignored by default
- Use strong passwords for all services
- Consider implementing authentication for exposed services
- NFS is restricted to Tailscale network only
- tsbridge OAuth client should only have `devices:write` scope
- Regularly review Tailscale device access in admin console

## Support

For issues and questions:
- [TRaSH Guides](https://trash-guides.info/)
- [Servarr Wiki](https://wiki.servarr.com/)
- [Tailscale Documentation](https://tailscale.com/kb/)
