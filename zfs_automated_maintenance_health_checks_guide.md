This guide covers setting up automated Cron jobs for daily ZFS pool health monitoring and monthly dataset scrubbing on Proxmox.

## 1. Create the ZFS Health Check Script

1. **Create the script file:**

   ```
   nano /root/zfs_health_check.sh
   
   ```

2. **Add the script content:**

```
#!/bin/bash

# Configuration
EMAIL="your-email@example.com"
HOSTNAME=$(hostname)
LOG_FILE="/var/log/zfs-monitor/health_check.log"
LOG_MAX_SIZE=$((10 * 1024 * 1024))  # 10MB in bytes

# Email configuration
FROM_EMAIL="admin@example.com"
FROM_NAME="Admin Name"
REPLY_TO="Admin Name <admin@example.com>"
RETURN_PATH="admin@example.com"

# Create required directories
mkdir -p "$(dirname "$LOG_FILE")"

# Initialize error log
errors=""

# Logging functions
rotate_log() {
    if [ -f "$LOG_FILE" ] && [ $(stat -f%z "$LOG_FILE" 2>/dev/null || stat -c%s "$LOG_FILE") -gt "$LOG_MAX_SIZE" ]; then
        mv "$LOG_FILE" "$LOG_FILE.old"
    fi
}

log_message() {
    echo -e "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log_error() {
    local message="$1"
    errors="${errors}\n$message"
    log_message "ERROR: $message"
}

# Check overall pool status
check_pool_status() {
    while IFS= read -r pool; do
        status=$(zpool status "$pool")

        # Check for common failure keywords
        if echo "$status" | grep -E "DEGRADED|FAULTED|OFFLINE|UNAVAIL|REMOVED|FAIL|DESTROYED|SUSPENDED" > /dev/null; then
            log_error "ALERT: Pool $pool is not healthy:\n$status"
        fi

        # Check for errors
        if echo "$status" | grep -v "No known data errors" | grep -i "errors:" > /dev/null; then
            log_error "ALERT: Pool $pool has errors:\n$status"
        fi

        # Check scrub status
        if echo "$status" | grep "scan" | grep -E "scrub canceled|scrub failed" > /dev/null; then
            log_error "ALERT: Pool $pool has unusual scrub status:\n$(echo "$status" | grep "scan")"
        fi
    done < <(zpool list -H -o name)
}

# Check individual device status
check_devices() {
    while IFS= read -r pool; do
        devices=$(zpool status "$pool" | awk '/ONLINE|DEGRADED|FAULTED|OFFLINE|UNAVAIL|REMOVED/ {print $1,$2}')

        echo "$devices" | while read -r device state; do
            if [ "$state" != "ONLINE" ] && [ "$device" != "pool" ] && [ "$device" != "mirror" ] && [ "$device" != "raidz1" ] && [ "$device" != "raidz2" ]; then
                log_error "ALERT: Device $device in pool $pool is $state"
            fi
        done
    done < <(zpool list -H -o name)
}

# Check capacity threshold (80% by default)
check_capacity() {
    while IFS= read -r pool; do
        capacity=$(zpool list -H -p -o capacity "$pool")
        if [ "$capacity" -ge 80 ]; then
            log_error "WARNING: Pool $pool is ${capacity}% full"
        fi
    done < <(zpool list -H -o name)
}

# Check dataset properties
check_dataset_properties() {
    while IFS= read -r dataset; do
        # Skip base pools
        if ! echo "$dataset" | grep "/" > /dev/null; then
            continue
        fi

        # Check if compression is enabled
        compression=$(zfs get -H compression "$dataset" | awk '{print $3}')
        if [ "$compression" = "off" ]; then
            log_error "WARNING: Compression is disabled on dataset $dataset"
        fi

        # Check if dataset is mounted
        mounted=$(zfs get -H mounted "$dataset" | awk '{print $3}')
        if [ "$mounted" = "no" ]; then
            log_error "WARNING: Dataset $dataset is not mounted"
        fi

        # Check available space
        available=$(zfs get -H available "$dataset" | awk '{print $3}')
        if [ "$available" = "0" ] || [ "$available" = "0B" ]; then
            log_error "CRITICAL: Dataset $dataset has no available space"
        fi
    done < <(zfs list -H -o name)
}

# Function to send email
send_email() {
    local subject="$1"
    local content="$2"
    
    {
        echo "Subject: $subject"
        echo "To: ${EMAIL}"
        echo "From: ${FROM_NAME} <${FROM_EMAIL}>"
        echo "Reply-To: ${REPLY_TO}"
        echo "Return-Path: ${RETURN_PATH}"
        echo "Content-Type: text/plain; charset=UTF-8"
        echo
        echo "$content"
    } | /usr/sbin/sendmail -t -f "${FROM_EMAIL}"
}

# Main execution
rotate_log
log_message "Starting ZFS health check"

# Run all checks
check_pool_status
check_devices
check_capacity
check_dataset_properties

# Send notification if there are errors
if [ -n "$errors" ]; then
    log_message "Issues detected - sending email alert"
    subject="Storage Alert: Issues Detected on ${HOSTNAME}"
    content=$(echo -e "ZFS Health Monitor Report from ${HOSTNAME}\n\nThe following issues were detected:${errors}")
    send_email "$subject" "$content"
else
    log_message "All ZFS checks passed successfully"
fi

```

3. **Make the script executable:**

   ```
   chmod +x /root/zfs_health_check.sh
   
   ```

## 2. Configure Crontab

1. **Open the system crontab editor:**

   ```
   crontab -e
   
   ```

2. **Add the schedule entries at the bottom of the file:**

   ```
   # ZFS Health Check - Run daily at 07:00
   0 7 * * * /root/zfs_health_check.sh >/dev/null 2>&1
   
   # ZFS scrub - Run on the first Sunday of the month at 02:00
   0 2 1-7 * * [ $(date +\%u) -eq 7 ] && /sbin/zpool scrub local-zfs
   
   ```

3. **Save and exit**