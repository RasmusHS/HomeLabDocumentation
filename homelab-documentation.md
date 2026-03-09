# Home Lab Documentation

## Resume Summary

Self-hosted cloud infrastructure built on repurposed hardware, providing secure file storage, collaborative document editing, and network-wide ad-blocking for multiple users.

**Technologies:** Proxmox VE, Docker, LXC containers, Nextcloud 33, Collabora Online, PostgreSQL 16, Redis 7, Traefik v3.3 (reverse proxy), Cloudflare Tunnel (zero-trust remote access), AdGuard Home (DNS/ad-blocking), Tailscale (admin VPN), Linux (Debian), YAML, bash scripting.

**Key accomplishments:**

- Designed and deployed a multi-service containerized stack using Docker Compose inside a Proxmox LXC container, optimized for 8GB RAM
- Configured Traefik as a reverse proxy with automatic Docker service discovery and Host-based routing for multiple services
- Implemented zero-trust remote access via Cloudflare Tunnel, eliminating port forwarding and providing automatic HTTPS with Cloudflare-managed certificates
- Deployed AdGuard Home on a dedicated Raspberry Pi for network-wide DNS resolution and ad-blocking, serving all LAN devices
- Set up Tailscale mesh VPN with subnet routing for secure remote administration of Proxmox and Docker infrastructure
- Configured Nextcloud with PostgreSQL, Redis caching, and Collabora Online for a self-hosted Google Workspace alternative with collaborative editing
- Implemented security hardening: Cloudflare WAF, HSTS headers, two-factor authentication, automated backups with retention policies

---

## Architecture

### Network Layout

```
Internet
   │
   ├── Cloudflare Edge (HTTPS termination, WAF, DDoS protection)
   │      │
   │      └── Cloudflare Tunnel (QUIC) ──► cloudflared container
   │
   ├── Tailscale (WireGuard mesh VPN for admin access)
   │
   └── Home Network (192.168.1.0/24)
          │
          ├── TP-Link Router ─── 192.168.1.1 (gateway, DHCP)
          │
          ├── Proxmox Host (ThinkPad X1 Carbon Gen 6) ─── 192.168.1.10
          │      │
          │      └── LXC Container: docker-host ─── 192.168.1.11
          │             │
          │             ├── traefik (reverse proxy, port 80 + 8080)
          │             ├── nextcloud (cloud storage)
          │             ├── collabora (document editing)
          │             ├── postgres (database)
          │             ├── redis (cache)
          │             └── cloudflared (Cloudflare Tunnel)
          │
          └── Raspberry Pi 4 (8GB) ─── 192.168.1.12
                 └── AdGuard Home (DNS, ad-blocking)
```

### Hardware

| Device | Role | IP | Specs |
|--------|------|-----|-------|
| ThinkPad X1 Carbon Gen 6 | Proxmox host | 192.168.1.10 | i5-8250U, 8GB RAM, 1TB NVMe |
| Docker host (LXC CT 100) | All services | 192.168.1.11 | 4 cores, 3072MB RAM, 1024MB swap, 400GB disk |
| Raspberry Pi 4 | AdGuard Home | 192.168.1.12 | 8GB RAM, Raspberry Pi OS Lite 64-bit |

### Docker Networks

| Network | Purpose | Services |
|---------|---------|----------|
| proxy | External-facing traffic | traefik, nextcloud, collabora, cloudflared |
| backend | Internal database/cache | nextcloud, postgres, redis |

### DNS Flow

All LAN devices → AdGuard Home (192.168.1.12) → upstream DNS. AdGuard provides ad-blocking and a `*.rhscloud.com` wildcard rewrite to 192.168.1.11 is **not** configured (local traffic goes through Cloudflare Tunnel to avoid HTTP/HTTPS mismatch). A `*.home.lab` wildcard rewrite to 192.168.1.11 can be added for direct local access to Traefik.

### External Access

| URL | Service | Path |
|-----|---------|------|
| https://nextcloud.rhscloud.com | Nextcloud | Browser → Cloudflare → cloudflared → nextcloud:80 |
| https://collabora.rhscloud.com | Collabora | Browser → Cloudflare → cloudflared → collabora:9980 |
| https://192.168.1.10:8006 | Proxmox UI | Via Tailscale only |

