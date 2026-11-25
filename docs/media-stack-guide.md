# Complete Media Stack Guide (*arr, Jellyfin, qBittorrent)

This guide explains the architecture, setup, hardlink optimization (TRaSH Guides), and hardware acceleration for the media stack in `docker-compose.yml`.

---

## TRaSH Guides Hardlink Principle

### Problem with Default Volume Maps

Suppose you map volumes like this example:

```yaml
volumes:
  - ~/media/downloads:/downloads
  - ~/media/movies:/movies
```

Docker treats `/downloads` and `/movies` as two separate virtual filesystems, even though both sit on the same ZFS dataset on the host. When Radarr imports a 40 GB movie, these problems happen:

* Radarr cannot make a hardlink (`EXDEV: Invalid cross-device link`).
* Radarr must do a slow file copy with high disk I/O.
* The file copy doubles disk usage (40 GB download plus 40 GB movie equals 80 GB used).

### Solution: Shared `/data` Mount

In `docker-compose.yml`, the download and automation containers (`qBittorrent`, `Sonarr`, `Radarr`, and `Bazarr`) share one root mount:

```yaml
volumes:
  - ${MEDIA_DIR:-~/media}:/data
```

Inside the containers, the directories use these paths:

* Torrents download to `/data/downloads`.
* Movies reside in `/data/movies`.
* TV shows reside in `/data/tv`.

> [!NOTE]
> Unlike download managers, `jellyfin` mounts `/data/movies:ro` and `/data/tv:ro` individually in read-only mode (`${MEDIA_DIR:-~/media}/movies:/data/movies:ro`). This keeps the same internal library paths and prevents changes or removals in your media files.

When Radarr or Sonarr imports a finished torrent, the Linux kernel makes an atomic hardlink with these results:

* **Duration**: Less than 1 millisecond.
* **Extra disk space**: 0 bytes.
* **Seed status**: Torrents stay active in the background with no interruption.

### Check That Hardlinks Work

1. Run this command in the host terminal:

```bash
ls -l -i ~/media/movies/SomeMovie/SomeMovie.mkv ~/media/downloads/SomeMovie.mkv
```

2. Look at the first number in each line of output. If hardlinks work, both files show the same inode number and a link count of `2`.

---

## Step-by-Step Service Setup

### 1. qBittorrent (`http://<ip>:8080`)

Recent `linuxserver/qbittorrent` images make a random temporary password when the container starts for the first time.

1. Run this command to get the temporary password from the logs:

   ```bash
   docker compose logs qbittorrent | grep -i "temporary password"
   ```

2. Open `http://<ip>:8080` and log in with username `admin` and the temporary password.
3. Go to **Tools** -> **Options** -> **Web UI** and change your password.
4. Go to **Tools** -> **Options** -> **Downloads** and set these paths:
   * **Default Save Path**: `/data/downloads`
   * **Keep incomplete torrents in**: `/data/downloads/incomplete`
5. Go to **Connection** and set the port for inbound connections to `6881`.
6. Go to **Speed** and set global rate limits to 80% of your maximum ISP upload speed to prevent network congestion.

> [!NOTE]
> If you use public trackers, route qBittorrent through a VPN container (such as Gluetun) to hide your home IP.

---

### 2. FlareSolverr (`http://<server-ip>:8191` or `http://flaresolverr:8191`)

FlareSolverr acts as a proxy for Prowlarr to solve Cloudflare and DDoS-Guard anti-bot checks on public torrent trackers.

* The host exposes port `8191:8191` so you can open `http://<server-ip>:8191` to check its status.
* Prowlarr connects directly to FlareSolverr over the internal Docker network at `http://flaresolverr:8191`.

---

### 3. Prowlarr (`http://<ip>:9696`)

Prowlarr manages all torrent and Usenet indexers from one place.

1. Add FlareSolverr as an indexer proxy:
   * Go to **Settings** -> **Indexers**.
   * Under **Proxy**, click **+** and select **FlareSolverr**.
   * Set **Host** to `http://flaresolverr:8191`.
   * Set **Name** to `FlareSolverr`.
   * Set **Tags** to `flaresolverr` and apply this tag to Cloudflare-protected trackers.
