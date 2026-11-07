This guide covers setting up ownCloud Infinite Scale (oCIS) using custom Docker Compose containers alongside Caddy and Tailscale Serve for reverse proxy routing.

## 1. LXC Storage & Dataset Setup (Proxmox Host)

If you are hosting oCIS inside an unprivileged LXC container with ZFS storage, create the storage dataset and configure the bind mount on the **Proxmox host** first.

```
# 1. Create the ZFS Dataset on the host
zfs create local-zfs/ocis-data

# 2. Configure the Bind Mount in LXC (Maps host dataset to container storage; replace [CTID])
pct set [CTID] -mp0 /local-zfs/ocis-data,mp=/var/lib/ocis

# 3. Apply Host Permissions for Unprivileged LXC (Maps UID 1000 in CT to UID 1001000 on Host)
chown -R 1001000:1001000 /local-zfs/ocis-data

# 4. Restart the LXC Container to apply changes
pct reboot [CTID]

```

## 2. Directory & Configuration Files Setup

### 2.1 Create Working Directory Structure

Run the following commands inside your LXC container to create the application directories:

```
mkdir -p ~/owncloud/ocis-config \
         ~/owncloud/tailscale/config \
         ~/owncloud/tailscale/state \
         ~/owncloud/caddy/data \
         ~/owncloud/caddy/config

cd ~/owncloud

```

### 2.2 Create Tailscale Serve Configuration

1. **Create `serve-config.json`:**

   ```
   nano tailscale/config/serve-config.json
   
   ```

2. **Paste configuration:**

```
{
  "TCP": {
    "443": {
      "HTTPS": true,
      "HTTP": false
    }
  },
  "Web": {
    "your-owncloud.your-tailnet.ts.net:443": {
      "Handlers": {
        "/": {
          "Proxy": "http://caddy:80"
        }
      }
    }
  }
}

```

### 2.3 Create Caddyfile

1. **Create `Caddyfile`:**

   ```
   nano caddy/Caddyfile
   
   ```

2. **Paste configuration:**

```
http://your-owncloud.your-tailnet.ts.net:80, https://your-owncloud.your-tailnet.ts.net:443 {
    tls internal

    reverse_proxy ocis:9200 {
        header_up Host {host}
        header_up X-Forwarded-Host {host}
        header_up X-Forwarded-Proto https
        header_up X-Forwarded-For {remote_host}
    }
}

```

### 2.4 Create Docker Compose File (`compose.yml`)

1. **Create `compose.yml`:**

   ```
   nano compose.yml
   
   ```

2. **Paste configuration:**

```
services:
  ocis:
    image: owncloud/ocis:latest
    container_name: ocis
    environment:
      - OCIS_URL=https://your-owncloud.your-tailnet.ts.net
      - OCIS_OIDC_ISSUER=https://your-owncloud.your-tailnet.ts.net
      - OCIS_INSECURE=true
      - PROXY_TLS=false
      - PROXY_HTTP_ADDR=0.0.0.0:9200
      - OWNCLOUD_LICENSE_ACCEPTED=true
      - OCIS_LOG_PRETTY=true
      - IDP_ACCESS_TOKEN_EXPIRATION=2629800
      - GOMEMLIMIT=3500MiB
# --- PosixFS Driver Settings ---
      - STORAGE_USERS_DRIVER=posix
      - STORAGE_USERS_POSIX_ROOT=/var/lib/ocis/storage
      - STORAGE_USERS_POSIX_WATCH_TYPE=poll
      - STORAGE_USERS_POSIX_POLL_INTERVAL=60s
      - STORAGE_USERS_ID_CACHE_STORE=nats-js-kv
      - STORAGE_USERS_POSIX_LOCK_TIMEOUT=100ms
      - STORAGE_USERS_POSIX_LOCK_RETRIES=30
    volumes:
      - ./ocis-config:/etc/ocis
      - /var/lib/ocis:/var/lib/ocis
    restart: unless-stopped
    networks:
      owncloud-net:
        aliases:
          - your-owncloud.your-tailnet.ts.net

  tailscale-owncloud:
    image: tailscale/tailscale:latest
    container_name: tailscale-owncloud
    hostname: owncloud
    environment:
      - TS_AUTHKEY=tskey-client-YOUR_OAUTH_KEY_HERE
      - TS_EXTRA_ARGS=--advertise-tags=tag:owncloud --accept-routes --reset
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_SERVE_CONFIG=/config/serve-config.json
    volumes:
      - ./tailscale/state:/var/lib/tailscale
      - ./tailscale/config:/config
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    restart: unless-stopped
    networks:
      - owncloud-net

  caddy:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile
      - ./caddy/data:/data
      - ./caddy/config:/config
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - ocis
      - tailscale-owncloud
    networks:
      owncloud-net:
        aliases:
          - your-owncloud.your-tailnet.ts.net

networks:
  owncloud-net:
    driver: bridge

```

## 3. Initialize & Deploy ownCloud Infinite Scale

### 3.1 Run Non-Interactive Initialization

Before starting the stack, oCIS must generate its initial configuration, certificates, and secrets.

```
# Clean existing data (if re-initializing)
docker compose down
sudo rm -rf ./ocis-config/* /var/lib/ocis/*
    
# Generate Config files
docker run --rm -v $(pwd)/ocis-config:/etc/ocis owncloud/ocis:latest init -insecure true

# Fix configuration permissions
sudo chown -R 1000:1000 ./ocis-config

```

### 3.2 Launch the Docker Stack

1. **Start all services:**

   ```
   docker compose up -d
   
   ```

2. **Retrieve Admin Credentials:**
   Check the oCIS initialization logs or startup logs to retrieve your initial admin password:

   ```
   docker compose logs ocis | grep -i password
   
   ```

3. **Access Web Interface:** Open `https://your-owncloud.your-tailnet.ts.net` in your browser.