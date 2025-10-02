# Synology Homelab Servarr Stack with Remote Seedbox

A production-ready Docker stack for Synology NAS that manages media with Sonarr, Radarr, and other Servarr applications, integrated with a remote seedbox via Tailscale and NFS.

## System Overview

This system provides:
- **Servarr Stack on Synology**: Sonarr, Radarr, Prowlarr, Bazarr, Overseerr, Plex, etc.
- **Remote Seedbox Integration**: qBittorrent on remote server accessed via NFS over Tailscale
- **Tailscale Access**: All services accessible via HTTPS URLs using tsbridge
- **Advanced YAML Configuration**: Minimal repetition using Docker Compose anchors
- **Future-Ready**: Clear migration path to Traefik when native tsnet support lands

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     SYNOLOGY NAS                             │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                  Docker Containers                     │  │
│  │                                                        │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │  │
│  │  │  Sonarr  │  │  Radarr  │  │    Prowlarr      │   │  │
│  │  │   8989   │  │   7878   │  │      9696        │   │  │
│  │  └──────────┘  └──────────┘  └──────────────────┘   │  │
│  │                                                        │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │  │
│  │  │  Bazarr  │  │Overseerr │  │      Plex        │   │  │
│  │  │   6767   │  │   5055   │  │     32400        │   │  │
│  │  └──────────┘  └──────────┘  └──────────────────┘   │  │
│  │                                                        │  │
│  │  ┌──────────────────────────────────────────────┐   │  │
│  │  │              tsbridge                         │   │  │
│  │  │   (Tailscale proxy for all containers)       │   │  │
│  │  └──────────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              NFS Volume Mounts                        │  │
│  │  torrents-hbd-nvme: /torrents-nvme (active downloads)│  │
│  │  torrents-hbd-hdd: /torrents-hdd (seeding)          │  │
│  └─────────────────────────┬────────────────────────────┘  │
└────────────────────────────┼────────────────────────────────┘
                             │ Tailscale Network (100.64.0.0/10)
                             │
┌────────────────────────────┼────────────────────────────────┐
│                            ↓        REMOTE SEEDBOX           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  qBittorrent                         │   │
│  │                  (Port 8080)                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              File Structure                           │  │
│  │  /home/fizzy/torrents/qbittorrent/ (NVMe - active)   │  │
│  │  /home/fizzy/storage/qbittorrent/  (HDD - seeding)   │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

## Project Structure

```
homelab/
├── appdata/
│   ├── docker-compose.yml          # Main compose file with YAML anchors
│   ├── .env                        # Environment configuration
│   ├── env.example                 # Template for .env
│   ├── [service-name]/             # Individual service configs
│   └── tsbridge/                   # tsbridge state storage
├── data/
│   └── media/                      # Local media library
│       ├── tv/
│       ├── movies/
│       ├── music/
│       └── books/
├── README.md                       # User documentation
├── claude.md                       # This file - project context
└── .gitignore                      # Excludes sensitive files
```

## Key Technologies

### tsbridge
- Provides Tailscale HTTPS access to all containers
- Uses Docker labels for service discovery
- Creates URLs like `https://sonarr.<tailnet>.ts.net`
- Requires OAuth credentials from Tailscale admin

### Docker Compose with YAML Anchors
The compose file uses advanced YAML anchors to minimize repetition:

```yaml
# Base configurations
x-base-service: &base-service
  <<: [*restart-policy, *default-logging]

x-servarr-service: &servarr-service
  <<: *base-service
  environment:
    <<: *common-variables

# Service using anchors
radarr:
  <<: *servarr-service
  labels:
    <<: [*pullio-labels, *tsbridge-labels]
    tsbridge.service.port: "7878"
```

### Native Docker NFS Volumes
```yaml
volumes:
  torrents-hbd-nvme:
    driver: local
    driver_opts:
      type: nfs
      o: addr=${REMOTE_HOST},nfsvers=4.2,...
      device: ":/home/fizzy/torrents/qbittorrent"
```

## Configuration Files

### docker-compose.yml
- Uses YAML anchors extensively
- Defines tsbridge service
- All services have tsbridge labels
- Native NFS volume definitions

