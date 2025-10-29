Drive Failure & Safety Strategy
-------------------------------

Your server uses three main drives:

-   **NVMe (boot drive)** -- Hosts Proxmox OS and core configuration.

-   **SATA SSD (`sata_1tb`)** -- Main data drive; holds Docker volumes, LXC root disks, and shared mounts.

-   **USB backup drive (`/mnt/pve/backup`)** -- External backup storage for all snapshots and config backups.

This section explains what happens if each fails and how to prepare.

* * * * *

### ⚙️ 1. Typical Lifespans and Failure Modes

| Drive | Typical Lifespan | Common Failure Mode | Notes |
| --- | --- | --- | --- |
| NVMe | 3--10 years | Sudden electronic failure or firmware corruption | Stores Proxmox itself |
| SATA SSD | 3--10 years | Bad blocks, controller death | Houses your containers and Docker data |
| USB Backup Drive | 3--5 years | Mechanical failure, filesystem corruption | Contains your backups; should not stay plugged in permanently |

* * * * *

### 🔍 2. Monitoring Drive Health

Use `smartctl` to check each drive:

```
smartctl -a /dev/nvme0n1     # For NVMe boot drive
smartctl -a /dev/sda         # For SATA SSD
smartctl -a /dev/sdb         # For USB backup drive
```

Look for:

-   `Reallocated_Sector_Ct` > 0 (bad sign)

-   `Media_Wearout_Indicator` low (< 10%)

-   `SMART overall-health self-assessment test result: FAILED`

Run this monthly and before long backup sessions.

⚠️ Key Attributes to Watch
--------------------------

| SMART Attribute | What It Means | Healthy Value | Warning Signs |
| --- | --- | --- | --- |
| **Reallocated_Sector_Ct** | Number of bad sectors remapped to spare area. | 0 | >0 means drive found physical damage. Increasing = dying disk. |
| **Current_Pending_Sector** | Sectors waiting for reallocation (unreadable for now). | 0 | Any non-zero = unstable media. If it increases → imminent failure. |
| **Offline_Uncorrectable** | Sectors that can't be fixed even after re-read. | 0 | >0 = data already lost in some spots. |
| **Reallocated_Event_Count** | Number of remap operations performed. | 0 | Rising count = spreading damage. |
| **Power_On_Hours** | How long the drive's been running. | N/A | >30,000 h (~3 yrs) = aging, watch closely. |
| **Temperature_Celsius** | Drive temp. | 25--45 °C (SSD), 30--50 °C (HDD) | >55 °C shortens lifespan; >60 °C dangerous. |
| **Power_Cycle_Count** | Number of power-ons. | N/A | Only concerning if extremely high (tens of thousands). |
| **Wear_Leveling_Count / Percentage_Used** (SSDs only) | Flash wear indicator. | 0--10% used | >80% = SSD nearing end of life. |
| **CRC_Error_Count** | Communication errors (bad SATA cable, power, or port). | 0 | Non-zero and rising = check cabling. |

* * * * *

### 🧯 3. Failure Scenarios & Recovery Steps

#### 💥 NVMe Boot Drive Fails

**Impact:**

-   Proxmox host will not boot.

-   VM/LXC data still safe on SATA SSD (`sata_1tb`).

**Recovery:**

1.  Replace NVMe and reinstall Proxmox.

2.  Mount the SATA SSD:

    `zpool import sata_1tb`

3.  Mount USB backup:

    `mount /dev/sdb1 /mnt/pve/backup`

4.  Restore Proxmox configuration:

    `tar -xzf /mnt/pve/backup/proxmox_config_backup_<DATE>.tar.gz -C /`

5.  Restore VMs/LXCs from backups in `/mnt/pve/backup/dump/`.

6.  Restore Docker volumes from Duplicati (see Docker Recovery section).

* * * * *

#### 💥 SATA SSD (`sata_1tb`) Fails

**Impact:**

-   All Docker volumes and LXC root disks lost.

-   Proxmox itself still boots (if NVMe is fine).

**Recovery:**

1.  Replace the SSD.

2.  Recreate the ZFS pool:

    `zpool create sata_1tb /dev/sda`

3.  Restore your VMs/LXCs from Proxmox backups:

    `qmrestore /mnt/pve/backup/dump/vm-XXX-DATE.vma.zst <NEW_ID> --storage sata_1tb`

4.  Restore Docker data using Duplicati:

    -   Import saved config from `/mnt/pve/backup/duplicati_config/`

    -   Restore each volume from `/mnt/pve/backup/duplicati_backups/`

* * * * *

#### 💥 USB Backup Drive Fails

**Impact:**

-   Backups lost, but live system unaffected.

**Recovery:**

1.  Replace the USB drive.

2.  Recreate mount point:

    `mkdir -p /mnt/pve/backup
    mount /dev/sdb1 /mnt/pve/backup`

3.  Recreate backup schedule or rerun backups manually:

    `/root/backup_proxmox_full_config.sh`

* * * * *

### 🧠 4. Preventive Maintenance

-   Replace drives **every 5 years** or sooner if showing SMART errors.

-   Keep **one extra USB backup drive** stored offline or offsite.

-   Perform a **test restore** every 6--12 months.

-   Always **unmount `/mnt/pve/backup`** before unplugging to prevent corruption:

    `umount /mnt/pve/backup`

* * * * *

### ✅ 5. Quick Reference: Recovery Priority

| Failure | Restore From | Key Steps |
| --- | --- | --- |
| NVMe (OS) | Config backup + VM backups | Reinstall Proxmox → import ZFS pool → restore configs |
| SATA SSD | VM backups + Duplicati | Replace drive → recreate pool → restore data |
| USB Backup | Fresh disk | Recreate mount → rerun backup scripts |
