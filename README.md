# 📸 Immich Self-Hosting on Proxmox VE with ZFS Bind Mounts

This guide documents the highly resource-efficient setup of **Immich** (Self-hosted Google Photos alternative) running in an **Unprivileged LXC Container** on a Proxmox VE host, utilizing a dedicated **ZFS pool** for large-scale media storage.

This approach ensures the container's small root disk remains clean while utilizing the superior data integrity and volume management of ZFS for photos and videos.

***

## 🚀 Part 1: Immich LXC Deployment (Helper Script Method)

We start by using the community helper script for a quick and standardized installation environment.

1.  **Access Proxmox Node Shell:** Go to your Proxmox Web UI and open the **Shell** for your main node.
2.  **Run the Immich Helper Script:** Copy and paste the command below to launch the automated setup wizard:
    ```bash
    bash -c "$(wget -qLO - [https://raw.githubusercontent.com/tteck/Proxmox/main/ct/immich.sh](https://raw.githubusercontent.com/tteck/Proxmox/main/ct/immich.sh))"
    ```
3.  **Configure Installation:** When prompted, select the **Advanced Setup** and configure the LXC resources:
    * **Container Type:** **Unprivileged** (Crucial for security.)
    * **Root Password:** Set a secure password.
    * **Installation Path:** Choose your fast local storage (like NVMe or SSD) for the LXC's rootfs. **DO NOT** select your large HDD/ZFS pool yet.
    * **Root Disk Size:** **16 GB** (or 20 GB). This is *only* for the OS and Docker.
    * **CPU/RAM:** Set **4 Cores** and **8 GB RAM**.
4.  **Machine Learning:** Click **Yes** for the OpenVINO machine learning installation.
5.  **Wait for Compilation:** The script will download and compile the Immich application. This process can take **10–30 minutes** depending on your CPU specifications.
6.  **Access Immich:** Immich will now be successfully installed. You can access the UI at `http://<LXC_IP_ADDRESS>:2283`.

***

## 💾 Part 2: Preparing the ZFS Media Storage (3TB Partition)

This section ensures your large HDD is partitioned and ready to form the ZFS pool used exclusively for photos.

1.  **Wipe and Partition HDD (Proxmox Host Shell):** Access the Proxmox Node Shell and use `gdisk` to prepare your disk.
    * Wipe the drive's partition table and create a new **GPT**.
    * Create two partitions (e.g., a **3TB** partition for Immich and a **~1TB** partition for other files).
    * *Example using `gdisk /dev/sdX`*: Create Partition 1 with size `+3T` (ZFS).
2.  **Create ZFS Pool (Proxmox GUI):**
    * Navigate to **Your Node** > **Disks** > **ZFS**.
    * Click the **Create: ZFS** button.
    * **Name:** Enter `immich`.
    * **Devices:** Select your **3TB partition** (`/dev/sdX1`).
    * **RAID Level:** Select **Single Disk** (for now).
    * **Compression:** Choose **LZ4** (Highly recommended for performance).
    * Click **Create**. The ZFS pool will be automatically mounted on the host at `/immich`.

***

## 🔗 Part 3: Fixing Storage and Permissions (The Core Fix)

The LXC needs to be told to use the ZFS pool, and we must fix the permissions so the unprivileged container can write to the ZFS volume.

### A. Establish the Bind Mount

1.  **Get LXC/User Details:**
    * **LXC Console:** Find the Immich user's UID and GID:
        ```bash
        grep immich /etc/passwd
        # Note the two numbers (e.g., 999:991)
        ```
    * **Proxmox Host Shell:** Bind the ZFS path (`/immich`) to a logical mount point inside the container (`/mnt/immich-data`).
        ```bash
        # Replace <LXC ID> with your container ID (e.g., 101)
        pct set <LXC ID> -mp0 /immich,mp=/mnt/immich-data
        ```
    * *Verification:* Check the LXC config file: `cat /etc/pve/lxc/<LXC ID>.conf`

### B. Prepare the Storage Path

1.  **Create the Mount Folder (Inside LXC):**
    * **LXC Console:**
        ```bash
        ls -lF /mnt/immich-data 
        # Output should be "Total 0"
        ```
2.  **Copy Immich's Required Directory Structure:** Copy the structure from the default installation path to the new mounted location.
    * **LXC Console:**
        ```bash
        cp -ar /opt/immich/upload /mnt/immich-data/
        ```
    * *Verification:* `ls -lF /mnt/immich-data/upload` should now show the subdirectories Immich uses.

### C. Update Configuration and Symlinks

1.  **Fix the `.env` File (Inside LXC):**
    * **LXC Console:** `nano /opt/immich/.env`
    * **Comment out the old line and add the correct paths:**
        ```env
        # IMMICH_MEDIA_LOCATION=/opt/immich/upload    # <-- COMMENT THIS OUT!
        UPLOAD_LOCATION=/mnt/immich-data/upload      # <-- Immich's upload base directory
        DB_DATA_LOCATION=/mnt/immich-data/postgres   # <-- Move DB off the LXC disk
        ```
2.  **Re-Link the Upload Symlinks (Inside LXC):**
    * **LXC Console (App Directory):**
        ```bash
        cd /opt/immich/app
        mv upload upload-original 
        ln -s /mnt/immich-data/upload upload
        chown -R immich:immich upload
        ```
    * **LXC Console (ML Directory):**
        ```bash
        cd /opt/immich/app/machine-learning
        mv upload upload-original 
        ln -s /mnt/immich-data/upload upload
        chown -R immich:immich upload
        ```
3.  **Reboot:** Reboot the LXC container from the Proxmox UI.

Your Immich installation will now store all new photos and database files on your 3TB ZFS pool.
