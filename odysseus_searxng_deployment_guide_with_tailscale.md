This guide covers deploying Odysseus (AI research tool) and SearXNG (metasearch engine) using custom Docker Compose configurations proxied through Tailscale Serve.

## 1. Debian LXC Preparation & Tailscale Prerequisites

1. **Create and prepare the Debian LXC Container:**
   * Download the latest Debian CT template in Proxmox.
   * Configure static IPv4 networking with router gateway and set DNS servers to `9.9.9.9` and `1.1.1.1`.

2. **Install Updates, Dependencies, and Docker:**

   ```
   apt update && apt upgrade -y
   apt install -y curl git sudo lxc
   curl -fsSL https://get.docker.com | bash
   
   ```

3. **Configure User Permissions:**

   ```
   adduser <username>
   usermod -aG sudo <username>
   usermod -aG docker <username>
   
   ```

4. **Generate Tailscale OAuth Key & Tag:**
   * Ensure `tag:odysseus` is added to your Tailscale ACLs.
   * Generate an OAuth key with write permissions for `tag:odysseus`.

## 2. Directory Structure & Repository Setup

Run the following commands inside the container to create the directory structure and clone the repository:

```
# Create working directory and clone repository
mkdir -p ~/odysseus && cd ~/odysseus
git clone https://github.com/pewdiepie-archdaemon/odysseus.git .

# Create Tailscale and SearXNG configuration directories
mkdir -p tailscale/state tailscale/config searxng

```

## 3. Configuration Files Setup

### 3.1 Create Tailscale Serve Configuration

Create `tailscale/config/serve-config.json`:

```
cat << 'EOF' > tailscale/config/serve-config.json
{
  "TCP": {
    "443": { "HTTPS": true }
  },
  "Web": {
    "your-odysseus.your-tailnet.ts.net:443": {
      "Handlers": {
        "/": { "Proxy": "http://odysseus-app:7000" }
      }
    }
  }
}
EOF

```

### 3.2 Create SearXNG Configuration

Create `searxng/settings.yml`:

```
cat << 'EOF' > searxng/settings.yml
use_default_settings: true

server:
  port: 8080
  bind_address: "0.0.0.0"
  secret_key: "YOUR_GENERATED_RANDOM_SECRET_KEY" # Replace with a random secret string

search:
  safe_search: 0
  formats:
    - html
    - json # Vital: Odysseus requires JSON output format
EOF

```

### 3.3 Create Docker Compose File (`docker-compose.yml`)

Create `docker-compose.yml` in `~/odysseus`:

```
cat << 'EOF' > docker-compose.yml
networks:
  odysseus-net:
    driver: bridge

services:
  odysseus-app:
    build:
      context: ./odysseus
      dockerfile: Dockerfile
      args:
        - INSTALL_OPTIONAL=true # Enables PDF & Office extractions
    container_name: odysseus-app
    restart: unless-stopped
    ports:
      - "7000:7000"
    environment:
      - APP_BIND=0.0.0.0       # Tells Odysseus to listen across the Docker network
      - APP_PORT=7000
      - AUTH_ENABLED=true
      - ODYSSEUS_ADMIN_PASSWORD= # add password
    volumes:
      - ./odysseus-data:/app/data
    networks:
      - odysseus-net
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

  tailscale-odysseus:
    image: tailscale/tailscale:latest
    container_name: tailscale-odysseus
    hostname: odysseus
    environment:
      - TS_AUTHKEY=tskey-client-YOUR_OAUTH_KEY_HERE
      - TS_EXTRA_ARGS=--advertise-tags=tag:odysseus --accept-routes --reset
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
      - odysseus-net
    restart: unless-stopped

  searxng:
    image: searxng/searxng:latest
    container_name: searxng
    restart: unless-stopped
    networks:
      - odysseus-net
    volumes:
      - ./searxng:/etc/searxng:rw
    environment:
      - SEARXNG_BASE_URL=http://searxng:8080/
EOF

```

## 4. Initial Deployment & First Login

1. **Build and start the stack:**

   ```
   docker compose up -d --build
   
   ```

2. **Retrieve default admin password:**

   ```
   docker compose logs odysseus-app | grep -i "password"
   
   ```

3. **Access Web Interface:**
   * Open `https://your-odysseus.your-tailnet.ts.net` in your browser.
   * Log in using username `admin` and the password retrieved from the logs.
   * **Change the default password immediately** under account settings.

4. **Connect SearXNG Search Engine:**
   * In Odysseus web UI settings, add SearXNG as a search provider.
   * Set the internal service URL to: `http://searxng:8080`

## 5. Updating Odysseus

To pull the latest code updates and rebuild the application:

```
cd ~/odysseus/odysseus
git pull
cd ..
docker compose down
docker compose up -d --build

```