---

## Recreation Guide

### Phase 1: Proxmox on ThinkPad

1. Flash Proxmox VE ISO to USB from proxmox.com/downloads
2. Boot ThinkPad from USB (F12 for boot menu). Requires USB-C to Ethernet adapter (no native Ethernet port on Gen 6)
3. Installer settings:
   - Filesystem: **ext4** (not ZFS — ZFS is too RAM-hungry for 8GB)
   - Hostname: `pve-thinkpad.home.lab`
   - Static IP: `192.168.1.10/24`
   - Gateway: `192.168.1.1`
   - DNS: `192.168.1.1`
4. Access web UI at `https://192.168.1.10:8006`
5. Run community post-install script (disables enterprise repos, removes subscription nag):

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/misc/post-pve-install.sh)"
```

6. Laptop-specific tweaks:

```bash
# Prevent lid-close suspend
sed -i 's/#HandleLidSwitch=suspend/HandleLidSwitch=ignore/' /etc/systemd/logind.conf
systemctl restart systemd-logind
```

7. Tune swappiness:

```bash
echo "vm.swappiness=30" >> /etc/sysctl.d/99-swappiness.conf
sysctl -p /etc/sysctl.d/99-swappiness.conf
```

### Phase 2: Docker Host LXC Container

1. Download Debian 12 template: Proxmox UI → local storage → CT Templates
2. Create CT 100:
   - Hostname: `docker-host`
   - Disk: 400GB on LVM-Thin
   - CPU: 4 cores
   - Memory: 3072MB, Swap: 1024MB
   - Network: Static 192.168.1.11/24, gateway 192.168.1.1, DNS 192.168.1.1
3. Before starting, edit the container config:

```bash
nano /etc/pve/lxc/100.conf
# Add: features: nesting=1,keyctl=1
```

4. Start container, then open console in Proxmox UI:

```bash
apt update && apt install -y openssh-server curl git
sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
systemctl restart sshd
```

5. Install Docker:

```bash
curl -fsSL https://get.docker.com | sh
```

6. Fix Docker API version for Traefik compatibility (Docker 29+ issue):

```bash
mkdir -p /etc/systemd/system/docker.service.d
cat > /etc/systemd/system/docker.service.d/min_api_version.conf << 'EOF'
[Service]
Environment="DOCKER_MIN_API_VERSION=1.24"
EOF
systemctl daemon-reload
systemctl restart docker
```

### Phase 3: Nextcloud Stack

1. Create directories and files:

```bash
mkdir -p /opt/stacks/nextcloud
cd /opt/stacks/nextcloud
```

2. Create the `.env` file (generate passwords with `openssl rand -base64 24`):

```
POSTGRES_DB=nextcloud
POSTGRES_USER=nextcloud
POSTGRES_PASSWORD=<generate>

REDIS_PASSWORD=<generate>

NEXTCLOUD_ADMIN_USER=admin
NEXTCLOUD_ADMIN_PASSWORD=<generate>

COLLABORA_ADMIN_USER=admin
COLLABORA_ADMIN_PASSWORD=<generate>
```

3. Create the HSTS config file:

```bash
echo 'Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"' > hsts.conf
```

4. Create the post-upgrade hook:

```bash
cat > post-upgrade.sh << 'EOF'
#!/bin/bash
php /var/www/html/occ maintenance:repair --include-expensive
php /var/www/html/occ db:add-missing-indices
EOF
chmod +x post-upgrade.sh
```

5. Create `docker-compose.yml` (see Configuration Files section below)

6. Deploy:

```bash
docker compose up -d
docker compose logs -f
```

7. After first startup, configure Nextcloud via occ:

```bash
# Set trusted proxies (use the Docker network CIDR range)
docker exec -u www-data nextcloud php occ config:system:set trusted_proxies 0 --value="172.16.0.0/12"

# Set phone region
docker exec -u www-data nextcloud php occ config:system:set default_phone_region --value="DK"

