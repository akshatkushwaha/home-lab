# 💾 ZFS Snapshot & Backup Guide

This guide details the complete storage architecture, automated snapshot scheduling, remote replication, and disaster recovery procedures for this homelab stack.

---

## 🏗️ Storage Architecture

Our storage model strictly isolates **critical state**, **personal memories & photos**, **personal cloud documents**, and **disposable bulk media**:

```
ZFS Pool: tank
│
├── tank/appdata (Mounted to ./configs inside ~/home-lab)
│   ├── Size: ~5 GB – 20 GB
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

## 🛠️ Step 1: Initial Pool & Dataset Creation

> [!IMPORTANT]
> Always clone the `~/home-lab` git repository **before** creating dataset mountpoints so `git clone` does not conflict with existing directories.

Run these commands on the Ubuntu ZFS host:

```bash
# 1. Install ZFS utilities (if not already installed)
sudo apt update && sudo apt install -y zfsutils-linux

# 2. Identify physical drives by ID (avoid /dev/sdX)
ls -l /dev/disk/by-id/

# 3. Create the ZFS storage pool named 'tank' (mirrored setup example):
sudo zpool create -o ashift=12 -O compression=lz4 tank mirror \
    /dev/disk/by-id/<disk-1-id> \
    /dev/disk/by-id/<disk-2-id>

# 4. Clone this repository into your home directory:
git clone <your-repo-url> ~/home-lab

# 5. Create the appdata dataset for container configs and databases:
sudo zfs create -o compression=lz4 -o atime=off -o xattr=sa -o mountpoint=/home/$USER/home-lab/configs tank/appdata

# 6. Create the dedicated photos dataset (Immich upload destination):
sudo zfs create -o compression=lz4 -o atime=off -o mountpoint=/home/$USER/photos tank/photos

# 7. Create the personal cloud storage dataset (SMB share & Syncthing):
sudo zfs create -o compression=lz4 -o atime=off -o mountpoint=/home/$USER/storage tank/storage

# 8. Create the bulk media dataset (1M recordsize for video streaming):
sudo zfs create -o compression=lz4 -o recordsize=1M -o mountpoint=/home/$USER/media tank/media

# 9. Create disposable media and transcode cache subdirectories:
mkdir -p /home/$USER/media/{downloads,movies,tv,cache/jellyfin,cache/immich-ml}

# 10. Set ownership to your user across all datasets:
sudo chown -R $USER:$USER /home/$USER/home-lab/configs /home/$USER/photos /home/$USER/storage /home/$USER/media
```

---

## ⏱️ Step 2: Automated Snapshot Management (Sanoid)

[Sanoid](https://github.com/jimsalterjrs/sanoid) is a policy-driven ZFS snapshot management tool that automatically creates, rotates, and prunes snapshots.

### Installation

```bash
sudo apt update
sudo apt install -y sanoid
```

### Configuration: `/etc/sanoid/sanoid.conf`

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

Enable the systemd timer:

```bash
sudo systemctl enable --now sanoid.timer
```

Verify snapshot execution:

```bash
sudo zfs list -t snapshot -r tank/appdata
sudo zfs list -t snapshot -r tank/photos
sudo zfs list -t snapshot -r tank/storage
```

---

## 🔄 Step 3: Remote Offsite Replication (Syncoid)

Syncoid transfers ZFS snapshots incrementally over SSH to another machine (NAS, secondary PC, or cloud VPS) using `zfs send | zfs recv`.

### Prerequisites for Remote Replication

1. Generate an SSH key on your server and copy it to the remote backup destination:
   ```bash
   ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
   ssh-copy-id backupuser@remote-nas.lan
   ```
2. Ensure `backupuser` has appropriate ZFS permissions (`zfs allow`) or root access on the target pool.

### Automated Nightly Replication Script

Create a script at `/usr/local/bin/backup-homelab.sh`:

```bash
#!/bin/bash
set -e

# 1. Take fresh snapshots
sanoid --take-snapshots

# 2. Replicate appdata (configs and databases)
syncoid --no-privilege-elevation \
        -r tank/appdata \
        backupuser@remote-nas.lan:backup-tank/homelab-configs

# 3. Replicate photos (Immich memories)
syncoid --no-privilege-elevation \
        -r tank/photos \
        backupuser@remote-nas.lan:backup-tank/homelab-photos

# 4. Replicate personal storage (SMB files & Syncthing documents)
syncoid --no-privilege-elevation \
        -r tank/storage \
        backupuser@remote-nas.lan:backup-tank/homelab-storage

echo "Homelab backup successfully completed on $(date)"
```

Make it executable and schedule via crontab:

```bash
sudo chmod +x /usr/local/bin/backup-homelab.sh
# Run nightly at 3:00 AM (sudo crontab -e):
# 0 3 * * * /usr/local/bin/backup-homelab.sh >> /var/log/homelab-backup.log 2>&1
```

---

## 🚨 Disaster Recovery Runbook

### Scenario A: Reverting a single corrupted service or database

If an update or accidental deletion corrupts a service (e.g. Radarr or Vaultwarden):

> [!CAUTION]
> **SQLite WAL Consistency**: Containers like Radarr and Sonarr write to `radarr.db-wal` and `radarr.db-shm`. You must remove active journal files before restoring the checkpointed `.db` file, otherwise newer WAL logs will replay on top of old data and corrupt the database.

```bash
# 1. Stop the affected container
cd ~/home-lab
docker compose stop radarr

# 2. Navigate to the hidden ZFS snapshot directory inside the configs folder
cd configs/.zfs/snapshot/

# 3. List available snapshots
ls -la

# 4. Clean up any active WAL / SHM files and restore the database from snapshot:
rm -f ~/home-lab/configs/radarr/radarr.db*
cp -a autosnap_2026-10-19_20:00:00_hourly/radarr/radarr.db* ~/home-lab/configs/radarr/

# 5. Restart the container
docker compose start radarr
```

---

### Scenario B: Full Disaster Recovery (Total Hardware Loss)

On a replacement server:

1. **Install OS, Docker & ZFS utilities**:

   ```bash
   sudo apt update && sudo apt install -y zfsutils-linux samba
   curl -fsSL https://get.docker.com | sh
   sudo usermod -aG docker $USER
   ```

2. **Clone the git repository first**:

   ```bash
   git clone <repo-url> ~/home-lab
   ```

3. **Restore all critical ZFS datasets from your remote backup server**:

   ```bash
   ssh backupuser@remote-nas.lan "zfs send -R backup-tank/homelab-configs@latest" | sudo zfs receive tank/appdata
   ssh backupuser@remote-nas.lan "zfs send -R backup-tank/homelab-photos@latest" | sudo zfs receive tank/photos
   ssh backupuser@remote-nas.lan "zfs send -R backup-tank/homelab-storage@latest" | sudo zfs receive tank/storage
   ```

4. **Set mountpoints and user ownership**:

   ```bash
   sudo zfs set mountpoint=/home/$USER/home-lab/configs tank/appdata
   sudo zfs set mountpoint=/home/$USER/photos tank/photos
   sudo zfs set mountpoint=/home/$USER/storage tank/storage
   sudo chown -R $USER:$USER /home/$USER/home-lab/configs /home/$USER/photos /home/$USER/storage
   ```

5. **Start the stack**:

   ```bash
   cd ~/home-lab
   # Restore your .env file or recreate from example.env
   docker compose up -d
   ```

All containers will start with 100% of their previous state, configurations, photos, documents, and databases intact.
