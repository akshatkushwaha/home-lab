---
layout: default
title: "Madcow Homelab"
description: "Self-Hosted Infrastructure, Services, and Private Cloud Portal"
---

# Madcow Homelab

Welcome to **madcow.fyi**, the central portal for the Madcow private homelab and cloud infrastructure.

Bare-metal hardware hosts all services. Docker Compose runs the containers, a ZFS pool stores the data, and NetBird gives private network access with wildcard SSL.

---

> [!NOTE]
> Public traffic routes through encrypted tunnels. Administrative and storage services need an active **NetBird** connection or local network access (`madcow.lan`).

---

## Services and Portals

<div style="display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr)); gap: 1.25rem; margin-top: 1.5rem; margin-bottom: 2rem;">

  <!-- Vaultwarden -->
  <div style="border: 1px solid #e1e4e8; border-radius: 8px; padding: 1.25rem; background: rgba(0, 0, 0, 0.02); display: flex; flex-direction: column; justify-content: space-between;">
    <div>
      <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 0.5rem;">
        <h3 style="margin: 0; font-size: 1.2rem;">Vaultwarden</h3>
        <span style="font-size: 0.75rem; background: #e3f2fd; color: #0d47a1; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Security</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        Bitwarden-compatible encrypted password and secrets manager with Web Crypto API protection.
      </p>
    </div>
    <a href="https://vault.madcow.fyi" target="_blank" style="display: inline-block; text-align: center; background: #0366d6; color: #ffffff; padding: 8px 14px; border-radius: 6px; text-decoration: none; font-weight: 600; font-size: 0.85rem;">
      Open Vault &rarr;
    </a>
  </div>

  <!-- Immich -->
  <div style="border: 1px solid #e1e4e8; border-radius: 8px; padding: 1.25rem; background: rgba(0, 0, 0, 0.02); display: flex; flex-direction: column; justify-content: space-between;">
    <div>
      <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 0.5rem;">
        <h3 style="margin: 0; font-size: 1.2rem;">Immich</h3>
        <span style="font-size: 0.75rem; background: #f3e5f5; color: #4a148c; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Photos</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        Photo and video backup server with face recognition and mobile synchronization.
      </p>
    </div>
    <a href="https://photos.madcow.fyi" target="_blank" style="display: inline-block; text-align: center; background: #0366d6; color: #ffffff; padding: 8px 14px; border-radius: 6px; text-decoration: none; font-weight: 600; font-size: 0.85rem;">
      Open Immich &rarr;
    </a>
  </div>

  <!-- Jellyfin -->
  <div style="border: 1px solid #e1e4e8; border-radius: 8px; padding: 1.25rem; background: rgba(0, 0, 0, 0.02); display: flex; flex-direction: column; justify-content: space-between;">
    <div>
      <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 0.5rem;">
        <h3 style="margin: 0; font-size: 1.2rem;">Jellyfin</h3>
        <span style="font-size: 0.75rem; background: #fff3e0; color: #e65100; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Media</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        Open-source media server for movie, television, and audio libraries on all devices.
      </p>
    </div>
    <a href="https://jellyfin.madcow.fyi" target="_blank" style="display: inline-block; text-align: center; background: #0366d6; color: #ffffff; padding: 8px 14px; border-radius: 6px; text-decoration: none; font-weight: 600; font-size: 0.85rem;">
      Open Jellyfin &rarr;
    </a>
  </div>

  <!-- Jellyseerr -->
  <div style="border: 1px solid #e1e4e8; border-radius: 8px; padding: 1.25rem; background: rgba(0, 0, 0, 0.02); display: flex; flex-direction: column; justify-content: space-between;">
    <div>
      <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 0.5rem;">
        <h3 style="margin: 0; font-size: 1.2rem;">Jellyseerr</h3>
        <span style="font-size: 0.75rem; background: #fff3e0; color: #e65100; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Requests</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        Media search and automatic request portal connected to Radarr and Sonarr.
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
        <h3 style="margin: 0; font-size: 1.2rem;">Stirling-PDF</h3>
        <span style="font-size: 0.75rem; background: #e8f5e9; color: #1b5e20; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Utilities</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        Local PDF tool suite to split, merge, convert, OCR, and edit PDF files.
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
        <h3 style="margin: 0; font-size: 1.2rem;">Syncthing</h3>
        <span style="font-size: 0.75rem; background: #ede7f6; color: #311b92; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Storage</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        Continuous file synchronization across laptops, phones, and the ZFS storage pool.
      </p>
    </div>
    <a href="https://sync.madcow.fyi" target="_blank" style="display: inline-block; text-align: center; background: #0366d6; color: #ffffff; padding: 8px 14px; border-radius: 6px; text-decoration: none; font-weight: 600; font-size: 0.85rem;">
      Open Syncthing &rarr;
    </a>
  </div>

  <!-- Portainer -->
  <div style="border: 1px solid #e1e4e8; border-radius: 8px; padding: 1.25rem; background: rgba(0, 0, 0, 0.02); display: flex; flex-direction: column; justify-content: space-between;">
    <div>
      <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 0.5rem;">
        <h3 style="margin: 0; font-size: 1.2rem;">Portainer CE</h3>
        <span style="font-size: 0.75rem; background: #e0f2f1; color: #004d40; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Ops</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        Web interface for Docker containers, stacks, logs, and system resources.
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
        <h3 style="margin: 0; font-size: 1.2rem;">Proxy Manager</h3>
        <span style="font-size: 0.75rem; background: #e0f2f1; color: #004d40; padding: 2px 8px; border-radius: 12px; font-weight: 600;">Network</span>
      </div>
      <p style="font-size: 0.9rem; color: #586069; margin: 0.5rem 0 1rem 0;">
        Reverse proxy console for wildcard SSL certificates and domain routes.
      </p>
    </div>
    <a href="https://npm.madcow.fyi" target="_blank" style="display: inline-block; text-align: center; background: #0366d6; color: #ffffff; padding: 8px 14px; border-radius: 6px; text-decoration: none; font-weight: 600; font-size: 0.85rem;">
      Proxy Console &rarr;
    </a>
  </div>

