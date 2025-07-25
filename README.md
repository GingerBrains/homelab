# Personal Ubuntu Server Container Setup

This guide documents the setup of my personal Ubuntu Server container in Proxmox, designed to host several self-hosted services for media, remote access, and network management.

## Purpose

This container is configured to run:
- **Nextcloud** (personal cloud storage)
- **Tailscale** (remote access VPN)
- **Pi-hole** (network-wide ad blocking)
- **Jellyfin** (media streaming)
- **qBittorrent** (torrent client)
- **Radarr** (movie downloader)
- **Sonarr** (TV show downloader)
- **Prowlarr** (indexer manager for Radarr/Sonarr)

---

## System Details

- **OS:** Ubuntu Server (version: `____`)
- **Proxmox Container Type:** LXC
- **Resources:** CPU: `____`, RAM: `____`, Storage: `____`
- **Hostname:** `____`
- **Network:** Static IP: `192.168.0.110`

---

## Network Configuration

- **Static IP** is configured for reliable access and port forwarding.
- **Tailscale** is used for secure remote access.
- **Pi-hole** is set as the DNS server for the local network.
- Ports are mapped as needed for external access (see below).

---

## Service Overview & Port Assignments

All services are dockerized using [linuxserver.io](https://www.linuxserver.io/) images for easy management and automatic startup when the container boots. Each service has its own `docker-compose` YAML file (see below for details).

| Service      | Host Port | Container Port | YAML File              |
|--------------|-----------|---------------|------------------------|
| Nextcloud    | 443       | 443           | nextcloud.yml          |
| MariaDB      | 3306      | 3306          | nextcloud.yml (service)|
| Jellyfin     | 8096      | 8096          | jellyfin.yml           |
|              | 8920      | 8920          | jellyfin.yml           |
|              | 7359/udp  | 7359/udp      | jellyfin.yml           |
|              | 1900/udp  | 1900/udp      | jellyfin.yml           |
| qBittorrent  | 8081      | 8080          | qbittorrent.yml        |
|              | 6881      | 6881          | qbittorrent.yml        |
|              | 6881/udp  | 6881/udp      | qbittorrent.yml        |
| Radarr       | 7878      | 7878          | radarr.yml             |
| Sonarr       | 8989      | 8989          | sonarr.yml             |
| Prowlarr     | 9696      | 9696          | prowlarr.yml           |
| Pi-hole      | 8053      | 53/udp,53/tcp | pihole.yml             |
|              | 8080      | 80            | pihole.yml             |
|              | 8443      | 443           | pihole.yml             |
| Tailscale    | (managed) | (managed)     | tailscale.yml (if any) |

- **Note:** Pi-hole's web interface is mapped to 8080 (host) to avoid conflict with Nextcloud's 443/HTTPS. Pi-hole's DNS is mapped to 8053 (host) to avoid conflicts. Jellyfin, qBittorrent, Radarr, Sonarr, and Prowlarr all use unique ports.
- All other services use their default ports unless otherwise specified.

---

## Installation Steps

### 1. Update & Upgrade
```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Core Utilities
```bash
sudo apt install curl wget git ufw -y
```

### 3. Install Tailscale
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```
_Follow the prompts to authenticate your device with your Tailscale account._

### 4. Install Docker (for app containers)
```bash
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install docker-ce docker-compose -y
```

### 5. Deploy Services

Each service has its own `docker-compose` YAML file. See the next section for the contents of each file. Start a service with:
```bash
sudo docker-compose -f <service>.yml up -d
```

---

## Tailscale & LXC Containers

For best results, run this container as a **privileged LXC** in Proxmox. Tailscale requires access to `/dev/net/tun` and certain kernel capabilities that are restricted in unprivileged containers.

- When creating the container, uncheck “Unprivileged container” in Proxmox.
- Add these lines to `/etc/pve/lxc/<CTID>.conf` (replace `<CTID>` with your container ID):
  ```
  lxc.cgroup2.devices.allow: c 10:200 rwm
  lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
  features: keyctl=1,nesting=1
  ```
- Restart the container after making changes.

**Security Note:** Privileged containers are less isolated from the host than unprivileged ones. For most homelab environments, this is an acceptable tradeoff for compatibility and ease of use.

If you must use an unprivileged container, additional configuration is required and some Tailscale features may not work.

---

## Docker Compose YAML Files

### nextcloud.yml
```yaml
version: "3.8"
services:
  nextcloud:
    image: lscr.io/linuxserver/nextcloud:latest
    container_name: nextcloud
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - /path/to/nextcloud/config:/config
      - /path/to/data:/data
    ports:
      - 443:443
    restart: unless-stopped
    depends_on:
      - db
  db:
    image: lscr.io/linuxserver/mariadb:latest
    container_name: mariadb
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - MYSQL_ROOT_PASSWORD=ROOT_ACCESS_PASSWORD
      - MYSQL_DATABASE=USER_DB_NAME #optional
      - MYSQL_USER=MYSQL_USER #optional
      - MYSQL_PASSWORD=DATABASE_PASSWORD #optional
      - REMOTE_SQL=http://URL1/your.sql,https://URL2/your.sql #optional
    volumes:
      - /path/to/mariadb/config:/config
    ports:
      - 3306:3306
    restart: unless-stopped
```

**Nextcloud Database Setup:**
- When installing Nextcloud, use the following database settings:
  - **Database user:** `MYSQL_USER` (from your environment)
  - **Database password:** `DATABASE_PASSWORD` (from your environment)
  - **Database name:** `USER_DB_NAME` (from your environment)
  - **Database host:** `mariadb:3306`
- The `mariadb` service name is used as the host because both containers are on the same Docker network.

### jellyfin.yml
```yaml
version: "3.8"
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - JELLYFIN_PublishedServerUrl=http://192.168.0.5 #optional
    volumes:
      - /path/to/jellyfin/library:/config
      - /path/to/tvseries:/data/tvshows
      - /path/to/movies:/data/movies
    ports:
      - 8096:8096
      - 8920:8920 #optional
      - 7359:7359/udp #optional
      - 1900:1900/udp #optional
    restart: unless-stopped
```

### qbittorrent.yml
```yaml
version: "3.8"
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - WEBUI_PORT=8080
    volumes:
      - /path/to/qbittorrent/config:/config
      - /path/to/downloads:/downloads
    ports:
      - 8081:8080
      - 6881:6881
      - 6881:6881/udp
    restart: unless-stopped
```

### radarr.yml
```yaml
version: "3.8"
services:
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - /path/to/radarr/data:/config
      - /path/to/movies:/movies #optional
      - /path/to/download-client-downloads:/downloads #optional
    ports:
      - 7878:7878
    restart: unless-stopped
```

### sonarr.yml
```yaml
version: "3.8"
services:
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - /path/to/sonarr/data:/config
      - /path/to/tvseries:/tv #optional
      - /path/to/downloadclient-downloads:/downloads #optional
    ports:
      - 8989:8989
    restart: unless-stopped
```

### prowlarr.yml
```yaml
version: "3.8"
services:
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - /path/to/prowlarr/data:/config
    ports:
      - 9696:9696
    restart: unless-stopped
```

### pihole.yml
```yaml
version: "3.8"
services:
  pihole:
    image: pihole/pihole
    container_name: pihole
    environment:
      - TZ=Etc/UTC
      - WEBPASSWORD=yourpassword
    volumes:
      - /path/to/pihole/etc-pihole:/etc/pihole
      - /path/to/pihole/etc-dnsmasq.d:/etc/dnsmasq.d
    ports:
      - 8053:53/tcp
      - 8053:53/udp
      - 8080:80
      - 8443:443
    cap_add:
      - NET_ADMIN
    restart: unless-stopped
```

---

## Scheduled Shutdown and Startup

Automate shutdown at 2:00 AM and startup at 8:00 AM using cronjobs:

1. Edit the root crontab:
   ```bash
   sudo crontab -e
   ```
2. Add these lines:
   ```cron
   0 2 * * * /sbin/shutdown -h now
   0 8 * * * /sbin/poweron  # (Replace with your method to start the container, e.g., via Proxmox task or script)
   ```
   _Note: Starting a container may require a Proxmox scheduled task or external script, as containers can't self-start when powered off._

---

## Accessing Services

- **Nextcloud:** `https://192.168.0.110` (port 443)
- **Jellyfin:** `http://192.168.0.110:8096`
- **qBittorrent:** `http://192.168.0.110:8081`
- **Radarr:** `http://192.168.0.110:7878`
- **Sonarr:** `http://192.168.0.110:8989`
- **Prowlarr:** `http://192.168.0.110:9696`
- **Pi-hole:** `http://192.168.0.110:8080` (web) or DNS on `8053`
- **Tailscale:** [Tailscale dashboard](https://login.tailscale.com/)

---

## Maintenance & Backup

- Regularly update containers:  
  `sudo docker-compose -f <service>.yml pull && sudo docker-compose -f <service>.yml up -d`
- Backup volumes (e.g., `/path/to/nextcloud/config`, `/path/to/data`, `/path/to/mariadb/config`, etc.)
- Monitor logs:  
  `sudo docker-compose -f <service>.yml logs -f`

---

## Troubleshooting

- Check container logs for errors.
- Ensure ports are not blocked by Proxmox or UFW.
- Verify Tailscale and Pi-hole are running and configured correctly.

---

## Future Plans
- [ ] Set up monitoring
- [ ] Expand storage for a NAS