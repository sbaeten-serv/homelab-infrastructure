*Prerequisite: The main sender machine must be on the same local network (subnet) as the target machine (no Tailscale network).*

## Step 1: BIOS/UEFI Configuration (Target Machine)

1. Boot into BIOS/UEFI (typically by pressing **F2**, **F12**, or **Del** at boot).
2. Enable **Wake-on-LAN**, **WOL**, or **Power On By PCI-E/PME**.
3. Disable **EuP/ErP Ready** or **Deep Sleep** (if present) to keep the Network Interface Card (NIC) powered while turned off.

## Step 2: Identify Network Details (Target Machine)

1. Find the target's physical interface name and MAC address:
    ```Bash
    ip a
    ```
2. Note down the physical interface (e.g., `enp1s0`) and MAC address (e.g., `30:65:ep:66:a2:02`).

## Step 3: Enable WoL in the OS (Target Machine)

1. Install `ethtool` and set Magic Packet mode:
    ```Bash
    apt update && apt install ethtool -y
    ethtool -s <interface> wol g
    ```
2. Verify:
    ```Bash
    ethtool <interface> | grep "Wake-on"
    ```
    _Expected output:_ `Wake-on: g`

## Step 4: Configure Persistence via Systemd Service (Target Machine)

> **Note:** Many laptops reset WoL to `d` (disabled) during boot/shutdown. Creating a shutdown service forces the NIC into `g` mode immediately before power loss.

1. Create the executable script:
    ```Bash
    nano /usr/local/bin/enable-wol.sh
    ```
    Paste the following content (replace `enp1s0` with your target interface name):
    ```Bash
    #!/bin/bash
    /usr/sbin/ethtool -s enp1s0 wol g
    ```
2. Make the script executable:
    ```Bash
    chmod +x /usr/local/bin/enable-wol.sh
    ```
3. Create the shutdown service unit:
    ```Bash
    nano /etc/systemd/system/wol-shutdown.service
    ```
    Paste the exact unit configuration:
    ```Ini, TOML
    [Unit]
    Description=Enable WoL at shutdown
    DefaultDependencies=no
    Before=shutdown.target reboot.target halt.target
    
    [Service]
    Type=oneshot
    ExecStart=/usr/local/bin/enable-wol.sh
    TimeoutStartSec=0
    
    [Install]
    WantedBy=shutdown.target reboot.target halt.target
    ```
4. Enable the service:
    ```Bash
    systemctl enable wol-shutdown.service
    ```

## Step 5: Configure the Sender (Main Server)

1. Install `wakeonlan`:
    ```Bash
    apt update && apt install wakeonlan -y
    ```
2. Add a persistent shortcut alias:
    ```Bash
    nano ~/.bashrc
    ```
    Add the following line at the bottom (replace `<MAC_ADDRESS>`):
    ```Bash
    alias boot-target='wakeonlan <MAC_ADDRESS>'
    ```
3. Apply changes:
    ```Bash
    source ~/.bashrc
    ```

## Step 6: Usage & Notes

- **Trigger Wake-up:** Send the Magic Packet by executing your alias on the sender machine:
    ```Bash
    boot-target
    ```
 
>Note: **AC Power Requirement:** Laptops usually cut power to the Ethernet port when running purely on battery to conserve charge. Ensure the target laptop remains plugged into wall power.