</div>

---

## Automation and Media Stack (*arr)

The media pipeline uses TRaSH Guides atomic hardlinks for instant file organization with zero extra disk space:

| Service | Primary Subdomain | Purpose |
| :--- | :--- | :--- |
| **Sonarr** | [`sonarr.madcow.fyi`](https://sonarr.madcow.fyi) | TV series tracker and file organizer |
| **Radarr** | [`radarr.madcow.fyi`](https://radarr.madcow.fyi) | Movie collection manager |
| **Prowlarr** | [`prowlarr.madcow.fyi`](https://prowlarr.madcow.fyi) | Central indexer and tracker proxy |
| **Bazarr** | [`bazarr.madcow.fyi`](https://bazarr.madcow.fyi) | Automatic subtitle manager |
| **qBittorrent** | [`qbit.madcow.fyi`](https://qbit.madcow.fyi) | BitTorrent download client |

---

## Infrastructure Highlights

- **ZFS Dataset Isolation**: Separate SSD (`fast/appdata`, `fast/photos`, `fast/storage`, `fast/cache`) and HDD (`tank/media`) datasets use Sanoid snapshots and Syncoid backup.
- **Zero-Trust Remote Access**: The router keeps all inbound ports closed, and a standalone Raspberry Pi runs NetBird and Pi-hole DNS for remote access.
- **Unified Reverse Proxy**: Nginx Proxy Manager gets wildcard Let's Encrypt certificates through DNS-01 validation.

For full infrastructure documentation, setup instructions, and recovery procedures, see the [Homelab Repository Documentation](https://github.com/akshatkushwaha/home-lab).
