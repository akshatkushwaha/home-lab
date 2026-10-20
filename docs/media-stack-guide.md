# 🎬 Complete Media Stack Guide (*arr, Jellyfin, qBittorrent)

This guide covers the architecture, configuration, hardlink optimization (TRaSH Guides), and hardware acceleration for the integrated media stack in `docker-compose.yml`.

---

## ⚡ The TRaSH Guides Hardlink Principle

### The Problem with Default Configurations

If you map volumes like this:

```yaml
volumes:
  - ~/media/downloads:/downloads
  - ~/media/movies:/movies
```

Docker treats `/downloads` and `/movies` as **two completely separate virtual filesystems**, even though on the host they sit on the exact same ZFS dataset!
When Radarr imports a 40GB movie:

* It cannot create a hardlink (`EXDEV: Invalid cross-device link`).
* It is forced to perform a **slow, I/O-heavy file copy**.
* It **doubles your disk usage** (40GB download + 40GB movie = 80GB used).

### The Solution: Unified `/data` Mount

In our `docker-compose.yml`, all media containers share a single root mount:

```yaml
volumes:
  - ${MEDIA_DIR:-~/media}:/data
```

Inside the containers:

* Torrents download to: `/data/downloads`
* Movies live in: `/data/movies`
* TV Shows live in: `/data/tv`

When Radarr or Sonarr imports a finished torrent, the Linux kernel creates an **atomic hardlink**:

* **Duration**: < 1 millisecond.
* **Disk space used**: 0 extra bytes.
* **Seeding**: Continues without interruption in the background.

### How to Verify Hardlinks are Working

Run this command inside the host terminal:

```bash
ls -l -i ~/media/movies/SomeMovie/SomeMovie.mkv ~/media/downloads/SomeMovie.mkv
```

If hardlinked, both files will display the **exact same inode number** (the first number in the output) and a link count of `2`.

---

## ⚙️ Step-by-Step Service Configuration

### 1. qBittorrent (`http://<ip>:8080`)

1. Log in (default: `admin` / `adminadmin` or check `docker compose logs qbittorrent`).
2. Go to **Tools** → **Options** → **Downloads**:
   * **Default Save Path**: `/data/downloads`
   * Check **Keep incomplete torrents in**: `/data/downloads/incomplete`
3. Go to **Connection**:
   * Port used for incoming connections: `6881`
4. Go to **Speed**:
   * Configure global rate limits to 80% of your ISP's max upload speed to prevent bufferbloat.

---

### 2. FlareSolverr (`http://flaresolverr:8191`)

* FlareSolverr is an internal proxy used by Prowlarr to bypass Cloudflare and DDoS-Guard anti-bot challenges on public torrent trackers.
* No Web UI configuration is needed; Prowlarr communicates with it directly over the internal Docker network.

---

### 3. Prowlarr (`http://<ip>:9696`)

Prowlarr is your central indexer manager.

1. **Add FlareSolverr**:
   * Navigate to **Settings** → **Indexers**.
   * Click **+** under **Proxy**.
   * Select **FlareSolverr**.
   * Host: `http://flaresolverr:8191`
   * Name: `FlareSolverr`
   * Tags: `flaresolverr` (apply this tag to Cloudflare-protected trackers).
2. **Add Applications (Sonarr & Radarr)**:
   * Navigate to **Settings** → **Apps**.
   * Click **+** → **Radarr**:
     * Prowlarr Server: `http://prowlarr:9696`
     * Radarr Server: `http://radarr:7878`
     * API Key: Copy from Radarr (**Settings** → **General** → **API Key**).
   * Click **+** → **Sonarr**:
     * Prowlarr Server: `http://prowlarr:9696`
     * Sonarr Server: `http://sonarr:8989`
     * API Key: Copy from Sonarr (**Settings** → **General** → **API Key**).
3. **Add Indexers**:
   * Navigate to **Indexers** → Click **+ Add Indexer**.
   * Search for your favorite public or private trackers.
   * Click **Sync App Profiles** to automatically push them into Sonarr and Radarr.

---

### 4. Radarr & Sonarr

#### Root Folders

* **Radarr**: **Settings** → **Media Management** → Root Folders → Add `/data/movies`.
* **Sonarr**: **Settings** → **Media Management** → Root Folders → Add `/data/tv`.

#### Download Client

In both Sonarr and Radarr:

1. Navigate to **Settings** → **Download Clients** → Click **+** → **qBittorrent**.
2. Host: `qbittorrent` (Docker internal name)
3. Port: `8080`
4. Username & Password: Your qBittorrent login.
5. Click **Test** and **Save**.

---

### 5. Bazarr (`http://<ip>:6767`)

Bazarr manages subtitles for both movies and TV shows.

1. Navigate to **Settings** → **Sonarr**:
   * Address: `sonarr`
   * Port: `8989`
   * API Key: Copy from Sonarr.
2. Navigate to **Settings** → **Radarr**:
   * Address: `radarr`
   * Port: `7878`
   * API Key: Copy from Radarr.
3. Navigate to **Settings** → **Providers**:
   * Add OpenSubtitles.com, Subdl, or YIFY Subtitles.

---

### 6. Jellyfin (`http://<ip>:8096`)

#### Libraries

1. Open Jellyfin and complete the initial wizard.
2. Add Library → **Movies**:
   * Folder: `/data/movies`
3. Add Library → **Shows**:
   * Folder: `/data/tv`

#### Transcoding Cache Isolation

1. Go to **Dashboard** → **Playback** → **Transcoding**.
2. **Transcode path**: Set to `/cache`.
   *(This ensures transcode chunks write to `~/media/cache/jellyfin`, keeping multi-gigabyte temporary files out of your ZFS config backup snapshots!)*

#### Hardware Acceleration (GPU Transcoding)

If your host server has an Intel CPU with QuickSync or an AMD/NVIDIA GPU:

1. In `docker-compose.yml`, uncomment the `devices` block under `jellyfin`:

   ```yaml
   devices:
     - /dev/dri:/dev/dri # Intel QuickSync / AMD VAAPI
   ```

2. Restart Jellyfin:

   ```bash
   docker compose up -d jellyfin
   ```

3. In Jellyfin, go to **Dashboard** → **Playback** → **Transcoding**:
   * Hardware acceleration: **Intel QuickSync (QSV)** or **VAAPI**.
   * Enable hardware decoding for H264, HEVC, and VP9.

---

### 7. Jellyseerr (`http://<ip>:5055`)

Jellyseerr provides a clean Netflix-style request interface for you and your family.

1. Open Jellyseerr and click **Sign in with Jellyfin**.
2. Enter your Jellyfin URL (`http://jellyfin:8096`) and admin credentials.
3. Configure Radarr and Sonarr integration:
   * Radarr: Server `http://radarr:7878`, paste API Key, select Root Folder `/data/movies`.
   * Sonarr: Server `http://sonarr:8989`, paste API Key, select Root Folder `/data/tv`.
4. Users can now search for any movie or TV series on Jellyseerr, click **Request**, and the entire stack will find, download, subtitle, and import it automatically!
