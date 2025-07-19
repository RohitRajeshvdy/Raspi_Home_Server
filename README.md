# 🍓 Raspberry Pi 4 Home Server Setup

A self-hosted, Docker-based home server running on Raspberry Pi 4. Includes media streaming, ad-blocking, automated downloads, photo management, and system monitoring.

---

## 📏 Table of Contents

1. [Hardware Used](#-hardware-used)
2. [Initial Raspberry Pi Setup](#-initial-raspberry-pi-setup)
3. [Static IP Setup](#-static-ip-setup)
4. [External Drive Formatting & Mounting](#-external-drive-formatting--mounting)
5. [Docker & Docker Compose Installation](#-docker--docker-compose-installation)
6. [Remote Access](#-remote-access)
7. [Services (Docker Compose)](#-services-docker-compose)

   * [Portainer](#-portainer)
   * [Nginx](#-nginx)
   * [Pi-hole](#-pi-hole)
   * [Glances](#-glances)
   * [Uptime Kuma](#-uptime-kuma)
   * [Immich](#-immich)
   * [Jellyfin + Jellyseerr](#-jellyfin--jellyseerr)
   * [\*arr Stack + qBittorrent](#-arr-stack--qbittorrent-setup)

     * [qBittorrent](#-qbittorrent)
     * [Prowlarr](#-prowlarr)
     * [Radarr](#-radarr)
     * [Sonarr](#-sonarr)
   * [Homarr](#-homarr)
   * [File Browser](#-file-browser)
8. [Backups & Data Safety](#-backups--data-safety)

---


## 🛠️ Hardware Used

* Raspberry Pi 4 (4GB or 8GB)
* SD Card for OS
* External SSD/HDD (formatted to ext4)
* Ethernet connection to router (recommended)

---

## 🛠️ Initial Raspberry Pi Setup

1. Flash **Raspberry Pi OS Lite** (64-bit) using [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
2. Enable SSH:

   * Enable it in the settings before flashing
3. Connect via SSH:

   ```bash
   ssh pi@<raspberry_ip>
   ```

---

## 📁 Static IP Setup

1. Open the DHCP configuration file:

   ```bash
   sudo nano /etc/dhcpcd.conf
   ```

2. Scroll to the bottom and add the following lines:

   ```ini
   interface eth0
   static ip_address=192.168.31.100/24
   static routers=192.168.31.1
   static domain_name_servers=1.1.1.1 8.8.8.8
   ```

   > Replace `192.168.31.100` with your desired static IP and `192.168.31.1` with your router's IP.

3. Save the file and reboot:

   ```bash
   sudo reboot
   ```

---

## 📍 External Drive Formatting & Mounting

1. Plug in your external SSD or HDD.

2. List available disks to identify your drive (usually `/dev/sda1`):

   ```bash
   lsblk
   ```

3. Format the drive to ext4 (WARNING: this erases all data on the drive):

   ```bash
   sudo mkfs.ext4 /dev/sda1
   ```

4. Create a mount point:

   ```bash
   sudo mkdir -p /mnt/hdd
   ```

5. Find the UUID of your drive:

   ```bash
   sudo blkid
   ```

6. Edit the `/etc/fstab` file to auto-mount on boot:

   ```bash
   sudo nano /etc/fstab
   ```

   Add this line at the end (replace `XXXX-XXXX` with your actual UUID):

   ```fstab
   UUID=XXXX-XXXX  /mnt/hdd  ext4  defaults,noatime  0  2
   ```

7. Mount the drive immediately:

   ```bash
   sudo mount -a
   ```

---

## 🐳 Docker & Docker Compose Installation

1. Update and upgrade your system:

   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. Install Docker using the official convenience script:

   ```bash
   curl -sSL https://get.docker.com | sh
   ```

3. Add your user to the Docker group:

   ```bash
   sudo usermod -aG docker $USER
   ```

   > Log out and back in or reboot to apply group changes.

4. Docker Compose is now included as a plugin in modern Docker versions. You can use it like this:

   ```bash
   docker compose version
   ```

   > No need to install it separately. Use `docker compose` (with a space) instead of `docker-compose`.

5. Confirm installation:

   ```bash
   docker --version
   docker compose version
   ```

---

## 🌐 Remote Access

To access your home server from outside your local network securely, use [Tailscale](https://tailscale.com), a zero-config WireGuard-based VPN.

### Install Tailscale:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

### Authenticate and Enable Subnet Routing:

```bash
sudo tailscale up --advertise-routes=192.168.31.0/24
```

This advertises your Raspberry Pi as a **subnet router** so you can access all local IP devices (e.g., `192.168.31.x`) from any device in your Tailscale network.

> Don't forget to approve the subnet route in the [Tailscale admin panel](https://login.tailscale.com/admin/machines) under your Pi's device settings.

### ⚠️ Optional Step:

Subnet routing is **not required** if you're okay with accessing services using only the Pi's Tailscale IP (e.g., `100.x.x.x`). In that case, you can simply use:

```bash
sudo tailscale up
```

### Check connection:

```bash
tailscale status
```

> After this, you can securely access your Raspberry Pi and local services using either its Tailscale IP (e.g., `100.x.x.x`) or local LAN IP (e.g., `192.168.31.x`).

---

## 🧩 Services

All services are organized in individual folders, each containing its own `docker-compose.yml` file.

### 📁 Clone Service Repository

Create a directory for your Docker setup and clone your repository:

```bash
mkdir ~/docker
cd ~/docker
git clone https://github.com/RohitRajeshvdy/Raspi-Home-Server.git .
```

> This will clone all folders and files from your GitHub repo into the `~/docker` directory on your Raspberry Pi.

---

## 🔧 Portainer

**Portainer** is a web UI for managing your Docker containers and stacks.

#### 📦 Deploy Portainer

```bash
cd ~/docker/portainer
docker compose up -d
```

> To modify the configuration:
```bash
nano docker-compose.yml
```

#### 🌐 Access Portainer

Open your browser and go to:

```
http://<raspberry_pi_ip>:9000
```

Set your admin password to complete the initial setup.

---

## 🌐 NGINX Proxy Manager

**NGINX Proxy Manager** provides an easy interface to manage reverse proxies, domain names, and SSL certificates.

#### 📦 Deploy NGINX

```bash
cd ~/docker/nginx
docker compose up -d
```

Access the web UI at:

```
http://<raspberry_pi_ip>:81
```

- **Default login:**
  - Username: `admin@example.com`
  - Password: `changeme`
- Change credentials on first login.

#### 🛡️ DuckDNS & SSL Setup

1. Go to [DuckDNS](https://www.duckdns.org/) and sign in using any provider.
2. Create a domain, e.g., `yourname.duckdns.org`
3. Update the IP to your Raspberry Pi’s local IP and copy the token.

In NGINX Proxy Manager:

1. Go to **SSL Certificates** → **Add SSL Certificate**.
2. Choose **Let’s Encrypt**, enter:
   - Domain Names: `yourname.duckdns.org`, `*.yourname.duckdns.org`
   - Enable **DNS Challenge**
   - Choose **DuckDNS**
   - Enter your DuckDNS token
3. If validation fails, increase propagation delay to 30 seconds.

> ✅ This is a one-time certificate setup. You do not need to repeat this for other services.

#### 🌍 Create Proxy Host (e.g., Portainer)

1. Navigate to **Hosts → Proxy Hosts → Add Proxy Host**
2. Fill in:
   - **Domain Names**: `portainer.yourname.duckdns.org`
   - **Forward Hostname/IP**: `<raspberry_pi_ip>`
   - **Forward Port**: `9000`
   - Enable:
     - `Websockets Support`
     - `Block Common Exploits`
     - `Websockets Support`
3. Go to the **SSL** tab:
   - Select the certificate you created
   - Enable `Force SSL` and `HTTP/2 Support`

You can now access Portainer via:

```
https://portainer.yourname.duckdns.org
```

> 🔁 Repeat these proxy host steps for each service you deploy below.

## 🚫 Pi-hole

### 📂 Step 1: Navigate to the Pi-hole directory

```bash
cd ~/docker/pihole
```

### ✏️ Step 2: Edit the docker-compose file

```bash
nano docker-compose.yml
```

Update the following lines:

* Change the `TZ` to your timezone (e.g., `Asia/Kolkata`)
* Set a secure `WEBPASSWORD` to access the Pi-hole dashboard

Example:

```yaml
    environment:
      - TZ=Asia/Kolkata
      - WEBPASSWORD=your_strong_password
```

> Modify other settings (like ports or volumes) only if needed.

### ▶️ Step 3: Start Pi-hole using Docker Compose

```bash
docker compose up -d
```

### 🌐 Step 4: Access the Pi-hole Web Interface

Open your browser and visit:

```
http://<your-raspberry-pi-ip>:8080
```

> Log in using the password you set in the `WEBPASSWORD` field.

---

## 🧠 How to Use Pi-hole for Network-Wide Ad Blocking

Choose one of the two DNS setup options below:

---

### ✅ Option 1: Set Router DNS (Recommended)

1. Login to your router admin panel (usually `192.168.1.1` or similar).
2. Find the DNS settings (under LAN or DHCP settings).
3. Set the **Primary DNS** to your Raspberry Pi IP (e.g., `192.168.31.100`)
4. Save and reboot the router.

> This routes all devices on your network through Pi-hole automatically.

---

### 🔧 Option 2: Set DNS Per Device

If router DNS change isn't possible:

1. On each device (PC, mobile, etc), go to network settings.
2. Manually set the DNS to your Raspberry Pi’s IP (e.g., `192.168.31.100`)
3. Leave the secondary DNS blank or use `1.1.1.1` as fallback (optional).

---

🎉 Done! Pi-hole should now be actively blocking ads for your chosen devices.


## 📈 Glances

### 📂 Step 1: Navigate to the Glances directory

```bash
cd ~/docker/glances
```

### ✏️ Step 2: Edit the docker-compose file

```bash
nano docker-compose.yml
```

Update the following line:

* Change the `TZ` to your timezone (e.g., `Asia/Kolkata`)

Example:

```yaml
    environment:
      - TZ=Asia/Kolkata
```

> No other changes needed unless you want to adjust port mapping or volume paths.

### ▶️ Step 3: Start Glances using Docker Compose

```bash
docker compose up -d
```

### 🌐 Step 4: Access the Glances Web UI

Open your browser and go to:

```
http://<your-raspberry-pi-ip>:61208
```

> Glances provides real-time system monitoring directly in your browser.

---

🎉 Done! You now have system resource monitoring via Glances.


## 🟢 Uptime Kuma

### 📂 Step 1: Navigate to the Uptime Kuma directory

```bash
cd ~/docker/uptime-kuma
```

### ▶️ Step 2: Start Uptime Kuma using Docker Compose

```bash
docker compose up -d
```

> No need to edit the compose file — it's ready to go!

### 🌐 Step 3: Access the Uptime Kuma Web Interface

Open your browser and go to:

```
http://<your-raspberry-pi-ip>:3001
```

Create an admin account on first login.

---

### 📊 Step 4: Add Services to Monitor

Once inside the dashboard:

1. Click **"Add New Monitor"**.
2. Enter a friendly name (e.g., "Pi-hole")
3. Enter the URL/IP and port (e.g., `http://192.168.31.100:8080`)
4. Click **"Save"**.

Repeat this process for each service you want to monitor (e.g., Glances, Jellyfin, qBittorrent).

---

🎉 Done! Uptime Kuma is now monitoring your services and will notify you of any downtime.


## 🖼️ Immich

> ⚠️ **Warning:** Immich is resource-heavy when Machine Learning is enabled. On a Raspberry Pi 4, it works perfectly fine **without** Machine Learning. You can still enable it, but expect **high RAM and CPU usage**.

**Machine Learning** in Immich enables:

* Face recognition
* Object detection
* Image classification
* Auto-tagging

If you **do not need these features**, it's strongly recommended to **comment out the entire `machine-learning` service** in the `docker-compose.yml` file to keep your system responsive.

---

### 📂 Step 1: Navigate to the Immich directory

```bash
cd ~/docker/immich
```

### ✏️ Step 2: Edit the docker-compose.yml file

```bash
nano docker-compose.yml
```

* Locate the section starting with `machine-learning:` and **comment out** that entire block using `#`.
* Save and exit.

---

### 🛠️ Step 3: Edit the .env file

```bash
nano .env
```

Update the storage path to your mounted external drive. Example:

```env
UPLOAD_LOCATION=/mnt/hdd/immich
```

> Make sure this folder exists. If not, create it using:

```bash
sudo mkdir -p /mnt/hdd/immich
sudo chown -R pi:pi /mnt/hdd/immich
```

* `mkdir -p` creates the folder (including any missing parent directories).
* `chown -R pi:pi` gives ownership of the folder (and all files inside) to the `pi` user.

---

### ▶️ Step 4: Start Immich using Docker Compose

```bash
docker compose up -d
```

---

### 🌐 Step 5: Access the Immich Web Interface

Open your browser and visit:

```
http://<your-raspberry-pi-ip>:2283
```

Create your admin account and start uploading and organizing your photos.

---

🎉 Done! Immich is now running. Use it to back up and manage your photo library.

> 🧠 If you ever upgrade to a more powerful server, you can re-enable the machine learning features for smarter photo organization.


## 🎬 Jellyfin + 🧠 Jellyseerr

### 📁 Step 1: Prepare Folder Structure

Before deploying Jellyfin and Jellyseerr, create the required directory structure on your external drive:

```bash
sudo mkdir -p /mnt/hdd/data/{books,movies,shows,downloads/qbittorrent/{completed/{radarr,sonarr,unsorted},incomplete/{radarr,sonarr,unsorted},torrents}}
sudo chown -R pi:pi /mnt/hdd/data
```

> This sets up dedicated folders for all your media content and download organization.

---

### 📂 Step 2: Navigate to the Jellyfin directory

```bash
cd ~/docker/jellyfin
```

### ✏️ Step 3: Edit the docker-compose.yml file

```bash
nano docker-compose.yml
```

Make the following changes:

* Set the correct timezone for both `jellyfin` and `jellyseerr` services:

```yaml
    environment:
      - TZ=Asia/Kolkata
```

* Update the volume mappings to match your Raspberry Pi setup:

```yaml
    volumes:
      - /home/mcper/docker/jellyfin/config:/config
      - /home/mcper/docker/jellyfin/cache:/cache
      - /mnt/hdd/data:/media
```

For Jellyseerr:

```yaml
    volumes:
      - /home/mcper/docker/jellyfin/jellyseerr/config:/app/config
```

> These mount paths ensure your media files and configurations are persisted and accessible by both containers.

---

### ▶️ Step 4: Start the Stack

```bash
docker compose up -d
```

---

### 🌐 Step 5: Access the Web Interfaces

* **Jellyfin:**

```
http://<your-raspberry-pi-ip>:8096
```

* **Jellyseerr:**

```
http://<your-raspberry-pi-ip>:5055
```

Set up your admin accounts and start configuring media libraries and request handling.

---

🎉 Done! Jellyfin will serve your media and Jellyseerr will help users request new content to be automatically downloaded.

## 🌪️ \*arr Stack + qBittorrent Setup

This section covers installing and configuring the full \*arr stack: qBittorrent, Sonarr, Radarr, Lidarr, Prowlarr, and Bazarr.

---

### 📂 Step 1: Navigate to the arr directory

```bash
cd ~/docker/arr
```

---

### ✏️ Step 2: Edit the `docker-compose.yml` File

```bash
nano docker-compose.yml
```

Ensure all services use the correct data mount:

```yaml
    volumes:
      - /mnt/hdd/data:/data
```

> This is required for media file access and consistent storage.

---

### ⚙️ Step 3: Edit the `.env` File

```bash
nano .env
```

Set your timezone and user IDs:

```env
TZ=Asia/Kolkata
PUID=1000
PGID=1000
```

---

### ▶️ Step 4: Start All Services

```bash
docker compose up -d
```

---

### 🧲 Step 5: qBittorrent Setup

After the services start, check qBittorrent logs to find the default login credentials:

```bash
docker logs qbittorrent
```

Access the web interface:

```
http://<raspberry-pi-ip>:8080
```

> Use the credentials from the logs, then immediately change them in:

```
Settings → Web UI → Username / Password
```

---

### 📺 Step 6: Configure the Other Services

* **Sonarr:** http\://<raspberry-pi-ip>:8989
* **Radarr:** http\://<raspberry-pi-ip>:7878
* **Lidarr:** http\://<raspberry-pi-ip>:8686
* **Bazarr:** http\://<raspberry-pi-ip>:6767
* **Prowlarr:** http\://<raspberry-pi-ip>:9696

All services are now running and accessible via your Pi’s IP address.

---

### 🎥 Final Step: Follow Setup Video

For full configuration of the \*arr automation system (Sonarr/Radarr + Prowlarr + qBittorrent), watch this video:

▶️ [Automated Torrent Media Server Setup - YouTube](https://youtu.be/twJDyoj0tDc?si=L5tVUr_hbUi_hDaJ)

It covers indexers, download clients, and organizing your media end-to-end.

---

🎉 Done! Your automated media server is live and ready to use.

## 🏠 Homarr

### 📂 Step 1: Navigate to the Homarr directory

```bash
cd ~/docker/homarr
```

### ▶️ Step 2: Deploy Homarr

```bash
docker compose up -d
```

> No file edits required — it's plug and play.

### 🌐 Step 3: Open the Dashboard

```
http://<raspberry-pi-ip>:7575
```

### 🧩 Step 4: Customize Your Homepage

1. Add widgets for Jellyfin, Pi-hole, etc.
2. Set logos, names, and links.
3. Save the layout for quick service access.

---

🎉 Homarr gives you a clean homepage to access and manage your server apps.

## 🗂️ File Browser

### 📂 Step 1: Navigate to the Filebrowser Directory

Create a new directory for Filebrowser:

```bash
mkdir -p ~/docker/filebrowser
cd ~/docker/filebrowser
```

---

### ✏️ Step 2: Edit the `docker-compose.yml` File

The `docker-compose.yml` file already exists. Open it and edit the Filebrowser service block to:

* Set the correct `user` ID (e.g., `1000:1000`) based on your system.
* Update the `volumes` paths to point to `/mnt/hdd/filebrowser/data` and `/mnt/hdd/filebrowser/config`.

---

### 📁 Step 3: Create and Set Permissions on Mount Folders

Create required folders and assign proper permissions:

```bash
sudo mkdir -p /mnt/hdd/filebrowser/{data,config}
sudo chown -R 1000:1000 /mnt/hdd/filebrowser
```

> Ensures Filebrowser has access to read/write data and config files.

---

### ▶️ Step 4: Start Filebrowser

```bash
docker compose up -d
```

After it starts, check the container logs for default login credentials:

```bash
docker logs filebrowser
```

---

### 🌐 Step 5: Access the Web Interface

Open your browser and visit:

```
http://<your-raspberry-pi-ip>:8443
```

Log in using the credentials from the logs.


Once logged in, change the username and password via:

```
Settings → User Management
```

---

🎉 Done! You now have a simple web-based file manager accessible from your browser.


## 🗄️ Backups & Data Safety

*Coming soon...*
