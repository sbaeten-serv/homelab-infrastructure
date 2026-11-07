This guide covers setting up Nextcloud using custom Docker Compose containers with Tailscale Serve as the reverse proxy.

## 1. Directory & Configuration Files Setup

### 1.1 Create Working Directory

1. **Create and enter directory:**
   ```
   mkdir nextcloud && cd nextcloud

   ```

### 1.2 Create Tailscale Directory & Serve File

1. **Create subfolder for Tailscale configurations:**

   ```
   mkdir tailscale

   ```

2. **Create `serve.json`:**

   ```
   sudo nano tailscale/serve.json

   ```

3. **Paste configuration:**

```
{
  "TCP": {
    "443": {
      "HTTPS": true
    }
  },
  "Web": {
    "${TS_CERT_DOMAIN}:443": {
      "Handlers": {
        "/": {
          "Proxy": "http://127.0.0.1:80"
        }
      }
    }
  }
}

```

### 1.3 Create Docker Compose File (`compose.yml`)

1. **Create file:**

   ```
   sudo nano compose.yml

   ```

2. **Paste configuration:**

```
services:
  db:
    image: mariadb:10.11
    container_name: nextcloud-db
    restart: unless-stopped
    command: --transaction-isolation=READ-COMMITTED --log-bin-trust-function-creators=1
    volumes:
      - db_data:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=your_secure_root_password
      - MYSQL_PASSWORD=your_nextcloud_db_password
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud

  redis:
    image: redis:alpine
    container_name: nextcloud-redis
    restart: unless-stopped

  tailscale:
    image: tailscale/tailscale:latest
    container_name: nextcloud-tailscale
    hostname: nextcloud
    environment:
      - TS_AUTHKEY=
      - TS_EXTRA_ARGS=--advertise-tags=tag:nextcloud --accept-routes --reset
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_SERVE_CONFIG=/config/serve.json
      - TS_USERSPACE=true
    volumes:
      - ./tailscale/state:/var/lib/tailscale
      - ./tailscale:/config
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    restart: unless-stopped

  app:
    image: nextcloud:apache
    container_name: nextcloud-app
    restart: unless-stopped
    depends_on:
      - db
      - redis
      - tailscale
    network_mode: service:tailscale
    environment:
      - MYSQL_HOST=db
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=your_nextcloud_db_password
      - REDIS_HOST=redis
      - NEXTCLOUD_DATA_DIR=/var/www/html/data
      - OVERWRITEPROTOCOL=https
      - OVERWRITECLIURL=https://your-nextcloud.your-tailnet.ts.net
      - NEXTCLOUD_DATADIR=/var/www/html/data
    volumes:
      - nextcloud_config:/var/www/html
      - /var/www/html/data:/var/www/html/data

volumes:
  db_data:
  nextcloud_config:
  tailscale_data:

```

## 2. Launch & Initialize Nextcloud

### 2.1 Start Docker Stack

1. **Deploy containers:**

   ```
   docker compose up -d

   ```

2. **Access Web Interface:** Open `https://your-nextcloud.your-tailnet.ts.net` in your browser.

3. **Complete Setup Wizard:** Create your admin user account. Database credentials will automatically populate from environment variables.

## 3. Post-Install Configuration & Maintenance (`config.php`)

To fix the critical reverse proxy IP header warning and optimize background maintenance tasks, apply a few configuration parameters directly to Nextcloud's configuration file.

### 3.1 Edit `config.php`

1. **Find & open `config.php` from your host directory:**

   ```
   docker volume inspect $(docker volume ls -q | grep nextcloud_config) --format '{{ .Mountpoint }}/config/config.php'
   sudo nano [the answer it just gave you]

   ```

2. **Add reverse proxy trust & maintenance window:** Inside the `$CONFIG = array (` array, add the following lines before the closing `);`:

   ```
     /* Trust Tailscale/Docker reverse proxy headers for accurate client IPs */
     'trusted_proxies' => array('127.0.0.1', '172.16.0.0/12', '10.0.0.0/8'),
   
     /* Schedule intensive background maintenance tasks to run at 01:00 UTC */
     'maintenance_window_start' => 1,
   
     /* Optional: Non-blocking background image preview generation */
     'enable_async_previews' => true,

   ```

3. **Restart the Nextcloud container to apply changes:**

   ```
   docker restart nextcloud-app

   ```

### 3.2 Safe-to-Ignore Security & Network Warnings

Because Nextcloud runs isolated inside Docker behind Tailscale, automated internal loopback HTTP requests will fail during self-checks. The following warnings under **Administration settings** -> **Overview** are harmless and can be safely ignored:

* **Loopback & Local Network Checks:** *Data directory protected*, *WebDAV endpoint*, *.well-known URLs*, *OCS provider resolving*, *Font file loading*, *HTTP headers*.
* **Development Tools:** *JavaScript source map / modules support*.
* **Optional Features:** *AppAPI deploy daemon*, *Email server configuration*, *Default phone region*, *Second factor configuration*.