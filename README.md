# 🧭 Full Proxmox Backup & Recovery Documentation

This document explains how to rebuild your Proxmox environment exactly as before in the event of SSD or system failure.  
It covers your hardware layout, mounting structure, Proxmox config restoration, and Docker/Duplicati data recovery.

---

## ⚙️ 1. System Overview

### 🖥️ Hardware Layout
- **Boot Drive (NVMe 256GB)**  
  - Proxmox OS installed here  
  - Handles system files and LXC/VM configuration  
  - Mounted automatically as `/`

- **SSD Storage Drive (1TB, ZFS pool named `sata_1tb`)**  
  - Houses all Docker volumes, LXC root disks, and shared data  
  - Mounted as `/sata_1tb`
  - Subvolumes include:
    ```
    /sata_1tb/nfs_share
    /sata_1tb/subvol-101-disk-0
    /sata_1tb/subvol-104-disk-0
    /sata_1tb/subvol-108-disk-0
    ```

- **USB Backup Drive**  
  - Mounted automatically by Proxmox as `/mnt/pve/backup`
  - Stores:
    ```
    /mnt/pve/backup/dump/                  → Proxmox VM & LXC backups
    /mnt/pve/backup/proxmox_config_backups → Proxmox config tarballs
    /mnt/pve/backup/duplicati_backups/     → Docker volume backups
    /mnt/pve/backup/duplicati_config/      → Duplicati configuration export
    /mnt/pve/backup/scripts/               → Backup & restore scripts
    ```

---

## 💽 2. Mounting the Drives After Fresh Install

After reinstalling Proxmox on a new NVMe drive:

### 🔹 Check available disks
```bash
lsblk
zpool list
zfs list
```
### 🔹 Import and mount the ZFS pool

If `sata_1tb` does not appear:

`zpool import
zpool import sata_1tb`

Then verify mount points:

`zfs list`

If needed, manually mount it:

`zfs mount sata_1tb`

It should now appear under `/sata_1tb`.

### 🔹 Mount the USB backup drive

Proxmox automounts `/mnt/pve/backup`, but if it does not:

`mount /dev/sdb1 /mnt/pve/backup`

(Replace `/dev/sdb1` with the correct device if different.)

* * * * *

🧩 3. Restore Proxmox Configurations
------------------------------------

### 📍 Script Locations

-   Backup script: `/mnt/pve/backup/scripts/backup_proxmox_full_config.sh`

-   Restore script: `/mnt/pve/backup/scripts/recover_proxmox_config.sh`

### ▶️ To back up (run from host):

`sudo /mnt/pve/backup/scripts/backup_proxmox_full_config.sh`

This creates a timestamped archive in:

`/mnt/pve/backup/proxmox_config_backups/proxmox_config_backup_<DATE>.tar.gz`

### ▶️ To restore (on new system):

`sudo /mnt/pve/backup/scripts/recover_proxmox_config.sh`

Then:

1.  Choose the backup file (e.g., `proxmox_config_backup_2025-10-27_14-00-23.tar.gz`)

2.  Confirm with `YES`

3.  Script automatically:

    -   Creates a safety backup of current `/etc/pve`

    -   Restores `/etc/pve`, `/etc/network`, `/etc/fstab`, `/usr/local/bin`, etc.

4.  Reboot after restore:

    `reboot`

### ✅ Verify after reboot

`cat /etc/fstab
cat /etc/network/interfaces
pct list && qm list`

* * * * *

🗄️ 4. Recover Proxmox VM & LXC Backups
---------------------------------------

If Proxmox itself is running and `/mnt/pve/backup/dump` contains `.vma.zst` or `.tar.zst` files:

### ▶️ Restore a VM

`qmrestore /mnt/pve/backup/dump/vm-100-<DATE>.vma.zst 100 --storage sata_1tb`

### ▶️ Restore an LXC Container

`pct restore <NEW_CT_ID> /mnt/pve/backup/dump/vzdump-lxc-<ID>-<DATE>.tar.zst --storage sata_1tb`

After restoration, adjust resources or mounts if needed in the Proxmox web UI.

* * * * *

☁️ 5. Restore Duplicati Configuration
-------------------------------------

If your Duplicati Docker container is running inside your VM:

1.  Locate your exported Duplicati configuration file:

    `/mnt/pve/backup/duplicati_config/Docker Volume Backup-duplicati-config.json`

2.  Open Duplicati Web UI → top-right menu → **Settings → Import Configuration**

3.  Choose the `.json` file above and import.

4.  Verify your backup jobs and storage paths (should point to `/mnt/usb_backup` inside container).

* * * * *

🐳 6. Restore Docker Volumes via Duplicati
------------------------------------------

### 🗂️ Backup Location

Duplicati backs up your Docker bind-mounted volumes under:

`/mnt/pve/backup/duplicati_backups/`

### ▶️ To restore:

1.  Open Duplicati Web UI inside the VM.

2.  Click **Restore** → **From backup location**.

3.  Choose the destination path that corresponds to your Docker bind mount (e.g. `/mnt/host/docker_volumes/Actual_budget`).

4.  Select the backup set and restore version you want.

5.  Restore to the **same location** to overwrite the old data.

6. Restore Docker Compose, Docker Volume, and Immich Docker Volume.
   
After restore:

`docker compose down
docker compose up -d`

Your containers will start using the restored volumes.

* * * * *

🧱 7. Verification Steps After Full Recovery
--------------------------------------------

Run these checks in order:

1.  **Check all drives are mounted**

    `df -h
    zpool status`

2.  **List all Proxmox entities**

    `pct list
    qm list`

3.  **Confirm Docker volumes exist in VM**

    `docker volume ls
    ls /mnt/host`

4.  **Open Duplicati Web UI**\
    Verify your backup jobs and restore points.

* * * * *

🧰 8. Summary of What's Protected
---------------------------------

| Category | Backup Method | Location |
| --- | --- | --- |
| Proxmox VMs & LXCs | Built-in Proxmox backups | `/mnt/pve/backup/dump/` |
| Proxmox Configs | Custom backup script | `/mnt/pve/backup/proxmox_config_backups/` |
| Duplicati Configuration | Exported via UI | `/mnt/pve/backup/duplicati_config/` |
| Docker Volumes | Duplicati inside VM | `/mnt/pve/backup/duplicati_backups/` |
| Custom Scripts | Stored manually | `/mnt/pve/backup/scripts/` |

* * * * *

🧠 Notes
--------

-   The USB backup drive (`/mnt/pve/backup`) is your **single source of truth**.

-   All backups are **timestamped** so you can restore from multiple points in time.

-   The ZFS SSD (`sata_1tb`) can fail independently --- you'll still retain all configs and data via these backups.

-   Always test restore workflows once every few months to ensure data integrity.
