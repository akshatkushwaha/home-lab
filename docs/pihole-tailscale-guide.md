# Standalone Raspberry Pi Guide (Pi-hole and Tailscale Subnet Router)

This guide gives step-by-step instructions to set up a Raspberry Pi with **Pi-hole** for network-wide ad block and **Tailscale** as a subnet router. You do not need an exit node for remote access.

---

## Architecture Goals

1. **High-Availability DNS**: DNS stays online when the main home server reboots, runs ZFS scrubs, or updates containers.
2. **Whole-Home Ad Block**: Pi-hole blocks ads, trackers, and telemetry on smart TVs, phones, tablets, and computers with no client software.
3. **Split-Tunnel Mobile DNS**: When you use 5G away from home, your phone connects to Tailscale for ad block and LAN access at full cellular speed.
4. **Subnet Route Bridge**: The Pi bridges your Tailnet to your home network (`192.168.0.0/24`) so you can reach local services remotely.

---

## Step 1: Install Raspberry Pi OS

1. Download the [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
2. Select **Raspberry Pi OS Lite (64-bit)** because a desktop interface is not necessary.
3. Click the gear icon (**Settings**) and set these options:
   * Set the hostname to `pihole`.
   * Turn on SSH with a password or an authorized public key.
   * Set your username and password.
   * Enter Wi-Fi credentials only if you cannot use an Ethernet cable (Ethernet gives better DNS reliability).
4. Write the image to the SD card and start your Raspberry Pi.
5. In your router admin panel, make a static DHCP reservation for the Pi (for example, `192.168.0.50`).

---

## Step 2: Install Pi-hole

1. Connect to the Raspberry Pi through SSH:

```bash
ssh <user>@192.168.0.50
```

2. Run the Pi-hole installation script:

```bash
curl -sSL https://install.pi-hole.net | bash
```

3. Select these options when the installer prompts you:
   * **Interface**: Select `eth0` (Ethernet) or `wlan0` (Wi-Fi).
   * **Upstream DNS Provider**: Select Cloudflare (`1.1.1.1`), Quad9 (`9.9.9.9`), or Google (`8.8.8.8`).
   * **Blocklist**: Accept the default StevenBlack unified hosts list.
   * **Web Admin Interface**: Select **Yes**.
   * **Web Server (lighttpd)**: Select **Yes**.

4. Set your web interface password after installation finishes:

```bash
sudo pihole -a -p
```

5. Open the Pi-hole dashboard at `http://192.168.0.50/admin`.

---

## Step 3: Install Tailscale and Set Up the Subnet Router

1. Run this command to install Tailscale on the Raspberry Pi:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

### Turn On Linux IP Forward

Tailscale needs Linux IP forward settings to work as a subnet router.

2. Run these commands to turn on IPv4 and IPv6 packet forward settings:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

### Advertise Your Home LAN Subnet

3. Run this command on the Raspberry Pi to find your LAN subnet:

```bash
ip route | grep default
# Example output: default via 192.168.1.1 dev eth0 -> Subnet is 192.168.1.0/24
```

4. Start Tailscale and replace `<your-lan-subnet>` with your network subnet (for example, `192.168.0.0/24` or `192.168.1.0/24`):

```bash
sudo tailscale up --advertise-routes=<your-lan-subnet> --accept-dns=false
```

5. Open the authentication link shown in the terminal to log in.

### Approve the Subnet Route in the Tailscale Admin Console

6. Open the [Tailscale Machines Admin Console](https://login.tailscale.com/admin/machines).
7. Find your `pihole` device.
8. Click the `...` menu and select **Edit route settings...**.
9. Select the box for your advertised subnet and click **Save**.

---

## Step 4: Set Up Split-Tunnel DNS for Mobile Devices

Set Pi-hole as the global DNS resolver across your Tailnet:

1. Open the [Tailscale DNS Admin Console](https://login.tailscale.com/admin/dns).
2. Under **Nameservers**, click **Add nameserver** and select **Custom**.
3. Enter the **Tailscale 100.x.y.z IP** of your Raspberry Pi (shown on the Machines page).
4. Set **Override local DNS** to **ON**.
5. Turn on Tailscale on your phone or laptop while on mobile data (5G) and do these checks:
   * Open `https://canyoublockit.com/testing/` to make sure ads do not load.
   * Check network speed to make sure only DNS queries go through the Pi while web traffic uses full cellular speed.
   * Open `http://<server-ip>:2283` or `smb://<server-ip>/HomeDrive` to make sure remote LAN access works.

---

## Step 5: Set Up Router DHCP (Home-Wide Ad Block)

Set your router DHCP server to send Pi-hole DNS to all home devices:

1. Log in to your home router admin panel (usually `http://192.168.0.1` or `http://192.168.1.1`).
2. Go to the **DHCP Server** or **LAN** settings.
3. Set **DNS Server 1 (Primary)** to the static IP of your Raspberry Pi (for example, `192.168.0.50`).
4. Keep **DNS Server 2 (Secondary)** empty, or enter a second Pi-hole IP address.
   * If you enter `8.8.8.8` as a secondary DNS server, devices can bypass Pi-hole and show ads.
5. Save your settings and renew DHCP leases (or restart the router).

---

## Step 6: Set Up Local DNS and Domain Names

> [!WARNING]
> Route Vaultwarden through your custom domain (`https://vault.madcow.fyi`) with a valid SSL certificate in Nginx Proxy Manager. Modern browsers need a secure HTTPS context (Web Crypto API) to encrypt Bitwarden vaults, and Let's Encrypt cannot issue certificates for local `.home.arpa` domains.

For services that do not need Web Crypto, or for internal routes:

1. In the Pi-hole Web UI, go to **Local DNS** -> **DNS Records**.
2. Add DNS records that point to your main server LAN IP address:
   * `immich.lan` -> `<main-server-ip>`
   * `jellyfin.lan` -> `<main-server-ip>`
3. Add the same proxy hosts in Nginx Proxy Manager if needed.
