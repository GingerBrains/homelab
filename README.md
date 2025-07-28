# Homelab Setup Guide

## Purpose
This Ubuntu Server container running on Proxmox hosts various services for personal use:
- **Nextcloud** (file storage and sharing)
- **Tailscale** (remote access)
- **Pi-hole** (network-wide ad blocking)
- **Jellyfin** (media streaming)
- **qBittorrent** (torrent client)
- **Radarr** (movie automation)
- **Sonarr** (TV show automation)
- **Prowlarr** (indexer management)
- **Flaresolverr** (cloudflare bypass for indexers)

## Prerequisites
Before starting this setup, ensure you have:
- **Proxmox VE** installed on your server
- **Ubuntu Server LXC container** created in Proxmox
- **Static IP** assigned to your container
- **SSD storage** for config files (default filesystem)
- **HDD storage** for media and bulk data (optional but recommended)
- **Basic Linux knowledge** (command line, file permissions, networking)

## System Details
- **Host**: Proxmox VE
- **Container**: Ubuntu Server LXC
- **Static IP**: 192.168.0.110 (replace with your IP)
- **Architecture**: x86_64

## Network Configuration
- **Static IP**: 192.168.0.110 (replace with your IP)
- **Gateway**: 192.168.0.1 (typical)
- **DNS**: 8.8.8.8, 8.8.4.4 (or your Pi-hole IP)

## Installation Steps

### 0. Initial Container Setup
```bash
# Update system and install essential packages
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git nano htop

# Create necessary directories
sudo mkdir -p /home/YOUR_USER/docker
sudo mkdir -p /media/movies /media/tvseries /media/downloads /storage

# Set proper permissions
sudo chown -R YOUR_USER:YOUR_USER /home/YOUR_USER/docker
```

### 1. Initial Setup
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker and Docker Compose
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
sudo systemctl enable docker
sudo systemctl start docker

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### 2. Install Tailscale
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

### 3. Scheduled Shutdown and Startup
Add to crontab (`crontab -e`):
```cron
0 2 * * * /sbin/shutdown -h now
0 8 * * * /sbin/poweron  # (Replace with your method to start the container, e.g., via Proxmox task or script)
```

### 4. Deploy Services
Each service has its own Docker Compose file for easy management:

```bash
# Deploy individual services
docker-compose -f nextcloud.yml up -d
docker-compose -f jellyfin.yml up -d
docker-compose -f qbittorrent.yml up -d
docker-compose -f prowlarr.yml up -d
docker-compose -f sonarr.yml up -d
docker-compose -f radarr.yml up -d
docker-compose -f flaresolverr.yml up -d
```

**Note**: Pi-hole is installed directly on the host system using the official installer, not via Docker.

### 5. Install Pi-hole (Host Installation)
```bash
# Install Pi-hole directly on the host
curl -sSL https://install.pi-hole.net | bash

# Follow the interactive installer prompts
# - Choose your network interface
# - Set static IP (if not already configured)
# - Choose upstream DNS providers
# - Select blocklists
# - Set admin password
# - Enable web interface
# - Enable web server
```

## Service Configuration Notes

### Nextcloud Setup
1. **First-time setup**: Access http://YOUR_IP:8080
2. **Create admin account** with secure password
3. **Database setup**: Use the MariaDB credentials from the Docker Compose file
4. **Trusted domains**: Add your IP to the trusted domains list

### Jellyfin Setup
1. **First-time setup**: Access http://YOUR_IP:8096
2. **Create admin account** with secure password
3. **Add media libraries**: Point to `/data/movies` and `/data/tvshows`
4. **Configure metadata**: Enable movie/TV show metadata scraping

### Media Automation (Sonarr/Radarr/Prowlarr)
1. **Prowlarr first**: Set up indexers and download clients
2. **Sonarr/Radarr**: Configure with your Prowlarr and qBittorrent
3. **Quality profiles**: Set up your preferred quality settings
4. **Root folders**: Point to `/tv` (Sonarr) and `/movies` (Radarr)

### Pi-hole Setup
1. **Installation**: Use the official installer script `curl -sSL https://install.pi-hole.net | bash`
2. **Web interface**: Access http://YOUR_IP (port 80)
3. **Admin password**: Set during installation or use `pihole -a -p` to change
4. **Configure DNS**: Set your router's DNS to point to Pi-hole IP
5. **Add blocklists**: Import popular ad-blocking lists via web interface

## Service Overview & Port Assignments

| Service | Host Port | Container Port | Installation Method | Purpose |
|---------|-----------|----------------|-------------------|---------|
| Nextcloud | 8080 | 80 | Docker | File storage |
| Jellyfin | 8096 | 8096 | Docker | Media streaming |
| qBittorrent | 8081 | 8081 | Docker | Torrent client |
| Prowlarr | 9696 | 9696 | Docker | Indexer management |
| Sonarr | 8989 | 8989 | Docker | TV automation |
| Radarr | 7878 | 7878 | Docker | Movie automation |
| Flaresolverr | 8191 | 8191 | Docker | Cloudflare bypass |
| Pi-hole | 80 | 80 | Host Install | DNS/ad blocking |

