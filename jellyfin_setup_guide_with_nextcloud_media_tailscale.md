This guide covers setting up Jellyfin on an LXC container using Nextcloud media directories via ZFS ACL permissions, alongside Tailscale Serve for reverse proxying.

## 1. ZFS Storage & LXC Bind Mount Setup (Proxmox Host)

Mount the Nextcloud media directory into your LXC container and configure POSIX ACL permissions to allow Jellyfin read/write access without altering Nextcloud user ownership.

```
# 1. Mount the folder to the LXC container (Replace [CTID] and [username])
pct set [CTID] -mp0 /local-zfs/nextcloud-data/[username]/files/Music,mp=/media/music
pct set [CTID] -mp1 /local-zfs/nextcloud-data/[username]/files/Videos,mp=/media/videos

# 2. Enable ACL support on the ZFS dataset (run once on the Proxmox Host)
apt install acl -y
zfs set acltype=posixacl local-zfs/nextcloud-data
zfs set xattr=sa local-zfs/nextcloud-data

# 3. Grant LXC user permissions using ACL
# Note: UID 1000 inside an unprivileged LXC maps to Host UID 101000
setfacl -R -m u:101000:rwx /local-zfs/nextcloud-data/[username]/files
setfacl -R -m d:u:101000:rwx /local-zfs/nextcloud-data/[username]/files

```

## 2. Directory & Configuration Setup

### 2.1 Create Working Directory

Run the following commands inside your LXC container:

```
mkdir -p ~/jellyfin/tailscale/config \
         ~/jellyfin/tailscale/state \
         ~/jellyfin/config \
         ~/jellyfin/cache

cd ~/jellyfin

```

### 2.2 Create Tailscale Serve Configuration

Create the proxy configuration file:

```
cat << 'EOF' > tailscale/config/serve-config.json
{
  "TCP": {
    "443": { "HTTPS": true }
  },
  "Web": {
    "your-jellyfin.your-tailnet.ts.net:443": {
      "Handlers": {
        "/": {
          "Proxy": "http://127.0.0.1:8096"
        }
      }
    }
  }
}
EOF

```

### 2.3 Create Docker Compose File (`compose.yml`)

Create `compose.yml` inside `~/jellyfin`:

```
cat << 'EOF' > compose.yml
name: jellyfin

services:
  tailscale-jellyfin:
    image: tailscale/tailscale:latest
    container_name: tailscale-jellyfin
    hostname: jellyfin # Result: your-jellyfin.your-tailnet.ts.net
    environment:
      - TS_AUTHKEY=
      - TS_EXTRA_ARGS=--advertise-tags=tag:jellyfin --accept-routes --reset
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_SERVE_CONFIG=/config/serve-config.json
    volumes:
      - ./tailscale/state:/var/lib/tailscale
      - ./tailscale/config:/config
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    networks:
      - jellyfin-net
    restart: unless-stopped

  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin-server
    network_mode: service:tailscale-jellyfin
    restart: unless-stopped
    environment:
      - JELLYFIN_UID=1000
      - JELLYFIN_GID=1000
    volumes:
      - ./config:/config
      - ./cache:/cache
      - /media/music:/media/music
      - /media/videos:/media/videos
    depends_on:
      - tailscale-jellyfin

networks:
  jellyfin-net:
    driver: bridge
EOF

```

## 3. Deploy & Troubleshooting

### 3.1 Start the Stack

```
docker compose up -d

```

### 3.2 Troubleshooting HTTPS / Tailscale Certificates

If Tailscale Serve reverts to plain `http` instead of securing the connection via `https`, issue a certificate request and restart the stack:

```
# Request certificate inside the Tailscale container
docker exec tailscale-jellyfin tailscale cert your-jellyfin.your-tailnet.ts.net

# Restart the stack
docker compose down && docker compose up -d

```