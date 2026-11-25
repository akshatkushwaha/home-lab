# ZFS Snapshot and Backup Guide

This guide explains the storage architecture, automatic snapshot schedule, remote replication, and disaster recovery procedures for this homelab.

---

## Storage Architecture

The storage model separates critical application state, personal photos, personal documents, and disposable bulk media into four datasets:

```
ZFS Pool: tank
│
├── tank/appdata (Mounted to ./configs inside ~/home-lab)
│   ├── Size: ~5 GB - 20 GB
│   ├── Contents: SQLite databases, Immich Postgres data, container configs, SSL certs
│   ├── Tuning: compression=lz4, atime=off, xattr=sa
│   ├── Snapshot Policy: Hourly (keep 48), Daily (keep 30), Monthly (keep 12)
│   └── Backup: Replicated offsite via zfs send / Syncoid
│
├── tank/photos (Mounted to ~/photos)
│   ├── Size: High priority, scalable (hundreds of GBs to TBs)
│   ├── Contents: Immich photo and video libraries (irreplaceable family memories)
│   ├── Tuning: compression=lz4, atime=off
│   ├── Snapshot Policy: Daily (keep 30), Monthly (keep 12), Yearly (keep 2)
│   └── Backup: Replicated offsite via Syncoid (CRITICAL BACKUP)
│
├── tank/storage (Mounted to ~/storage)
│   ├── Size: Scalable based on personal documents
│   ├── Contents: Native SMB Share & Syncthing Offline Drive (Google Drive replacement)
│   ├── Tuning: compression=lz4, atime=off
│   ├── Snapshot Policy: Daily (keep 30), Monthly (keep 12), Yearly (keep 2)
│   └── Backup: Replicated offsite via Syncoid (CRITICAL BACKUP)
│
└── tank/media (Mounted to ~/media)
    ├── Size: Multiple Terabytes
    ├── Subdirectories:
    │   ├── downloads/ (Torrent download staging)
    │   ├── movies/    (Curated movie files)
    │   ├── tv/        (Curated TV show files)
    │   └── cache/     (Jellyfin transcoding chunks & Immich ML model cache)
    ├── Tuning: compression=lz4, recordsize=1M
    ├── Snapshot Policy: Optional weekly local snapshot (no long retention)
    └── Backup: EXCLUDED from remote replication (disposable & re-downloadable)
```

---

## Step 1: Make the ZFS Pool and Datasets

> [!WARNING]
> Clone the `~/home-lab` git repository before you make the dataset mountpoints. If the mountpoint directories exist first, `git clone` fails.

1. Run these commands on the Ubuntu ZFS host to set up the pool, datasets, and folders:

```bash
# 1. Install ZFS utilities if not installed
sudo apt update && sudo apt install -y zfsutils-linux

# 2. List physical drives by ID (avoid /dev/sdX)
ls -l /dev/disk/by-id/

# 3. Make the ZFS pool named 'tank' (mirrored setup example)
sudo zpool create -o ashift=12 -O compression=lz4 tank mirror \
    /dev/disk/by-id/<disk-1-id> \
    /dev/disk/by-id/<disk-2-id>

# 4. Clone this repository into your home directory
git clone <your-repo-url> ~/home-lab

# 5. Make the appdata dataset for container configs and databases
sudo zfs create -o compression=lz4 -o atime=off -o xattr=sa -o mountpoint=/home/$USER/home-lab/configs tank/appdata

# 6. Make the photos dataset for Immich uploads
sudo zfs create -o compression=lz4 -o atime=off -o mountpoint=/home/$USER/photos tank/photos

# 7. Make the personal storage dataset for SMB and Syncthing
sudo zfs create -o compression=lz4 -o atime=off -o mountpoint=/home/$USER/storage tank/storage

# 8. Make the bulk media dataset (1M recordsize for video streams)
sudo zfs create -o compression=lz4 -o recordsize=1M -o mountpoint=/home/$USER/media tank/media

# 9. Make subdirectories for media and transcode caches
mkdir -p /home/$USER/media/{downloads,movies,tv,cache/jellyfin,cache/immich-ml}

# 10. Give ownership of all datasets to your user
sudo chown -R $USER:$USER /home/$USER/home-lab/configs /home/$USER/photos /home/$USER/storage /home/$USER/media
```

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
[tank/appdata]
    use_template = production
    recursive = yes

[tank/photos]
    use_template = personal_files
    recursive = yes

[tank/storage]
    use_template = personal_files
    recursive = yes

[tank/media]
    use_template = media_bulk
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

[template_media_bulk]
    frequently = 0
    hourly = 0
    daily = 7
    monthly = 1
    yearly = 0
    autosnap = yes
    autoprune = yes
```

3. Start the systemd timer for Sanoid:

```bash
sudo systemctl enable --now sanoid.timer
```

4. Run these commands to check that Sanoid made snapshots:

```bash
sudo zfs list -t snapshot -r tank/appdata
sudo zfs list -t snapshot -r tank/photos
sudo zfs list -t snapshot -r tank/storage
```

---

## Step 3: Set Up Remote Offsite Replication (Syncoid)

Syncoid sends ZFS snapshots over SSH to a remote machine (a NAS, a second PC, or a cloud VPS) with `zfs send | zfs recv`.

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
        -r tank/appdata \
        backupuser@remote-nas.lan:backup-tank/homelab-configs

# 3. Replicate photos (Immich library)
syncoid --no-privilege-elevation \
        -r tank/photos \
        backupuser@remote-nas.lan:backup-tank/homelab-photos

# 4. Replicate personal storage (SMB files and Syncthing documents)
syncoid --no-privilege-elevation \
        -r tank/storage \
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

### Scenario B: Full Disaster Recovery (Total Hardware Loss)

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

3. Restore all critical ZFS datasets from your remote backup server:

   ```bash
   # Get the latest snapshot names
   LATEST_CONFIGS=$(ssh backupuser@remote-nas.lan "zfs list -t snapshot -o name -s creation -r backup-tank/homelab-configs | tail -n 1")
   LATEST_PHOTOS=$(ssh backupuser@remote-nas.lan "zfs list -t snapshot -o name -s creation -r backup-tank/homelab-photos | tail -n 1")
   LATEST_STORAGE=$(ssh backupuser@remote-nas.lan "zfs list -t snapshot -o name -s creation -r backup-tank/homelab-storage | tail -n 1")

   # Receive snapshots into the local pool
   ssh backupuser@remote-nas.lan "zfs send -R ${LATEST_CONFIGS}" | sudo zfs receive tank/appdata
   ssh backupuser@remote-nas.lan "zfs send -R ${LATEST_PHOTOS}" | sudo zfs receive tank/photos
   ssh backupuser@remote-nas.lan "zfs send -R ${LATEST_STORAGE}" | sudo zfs receive tank/storage
   ```

4. Set the dataset mountpoints and user ownership:

   ```bash
   sudo zfs set mountpoint=/home/$USER/home-lab/configs tank/appdata
   sudo zfs set mountpoint=/home/$USER/photos tank/photos
   sudo zfs set mountpoint=/home/$USER/storage tank/storage
   sudo chown -R $USER:$USER /home/$USER/home-lab/configs /home/$USER/photos /home/$USER/storage
   ```

5. Restore or recreate your `.env` file, then start the stack:

   ```bash
   cd ~/home-lab
   # Restore your .env file or copy it from example.env
   docker compose up -d
   ```

All containers start with their previous state, configs, photos, documents, and databases intact.