## Accessing Services
- **Nextcloud**: http://YOUR_IP:8080
- **Jellyfin**: http://YOUR_IP:8096
- **qBittorrent**: http://YOUR_IP:8081
- **Prowlarr**: http://YOUR_IP:9696
- **Sonarr**: http://YOUR_IP:8989
- **Radarr**: http://YOUR_IP:7878
- **Flaresolverr**: http://YOUR_IP:8191
- **Pi-hole**: http://YOUR_IP (port 80)

## Storage Configuration
This setup uses a hybrid storage approach:
- **SSD**: Config files and Docker containers (default filesystem)
- **HDD**: Bulk media and data storage (mounted at `/media/` and `/storage/`)

### Volume Paths
- **Nextcloud**: 
  - Config: `/home/YOUR_USER/docker/nextcloud/appdata:/config`
  - Data: `/storage:/data`
  - Database: `/home/YOUR_USER/docker/nextcloud/db:/var/lib/mysql`
- **Jellyfin**: 
  - Config: `/home/YOUR_USER/docker/jellyfin/config:/config`
  - TV: `/media/tvseries:/data/tvshows`
  - Movies: `/media/movies:/data/movies`
- **Download Services**:
  - Configs: `/home/YOUR_USER/docker/[service]:/config`
  - Downloads: `/media/downloads:/downloads`
  - Media: `/media/[type]:/[type]`

## Nextcloud Database Setup
When setting up Nextcloud, use these database details:
- **Database user**: `nextcloud`
- **Database password**: `YOUR_DB_PASSWORD` (replace with secure password)
- **Database name**: `nextcloud`
- **Database host**: `mariadb:3306`
- The `mariadb` service name is used as the host because both containers are on the same Docker network.

## Tailscale & LXC Containers
Tailscale works best in **privileged** LXC containers. Add these lines to your LXC container configuration:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
features: keyctl=1,nesting=1
```

## Storage Customization: Mounting HDDs for Media & Data

### Mounting HDD in Proxmox LXC
1. **Add disk to LXC container** in Proxmox web interface
2. **Mount in container**:
```bash
# Create mount point
sudo mkdir -p /media/movies /media/tvseries /media/downloads /storage

# Mount (replace /dev/sdX with your actual device)
sudo mount /dev/sdX1 /media/movies
sudo mount /dev/sdX2 /media/tvseries
sudo mount /dev/sdX3 /media/downloads
sudo mount /dev/sdX4 /storage

# Add to /etc/fstab for persistence
echo "/dev/sdX1 /media/movies ext4 defaults 0 0" | sudo tee -a /etc/fstab
echo "/dev/sdX2 /media/tvseries ext4 defaults 0 0" | sudo tee -a /etc/fstab
echo "/dev/sdX3 /media/downloads ext4 defaults 0 0" | sudo tee -a /etc/fstab
echo "/dev/sdX4 /storage ext4 defaults 0 0" | sudo tee -a /etc/fstab
```

### Example Docker Volume Paths
```yaml
volumes:
  - /home/YOUR_USER/docker/jellyfin/config:/config    # SSD for configs
  - /media/movies:/data/movies                      # HDD for bulk data
```

## Helpful Resources

These YouTube videos were instrumental in setting up this homelab:

- **[Proxmox Setup Guide](https://www.youtube.com/watch?v=qmSizZUbCOA&t=614s)** - Comprehensive guide for setting up Proxmox VE
- **[Nextcloud Installation](https://www.youtube.com/watch?v=DFUmfHqQWyg&t=989s)** - Step-by-step Nextcloud setup with Docker
- **[Jellyfin Media Server](https://www.youtube.com/watch?v=twJDyoj0tDc&t=719s)** - Complete Jellyfin installation and configuration

## Maintenance & Backup

### Update Services
```bash
# Update individual services
docker-compose -f <service>.yml pull
docker-compose -f <service>.yml up -d
```

### Backup Important Data
```bash
# Backup configs (SSD)
sudo tar -czf backup-configs-$(date +%Y%m%d).tar.gz /home/YOUR_USER/docker/

# Backup media (HDD) - if needed
sudo tar -czf backup-media-$(date +%Y%m%d).tar.gz /media/ /storage/
```

### Logs
```bash
# View logs for specific service
docker-compose -f <service>.yml logs -f

# View all container logs
docker logs <container_name>
```

## Troubleshooting

### Common Issues
1. **Port conflicts**: Check if ports are already in use with `netstat -tulpn`
2. **Permission issues**: Ensure PUID/PGID match your user (1000:1000)
3. **Storage not accessible**: Verify HDD mounts with `df -h`
4. **Tailscale not working**: Ensure LXC container is privileged with proper config

### Useful Commands
```bash
# Check container status
docker ps -a

# Restart specific service
docker-compose -f <service>.yml restart

# View resource usage
docker stats

# Clean up unused images/containers
docker system prune -a
```

## Future Plans
- [ ] Add monitoring (Grafana/Prometheus)
- [ ] Implement automated backups
- [ ] Add reverse proxy (Nginx/Traefik)
- [ ] Set up SSL certificates
- [ ] Add more media automation tools