# Set maintenance window (3 AM UTC)
docker exec -u www-data nextcloud php occ config:system:set maintenance_window_start --type=integer --value=3

# Set server ID
docker exec -u www-data nextcloud php occ config:system:set serverid --value="1"
```

8. In Nextcloud web UI:
   - Apps → install "Two-Factor TOTP" → enable in Personal Settings → Security
   - Apps → install "Nextcloud Office"
   - Administration Settings → Nextcloud Office → Use your own server → `https://collabora.rhscloud.com`

### Phase 4: Cloudflare Tunnel

1. Register a domain through Cloudflare (e.g., rhscloud.com)
2. Cloudflare dashboard → SSL/TLS → set encryption mode to **Full**
3. SSL/TLS → Edge Certificates → enable **Always Use HTTPS** and **HSTS** (max-age 15552000, include subdomains)
4. Security → Settings → ensure Cloudflare managed ruleset is active
5. Cloudflare Zero Trust → Networks → Tunnels → Create a tunnel:
   - Name: `homelab`
   - Copy the tunnel token
   - Add public hostnames:
     - `nextcloud.rhscloud.com` → `http://nextcloud:80`
     - `collabora.rhscloud.com` → `http://collabora:9980`
6. Add the token to the `TUNNEL_TOKEN` field in `docker-compose.yml`
7. Redeploy: `docker compose down && docker compose up -d`

**Important:** If the tunnel stops passing HTTP requests despite showing as "Healthy" (QUIC data channel failure in LXC), delete the tunnel in Cloudflare, create a new one with a new token, update `docker-compose.yml`, and redeploy. The watchdog script (see below) handles transient failures automatically.

### Phase 5: AdGuard Home on Raspberry Pi

1. Flash Raspberry Pi OS Lite (64-bit) to SD card, enable SSH
2. Set static IP via NetworkManager:

```bash
sudo nmcli con mod "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 192.168.1.12/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns 1.1.1.1
sudo nmcli con up "Wired connection 1"
```

3. Install AdGuard Home:

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

4. Complete setup wizard at `http://192.168.1.12:3000`:
   - Admin Web Interface: All interfaces, port 80
   - DNS server: All interfaces, port 53
5. In AdGuard Home dashboard → Filters → DNS rewrites:
   - `*.home.lab` → `192.168.1.11` (optional, for direct local access)
6. Configure router DHCP to set primary DNS to `192.168.1.12`, secondary to `1.1.1.1`
7. On client devices, ensure DNS is set to DHCP (not manually overridden)

### Phase 6: Tailscale

1. On Proxmox host:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

If enterprise repos block apt, disable them first:

```bash
echo "Enabled: no" >> /etc/apt/sources.list.d/pve-enterprise.sources
echo "Enabled: no" >> /etc/apt/sources.list.d/ceph.sources
apt-get update
apt-get install -y tailscale
```

2. Enable IP forwarding:

```bash
cat > /etc/sysctl.d/99-tailscale.conf << 'EOF'
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
net.core.rmem_max = 7500000
net.core.wmem_max = 7500000
EOF
sysctl -p /etc/sysctl.d/99-tailscale.conf
```

3. Start Tailscale with subnet routing:

```bash
tailscale up --advertise-routes=192.168.1.0/24
```

4. Follow the authentication URL
5. In Tailscale admin console, approve the subnet route
6. Install Tailscale on phone and PC from tailscale.com/download

### Phase 7: Backups & Monitoring

Automated weekly backup of Docker host container (Proxmox host):

```bash
cat > /etc/cron.d/backup-docker-host << 'EOF'
0 4 * * 0 root vzdump 100 --mode snapshot --compress zstd --storage local --mailnotification failure --prune-backups keep-last=3
EOF
```

Tunnel watchdog (Docker host):