2. Add Radarr and Sonarr under **Settings** -> **Apps**:
   * Click **+** and select **Radarr**.
   * Set **Prowlarr Server** to `http://prowlarr:9696`.
   * Set **Radarr Server** to `http://radarr:7878`.
   * Copy the API key from Radarr (**Settings** -> **General** -> **API Key**) and paste it into **API Key**.
   * Click **+** and select **Sonarr**.
   * Set **Prowlarr Server** to `http://prowlarr:9696`.
   * Set **Sonarr Server** to `http://sonarr:8989`.
   * Copy the API key from Sonarr (**Settings** -> **General** -> **API Key**) and paste it into **API Key**.
3. Add indexers and sync profiles:
   * Go to **Indexers** and click **+ Add Indexer**.
   * Select your public or private trackers.
   * Click **Sync App Profiles** to send the indexers to Sonarr and Radarr.

---

### 4. Radarr and Sonarr

#### Set Root Folders

1. In Radarr, go to **Settings** -> **Media Management** -> **Root Folders** and add `/data/movies`.
2. In Sonarr, go to **Settings** -> **Media Management** -> **Root Folders** and add `/data/tv`.

#### Add the Download Client

Do these steps in both Sonarr and Radarr:

1. Go to **Settings** -> **Download Clients**, click **+**, and select **qBittorrent**.
2. Set **Host** to `qbittorrent` (the internal Docker container name).
3. Set **Port** to `8080`.
4. Enter your qBittorrent username and password.
5. Set **Category** to `tv` in Sonarr and `movies` in Radarr.
   * Categories put downloads into `/data/downloads/tv` and `/data/downloads/movies` so Sonarr and Radarr do not interfere with each other.
6. Click **Test** and then click **Save**.

---

### 5. Bazarr (`http://<ip>:6767`)

Bazarr manages subtitles for movies and TV shows.

1. Go to **Settings** -> **Sonarr** and enter these values:
   * **Address**: `sonarr`
   * **Port**: `8989`
   * **API Key**: Your Sonarr API key
2. Go to **Settings** -> **Radarr** and enter these values:
   * **Address**: `radarr`
   * **Port**: `7878`
   * **API Key**: Your Radarr API key
3. Go to **Settings** -> **Providers** and add OpenSubtitles.com, Subdl, or YIFY Subtitles.

---

### 6. Jellyfin (`http://<ip>:8096`)

#### Add Media Libraries

1. Open Jellyfin and complete the initial setup wizard.
2. Add a **Movies** library and select `/data/movies` as the folder.
3. Add a **Shows** library and select `/data/tv` as the folder.

> [!NOTE]
> The `docker-compose.yml` file mounts `/data/movies` and `/data/tv` as read-only (`:ro`) so Jellyfin cannot change your media files. Save library metadata and images inside the Jellyfin config directory instead of the media folders.

#### Isolate the Transcode Cache

1. Go to **Dashboard** -> **Playback** -> **Transcoding**.
2. Set **Transcode path** to `/cache`.
   * Jellyfin writes temporary transcode files to `~/media/cache/jellyfin` instead of your ZFS config backup snapshots.

#### Set Up Hardware Acceleration (GPU Transcode)

If your server has an Intel CPU with QuickSync or an AMD/NVIDIA GPU, do these steps:

1. In `docker-compose.yml`, remove the comment markers from the `devices` block under `jellyfin`:

   ```yaml
   devices:
     - /dev/dri:/dev/dri # Intel QuickSync / AMD VAAPI
   ```

2. Run this command to restart Jellyfin:

   ```bash
   docker compose up -d jellyfin
   ```

3. In Jellyfin, go to **Dashboard** -> **Playback** -> **Transcoding**.
4. Set **Hardware acceleration** to **Intel QuickSync (QSV)** or **VAAPI**.
5. Select hardware decode options for H264, HEVC, and VP9.

---

### 7. Jellyseerr (`http://<ip>:5055`)

Jellyseerr gives users a media request interface for movies and TV shows.

1. Open Jellyseerr and click **Sign in with Jellyfin**.
2. Enter your Jellyfin URL (`http://jellyfin:8096`) and admin credentials.
3. Connect Radarr with server `http://radarr:7878`, your Radarr API key, and root folder `/data/movies`.
4. Connect Sonarr with server `http://sonarr:8989`, your Sonarr API key, and root folder `/data/tv`.

Users can now search for movies or TV shows in Jellyseerr and click **Request** to start automatic download and import.
