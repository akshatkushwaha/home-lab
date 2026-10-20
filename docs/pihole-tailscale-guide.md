# 🛡️ Standalone Raspberry Pi Guide (Pi-hole + Tailscale Subnet Router)

This guide walks you through setting up a dedicated Raspberry Pi running **Pi-hole** for network-wide ad blocking and **Tailscale** as a Subnet Router for secure remote access without requiring an exit node.

---

## 🎯 Architecture Goals

1. **High Availability DNS ("The Family Approval Factor")**: DNS remains 100% online even when your main home server is rebooting, running ZFS scrubs, or updating containers.
2. **Whole-Home Ad-Blocking**: Blocks ads, trackers, and telemetry on smart TVs, mobile phones, tablets, and PCs without client-side extensions.
3. **Split-Tunnel Mobile DNS**: When you are outside on 5G, your phone connects to Tailscale for ad-blocking and local homelab access with **full native 5G speed** (no slow exit node routing).
4. **Subnet Routing**: The Pi bridges your Tailnet to your entire home network (`192.168.0.0/24`). You can access all home server apps (Jellyfin, Immich, Portainer, etc.) remotely at their local IPs.

---

## 📋 Step 1: Raspberry Pi OS Installation

1. Download the [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
2. Select **Raspberry Pi OS Lite (64-bit)** (headless, no desktop environment needed).
3. Click the gear icon (Settings):
   * Set hostname: `pihole`
   * Enable SSH with password or authorized public key.
   * Set your username and password.
   * (Optional) Set Wi-Fi credentials if not using an Ethernet cable (Ethernet is strongly recommended for DNS).
4. Flash the SD card and boot your Raspberry Pi.
5. In your router's admin panel, create a **Static DHCP Reservation** for your Pi (e.g. `192.168.0.50`).

---

## 🛡️ Step 2: Pi-hole Installation

SSH into the Raspberry Pi:

```bash
ssh <user>@192.168.0.50
```

Run the official Pi-hole installation script:

```bash
curl -sSL https://install.pi-hole.net | bash
```

Follow the on-screen installer prompts:

* **Interface**: Choose `eth0` (Ethernet) or `wlan0` (Wi-Fi).
* **Upstream DNS Provider**: Choose Cloudflare (`1.1.1.1`), Quad9 (`9.9.9.9`), or Google (`8.8.8.8`).
* **Blocklist**: Accept the default StevenBlack unified hosts list.
* **Web Admin Interface**: Select **Yes**.
* **Web Server (lighttpd)**: Select **Yes**.

Once installation finishes, set your admin web interface password:

```bash
sudo pihole -a -p
```

Access the Pi-hole dashboard at `http://192.168.0.50/admin`.

---

## 🔒 Step 3: Tailscale Installation & Subnet Router

Install Tailscale on the Raspberry Pi:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

### Enable Linux IP Forwarding

Tailscale needs IP forwarding to act as a subnet router:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

### Start Tailscale Advertising Your Home LAN Subnet

Find your actual LAN subnet by checking your local IP and route on the Raspberry Pi:

```bash
ip route | grep default
# Example output: default via 192.168.1.1 dev eth0 -> Subnet is 192.168.1.0/24
```

Start Tailscale, replacing `<your-lan-subnet>` with your actual network subnet (e.g., `192.168.0.0/24` or `192.168.1.0/24`):

```bash
sudo tailscale up --advertise-routes=<your-lan-subnet> --accept-dns=false
```

*(Log in via the authentication link printed in the terminal).*

### Approve the Subnet Route in Tailscale Admin Console

1. Open the [Tailscale Machines Admin Console](https://login.tailscale.com/admin/machines).
2. Locate your `pihole` device.
3. Click the `...` menu → **Edit route settings...**.
4. Check the box for your advertised subnet and click **Save**.

---

## 📱 Step 4: Configure Split-Tunnel DNS for Mobile Devices

Now, make Pi-hole your global DNS resolver across your entire Tailnet:

1. Open the [Tailscale DNS Admin Console](https://login.tailscale.com/admin/dns).
2. Under **Nameservers**, click **Add nameserver** → **Custom**.
3. Enter your Raspberry Pi's **Tailscale 100.x.y.z IP** (found on the Machines page).
4. Toggle **Override local DNS** to **ON**.
5. Enable Tailscale on your phone or laptop while on mobile data (5G).
   * **Test Ad-blocking**: Visit `https://canyoublockit.com/testing/` to verify ads are blocked.
   * **Test Speed**: Runs at full cellular speed because only DNS queries route through the Pi, not bulk media or web traffic!
   * **Test Home Server Access**: Open your home server's LAN IP (e.g. `http://<server-ip>:2283` or `smb://<server-ip>/HomeDrive`) to access Immich or your personal files remotely.

---

## 🏠 Step 5: Router DHCP Configuration (Home-Wide Ad Blocking)

To protect all smart TVs, laptops, and home devices without installing software on each:

1. Log in to your home router admin panel (usually `http://192.168.0.1` or `http://192.168.1.1`).
2. Navigate to the **DHCP Server / LAN** settings.
3. Change **DNS Server 1 (Primary)** to your Raspberry Pi's reserved static IP (e.g. `192.168.0.50`).
4. Leave **DNS Server 2 (Secondary)** empty (or point to a secondary Pi-hole if running redundant DNS).
   *(Note: If you put 8.8.8.8 as secondary DNS, devices will periodically query Google DNS and show ads).*
5. Save settings and renew DHCP leases (or reboot the router).

---

## 🏷️ Step 6: Local DNS & Domain Resolution

To access your homelab services via friendly domain names:

> [!WARNING]
> **HTTPS Requirement for Vaultwarden**: Modern browsers strictly require a secure HTTPS context (via the Web Crypto API) to encrypt and decrypt Bitwarden vaults. Let's Encrypt cannot issue public SSL certificates for local domains like `.home.arpa`. For Vaultwarden, route through your DuckDNS domain (`https://vault.yourdomain.duckdns.org`) with the wildcard SSL certificate in Nginx Proxy Manager.

For services that do not require Web Crypto (or when routing internally):

1. In the Pi-hole Web UI, navigate to **Local DNS** → **DNS Records**.
2. Add records pointing to your main server LAN IP:
   * `immich.lan` → `<main-server-ip>`
   * `jellyfin.lan` → `<main-server-ip>`
3. Configure Nginx Proxy Manager to route these hostnames if desired.
