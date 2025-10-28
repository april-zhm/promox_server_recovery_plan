🛠 Homelab Backup & Recovery Documentation
==========================================

**Purpose:** Restore the full environment (Proxmox host, VMs, LXCs, and important data) to a new machine or after a hardware failure.

* * * * *

1️⃣ Hardware & Storage Layout
-----------------------------

| Drive | Mount Point | Purpose |
| --- | --- | --- |
| NVMe 256 GB | `/` | Proxmox OS |
| SSD 1 TB (`sata_1tb`) | `/sata_1tb` | Storage for LXC root disks and shared docker volumes |
| External USB | `/mnt/pve/backup` | Proxmox vzdump backup destination |

* * * * *

2️⃣ Mounting & Bind Mounts
--------------------------

### Host

-   `sata_1tb` is a ZFS pool mounted at `/sata_1tb`

-   `nfs_share` folder inside `sata_1tb` contains data shared with Docker containers

-   External USB is mounted at `/mnt/pve/backup` for backups

3️⃣ Backup Strategy
-------------------

### 3.1 Proxmox Backup (VMs & LXCs)

-   Tool: Proxmox **vzdump** (or Proxmox Backup Server if desired in future)

-   Scope:

    -   Entire VM and LXC root disks (including those stored on `sata_1tb`)

    -   LXC configuration files (`/etc/pve/lxc/<id>.conf`)

-   Destination: `/mnt/pve/backup` (USB drive)

-   Schedule: Weekly (or as preferred)

-   Notes:

    -   All container apps and configs are included

    -   Root disks on `sata_1tb` are fully captured

### 3.2 Duplicati Backup (Shared Data)

-   Tool: **Duplicati Docker container** running inside VM 101

-   Source: `/mnt/sata_1tb/nfs_share`

-   Destination: `/mnt/usb_backup/duplicati_backups/sata_1tb`

-   Purpose: Backup all shared Docker volumes for all other containers

-   Container: **Privileged** (to ensure read access)

-   Schedule: Weekly

-   Notes:

    -   Only backs up `nfs_share` --- no need to back up other LXC root disks (already in vzdump)

* * * * *

4️⃣ Restore Workflow
--------------------

### Scenario: Full host failure

1.  **Install new Proxmox host** on new NVMe drive.

2.  **Attach SSD (`sata_1tb`)** and USB backup. Ensure they mount properly (`/sata_1tb` and `/mnt/pve/backup`).

3.  **Restore VMs & LXCs** from Proxmox backup:

    `vzdump --restore /mnt/pve/backup/vzdump-lxc-101.vma.zst 101`

    -   This restores LXC root filesystem, config, and apps

    -   LXCs with root disks on `sata_1tb` are fully restored

4.  **Restore shared Docker data** using Duplicati:

    -   Start Duplicati VM (101)

    -   Navigate to `/mnt/usb_backup/duplicati_backups/sata_1tb`

    -   Run restore to `/mnt/sata_1tb/nfs_share`

5.  **Start Docker containers** inside the VM --- they will automatically use `/mnt/sata_1tb/nfs_share` for volumes.

### Notes

-   After restore, all Docker containers, VMs, and LXC configurations will be **exactly as before**.

-   If new containers are added in future, just ensure their persistent volumes live in `nfs_share` or other mount points backed by Duplicati.

* * * * *

5️⃣ Verification After Restore
------------------------------

1.  Check disk mounts:

    `mount | grep sata_1tb`

2.  Check container bind mounts:

    `pct exec <container_id> -- ls -l /mnt/sata_1tb/nfs_share`

3.  Verify Duplicati backups exist and are restorable.

4.  Start Docker containers and confirm volumes are intact.

* * * * *

✅ This strategy ensures:

-   LXC/VM OS and configs are safe in Proxmox backups

-   All shared Docker volumes (`nfs_share`) are backed up via Duplicati

-   No redundancy or double-backups for root disks
