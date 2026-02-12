# Proxmox Offsite Backup Documentation

**Last Updated:** February 12, 2026
**Server:** `dragonpx` (Proxmox Host)

## 1. System Overview

* **Goal:** Encrypted offsite backup of critical data (Photos, Docker, Airbnb configs) and VM snapshots to Google Drive.
* **Tools:**
* **Restic:** Handles encryption, deduplication, and versioning.
* **Rclone:** Handles the transport to Google Drive.


* **Storage Location:** Google Drive folder `ProxmoxBackups/ResticRepo`
* **Critical Password:** `Able1593.` (REQUIRED to restore data. **Do not lose this.**)

## 2. Paths & Configs

| Item | Path / Value |
| --- | --- |
| **Rclone Config** | `/root/.config/rclone/rclone.conf` |
| **Restic Repo** | `rclone:gdrive:ProxmoxBackups/ResticRepo` |
| **Source Data** | `/sata_1tb/nfs_share` (Docker, Photos) |
| **VM Dump Path** | `/var/lib/vz/dump` |
| **VM ID** | `110` (Docker VM) |

---

## 3. The Scripts

These are the two scripts currently running your backup pipeline.

### Script A: VM Snapshot Preparer

**File:** `/root/cloud-backup.sh`
**Purpose:** Creates a local compressed snapshot of the VM so Restic can upload it.

```bash
#!/usr/bin/env bash
# --- VM PREPARATION ONLY ---
VMID=110
LOCAL_DUMP="/var/lib/vz/dump"

echo "Creating local Snapshot for VM $VMID..."
# Uses absolute path to ensure Cron finds the command
/usr/sbin/vzdump $VMID --storage local --mode snapshot --compress zstd

echo "VM snapshot complete. Restic will handle the upload next."

```

### Script B: The Master Restic Backup

**File:** `/root/restic-backup.sh`
**Purpose:** Encrypts Photos, Docker files, and the VM Snapshot, then uploads to Google Drive.
**Key Features:** Includes a "Speed Limit" to prevent Google API bans and excludes the massive Jellyfin folder.

```bash
#!/bin/bash

# --- CONFIGURATION ---
export RCLONE_CONFIG="/root/.config/rclone/rclone.conf"
export RESTIC_REPOSITORY="rclone:gdrive:ProxmoxBackups/ResticRepo"
export RESTIC_PASSWORD="Able1593."

# Speed Limit to prevent "Quota Exceeded (403)" errors from Google
# Limits upload to 3 transactions per second
LIMITS="--option rclone.args='--tpslimit 3 --tpslimit-burst 3'"

NFS_PATH="/sata_1tb/nfs_share"
VM_DUMP="/var/lib/vz/dump"

# --- HELPER FUNCTION ---
safe_backup() {
    # $1 = Folder Path, $2 = Tag, $3 = Extra Args (Excludes)
    if [ -d "$1" ] || [ -f "$1" ]; then
        echo "Backing up $1..."
        restic backup "$1" --tag "$2" $LIMITS $3
    else
        echo "WARNING: Path $1 not found. Skipping $2."
    fi
}

echo "--- Starting Speed-Limited Backup ---"

# 1. Backup Photos
safe_backup "$NFS_PATH/Photos" "photo"

# 2. Backup Docker (Excluding heavy Jellyfin media/metadata)
# Note: The exclude path is absolute to be safe
JELLYFIN_EXCLUDE="--exclude /sata_1tb/nfs_share/docker-volumes/File_Browser/files/aprilz/Jellyfin"
safe_backup "$NFS_PATH/docker-volumes" "docker" "$JELLYFIN_EXCLUDE"
safe_backup "$NFS_PATH/docker-compose" "docker"

# 3. Backup VM Snapshot (Checks if file exists first)
if ls $VM_DUMP/vzdump-qemu-110-* 1> /dev/null 2>&1; then
    echo "VM file found. Backing up..."
    restic backup "$VM_DUMP" --tag vm $LIMITS
    
    echo "Upload complete. Deleting local VM snapshot..."
    rm -f $VM_DUMP/vzdump-qemu-110-*
else
    echo "No VM snapshot found. Did cloud-backup.sh run?"
fi

# 4. Maintenance (Keep last 3 snapshots, prune old data)
echo "Pruning old backups..."
restic forget --keep-last 3 --prune $LIMITS

echo "--- Backup Finished Successfully ---"

```

---

## 4. Automation (Cron Schedule)

Run `crontab -e` to view or edit.

```bash
# 1. Create VM Snapshot at 3:00 AM every Sunday
0 3 * * 0 /root/cloud-backup.sh

# 2. Start Upload at 3:30 AM (Gives VM time to finish dumping)
30 3 * * 0 /root/restic-backup.sh >> /root/restic.log 2>&1

```

---

## 5. Emergency Manual & Troubleshooting

### How to Run Manually

If you need to force a backup *right now*:

1. Run prep: `/root/cloud-backup.sh`
2. Run upload: `/root/restic-backup.sh`

### How to Fix "Repository Locked" Error

If a backup crashes or is stopped manually, Restic will lock the vault.

1. **Kill background processes:**
```bash
killall -9 restic rclone

```


2. **Unlock the vault:**
```bash
export RESTIC_REPOSITORY="rclone:gdrive:ProxmoxBackups/ResticRepo"
export RESTIC_PASSWORD="Able1593."
restic unlock

```



### How to Restore Files

**Option A: Browse files like a USB drive (Easiest)**

```bash
mkdir /mnt/restore_view
export RESTIC_PASSWORD="Able1593."
export RESTIC_REPOSITORY="rclone:gdrive:ProxmoxBackups/ResticRepo"
restic mount /mnt/restore_view
# Now you can browse /mnt/restore_view using `ls` or a file manager

```

**Option B: Restore everything to a folder**

```bash
restic restore latest --target /path/to/restore/location --tag photo

```

### Rclone Re-Authorization

If Rclone stops working (token expires) and you are on the headless Proxmox shell:

1. On your **Laptop**: Run `rclone authorize "drive"`
2. Log in to Google in the browser pop-up.
3. Copy the `{ "access_token": ... }` text it gives you.
4. On **Proxmox**: Run `rclone config`, edit the `gdrive` remote, and paste the token when asked for `config_token`.
