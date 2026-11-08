---
layout: default
title: "Madcow Homelab"
description: "Self-Hosted Infrastructure, Services & Private Cloud Portal"
---

# 🛸 Madcow Homelab

Welcome to **madcow.fyi** — the central access portal for the Madcow private homelab and cloud infrastructure.

All services are hosted on dedicated bare-metal hardware, orchestrated via **Docker Compose**, backed by an automated **ZFS storage pool**, and secured with wildcard SSL and isolated private network routing via **Tailscale**.

---

> [!NOTE]
> **Access Notice**: Public internet traffic routes securely through encrypted tunnels. Many administrative and storage services require an active **Tailscale mesh connection** or local network access (`madcow.lan`).

---

## 🚀 Services & Portals

<div style="display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr)); gap: 1.25rem; margin-top: 1.5rem; margin-bottom: 2rem;">

  <!-- Vaultwarden -->
  <div style="border: 1px solid #e1e4e8; border-radius: 8px; padding: 1.25rem; background: rgba(0, 0, 0, 0.02); display: flex; flex-direction: column; justify-content: space-between;">
    <div>
      <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 0.5rem;">
        <h3 style="margin: 0; font-size: 1.2rem;">🔐 Vaultwarden</h3>
        <span style="font-size: 0.75rem; background: #e3f2fd; color: #0d47a1; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Security</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        Bitwarden-compatible end-to-end encrypted password and secrets manager with Web Crypto API security.
      </p>
    </div>
    <a href="https://vault.madcow.fyi" target="_blank" style="display: inline-block; text-align: center; background: #0366d6; color: #ffffff; padding: 8px 14px; border-radius: 6px; text-decoration: none; font-weight: 600; font-size: 0.85rem;">
      Launch Vault &rarr;
    </a>
  </div>

  <!-- Immich -->
  <div style="border: 1px solid #e1e4e8; border-radius: 8px; padding: 1.25rem; background: rgba(0, 0, 0, 0.02); display: flex; flex-direction: column; justify-content: space-between;">
    <div>
      <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 0.5rem;">
        <h3 style="margin: 0; font-size: 1.2rem;">📸 Immich</h3>
        <span style="font-size: 0.75rem; background: #f3e5f5; color: #4a148c; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Photos</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        High-performance photo and video backup solution with ML face recognition and mobile synchronization.
      </p>
    </div>
    <a href="https://photos.madcow.fyi" target="_blank" style="display: inline-block; text-align: center; background: #0366d6; color: #ffffff; padding: 8px 14px; border-radius: 6px; text-decoration: none; font-weight: 600; font-size: 0.85rem;">
      Open Immich &rarr;
    </a>
  </div>

  <!-- Paperless-ngx -->
  <div style="border: 1px solid #e1e4e8; border-radius: 8px; padding: 1.25rem; background: rgba(0, 0, 0, 0.02); display: flex; flex-direction: column; justify-content: space-between;">
    <div>
      <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 0.5rem;">
        <h3 style="margin: 0; font-size: 1.2rem;">📄 Paperless-ngx</h3>
        <span style="font-size: 0.75rem; background: #e8f5e9; color: #1b5e20; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Documents</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        Full-text searchable automated document archiving with OCR, tagging, and correspondence indexing.
      </p>
    </div>
    <a href="https://paperless.madcow.fyi" target="_blank" style="display: inline-block; text-align: center; background: #0366d6; color: #ffffff; padding: 8px 14px; border-radius: 6px; text-decoration: none; font-weight: 600; font-size: 0.85rem;">
      Open Documents &rarr;
    </a>
  </div>

  <!-- Jellyfin -->
  <div style="border: 1px solid #e1e4e8; border-radius: 8px; padding: 1.25rem; background: rgba(0, 0, 0, 0.02); display: flex; flex-direction: column; justify-content: space-between;">
    <div>
      <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 0.5rem;">
        <h3 style="margin: 0; font-size: 1.2rem;">🍿 Jellyfin</h3>
        <span style="font-size: 0.75rem; background: #fff3e0; color: #e65100; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Media</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        Open-source media streaming server for movies, television, and audio collections across all screens.
      </p>
    </div>
    <a href="https://jellyfin.madcow.fyi" target="_blank" style="display: inline-block; text-align: center; background: #0366d6; color: #ffffff; padding: 8px 14px; border-radius: 6px; text-decoration: none; font-weight: 600; font-size: 0.85rem;">
      Watch Jellyfin &rarr;
    </a>
  </div>

  <!-- Jellyseerr -->
  <div style="border: 1px solid #e1e4e8; border-radius: 8px; padding: 1.25rem; background: rgba(0, 0, 0, 0.02); display: flex; flex-direction: column; justify-content: space-between;">
    <div>
      <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 0.5rem;">
        <h3 style="margin: 0; font-size: 1.2rem;">✨ Jellyseerr</h3>
        <span style="font-size: 0.75rem; background: #fff3e0; color: #e65100; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Requests</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        Clean media discovery and automated request system integrated with Radarr and Sonarr.
      </p>
    </div>
    <a href="https://requests.madcow.fyi" target="_blank" style="display: inline-block; text-align: center; background: #0366d6; color: #ffffff; padding: 8px 14px; border-radius: 6px; text-decoration: none; font-weight: 600; font-size: 0.85rem;">
      Request Media &rarr;
    </a>
  </div>

  <!-- Stirling PDF -->
  <div style="border: 1px solid #e1e4e8; border-radius: 8px; padding: 1.25rem; background: rgba(0, 0, 0, 0.02); display: flex; flex-direction: column; justify-content: space-between;">
    <div>
      <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 0.5rem;">
        <h3 style="margin: 0; font-size: 1.2rem;">📑 Stirling-PDF</h3>
        <span style="font-size: 0.75rem; background: #e8f5e9; color: #1b5e20; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Utilities</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        Locally hosted PDF manipulation Swiss Army knife: split, merge, convert, OCR, and edit.
      </p>
    </div>
    <a href="https://pdf.madcow.fyi" target="_blank" style="display: inline-block; text-align: center; background: #0366d6; color: #ffffff; padding: 8px 14px; border-radius: 6px; text-decoration: none; font-weight: 600; font-size: 0.85rem;">
      Open PDF Tools &rarr;
    </a>
  </div>

  <!-- Syncthing -->
  <div style="border: 1px solid #e1e4e8; border-radius: 8px; padding: 1.25rem; background: rgba(0, 0, 0, 0.02); display: flex; flex-direction: column; justify-content: space-between;">
    <div>
      <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 0.5rem;">
        <h3 style="margin: 0; font-size: 1.2rem;">🔄 Syncthing</h3>
        <span style="font-size: 0.75rem; background: #ede7f6; color: #311b92; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Storage</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        Decentralized, continuous file synchronization across laptops, phones, and the central ZFS storage pool.
      </p>
    </div>
    <a href="https://sync.madcow.fyi" target="_blank" style="display: inline-block; text-align: center; background: #0366d6; color: #ffffff; padding: 8px 14px; border-radius: 6px; text-decoration: none; font-weight: 600; font-size: 0.85rem;">
      Manage Sync &rarr;
    </a>
  </div>

  <!-- Portainer -->
  <div style="border: 1px solid #e1e4e8; border-radius: 8px; padding: 1.25rem; background: rgba(0, 0, 0, 0.02); display: flex; flex-direction: column; justify-content: space-between;">
    <div>
      <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 0.5rem;">
        <h3 style="margin: 0; font-size: 1.2rem;">🐳 Portainer CE</h3>
        <span style="font-size: 0.75rem; background: #e0f2f1; color: #004d40; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Ops</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        Web interface for Docker container lifecycle, stack inspection, logs, and resource management.
      </p>
    </div>
    <a href="https://portainer.madcow.fyi" target="_blank" style="display: inline-block; text-align: center; background: #0366d6; color: #ffffff; padding: 8px 14px; border-radius: 6px; text-decoration: none; font-weight: 600; font-size: 0.85rem;">
      Docker Admin &rarr;
    </a>
  </div>

  <!-- Nginx Proxy Manager -->
  <div style="border: 1px solid #e1e4e8; border-radius: 8px; padding: 1.25rem; background: rgba(0, 0, 0, 0.02); display: flex; flex-direction: column; justify-content: space-between;">
    <div>
      <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 0.5rem;">
        <h3 style="margin: 0; font-size: 1.2rem;">🌐 Proxy Manager</h3>
        <span style="font-size: 0.75rem; background: #e0f2f1; color: #004d40; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Networking</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        Reverse proxy administration, wildcard SSL certificate provisioning, and ingress domain routing.
      </p>
    </div>
    <a href="https://npm.madcow.fyi" target="_blank" style="display: inline-block; text-align: center; background: #0366d6; color: #ffffff; padding: 8px 14px; border-radius: 6px; text-decoration: none; font-weight: 600; font-size: 0.85rem;">
      Proxy Console &rarr;
    </a>
  </div>

  <!-- Uptime Kuma -->
  <div style="border: 1px solid #e1e4e8; border-radius: 8px; padding: 1.25rem; background: rgba(0, 0, 0, 0.02); display: flex; flex-direction: column; justify-content: space-between;">
    <div>
      <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 0.5rem;">
        <h3 style="margin: 0; font-size: 1.2rem;">📊 Uptime Kuma</h3>
        <span style="font-size: 0.75rem; background: #e8f5e9; color: #1b5e20; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Monitoring</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        Real-time uptime status monitoring, response time metrics, and service heartbeat alerts.
      </p>
    </div>
    <a href="https://status.madcow.fyi" target="_blank" style="display: inline-block; text-align: center; background: #0366d6; color: #ffffff; padding: 8px 14px; border-radius: 6px; text-decoration: none; font-weight: 600; font-size: 0.85rem;">
      Status Page &rarr;
    </a>
  </div>

  <!-- Homepage Dashboard -->
  <div style="border: 1px solid #e1e4e8; border-radius: 8px; padding: 1.25rem; background: rgba(0, 0, 0, 0.02); display: flex; flex-direction: column; justify-content: space-between;">
    <div>
      <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 0.5rem;">
        <h3 style="margin: 0; font-size: 1.2rem;">⚡ Homepage</h3>
        <span style="font-size: 0.75rem; background: #fce4ec; color: #880e4f; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Dashboard</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        Consolidated homelab dashboard integrating live metrics, system resources, and service shortcuts.
      </p>
    </div>
    <a href="https://dash.madcow.fyi" target="_blank" style="display: inline-block; text-align: center; background: #0366d6; color: #ffffff; padding: 8px 14px; border-radius: 6px; text-decoration: none; font-weight: 600; font-size: 0.85rem;">
      Homelab Dashboard &rarr;
    </a>
  </div>

