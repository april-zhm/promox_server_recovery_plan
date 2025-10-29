🧠 Drive Health Monitoring (SMART Setup)
----------------------------------------------------------

Keeping track of your NVMe, SSD, and backup drives' health is essential. SMART monitoring helps detect early warning signs such as reallocated sectors, read errors, or rising temperatures.

* * * * *

### ⚙️ Step 1: Install Required Tools

`apt update
apt install smartmontools -y`

* * * * *

### 🧩 Step 2: Configure `smartd`

Edit the configuration file:

```
nano /etc/smartd.conf
```

Add this line at the bottom:

```
DEVICESCAN -l error -l selftest -s (S/../.././02) -M exec /usr/local/bin/smart_warning.sh
```

**Explanation:**

-   `DEVICESCAN` --- automatically detects all drives (NVMe, SATA, USB).

-   `-l error -l selftest` --- logs device errors and self-test results.

-   `-s (S/../.././02)` --- runs a short self-test every day at 2 AM.

-   `-M exec` --- executes a custom script when warnings occur (instead of sending email).

* * * * *

### 🪄 Step 3: Create the Logging Script

```
nano /usr/local/bin/smart_warning.sh
```

Paste this content:

```
#!/bin/bash
LOGFILE="/var/log/smartd_warnings.log"
echo "[$(date)] SMART warning detected on device $SMARTD_DEVICE ($SMARTD_FAILTYPE)" >> "$LOGFILE"
```

Make it executable:

```
chmod +x /usr/local/bin/smart_warning.sh
```

* * * * *

### 🚀 Step 4: Enable and Start the Service

```
systemctl enable smartd
systemctl restart smartd
systemctl status smartd
```

You should see something like:

`smartd[1234]: Device scan detected /dev/nvme0n1, /dev/sda, /dev/sdb`

That means all your disks are being monitored.

* * * * *

### 🧾 Step 5: Check Health and Logs

To view current health status of each drive:

```smartctl -a /dev/nvme0n1
smartctl -a /dev/sda
```

To read recent SMART logs:

```
journalctl -u smartd -n 20
cat /var/log/smartd_warnings.log
```

* * * * *

### ⚠️ How to Interpret Warnings

If `smartd` logs a warning, check for:

-   **Reallocated_Sector_Ct** > 0 --- bad sectors remapped.

-   **Current_Pending_Sector** > 0 --- sectors waiting for reallocation.

-   **Temperature_Celsius** unusually high (e.g., > 60°C).

-   **Media_Wearout_Indicator** (on SSDs) --- low value means nearing end of life.

* * * * *

### 💡 Optional: Manual Health Check

You can manually run a quick or long self-test anytime:

```
smartctl -t short /dev/nvme0n1
smartctl -t long /dev/sda
```

Then check the result later:

```
smartctl -a /dev/nvme0n1 | grep -A 5 "Self-test"
```

* * * * *

### 🧷 Summary

With this setup:

-   All drives (NVMe, SATA, and backup USB) are auto-monitored.

-   Self-tests run daily.

-   Warnings are logged to `/var/log/smartd_warnings.log`.

-   No email setup required.

-   You can periodically review logs or include this file in your Duplicati backups.

* * * * *

Would you like me to add a **"Maintenance Routine"** section next (e.g. how often to check logs, verify backup integrity, and test restore quarterly)? It complements this SMART monitoring section nicely.
