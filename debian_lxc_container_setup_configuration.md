## 1. Create Debian LXC Container

### 1.1 Download Container Template

1. In the Proxmox Web GUI, select your **local** storage drive under your host node.
2. Click **CT Templates** -> **Templates**.
3. Search for and download the latest **Debian** container template.

### 1.2 Create the Container

1. Click **Create CT** in the top right corner of the Proxmox GUI.
2. Complete the setup wizard, ensuring the following settings:
   * **Network (IPv4):** Set to static. Use your chosen container IP with the CIDR suffix (e.g., `192.168.0.100/24`; ensure the IP is unique on your network).
   * **Gateway:** Set to your router's IP address (typically `192.168.0.1`).
   * **DNS Servers:** Set to `9.9.9.9 1.1.1.1` (required if the Proxmox host is running Tailscale).
   * Leave DNS domain as default.
3. Confirm settings and start the container upon completion.

---

## 2. Install Updates, Dependencies & Docker

### 2.1 Update System Packages

1. Check for package updates: `apt update`
2. Apply pending system upgrades: `apt upgrade -y`

### 2.2 Install Base Dependencies & Docker

1. Install essential system utilities: `apt install -y curl git sudo lxc`
2. Install Docker using the official automated script: `curl -fsSL https://get.docker.com | bash`

---

## 3. Configure User Permissions & Groups

### 3.1 Create User and Grant Privileges

1. Add the main user account: `adduser <username>`
2. Add user to the `sudo` administrative group: `usermod -aG sudo <username>`
3. Add user to the `docker` group to manage containers without root: `usermod -aG docker <username>`

---

## 4. Bind Mount ZFS Storage to Container

> **Note:** This step is optional and only necessary if the service you are spinning up requires large storage capacity.

### 4.1 Execute Host Dataset & Bind Mount Setup

1. In the **Proxmox host terminal**, adjust the values in brackets and run the configuration commands.
   *(Note: UID and container destination paths may vary depending on the service; example below is configured for Nextcloud)*

```bash
# 1. Create the ZFS Dataset on the host
zfs create [YourPool]/nextcloud-data

# 2. Configure the Bind Mount in LXC (Maps host dataset to container path)
pct set [CTID] -mp0 /[YourPool]/nextcloud-data,mp=/var/www/html/data
# (or edit the container config file: nano /etc/pve/lxc/101.conf)

# 3. Apply Host Permissions (Maps container internal UID to host storage)
chown -R 1000[NextcloudUID]:1000[NextcloudUID] /[YourPool]/nextcloud-data

# 4. Restart the container to apply changes
pct reboot [CTID]
```