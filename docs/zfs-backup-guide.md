# ZFS Snapshot and Backup Guide

This guide explains the two-pool ZFS storage architecture, automatic snapshot schedule, remote replication, and disaster recovery procedures for this homelab.

---

## Storage Architecture

The storage model puts critical application state, personal photos, documents, and disposable caches on a mirrored SSD pool (`fast`). Bulk video files reside on a separate single HDD pool (`tank`) with no mirror:

```
ZFS SSD Mirror Pool: fast (2x SSD mirror)
│
├── fast/appdata (Mounted to ./configs inside ~/home-lab)
│   ├── Size: ~10 GB - 40 GB
│   ├── Contents: SQLite databases, Immich Postgres data, container configs, SSL certs
│   ├── Tuning: compression=lz4, atime=off, xattr=sa
│   ├── Snapshot Policy: Hourly (keep 48), Daily (keep 30), Monthly (keep 12)
│   └── Backup: Replicated offsite via zfs send / Syncoid
│
├── fast/photos (Mounted to ~/photos)
│   ├── Size: ~140 GB+ (Immich photo and video libraries)
│   ├── Contents: Irreplaceable family photos and videos
│   ├── Tuning: compression=lz4, atime=off
│   ├── Snapshot Policy: Daily (keep 30), Monthly (keep 12), Yearly (keep 2)
│   └── Backup: Replicated offsite via Syncoid (CRITICAL BACKUP)
│
├── fast/storage (Mounted to ~/storage)
│   ├── Size: ~1 GB+ (personal cloud documents)
│   ├── Contents: Native SMB Share & Syncthing Offline Drive
│   ├── Tuning: compression=lz4, atime=off
│   ├── Snapshot Policy: Daily (keep 30), Monthly (keep 12), Yearly (keep 2)
│   └── Backup: Replicated offsite via Syncoid (CRITICAL BACKUP)
│
└── fast/cache (Mounted to ~/cache)
    ├── Size: ~10 GB - 30 GB
    ├── Subdirectories:
    │   ├── jellyfin/  (Transcode chunks)
    │   └── immich-ml/ (Downloaded machine learning model weights)
    ├── Tuning: compression=lz4, atime=off
    └── Backup: EXCLUDED from snapshots and remote replication (disposable cache)

ZFS HDD Pool: tank (1x HDD, unmirrored)
│
└── tank/media (Mounted to ~/media)
    ├── Size: Multiple Terabytes
    ├── Subdirectories:
    │   ├── downloads/ (Torrent download staging)
    │   ├── movies/    (Curated movie files)
    │   └── tv/        (Curated TV show files)
    ├── Tuning: compression=lz4, recordsize=1M
    ├── Snapshot Policy: None (or optional short local retention)
    └── Backup: EXCLUDED from remote replication (Sonarr and Radarr DBs on fast/appdata automate recovery)
```

---

## Step 1: Make the ZFS Pools and Datasets

> [!WARNING]
> Clone the `~/home-lab` git repository before you make the dataset mountpoints. If the mountpoint directories exist first, `git clone` fails.

1. Run these commands on the Ubuntu host to set up both pools, the ARC cap, datasets, and folders:

```bash
# 1. Install ZFS utilities if not installed
sudo apt update && sudo apt install -y zfsutils-linux

# 2. List physical drives by ID (avoid /dev/sdX)
ls -l /dev/disk/by-id/

# 3. Make the mirrored SSD pool named 'fast'
sudo zpool create -o ashift=12 -O compression=lz4 fast mirror \
    /dev/disk/by-id/<ssd-1-id> \
    /dev/disk/by-id/<ssd-2-id>

# 4. Make the single-HDD pool named 'tank' for bulk media
sudo zpool create -o ashift=12 -O compression=lz4 tank /dev/disk/by-id/<hdd-1-id>

# 5. Cap ZFS ARC at 4 GB on a 16 GB RAM host so Immich ML keeps free memory
echo "options zfs zfs_arc_max=4294967296" | sudo tee /etc/modprobe.d/zfs.conf
sudo update-initramfs -u

# 6. Clone this repository into your home directory
git clone <your-repo-url> ~/home-lab

# 7. Make the SSD datasets (appdata, photos, storage, and cache)
sudo zfs create -o compression=lz4 -o atime=off -o xattr=sa -o mountpoint=/home/$USER/home-lab/configs fast/appdata
sudo zfs create -o compression=lz4 -o atime=off -o mountpoint=/home/$USER/photos fast/photos
sudo zfs create -o compression=lz4 -o atime=off -o mountpoint=/home/$USER/storage fast/storage
sudo zfs create -o compression=lz4 -o atime=off -o mountpoint=/home/$USER/cache fast/cache

# 8. Make the bulk media dataset on HDD (1M recordsize for video streams)
sudo zfs create -o compression=lz4 -o recordsize=1M -o mountpoint=/home/$USER/media tank/media

# 9. Make subdirectories for cache and media
mkdir -p /home/$USER/cache/{jellyfin,immich-ml}
mkdir -p /home/$USER/media/{downloads,movies,tv}

# 10. Give ownership of all datasets to your user
sudo chown -R $USER:$USER /home/$USER/home-lab/configs /home/$USER/photos /home/$USER/storage /home/$USER/cache /home/$USER/media
```

