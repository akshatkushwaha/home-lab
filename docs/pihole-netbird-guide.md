# Standalone Raspberry Pi Guide (Pi-hole and NetBird Cloud)

This guide gives step-by-step instructions to set up a Raspberry Pi with **Pi-hole** for network-wide ad block and **NetBird Cloud** as a routing peer. You do not need open router ports or an exit node for remote access.

---

## Architecture Goals

1. **High-Availability DNS**: DNS stays online when the main home server reboots, runs ZFS scrubs, or updates containers.
2. **Whole-Home Ad Block**: Pi-hole blocks ads, trackers, and telemetry on smart TVs, phones, tablets, and computers with no client software.
3. **Split-Tunnel Mobile DNS**: When you use 5G away from home, your phone connects to NetBird for ad block and LAN access at full cellular speed.
4. **Subnet Route Bridge**: The Pi bridges your NetBird network to your home network (`192.168.0.0/24`) so you can reach local services remotely.

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
6. Go to **Settings** -> **DNS** -> **Interface settings**, select **Permit all origins**, and click **Save**.
   * This setting lets Pi-hole answer DNS queries from the NetBird tunnel interface (`wt0`).

---

## Step 3: Install NetBird and Set Up Linux IP Forward

1. Open the [NetBird Cloud Dashboard](https://app.netbird.io) and make a free account.
2. Run this command to install NetBird on the Raspberry Pi:

```bash
curl -fsSL https://pkgs.netbird.io/install.sh | sh
```

### Turn On Linux IP Forward

NetBird needs Linux IP forward settings to route traffic to your home LAN subnet.

3. Run these commands to turn on IPv4 and IPv6 packet forward settings:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-netbird.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-netbird.conf
sudo sysctl -p /etc/sysctl.d/99-netbird.conf
```

### Connect the Raspberry Pi to NetBird Cloud

4. Run this command on the Raspberry Pi to start NetBird:

```bash
sudo netbird up
```

5. Open the authentication link shown in the terminal and log in to your NetBird Cloud account.

> [!NOTE]
> For headless setup, make a setup key under **Setup Keys** in NetBird Cloud and run `sudo netbird up --setup-key <your-setup-key>`.

---

## Step 4: Set Up Groups, Network Routes, and Policies

NetBird enforces access rules between client groups and network resources. Use separate network resources so `Friends and Family` reaches only `SHARED_IP` (`192.168.0.101`).

### A. Find Your LAN Subnet

1. Run this command on the Raspberry Pi to find your LAN subnet:

```bash
ip route | grep default
# Example output: default via 192.168.1.1 dev eth0 -> Subnet is 192.168.1.0/24
```

### B. Make the Groups in NetBird Cloud

2. Open the [NetBird Cloud Dashboard](https://app.netbird.io) and go to **Peers**.
3. Find your `pihole` peer and note its **NetBird IP (`100.x.y.z`)**.
4. Go to **Access Control** -> **Groups** and make these six groups:
   * `Admin`: Assign your personal laptop, phone, and workstation to this group.
   * `Raspberry`: Assign your Raspberry Pi (`pihole`) peer to this group.
   * `Friends and Family`: Assign devices for friends and family to this group.
   * `Private Services`: Target group for `192.168.0.100/32` (`PRIVATE_IP`).
   * `Shared Services`: Target group for `192.168.0.101/32` (`SHARED_IP`).
   * `Complete LAN`: Target group for your full LAN resource (`192.168.0.0/24`).

> [!NOTE]
> Remove `Admin` from the `pihole` peer if your user account auto-assigns groups. Do not use the built-in `All` group in policies.

### C. Add Three Network Resources

5. Go to **Networks** and click **Add Network** (for example, `Homelab LAN`).
6. Add the private server IP resource:
   * Click **Add Resource** and enter `192.168.0.100/32` (`PRIVATE_IP`).
   * Assign the resource to group `Private Services`.
   * Select `Raspberry` as the **Routing Peer**.
7. Add the shared server IP resource:
   * Click **Add Resource** and enter `192.168.0.101/32` (`SHARED_IP`).
   * Assign the resource to group `Shared Services`.
   * Select `Raspberry` as the **Routing Peer**.
8. Add the full LAN resource:
   * Click **Add Resource** and enter `192.168.0.0/24`.
   * Assign the resource to group `Complete LAN`.
   * Select `Raspberry` as the **Routing Peer**.

### D. Set Up Access Control Policies

9. Go to **Access Control** -> **Policies** and remove or turn off the default `All -> All` policy.
10. Add a policy for full `Admin` access:
    * **Source**: `Admin`
    * **Destination**: `Raspberry`, `Complete LAN`
    * **Protocol**: `All`
11. Add a policy for `Friends and Family` shared apps:
    * **Source**: `Friends and Family`
    * **Destination**: `Shared Services`
    * **Protocol and Ports**: `TCP` ports `443`, `8096`, `5055`, and `2283`
12. Add a policy for Pi-hole DNS queries:
    * **Source**: `Admin`, `Friends and Family`
    * **Destination**: `Raspberry`
    * **Protocol and Ports**: `UDP` port `53` and `TCP` port `53`

> [!NOTE]
> `Complete LAN` (`192.168.0.0/24`) already includes `.100` and `.101`, so `Admin` does not need separate `/32` routes. Keep `Private Services` ready for future per-host policies.

---

## Step 5: Set Up Split-Tunnel DNS for Mobile Devices

Set Pi-hole as the primary DNS nameserver across your NetBird network:

1. In the NetBird Cloud dashboard, go to **DNS** -> **Nameservers**.
2. Click **Add Nameserver**.
3. Enter the **NetBird `100.x.y.z` IP** of your Raspberry Pi (port `53`).
4. Leave **Match Domains** blank so all DNS queries use Pi-hole.
5. Assign the nameserver to `Admin` and `Friends and Family`, then click **Save**.
6. Install and turn on NetBird on your phone or laptop while on mobile data (5G), then do these checks:
   * Open `https://canyoublockit.com/testing/` to make sure ads do not load.
   * Check network speed to make sure only DNS queries go through the Pi while web traffic uses full cellular speed.
   * Open `http://<shared-ip>:2283` or `smb://<private-ip>/HomeDrive` to make sure remote LAN access works.

---

## Step 6: Set Up Router DHCP (Home-Wide Ad Block)

Set your router DHCP server to send Pi-hole DNS to all home devices:

1. Log in to your home router admin panel (usually `http://192.168.0.1` or `http://192.168.1.1`).
2. Go to the **DHCP Server** or **LAN** settings.
3. Set **DNS Server 1 (Primary)** to the static IP of your Raspberry Pi (for example, `192.168.0.50`).
4. Keep **DNS Server 2 (Secondary)** empty, or enter a second Pi-hole IP address.
   * If you enter `8.8.8.8` as a secondary DNS server, devices can bypass Pi-hole and show ads.
5. Save your settings and renew DHCP leases (or restart the router).

---

## Step 7: Set Up Local DNS and Domain Names

> [!WARNING]
> Route Vaultwarden through your custom domain (`https://vault.madcow.fyi`) with a valid SSL certificate in Shared NPM. Modern browsers need a secure HTTPS context (Web Crypto API) to encrypt Bitwarden vaults, and Let's Encrypt cannot issue certificates for local `.home.arpa` domains.

Point shared domains to `SHARED_IP` (`192.168.0.101`) and private domains to `PRIVATE_IP` (`192.168.0.100`) so NetBird blocks guest access to private domains:

1. In the Pi-hole Web UI, go to **Local DNS** -> **DNS Records**.
2. Add shared service records that point to `SHARED_IP` (`192.168.0.101`):
   * `jellyfin.madcow.fyi` -> `192.168.0.101`
   * `requests.madcow.fyi` -> `192.168.0.101`
   * `photos.madcow.fyi` -> `192.168.0.101`
   * `vault.madcow.fyi` -> `192.168.0.101`
3. Add private service records that point to `PRIVATE_IP` (`192.168.0.100`):
   * `portainer.madcow.fyi` -> `192.168.0.100`
   * `npm.madcow.fyi` -> `192.168.0.100`
   * `sonarr.madcow.fyi` -> `192.168.0.100`
   * `radarr.madcow.fyi` -> `192.168.0.100`
4. Add each proxy host in the correct Nginx Proxy Manager instance (`:81` for Private, `:8181` for Shared).