</div>

---

## ⚡ Automation & Media Stack (*arr)

The automated content pipeline relies on TRaSH Guides zero-copy atomic hardlinks for instant, zero-storage file organization:

| Service | Primary Subdomain | Purpose |
| :--- | :--- | :--- |
| **Sonarr** | [`sonarr.madcow.fyi`](https://sonarr.madcow.fyi) | TV series tracker & automated file organizer |
| **Radarr** | [`radarr.madcow.fyi`](https://radarr.madcow.fyi) | Movie collection manager & automated pipeline |
| **Prowlarr** | [`prowlarr.madcow.fyi`](https://prowlarr.madcow.fyi) | Centralized indexer and tracker proxy manager |
| **Bazarr** | [`bazarr.madcow.fyi`](https://bazarr.madcow.fyi) | Automated multi-language subtitle synchronizer |
| **qBittorrent** | [`qbit.madcow.fyi`](https://qbit.madcow.fyi) | High-throughput BitTorrent download client |

---

## 🛡️ Infrastructure Highlights

- **ZFS Dataset Isolation**: Dedicated ZFS datasets (`tank/appdata`, `tank/photos`, `tank/storage`, `tank/media`) with Sanoid automated snapshots and Syncoid offsite backup.
- **Zero-Trust Remote Access**: No exposed inbound router ports; remote connectivity is established via an isolated Raspberry Pi running Tailscale and Pi-hole DNS.
- **Unified Reverse Proxy**: Wildcard Let's Encrypt certificates generated via DNS-01 validation.

For the full infrastructure documentation, deployment instructions, and disaster recovery guides, visit the [Homelab Repository Documentation](https://github.com/akshatkushwaha/home-lab).

