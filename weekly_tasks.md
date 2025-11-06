🗓 Weekly Backup & Recovery Tasks
=================================

1\. Hardware & Mount Layout
---------------------------

-   **NVMe (Proxmox OS)**: `/` --- contains Proxmox installation.

-   **SSD (Data storage)**: `/sata_1tb` --- contains all Docker volumes and other VM data.

-   **USB Backup Drive**: `/mnt/pve/backup` --- stores Proxmox VM/LXC backups, Duplicati backups, and config backups.

**Mounts in VM:**

```
mp0: /sata_1tb,mp=/mnt/host
mp1: /mnt/pve/backup,mp=/mnt/usb_backup
```

* * * * *

2\. Weekly Backup Tasks
-----------------------

### A. Proxmox VM & LXC Backup (Manual)

**Steps:**

1.  **Plug in USB backup drive** and ensure it is mounted:

```
df -h | grep backup
```

-   If it is busy or inaccessible, make sure no shell is inside `/mnt/pve/backup` and stop any processes using it.

1.  **Run manual backup script:**

```
/mnt/pve/backup/scripts/manual_proxmox_backup.sh
```

**What it does:**

-   Backs up all VMs and LXCs to `/mnt/pve/backup`.

-   Keeps the **2 most recent backups per VM/LXC**.

-   Uses **snapshot mode** for running VMs to avoid hangs.

-   Stops briefly running VMs if necessary (fallback stop mode).

-   Logs progress to `/mnt/pve/backup/log/manual_proxmox_backup.log`.

* * * * *

### B. Proxmox Configuration Backup

**Script:**

```
/mnt/pve/backup/scripts/backup_proxmox_full_config.sh
```

**Run manually each week.**

* * * * *

### C. Duplicati Backup (Manual)

**Purpose:** Backup Docker-related data and immich.

**Items to backup:**

1.  **Docker Compose files**

2.  **Docker volume folders** (all mounted under `/mnt/host`)

3.  **Immich backups**

**Location:** `/mnt/pve/backup/duplicati_backups/`

* * * * *
### D. Optional Safety Notes

-   Check NVMe, SSD, and USB health weekly with `smartctl -a /dev/nvme0n1` or `smartctl -a /dev/sdb`.

-   Check smartd logs:
  ```
journalctl -u smartd -n 20
cat /var/log/smartd_warnings.log
```

-   Ensure USB drive filesystem is healthy (`fsck`) if any errors appear.

-   Keep backup rotation limited (Proxmox script: 2 per VM, Duplicati: 3 per type) to avoid full disk issues.

* * * * *

### E. Safe USB Drive Handling

**Before unplugging USB drive:**

1.  Ensure no terminal or process is in `/mnt/pve/backup`:

```
cd ~
```

1.  Unmount safely:

```
umount /mnt/pve/backup
```

-   If busy, use lazy unmount:

```
umount -l /mnt/pve/backup
```

* * * * *

