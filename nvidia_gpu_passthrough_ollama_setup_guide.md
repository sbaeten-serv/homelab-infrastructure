This guide covers installing NVIDIA drivers on both the Proxmox VE host and an unprivileged LXC container, configuring LXC GPU passthrough devices, installing the NVIDIA Container Toolkit for Docker, and deploying Ollama.

> **Important:** The NVIDIA driver version installed inside the LXC container **must exactly match** the driver version installed on the Proxmox host.

## 1. Proxmox Host Configuration & Driver Installation

### 1.1 Install Host NVIDIA Drivers & Dependencies

1. **Open Proxmox Host Terminal** and install required build dependencies:

   ```bash
   apt update && apt install -y pve-headers build-essential dkms
   ```

2. **Download NVIDIA Linux Driver on Host:**

   ```bash
   wget https://us.download.nvidia.com/XFree86/Linux-x86_64/595.84/NVIDIA-Linux-x86_64-595.84.run # change version number to most recent
   ```

3. **Make file executable & run installer:**

   ```bash
   chmod +x NVIDIA-Linux-x86_64-595.84.run
   ./NVIDIA-Linux-x86_64-595.84.run
   ```

4. **Verify Host Installation:**

   ```bash
   nvidia-smi
   ```

   *Verify that your GPU and driver version (e.g., `595.84`) are displayed correctly.*

### 1.2 Attach GPU Devices to LXC Container

1. **Stop the target LXC container:**

   ```bash
   pct stop [CTID]
   ```

2. **Edit LXC configuration on Proxmox Host:**

   ```bash
   nano /etc/pve/lxc/[CTID].conf
   ```

3. **Add device passthrough mappings** at the bottom of the configuration file:

   ```ini
   dev0: /dev/nvidia0
   dev1: /dev/nvidiactl
   dev2: /dev/nvidia-uvm
   dev3: /dev/nvidia-uvm-tools
   dev4: /dev/nvidia-caps/nvidia-cap1
   dev5: /dev/nvidia-caps/nvidia-cap2
   ```

4. **Start the LXC Container:**

   ```bash
   pct start [CTID]
   ```

## 2. LXC Container Setup & Ollama Deployment

### 2.1 Install Container NVIDIA Drivers (No Kernel Modules)

Since the LXC container shares the Linux kernel with the Proxmox host, you must install the NVIDIA driver **without** kernel modules inside the container.

1. **Access LXC Container Shell.**

2. **Download matching driver version inside LXC:**

   ```bash
   wget "https://us.download.nvidia.com/XFree86/Linux-x86_64/595.84/NVIDIA-Linux-x86_64-595.84.run" # append version if needed 
   ```

3. **Make file executable:**

   ```bash
   chmod +x NVIDIA-Linux-x86_64-595.84.run
   ```

4. **Execute driver installer without kernel modules:**

   ```bash
   sudo ./NVIDIA-Linux-x86_64-595.84.run --no-kernel-modules
   ```

   *Accept default prompt choices during installation. (MIT/GPL, yes, ...)*

5. **Verify LXC Driver Recognition:**

   ```bash
   nvidia-smi
   ```

### 2.2 Install NVIDIA Container Toolkit (For Docker GPU Access)

If you plan to pass the GPU into Docker containers running inside this LXC:

1. **Configure repository key and sources list:**

   ```bash
   curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
     && curl -s -L https://nvidia.github.io/libnvidia-container/experimental/deb/libnvidia-container.list | \
     sed 's#deb [^ ]*#& [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg]#g' | \
     sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
   ```

2. **Install toolkit and restart Docker:**

   ```bash
   sudo apt update && sudo apt install -y nvidia-container-toolkit
   sudo nvidia-ctk runtime configure --runtime=docker
   sudo systemctl restart docker
   ```

## 3. Install & Configure Ollama

### 3.1 Install Dependencies & Ollama

1. **Install required tools:**

   ```bash
   sudo apt-get install -y zstd lshw
   ```

2. **Run official Ollama automated installation script:**

   ```bash
   curl -fsSL https://ollama.com/install.sh | sh
   ```

### 3.2 Bind Ollama Service to Local Network

By default, Ollama only listens on `127.0.0.1` (localhost). Modify the service override to accept connections across your local network.

1. **Open systemd service editor:**

   ```bash
   sudo systemctl edit ollama.service
   ```

2. **Insert environment variable configuration:**

   ```ini
   [Service]
   Environment="OLLAMA_HOST=0.0.0.0"
   ```

3. **Reload daemon and restart Ollama:**

   ```bash
   systemctl daemon-reload
   systemctl restart ollama
   ```

### 3.3 Network Verification

You can now point web interfaces or APIs (like Open WebUI) to your container's IP address and Ollama default port:

`http://<LXC_IP_ADDRESS>:11434`