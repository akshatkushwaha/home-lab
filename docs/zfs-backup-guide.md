# 💾 ZFS Snapshot & Backup Guide

This guide details the complete storage architecture, automated snapshot scheduling, remote replication, and disaster recovery procedures for this homelab stack.

---

## 🏗️ Storage Architecture

Our storage model isolates **critical state** from **disposable bulk data**:

```
ZFS Pool: tank
│
├── tank/appdata (Mounted to ~/home-lab/configs)
│   ├── Size: 5 GB – 30 GB
│   ├── Frequency of change: Continuous (SQLite databases, session tokens, user state)
│   ├── Snapshot Policy: Hourly (keep 48), Daily (keep 30), Monthly (keep 12)
│   └── Backup: Replicated offsite via zfs send / Syncoid
│
└── tank/media (Mounted to ~/media)
    ├── Size: Multiple Terabytes
    ├── Subdirectories:
    │   ├── downloads/ (Torrent download staging)
    │   ├── movies/    (Movie files)
    │   ├── tv/        (TV show files)
    │   └── cache/     (Jellyfin transcode scratch & image cache)
    ├── Snapshot Policy: Optional weekly local snapshot (no long retention)
    └── Backup: Excluded from routine remote replication
```

---

## 🛠️ Step 1: Initial Dataset Creation

Run these commands on the ZFS host:

```bash
# 1. Create the appdata dataset with LZ4 compression enabled
sudo zfs create -o compression=lz4 -o mountpoint=/home/$USER/home-lab/configs tank/appdata

# 2. Create the media dataset (set recordsize=1M for optimal streaming/large media)
sudo zfs create -o compression=lz4 -o recordsize=1M -o mountpoint=/home/$USER/media tank/media

# 3. Create media subdirectories
mkdir -p /home/$USER/media/{downloads,movies,tv,cache/jellyfin}

# 4. Set ownership to your user
sudo chown -R $USER:$USER /home/$USER/home-lab/configs /home/$USER/media
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

Verify snapshots:

```bash
sudo zfs list -t snapshot -r tank/appdata
```

---

## 🔄 Step 3: Remote Offsite Replication (Syncoid)

Syncoid transfers ZFS snapshots incrementally over SSH to another machine (NAS, secondary PC, or cloud VPS) using `zfs send | zfs recv`.

### Basic Syncoid Command

```bash
# Push appdata snapshots to a remote backup server over SSH:
syncoid -r tank/appdata backupuser@nas.local:backuppool/homelab-appdata
```

### Automated Nightly Cron Job

Create a script `/usr/local/bin/backup-homelab.sh`:

```bash
#!/bin/bash
set -e

# Take fresh snapshot
sanoid --take-snapshots

# Replicate appdata to remote backup destination
syncoid --no-privilege-elevation \
        -r tank/appdata \
        backupuser@remote-nas.lan:backup-tank/homelab-configs

echo "Homelab backup successfully completed on $(date)"
```

Make it executable and add to crontab:

```bash
sudo chmod +x /usr/local/bin/backup-homelab.sh
# Run nightly at 3:00 AM:
# 0 3 * * * /usr/local/bin/backup-homelab.sh >> /var/log/homelab-backup.log 2>&1
```

---

## 🚨 Disaster Recovery Runbook

### Scenario A: Reverting a single corrupted service or database

If an update corrupts a service (e.g. Radarr or Vaultwarden):

```bash
# 1. Stop the affected container
cd ~/home-lab
docker compose stop radarr

# 2. Access the hidden ZFS snapshot directory
cd ~/home-lab/configs/.zfs/snapshot/

# 3. List available snapshots
ls -la

# 4. Copy back the database from a known good snapshot (e.g. 2 hours ago)
cp -a autosnap_2026-10-19_20:00:00_hourly/radarr/radarr.db ~/home-lab/configs/radarr/radarr.db

# 5. Restart the container
docker compose start radarr
```

### Scenario B: Full Disaster Recovery (Total Hardware Loss)

On a brand new machine:

1. **Install OS & Docker**:

   ```bash
   curl -fsSL https://get.docker.com | sh
   sudo usermod -aG docker $USER
   ```

2. **Restore the ZFS dataset from your backup server**:

   ```bash
   ssh backupuser@remote-nas.lan "zfs send -R backup-tank/homelab-configs@latest" | sudo zfs receive tank/appdata
   ```

3. **Mount the restored dataset**:

   ```bash
   sudo zfs set mountpoint=/home/$USER/home-lab/configs tank/appdata
   sudo chown -R $USER:$USER /home/$USER/home-lab/configs
   ```

4. **Clone the git repository & start the stack**:

   ```bash
   git clone <repo-url> ~/home-lab
   cd ~/home-lab
   # Ensure .env is restored or recreated
   docker compose up -d
   ```

All 22 containers will boot up with 100% of their previous state, databases, users, watch histories, and SSL certificates intact in under 10 minutes.