```bash
cat > /opt/stacks/nextcloud/tunnel-watchdog.sh << 'EOF'
#!/bin/bash
RESPONSE=$(curl -sI --max-time 10 https://nextcloud.rhscloud.com 2>/dev/null | head -1)
if echo "$RESPONSE" | grep -q "502\|503\|504"; then
    echo "$(date): Tunnel unhealthy, restarting cloudflared" >> /var/log/tunnel-watchdog.log
    cd /opt/stacks/nextcloud && docker restart cloudflared
fi
EOF
chmod +x /opt/stacks/nextcloud/tunnel-watchdog.sh

echo "*/5 * * * * root /opt/stacks/nextcloud/tunnel-watchdog.sh" > /etc/cron.d/tunnel-watchdog
```

---

## Configuration Files

### docker-compose.yml

```yaml
services:

  traefik:
    image: traefik:v3.3
    container_name: traefik
    restart: unless-stopped
    command:
      - "--api.dashboard=true"
      - "--api.insecure=true"
      - "--entrypoints.web.address=:80"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--log.level=WARN"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      proxy:

  postgres:
    image: postgres:16-alpine
    container_name: postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      backend:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 30s
      timeout: 10s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 128mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    networks:
      backend:
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

  nextcloud:
    image: nextcloud:33-apache
    container_name: nextcloud
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      POSTGRES_HOST: postgres
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      REDIS_HOST: redis
      REDIS_HOST_PASSWORD: ${REDIS_PASSWORD}
      NEXTCLOUD_ADMIN_USER: ${NEXTCLOUD_ADMIN_USER}
      NEXTCLOUD_ADMIN_PASSWORD: ${NEXTCLOUD_ADMIN_PASSWORD}
      NEXTCLOUD_TRUSTED_DOMAINS: "nextcloud.rhscloud.com nextcloud.home.lab 192.168.1.11"
      TRUSTED_PROXIES: "172.16.0.0/12"
      OVERWRITEPROTOCOL: "https"
      OVERWRITEHOST: "nextcloud.rhscloud.com"
      PHP_MEMORY_LIMIT: 512M
      PHP_UPLOAD_LIMIT: 10G
    volumes:
      - nextcloud_html:/var/www/html
      - nextcloud_data:/var/www/html/data
      - ./hsts.conf:/etc/apache2/conf-enabled/hsts.conf:ro
      - ./post-upgrade.sh:/docker-entrypoint-hooks.d/post-upgrade/post-upgrade.sh:ro
    networks:
      proxy:
        aliases:
          - nextcloud.home.lab
      backend:
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.nextcloud.rule=Host(`nextcloud.home.lab`)"
      - "traefik.http.routers.nextcloud.entrypoints=web"
      - "traefik.http.services.nextcloud.loadbalancer.server.port=80"
      - "traefik.http.middlewares.nextcloud-headers.headers.stsSeconds=15552000"
      - "traefik.http.middlewares.nextcloud-headers.headers.stsIncludeSubdomains=true"
      - "traefik.http.middlewares.nextcloud-redirectregex.redirectregex.permanent=true"
      - "traefik.http.middlewares.nextcloud-redirectregex.redirectregex.regex=https?://([^/]*)/\\.well-known/(?:card|cal)dav"
      - "traefik.http.middlewares.nextcloud-redirectregex.redirectregex.replacement=http://$${1}/remote.php/dav"
      - "traefik.http.routers.nextcloud.middlewares=nextcloud-headers,nextcloud-redirectregex"
      - "traefik.docker.network=proxy"
      - "traefik.http.routers.nextcloud-public.rule=Host(`nextcloud.rhscloud.com`)"
      - "traefik.http.routers.nextcloud-public.entrypoints=web"
      - "traefik.http.routers.nextcloud-public.service=nextcloud"
      - "traefik.http.routers.nextcloud-public.middlewares=nextcloud-headers,nextcloud-redirectregex"

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    command: tunnel run
    environment:
      TUNNEL_TOKEN: "<your-tunnel-token>"
    networks:
      proxy:

  collabora:
    image: collabora/code:latest
    container_name: collabora
    restart: unless-stopped
    environment:
      aliasgroup1: "https://nextcloud\\.rhscloud\\.com"
      server_name: "collabora.rhscloud.com"
      username: ${COLLABORA_ADMIN_USER}
      password: ${COLLABORA_ADMIN_PASSWORD}
      extra_params: "--o:ssl.enable=false --o:ssl.termination=true --o:num_prespawn_children=1 --o:user_interface.use_integration_theme=true --o:per_document.doc_background_color=16777215"
    cap_add:
      - MKNOD
      - SYS_ADMIN
    networks:
      proxy:
        aliases:
          - collabora.home.lab
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.collabora.rule=Host(`collabora.home.lab`)"
      - "traefik.http.routers.collabora.entrypoints=web"
      - "traefik.http.services.collabora.loadbalancer.server.port=9980"
      - "traefik.http.routers.collabora-public.rule=Host(`collabora.rhscloud.com`)"
      - "traefik.http.routers.collabora-public.entrypoints=web"
      - "traefik.http.routers.collabora-public.service=collabora"

volumes:
  postgres_data:
  redis_data:
  nextcloud_html:
  nextcloud_data:

networks:
  proxy:
    name: proxy
  backend:
    name: backend
```

