*Reference: [https://github.com/nextcloud/all-in-one/discussions/5439](https://github.com/nextcloud/all-in-one/discussions/5439)*

## 1. Directory & Configuration Files Setup

### 1.1 Create Working Directory

1. **Create and enter directory:**

   ```
   mkdir nextcloud-aio && cd nextcloud-aio
   
   ```

### 1.2 Create Caddy Dockerfile (`Caddy.Dockerfile`)

1. **Create file:** `nano Caddy.Dockerfile`

2. **Paste configuration:**

```Dockerfile
FROM caddy:2.9.1-builder-alpine AS builder
RUN xcaddy build --with github.com/mholt/caddy-l4@87e3e5e2c7f986b34c0df373a5799670d7b8ca03

FROM caddy:2.9.1-alpine
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

### 1.3 Create Caddy Configuration (`Caddyfile`)

1. **Create file:** `nano Caddyfile`

2. **Paste configuration:**

```Caddyfile
{
    layer4 {
        127.0.0.1:3478 {
            route {
                proxy {
                    upstream nextcloud-aio-talk:3478
                }
            }
        }
        127.0.0.1:3479 {
            route {
                proxy {
                    upstream nextcloud-aio-talk:3479
                }
            }
        }
    }
}

https://{$NC_DOMAIN} {
    reverse_proxy nextcloud-aio-apache:11000 {
        header_up X-Forwarded-Proto "https"
        header_up Host {host}
    }
}

http://{$NC_DOMAIN} {
    reverse_proxy nextcloud-aio-apache:11000 {
        header_up X-Forwarded-Proto "http"
        header_up Host {host}
    }
}
```

### 1.4 Create Docker Compose Configuration (`compose.yml`)

1. **Create file:** `nano compose.yml`

2. **Paste configuration:** *(Ensure `NC_DOMAIN` matches your `.ts.net` domain and `TS_AUTH_KEY` contains your generated Tailscale key)*

```YAML
services:
  nextcloud-aio-mastercontainer:
    image: ghcr.io/nextcloud-releases/all-in-one:latest
    init: true
    restart: always
    container_name: nextcloud-aio-mastercontainer
    volumes:
      - nextcloud_aio_mastercontainer:/mnt/docker-aio-config
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - nextcloud-aio
    ports:
      - 0.0.0.0:8080:8080
    environment:
      APACHE_PORT: 11000
      APACHE_IP_BINDING: 127.0.0.1
      SKIP_DOMAIN_VALIDATION: true
      NEXTCLOUD_DATADIR: /var/www/html/data

  caddy:
    build:
      context: .
      dockerfile: Caddy.Dockerfile
    depends_on:
      tailscale:
        condition: service_healthy
    restart: unless-stopped
    environment:
      NC_DOMAIN: your-nextcloud.your-tailnet.ts.net
    volumes:
      - type: bind
        source: ./Caddyfile
        target: /etc/caddy/Caddyfile
      - type: volume
        source: caddy_certs
        target: /certs
      - type: volume
        source: caddy_data
        target: /data
      - type: volume
        source: caddy_config
        target: /config
      - type: volume
        source: tailscale_sock
        target: /var/run/tailscale/
        read_only: true
    network_mode: service:tailscale

  tailscale:
    image: tailscale/tailscale:stable
    environment:
      TS_HOSTNAME: nextcloud
      TS_AUTH_KEY: tskey-client-YOUR_OAUTH_KEY_HERE
      TS_EXTRA_ARGS: --advertise-tags=tag:nextcloud --reset
    init: true
    healthcheck:
      test: tailscale status --peers=false --json | grep 'Online.*true'
      start_period: 10s
      interval: 5s
      retries: 5
    restart: unless-stopped
    devices:
      - /dev/net/tun:/dev/net/tun
    volumes:
      - type: volume
        source: tailscale
        target: /var/lib/tailscale
      - type: volume
        source: tailscale_sock
        target: /tmp
    cap_add:
      - NET_ADMIN
    networks:
      - nextcloud-aio

volumes:
  nextcloud_aio_mastercontainer:
    name: nextcloud_aio_mastercontainer
  caddy_certs:
  caddy_config:
  caddy_data:
  tailscale:
  tailscale_sock:

networks:
  nextcloud-aio:
    name: nextcloud-aio
    driver: bridge
    enable_ipv6: false
    driver_opts:
      com.docker.network.driver.mtu: "1280"
      com.docker.network.bridge.host_binding_ipv4: "127.0.0.1"
      com.docker.network.bridge.enable_icc: "true"
      com.docker.network.bridge.default_bridge: "false"
      com.docker.network.bridge.enable_ip_masquerade: "true"
```

## 2. Launch Nextcloud AIO

### 2.1 Start Docker Stack

1. **Deploy containers:**

   ```
   docker compose up -d
   
   ```

2. **Access AIO Setup Interface:** Open `https://<CONTAINER_IP>:8080` in your web browser.

3. **Save Passphrase:** Copy and record the generated AIO setup passphrase in a password manager.

4. **Domain Setup:** Submit your configured domain (`your-nextcloud.your-tailnet.ts.net`) in the setup wizard.

5. **Configure Optional Services:** Select **OnlyOffice** as the only additional service to deploy.

## 3. Nextcloud Performance Optimizations (`config.php`)

### 3.1 Locate and Backup Configuration File

1. **Find Nextcloud Docker Volume:**

   ```
   docker volume ls
   
   ```

2. **Verify Mountpoint Path:**

   ```
   docker inspect nextcloud_aio_nextcloud | grep Mountpoint
   
   ```

3. **Backup existing config:**

   ```
   sudo cp /var/lib/docker/volumes/nextcloud_aio_nextcloud/_data/config/config.php /var/lib/docker/volumes/nextcloud_aio_nextcloud/_data/config/config.php.bak
   
   ```

### 3.2 Edit Configuration Settings

1. **Open configuration file:**

   ```
   sudo nano /var/lib/docker/volumes/nextcloud_aio_nextcloud/_data/config/config.php
   
   ```

2. **Set default phone region:** At the end of the file, ensure the closing section matches:

   ```PHP
     'default_phone_region' => 'BE',
   );
   ```

3. **Enable asynchronous previews:** Directly after `'enable_previews' => true,`, add:

   ```PHP
     'enable_async_previews' => true,
   ```

4. **Restart Nextcloud container:**

   ```
   docker restart nextcloud-aio-nextcloud
   
   ```

## 4. Post-Install Administration & Email Setup

### 4.1 Admin Account Security & Brevo Email Setup

1. **Update Admin Password:** Log into the Nextcloud Web UI and change the default admin password.

2. **Configure Brevo SMTP:**

   * Set **Send mode:** `SMTP`

   * Set **Encryption:** `STARTTLS` (Port `587`) or `SSL/TLS` (Port `465`)

   * Set **SMTP Server:** `smtp-relay.brevo.com`

   * Enter your **Brevo Login Email** and **SMTP Key** (generated under Brevo -> Settings -> SMTP & API).

### 4.2 Safe-to-Ignore Admin Security Warnings

The following security warnings occur because Nextcloud routes through Tailscale loopbacks and cannot reach itself directly. They can be safely ignored:

* Data directory protected

* JavaScript source map / modules support

* OCS provider resolving

* WebDAV endpoint & .well-known URLs

* Font file loading & HTTP headers