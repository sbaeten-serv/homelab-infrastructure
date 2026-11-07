This guide covers setting up Immich on an LXC container with ZFS bind mounts, Tailscale Serve proxying, standard machine learning, and data import using `immich-go`.

## 1. ZFS Storage & LXC Bind Mount Setup (Proxmox Host)

Create the ZFS storage datasets on the Proxmox host and map them to your LXC container.

```Bash
# 1. Create ZFS Datasets for photo uploads and database storage
zfs create [YourPool]/immich-photos
zfs create [YourPool]/immich-db

# 2. Configure Bind Mounts in LXC (Replace [CTID] and [YourPool])
pct set [CTID] -mp0 /[YourPool]/immich-photos,mp=/mnt/immich-photos
pct set [CTID] -mp1 /[YourPool]/immich-db,mp=/mnt/immich-db

# 3. Apply Host Permissions for Unprivileged LXC
# Root (0) inside LXC maps to UID 100000 on Host
# Postgres (UID 999 inside LXC) maps to UID 100999 on Host
chown -R 100000:100000 /[YourPool]/immich-photos
chown -R 100999:100999 /[YourPool]/immich-db

# 4. Restart the LXC Container
pct reboot [CTID]
```

## 2. Application Directory & Configuration Setup
### 2.1 Create Working Directory

Run the following commands inside your LXC container:
```Bash
mkdir -p ~/immich/tailscale/config ~/immich/tailscale/state
cd ~/immich
```

### 2.2 Create Tailscale Serve Configuration
Create the Tailscale proxy configuration file:

```Bash
cat << 'EOF' > tailscale/config/serve-config.json
{
  "TCP": {
    "443": { "HTTPS": true }
  },
  "Web": {
    "your-domain.your-tailnet.ts.net:443": {
      "Handlers": {
        "/": { "Proxy": "http://127.0.0.1:2283" }
      }
    }
  }
}
EOF
```

### 2.3 Create Docker Compose File (`docker-compose.yml`)
Create `docker-compose.yml` inside `~/immich`:

```Bash
cat << 'EOF' > docker-compose.yml
name: immich

services:
  tailscale-immich:
    image: tailscale/tailscale:latest
    container_name: tailscale-immich
    hostname: immich
    environment:
      - TS_AUTHKEY=your_tailscale_auth_key
      - TS_EXTRA_ARGS=--advertise-tags=tag:immich --accept-routes --reset
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
      - immich-net
    restart: unless-stopped

  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:release
    volumes:
      - /mnt/immich-photos:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    environment:
      - DB_PASSWORD=your_secure_password
      - DB_USERNAME=postgres
      - DB_DATABASE_NAME=immich
      - DB_HOSTNAME=immich_postgres
      - REDIS_HOSTNAME=immich_redis
    depends_on:
      - tailscale-immich
      - redis
      - database
    network_mode: "service:tailscale-immich"
    restart: unless-stopped

  immich-machine-learning:
    container_name: immich_machine_learning
    image: ghcr.io/immich-app/immich-machine-learning:release
    volumes:
      - model-cache:/cache
    environment:
      - DB_PASSWORD=your_secure_password
    networks:
      - immich-net
    restart: unless-stopped

  redis:
    container_name: immich_redis
    image: docker.io/valkey/valkey:9
    restart: unless-stopped
    networks:
      - immich-net

  database:
    container_name: immich_postgres
    image: ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0
    environment:
      - POSTGRES_PASSWORD=your_secure_password
      - POSTGRES_USER=postgres
      - POSTGRES_DB=immich
      - POSTGRES_INITDB_ARGS=--data-checksums
    volumes:
      - /mnt/immich-db:/var/lib/postgresql/data
    networks:
      - immich-net
    restart: unless-stopped

volumes:
  model-cache:

networks:
  immich-net:
    driver: bridge
EOF
```

### 2.4 Deploy Stack

Start the containers in detached mode:
```Bash
sudo docker compose up -d
```

## 3. Bulk Data Import (`immich-go`)

Use `immich-go` to upload external archives (such as Google Takeout exports) directly to your Immich instance.

1. Download and extract the latest version of `immich-go` from [Github](https://github.com/simulot/immich-go).
2. Open a terminal inside the extracted directory.
3. Run the import command:

```PowerShell
.\immich-go.exe upload from-google-photos `
  --server=http://<tailscale-ip>:2283 `
  --api-key=YOUR_API_KEY `
  --concurrent-tasks=12 `
  --manage-raw-jpeg=StackCoverRaw `
  --manage-burst=Stack `
  --manage-heic-jpeg=StackCoverJPG `
  --on-errors=continue `
  "C:\path\to\your\takeout-*.zip"
```

_Note: If uploading to a non-admin account, append `--admin-api-key=YOUR_ADMIN_KEY` to the command._