### .env template

```
POSTGRES_DB=nextcloud
POSTGRES_USER=nextcloud
POSTGRES_PASSWORD=CHANGE_ME_generate_with_openssl_rand_base64_24

REDIS_PASSWORD=CHANGE_ME_generate_with_openssl_rand_base64_24

NEXTCLOUD_ADMIN_USER=admin
NEXTCLOUD_ADMIN_PASSWORD=CHANGE_ME_generate_with_openssl_rand_base64_24

COLLABORA_ADMIN_USER=admin
COLLABORA_ADMIN_PASSWORD=CHANGE_ME_generate_with_openssl_rand_base64_24
```

---

## Known Issues & Troubleshooting

### Cloudflare Tunnel QUIC failure in LXC

**Symptom:** Tunnel shows as "Healthy" in Cloudflare dashboard but HTTP requests return 502. Cloudflared logs show no incoming requests. Config updates still arrive (QUIC control channel works, data channel doesn't).

**Cause:** QUIC (UDP) is unreliable inside LXC containers. IP forwarding changes, host reboots, or network changes can silently break the UDP data path.

**Fix:** Delete the tunnel in Cloudflare Zero Trust dashboard, create a new one, update the `TUNNEL_TOKEN` in docker-compose.yml, redeploy.

**Mitigation:** The tunnel-watchdog cron job restarts cloudflared every 5 minutes if it detects 502/503/504 responses. This handles transient failures. For persistent failures, manual tunnel recreation is required.

### Docker API version mismatch (Traefik + Docker 29)

**Symptom:** Traefik error: "client version 1.24 is too old. Minimum supported API version is 1.44"

**Cause:** Docker 29 removed backward compatibility for old API versions. Traefik's Go client starts negotiation at 1.24.

**Fix:** Set `DOCKER_MIN_API_VERSION=1.24` as a systemd override for the Docker daemon (see Phase 2 step 6).

### Traefik 502 with multi-network containers

**Symptom:** Traefik returns 502 for Nextcloud, which is on both `proxy` and `backend` networks.

**Cause:** Traefik picks a random network to reach the container. If it picks `backend` (which Traefik isn't on), the connection fails.

**Fix:** Add label `traefik.docker.network=proxy` to the Nextcloud service.

### Nextcloud trusted_proxies overridden by environment variable

**Symptom:** `TRUSTED_PROXIES` in docker-compose.yml overrides the value set via `occ` on every container recreate.

**Fix:** Set the correct value in docker-compose.yml (`172.16.0.0/12` covers all Docker networks). If the proxy network subnet changes after a recreate, this CIDR still covers it.

### Nextcloud redirect loop behind Cloudflare

**Symptom:** Nextcloud endlessly redirects between HTTP and HTTPS, resulting in 502.

**Cause:** `OVERWRITEPROTOCOL` is set to `https` but `TRUSTED_PROXIES` doesn't include the proxy's IP, so Nextcloud ignores the `X-Forwarded-Proto: https` header.

**Fix:** Ensure `TRUSTED_PROXIES` covers the Docker network CIDR (`172.16.0.0/12`).
