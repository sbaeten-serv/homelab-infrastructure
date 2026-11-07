This guide covers installing and configuring `zpbs-backup` (a ZFS backup wrapper for `proxmox-backup-client`), configuring API authentication to Proxmox Backup Server (PBS), setting dataset flags, automating backups with email alerts, and excluding specific folders.

## 1. Install `proxmox-backup-client`

Ensure the client package is installed on your Proxmox host:

```Bash
apt update
apt install -y proxmox-backup-client
```

## 2. Install `zpbs-backup`

1. Download the latest release `.deb` package from the [zpbs-backup GitHub Releases](https://github.com/ndemarco/zpbs-backup/releases) (check the version number!):
    ```Bash
    wget https://github.com/ndemarco/zpbs-backup/releases/download/v0.5.2/zpbs-backup_0.5.2_amd64.deb
    ```
    
2. Install the package via `apt`:
    ```Bash
    apt install ./zpbs-backup_0.5.2_amd64.deb
    ```

## 3. Configure PBS Authentication & Repository Settings

### 3.1 Create PBS API Token & Set Permissions

1. Log into your **Proxmox Backup Server (PBS)** Web GUI.
2. Navigate to **Access Control** -> **API Token** -> **Add**.
3. Create a token (e.g., User: `root@pam`, Token ID: `zpbs-backup`). **Save the generated secret key.**
4. Go to **Access Control** -> **Permissions** -> **Add**. 
    - **Path:** `/datastore/zfs-pool` (adjust for your datastore name)
    - **User/Token:** Select your newly created API token
    - **Role:** `DatastoreBackup`
5. Copy your **PBS Fingerprint** from the PBS Dashboard / Home page.

### 3.2 Create Configuration File

1. Create the configuration file:
    ```Bash
    nano /etc/zpbs-backup/pbs.conf
    ```
2. Add your environment credentials:
    ```Bash
    export PBS_REPOSITORY="root@pam!zpbs-backup@<PBS_IP_OR_HOSTNAME>:zfs-pool"
    export PBS_PASSWORD="489a0211-6acd-451a-9911" # EXAMPLE, CHANGE THIS!!!
    export PBS_FINGERPRINT="4b:5a:76:b8:37:8b:49:55:f8:bb:64:99:a1:6d:b2:0e:6f:3b:30:33:fa:c0:59:6c:9b:f4" # EXAMPLE, CHANGE THIS!!!
    export PBS_NAMESPACE="pve"
    ```
    
3. Test configuration and connectivity:
    ```Bash
    zpbs-backup show-config
    ```

## 4. Select ZFS Datasets for Backup

1. Set the backup flag on your target ZFS pool/datasets:
    ```Bash
    zpbs-backup set zpbs:backup=true local-zfs
    ```
2. Perform a dry run to verify which datasets are selected:
    ```Bash
    zpbs-backup run --dry-run
    ```

## 5. Configure Automated Backup Script

1. Create the backup script:
    ```Bash
    nano /usr/local/bin/run-pbs-backups.sh
    ```
2. Paste the execution script:
```Bash
#!/bin/bash
# ZPBS Backup Script with Failure Email Alerts

PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Configuration
RECIPIENT="your-email@example.com"
SUBJECT="[ALERT] ZPBS Backup Failed on $(hostname)"
LOG_FILE=$(mktemp)

# Run backup and capture stdout/stderr
if ! zpbs-backup run --force > "$LOG_FILE" 2>&1; then
    # Send email if the backup failed (non-zero exit code)
    mail -s "$SUBJECT" "$RECIPIENT" < "$LOG_FILE"
fi

# Clean up temporary log
rm -f "$LOG_FILE"
```

3. Make the script executable:
    ```Bash
    chmod +x /usr/local/bin/run-pbs-backups.sh
    ```
4. Configure Cron Automation:
    ```Bash
    crontab -e
    ```
    
    Add the schedule entry (runs daily at 02:00):
    ```Code snippet
    0 2 * * * /usr/local/bin/run-pbs-backups.sh >> /var/log/zpbs-backup.log 2>&1
    ```
    
3. Test the script manually in a restricted environment:
    ```Bash
    sudo env -i PATH=/usr/bin:/bin /usr/local/bin/run-pbs-backups.sh
    ```
    _(Note: For initial full backups, consider running inside `tmux` or `screen` if dataset size is large)._

## 6. Exclude Folders from Backup (`.pxarexclude`)

To exclude specific folders within a ZFS dataset from being backed up:

1. Create a `.pxarexclude` file in the **root directory of the dataset**:
    ```Bash
    nano /local-zfs/nextcloud-data/.pxarexclude
    ```
2. Define *relative* directory paths starting with `/` and ending with `/`:
    ```Plaintext
    /admin/files/Videos/
    ```
3. Save the file. `proxmox-backup-client` will automatically skip the excluded paths during subsequent runs.


> ### ⚠️ Legacy Backup Script Reference (Deprecated)
> 
> _The following legacy script managed container shutdowns and restarts during backups. It is kept here for historical reference only._
> 
> ```Bash
> #!/bin/bash
> set -e
> PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
> 
> CONTAINERS=(100 102 103)
> RUNNING_CONTAINERS=()
> EMAIL="your-email@example.com"
> TEMP_LOG="/tmp/pbs_backup_run.log"
> 
> for id in "${CONTAINERS[@]}"; do
>     if pct status "$id" | grep -q "status: running"; then
>         RUNNING_CONTAINERS+=("$id")
>     fi
> done
> 
> cleanup() {
>     local EXIT_CODE=$?
>     echo "$(date '+%Y-%m-%d %H:%M:%S') - Running cleanup and checking for errors..."
>     for id in "${RUNNING_CONTAINERS[@]}"; do
>         /usr/sbin/pct start $id || true
>     done
> 
>     if [ "${EXIT_CODE:-0}" -ne 0 ]; then
>         (
>           echo "The backup script failed on $(hostname) at $(date '+%Y-%m-%d %H:%M:%S')"
>           echo "-------------------------------------------"
>           echo "Last 10 lines of the log:"
>           echo "-------------------------------------------"
>           tail -n 10 "$TEMP_LOG"
>           echo "-------------------------------------------"
>         ) | mail -s "ALERT: Backup Failed on $(hostname)" "$EMAIL"
>     fi
>     rm -f "$TEMP_LOG"
> }
> trap cleanup EXIT
> 
> if [ ${#RUNNING_CONTAINERS[@]} -eq 0 ]; then
>     echo "$(date '+%Y-%m-%d %H:%M:%S') No target containers are running. Proceeding with backup..."
> else
>     echo "$(date '+%Y-%m-%d %H:%M:%S') Stopping running containers: ${RUNNING_CONTAINERS[*]}..."
>     for id in "${RUNNING_CONTAINERS[@]}"; do
>         pct stop "$id"
>     done
> fi
> 
> echo "$(date '+%Y-%m-%d %H:%M:%S') Starting zpbs-backup..."
> zpbs-backup run --force
> 
> echo "$(date '+%Y-%m-%d %H:%M:%S') Starting containers: ${CONTAINERS[*]}..."
> for id in "${RUNNING_CONTAINERS[@]}"; do
>     /usr/sbin/pct start $id
> done
> 
> echo "$(date '+%Y-%m-%d %H:%M:%S') Backup process complete."
> ```