### .env
```env
# Synology paths
DOCKERCONFDIR=/volume1/docker/appdata
DOCKERSTORAGEDIR=/volume1/data

# User/Group (Synology docker user)
PUID=1026
PGID=100

# Remote seedbox
REMOTE_HOST=100.64.0.2

# Tailscale OAuth
TS_OAUTH_CLIENT_ID=xxx
TS_OAUTH_CLIENT_SECRET=xxx
```

## Service Access

### Local Network
- `http://synology-ip:8989` - Sonarr
- `http://synology-ip:7878` - Radarr
- `http://synology-ip:9696` - Prowlarr
- etc.

### Via Tailscale (from anywhere)
- `https://sonarr.<tailnet>.ts.net`
- `https://radarr.<tailnet>.ts.net`
- `https://prowlarr.<tailnet>.ts.net`
- etc.

## Setup Process

1. **Configure Tailscale OAuth**:
   - Go to https://login.tailscale.com/admin/settings/oauth
   - Create OAuth client with `devices:write` scope
   - Save credentials

2. **Synology Setup**:
   ```bash
   cd /volume1/docker
   git clone <repo> homelab
   cd homelab/appdata
   # Edit .env with OAuth credentials and paths
   docker-compose up -d
   ```

3. **Remote Seedbox NFS**:
   ```bash
   # /etc/exports on remote
   /home/fizzy/torrents/qbittorrent 100.64.0.0/10(rw,async,no_subtree_check)
   /home/fizzy/storage/qbittorrent 100.64.0.0/10(rw,async,no_subtree_check)
   ```

## Remote Path Mappings

Configure in Sonarr/Radarr:

**For active downloads (NVMe)**:
- Remote: `/home/fizzy/torrents/qbittorrent/`
- Local: `/torrents-nvme/`

**For completed/seeding (HDD)**:
- Remote: `/home/fizzy/storage/qbittorrent/`
- Local: `/torrents-hdd/`

## Future Migration to Traefik

When [Traefik PR #11733](https://github.com/traefik/traefik/pull/11733) is merged:

1. Replace tsbridge with Traefik
2. Update labels from `tsbridge.*` to `traefik.*`
3. Configure Traefik's native tsnet entrypoint
4. Benefit from advanced routing and middleware

Benefits:
- Native Tailscale integration without OAuth
- Advanced routing capabilities
- Built-in middleware (auth, rate limiting)
- Better observability and metrics

## Important Notes

### Synology-Specific
- Docker user is typically UID 1026
- Use `/volume1/` paths for storage
- DSM has native Tailscale package installed
- Docker socket at standard `/var/run/docker.sock`

### Security
- `.env` file contains OAuth secrets (gitignored)
- All service configs excluded from git
- NFS restricted to Tailscale network only
- tsbridge OAuth needs only `devices:write` scope

### Performance
- NVMe mount for active downloads (race categories)
- HDD mount for long-term seeding
- NFS optimized with 1MB buffers
- Soft mounts prevent hanging

## Troubleshooting

### Check tsbridge
```bash
docker logs tsbridge
# Should show services being registered
```

### Verify NFS mounts
```bash
docker exec radarr df -h | grep torrents
# Should show both NFS mounts
```

### Test Tailscale access
```bash
curl https://sonarr.<tailnet>.ts.net
# Should return service page
```

## Maintenance

### Update containers
```bash
docker-compose pull
docker-compose up -d
```

### Backup configs
```bash
tar -czf backup-$(date +%Y%m%d).tar.gz /volume1/docker/appdata
```

### Monitor logs
```bash
docker-compose logs -f [service-name]
```

# Key Decisions

1. **tsbridge over individual Tailscale sidecars**: Simpler OAuth-based setup
2. **Native Docker NFS volumes**: Cleaner than separate mount container
3. **YAML anchors**: Reduces repetition, improves maintainability
4. **Synology paths**: Using standard `/volume1/` structure
5. **Future Traefik migration**: Ready to switch when native tsnet lands

# Development Notes

- Always test compose changes with `docker-compose config` first
- Keep sensitive data in `.env` only
- Document any path changes for Synology compatibility
- Monitor tsbridge project for updates
- Track Traefik PR #11733 for migration timing