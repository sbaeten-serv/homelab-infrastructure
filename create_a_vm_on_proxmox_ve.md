## 1. Create and Configure Ubuntu VM

### 1.1 Create the VM
1. **Launch Wizard:** In the Proxmox Web GUI, click the **Create VM** button in the top right corner.
2. **Configure General & OS Settings:** Give the VM a name and select your uploaded Ubuntu Server ISO image.
3. **Allocate Resources:** Assign the desired CPU cores, RAM, and disk size based on your workload needs, then finish the wizard and start the VM.

### 1.2 Enable SSH Service
1. **Open Console:** After the VM boots, open its **Console** tab in the Proxmox GUI.
2. **Start SSH Service:** Enable and start the SSH server: `sudo systemctl start ssh`

### 1.3 Fix SSH Permissions (If Required)
1. **Open SSH Configuration:** If you encounter a `Permission denied` error, open the SSH configuration file: `sudo nano /etc/ssh/sshd_config`
2. **Enable Password Authentication:** Ensure the following setting is enabled: `PasswordAuthentication yes`
3. **Restart Service:** Apply the changes by restarting the SSH service: `sudo systemctl restart ssh`

### 1.4 Access via SSH
1. **Connect from Laptop:** Open terminal on your laptop and SSH into the VM using its local IP address: `ssh your_username@192.168.0.123`