> [!NOTE]
> Keep `fast` below 80% used capacity so ZFS copy-on-write stays fast. Also keep `downloads`, `movies`, and `tv` inside `tank/media` so atomic hardlinks work.

---

## Step 2: Set Up Automatic Snapshots (Sanoid)

[Sanoid](https://github.com/jimsalterjrs/sanoid) makes, rotates, and prunes ZFS snapshots based on retention rules.

### Install Sanoid

1. Run these commands to install Sanoid:

```bash
sudo apt update
sudo apt install -y sanoid
```

### Edit `/etc/sanoid/sanoid.conf`

2. Put this configuration into `/etc/sanoid/sanoid.conf`:

```ini
[fast/appdata]
    use_template = production
    recursive = yes

[fast/photos]
    use_template = personal_files
    recursive = yes

[fast/storage]
    use_template = personal_files
    recursive = yes

[fast/cache]
    use_template = ignore
    recursive = yes

[tank/media]
    use_template = ignore
    recursive = yes

#############################
# TEMPLATES
#############################

[template_production]
    frequently = 0
    hourly = 48
    daily = 30
    monthly = 12
    yearly = 1
    autosnap = yes
    autoprune = yes

[template_personal_files]
    frequently = 0
    hourly = 12
    daily = 30
    monthly = 12
    yearly = 2
    autosnap = yes
    autoprune = yes

[template_ignore]
    autosnap = no
    autoprune = no
```

3. Start the systemd timer for Sanoid:

```bash
sudo systemctl enable --now sanoid.timer
```

4. Run these commands to check that Sanoid made snapshots:

```bash
sudo zfs list -t snapshot -r fast/appdata
sudo zfs list -t snapshot -r fast/photos
sudo zfs list -t snapshot -r fast/storage
```

---

## Step 3: Set Up Remote Offsite Replication (Syncoid)

Syncoid sends ZFS snapshots over SSH to a remote machine (a NAS, a second PC, or a cloud VPS) with `zfs send | zfs recv`.

> [!NOTE]
> A ZFS mirror protects against one drive failure, not theft, fire, or accidental removal. If you do not run a remote ZFS host, back up `~/home-lab/configs`, `~/photos`, and `~/storage` with an encrypted cloud tool such as `restic` or `rclone`.

### Set Up SSH Access

1. Make an SSH key on your server and copy it to the remote backup host:
   ```bash
   ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
   ssh-copy-id backupuser@remote-nas.lan
   ```
2. Make sure `backupuser` has ZFS permissions (`zfs allow`) or root access on the target pool.

### Set Up the Nightly Replication Script

1. Put this script into `/usr/local/bin/backup-homelab.sh`:

```bash
#!/bin/bash
set -e

# 1. Take fresh snapshots
sanoid --take-snapshots

# 2. Replicate appdata (configs and databases)
syncoid --no-privilege-elevation \
        -r fast/appdata \
        backupuser@remote-nas.lan:backup-tank/homelab-configs

# 3. Replicate photos (Immich library)
syncoid --no-privilege-elevation \
        -r fast/photos \
        backupuser@remote-nas.lan:backup-tank/homelab-photos

# 4. Replicate personal storage (SMB files and Syncthing documents)
syncoid --no-privilege-elevation \
        -r fast/storage \
        backupuser@remote-nas.lan:backup-tank/homelab-storage

echo "Homelab backup completed on $(date)"
```

2. Make the script executable and add a nightly cron job at 3:00 AM:

```bash
sudo chmod +x /usr/local/bin/backup-homelab.sh
# Run nightly at 3:00 AM (sudo crontab -e):
# 0 3 * * * /usr/local/bin/backup-homelab.sh >> /var/log/homelab-backup.log 2>&1
```

---

## Disaster Recovery Runbook

### Scenario A: Restore One Corrupted Service or Database

Use this procedure when an update or accidental removal corrupts a service (for example, Radarr or Vaultwarden).

> [!CAUTION]
> Remove active journal files (`-wal` and `-shm`) before you restore a `.db` file. Containers such as Radarr and Sonarr write to `radarr.db-wal` and `radarr.db-shm`. If you keep those files, newer WAL logs replay on top of old data and cause database corruption.

1. Run these commands to stop the container, restore the database from a snapshot, and restart the container:

```bash
# 1. Stop the affected container
cd ~/home-lab
docker compose stop radarr

# 2. Go to the hidden ZFS snapshot directory inside configs
cd configs/.zfs/snapshot/

# 3. List available snapshots
ls -la

# 4. Remove active WAL and SHM files, then copy the database from a snapshot
rm -f ~/home-lab/configs/radarr/radarr.db*
# Replace <snapshot_name> with the snapshot folder name from step 3:
cp -a <snapshot_name>/radarr/radarr.db* ~/home-lab/configs/radarr/

# 5. Start the container again
docker compose start radarr
```

---

### Scenario B: Media HDD Failure (`tank/media`)

Use this procedure when the single unmirrored media HDD fails:

1. Replace the failed HDD and make a new `tank` pool and `tank/media` dataset:

   ```bash
   sudo zpool create -f -o ashift=12 -O compression=lz4 tank /dev/disk/by-id/<new-hdd-id>
   sudo zfs create -o compression=lz4 -o recordsize=1M -o mountpoint=/home/$USER/media tank/media
   mkdir -p /home/$USER/media/{downloads,movies,tv}
   sudo chown -R $USER:$USER /home/$USER/media
   ```

2. Restart the media containers (`docker compose up -d qbittorrent sonarr radarr bazarr jellyfin`).
3. In **Radarr** and **Sonarr**, open **Wanted** -> **Missing** and search so both apps download your library again.

---

### Scenario C: Full Disaster Recovery (Total Hardware Loss)

Run these steps on a replacement server:

1. Install ZFS utilities, Samba, and Docker:

   ```bash
   sudo apt update && sudo apt install -y zfsutils-linux samba
   curl -fsSL https://get.docker.com | sh
   sudo usermod -aG docker $USER
   ```

2. Clone the git repository into your home directory first:

   ```bash
   git clone <repo-url> ~/home-lab
   ```

3. Restore all critical ZFS datasets from your remote backup server into `fast`:

   ```bash
   # Get the latest snapshot names
   LATEST_CONFIGS=$(ssh backupuser@remote-nas.lan "zfs list -t snapshot -o name -s creation -r backup-tank/homelab-configs | tail -n 1")
   LATEST_PHOTOS=$(ssh backupuser@remote-nas.lan "zfs list -t snapshot -o name -s creation -r backup-tank/homelab-photos | tail -n 1")
   LATEST_STORAGE=$(ssh backupuser@remote-nas.lan "zfs list -t snapshot -o name -s creation -r backup-tank/homelab-storage | tail -n 1")

   # Receive snapshots into the local SSD pool
   ssh backupuser@remote-nas.lan "zfs send -R ${LATEST_CONFIGS}" | sudo zfs receive fast/appdata
   ssh backupuser@remote-nas.lan "zfs send -R ${LATEST_PHOTOS}" | sudo zfs receive fast/photos
   ssh backupuser@remote-nas.lan "zfs send -R ${LATEST_STORAGE}" | sudo zfs receive fast/storage
   ```

4. Make the disposable `fast/cache` and `tank/media` datasets, then set mountpoints and ownership:

   ```bash
   sudo zfs set mountpoint=/home/$USER/home-lab/configs fast/appdata
   sudo zfs set mountpoint=/home/$USER/photos fast/photos
   sudo zfs set mountpoint=/home/$USER/storage fast/storage
   sudo zfs create -o compression=lz4 -o atime=off -o mountpoint=/home/$USER/cache fast/cache
   sudo zfs create -o compression=lz4 -o recordsize=1M -o mountpoint=/home/$USER/media tank/media
   mkdir -p /home/$USER/cache/{jellyfin,immich-ml} /home/$USER/media/{downloads,movies,tv}
   sudo chown -R $USER:$USER /home/$USER/home-lab/configs /home/$USER/photos /home/$USER/storage /home/$USER/cache /home/$USER/media
   ```

5. Restore or recreate your `.env` file, then start the stack:

   ```bash
   cd ~/home-lab
   # Restore your .env file or copy it from example.env
   docker compose up -d
   ```

All containers start with their previous state, configs, photos, documents, and databases intact.
