# 🍓 Raspberry Pi 4 Home Server

[![Platform](https://img.shields.io/badge/Hardware-Raspberry%20Pi%204B-C51A4A?logo=raspberry-pi&logoColor=white)](#hardware--prerequisites)
[![OS](https://img.shields.io/badge/OS-Debian%2013%20%2F%20RPi%20OS%2064--bit-A80030?logo=debian&logoColor=white)](#initial-system-setup)
[![Docker](https://img.shields.io/badge/Engine-Docker%20Compose-2496ED?logo=docker&logoColor=white)](#docker--docker-compose-installation)
[![Tailscale](https://img.shields.io/badge/Mesh%20VPN-Tailscale-24292E?logo=tailscale&logoColor=white)](#remote-access--tailscale-ssh)
[![Caddy](https://img.shields.io/badge/Reverse%20Proxy-Caddy%20v2-1F88C0?logo=caddy&logoColor=white)](#core-infrastructure-caddy-reverse-proxy)

A lightweight, secure, self-hosted home server architecture built on the **Raspberry Pi 4 Model B**. All core services run in isolated Docker containers behind a custom **Caddy** reverse proxy, providing automated wildcard SSL through **DuckDNS** (DNS-01 challenge). Access is restricted strictly to a private **Tailscale** WireGuard mesh network—leaving zero inbound ports exposed to the public internet.

---

## 📑 Table of Contents

1. [Service & Subdomain Directory](#service--subdomain-directory)
2. [Hardware & Prerequisites](#hardware--prerequisites)
3. [Initial System Setup](#initial-system-setup)
4. [Remote Access & Tailscale SSH](#remote-access--tailscale-ssh)
5. [External Storage Setup & Auto-Mount](#external-storage-setup--auto-mount)
6. [Docker & Docker Compose Installation](#docker--docker-compose-installation)
7. [Core Infrastructure: Caddy Reverse Proxy](#core-infrastructure-caddy-reverse-proxy)
8. [Services Deployment](#services-deployment)
   - [Portainer](#portainer)
   - [Pi-hole](#pi-hole)
   - [Vaultwarden](#vaultwarden)
   - [FileBrowser Quantum](#filebrowser-quantum)
   - [Glance Dashboard](#glance-dashboard)
9. [Central Caddyfile Reference](#central-caddyfile-reference)
10. [Troubleshooting & Verification](#troubleshooting--verification)

---

## 🌐 Service & Subdomain Directory

| Service | Subdomain Route | Internal Port | Primary Purpose |
| :--- | :--- | :--- | :--- |
| **Portainer** | `portainer.<domain>.duckdns.org` | `9443` (HTTPS) | Container & stack management |
| **Pi-hole** | `pihole.<domain>.duckdns.org` | `80` (Admin) | Network ad-blocking & local DNS |
| **Vaultwarden** | `vault.<domain>.duckdns.org` | `80` | Bitwarden-compatible password vault |
| **FileBrowser** | `files.<domain>.duckdns.org` | `80` | Lightweight private file management |
| **Glance** | `glance.<domain>.duckdns.org` | `8080` | Unified feeds, system stats & service dashboard |

---

## 🛠️ Hardware & Prerequisites

* **SBC:** Raspberry Pi 4 Model B (4GB or 8GB recommended).
* **Storage:** 
  * 32GB+ High-Endurance microSD card (for the OS).
  * External USB 3.0 SSD/HDD formatted to `ext4` (for persistent data & files).
* **Power:** Official 5.1V / 3.0A USB-C Raspberry Pi power supply.
* **Network:** Gigabit Ethernet connection to your router.
* **Accounts & Tokens:**
  * Free [Tailscale Account](https://tailscale.com).
  * Free [DuckDNS Account](https://www.duckdns.org) with a registered subdomain and API token.
  * GitHub Personal Access Token (classic, `public_repo` or read-only scope for software release tracking).

---

## ⚙️ Initial System Setup

1. Flash your microSD card with **Raspberry Pi OS Lite (64-bit)** or **Debian 13 (Trixie) 64-bit** using [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
2. In the OS Customization settings:
   * Set your hostname (e.g., `raspberrypi`).
   * Set your primary non-root username and password (e.g., `pi`).
   * Set your timezone (e.g., `Asia/Kolkata`).
   * Enable SSH with password authentication or your public SSH key.
3. Insert the card into the Pi, connect the Ethernet cable, and power it on.
4. SSH into the Pi from your workstation:
   ```bash
   ssh pi@raspberrypi.local
   ```
   > ⚠️ **Placeholder Reminder:** Replace `pi` and `raspberrypi.local` with the username and hostname you configured in the imager.
5. Update repository packages and upgrade the base system:
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

---

## 🌐 Remote Access & Tailscale SSH

Tailscale creates an encrypted WireGuard mesh network connecting all of your personal devices. Tailscale SSH eliminates the need for manual router port forwarding or exposing port 22 to the public internet.

### 1. Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

### 2. Authenticate and Enable Tailscale SSH

```bash
sudo tailscale up --ssh
```

1. Open the URL shown in your terminal.
2. Authorize the machine in your browser to attach the Raspberry Pi to your tailnet.

### 3. Verify Status & Note Your IP

```bash
tailscale ip -4
tailscale status
```
*Take note of the assigned `100.x.y.z` IPv4 address.*

### 4. Connect via Tailscale SSH

From any device logged into your Tailscale account:

```bash
ssh pi@<tailscale-ip-or-magicdns-hostname>
```
> ⚠️ **Placeholder Reminder:** Replace `<tailscale-ip-or-magicdns-hostname>` with your Pi's `100.x.y.z` Tailscale IP or MagicDNS hostname (e.g., `ssh pi@100.101.102.103`).

---

## 📍 External Storage Setup & Auto-Mount

To protect the microSD card from write wear, persistent data and file shares live on an external drive.

1. Connect your external drive to one of the blue **USB 3.0 ports**.
2. Identify the partition path (e.g., `/dev/sda1`):
   ```bash
   lsblk
   ```
3. Format the target partition to `ext4` (*warning: erases all data on that partition*):
   ```bash
   sudo mkfs.ext4 -L Storage /dev/sda1
   ```
4. Create the system mount point:
   ```bash
   sudo mkdir -p /mnt/hdd
   ```
5. Retrieve the drive UUID:
   ```bash
   sudo blkid /dev/sda1
   ```
   *Copy the alphanumeric UUID string (e.g., `UUID="12345678-1234-1234-1234-123456789abc"`).*
6. Configure auto-mount at boot:
   ```bash
   sudo nano /etc/fstab
   ```
   Append this line to `/etc/fstab`:
   ```fstab
   UUID=YOUR_UUID_HERE /mnt/hdd ext4 defaults,noatime,nofail 0 2
   ```
   > ⚠️ **Placeholder Reminder:** Replace `YOUR_UUID_HERE` with your actual partition UUID from step 5.
7. Test the mount and set ownership permissions to your user (`1000:1000`):
   ```bash
   sudo mount -a
   sudo chown -R $USER:$USER /mnt/hdd
   ```

---

## 🐳 Docker & Docker Compose Installation

1. Install Docker Engine and the Compose plugin:
   ```bash
   curl -fsSL https://get.docker.com | sh
   ```
2. Add your current user to the `docker` group:
   ```bash
   sudo usermod -aG docker $USER
   ```
3. Apply group membership without logging out:
   ```bash
   newgrp docker
   ```
4. Confirm installation:
   ```bash
   docker --version && docker compose version
   ```

---

## 🛡️ Core Infrastructure: Caddy Reverse Proxy

Caddy automatically provisions wildcard Let's Encrypt certificates using the DuckDNS DNS-01 challenge. Services communicate over an internal bridge network (`caddy_net`).

### Step 1: Update DuckDNS

1. Retrieve your Raspberry Pi's Tailscale IP:
   ```bash
   tailscale ip -4
   ```
2. Log into [DuckDNS](https://www.duckdns.org/).
3. Update your subdomain to point to your `100.x.y.z` Tailscale IP.
4. Copy your DuckDNS API token from the top of the account page.

### Step 2: Create the Shared Network

```bash
docker network create caddy_net
```

### Step 3: Build & Configure Caddy

1. Create the Caddy project folder:
   ```bash
   mkdir -p ~/docker/caddy && cd ~/docker/caddy
   ```
2. Create the custom `Dockerfile` containing the DuckDNS DNS module:
   ```dockerfile
   FROM caddy:builder AS builder

   RUN xcaddy build \
       --with github.com/caddy-dns/duckdns

   FROM caddy:latest

   COPY --from=builder /usr/bin/caddy /usr/bin/caddy
   ```
3. Create `docker-compose.yml`:
   ```yaml
   services:
     caddy:
       build: .
       container_name: caddy
       restart: unless-stopped
       ports:
         - "80:80"
         - "443:443"
         - "443:443/udp" # HTTP/3 QUIC
       environment:
         - DUCKDNS_TOKEN=YOUR_DUCKDNS_TOKEN_HERE
       volumes:
         - ./Caddyfile:/etc/caddy/Caddyfile:ro
         - caddy_data:/data
         - caddy_config:/config
       networks:
         - caddy_net

   volumes:
     caddy_data:
       name: caddy_data
     caddy_config:
       name: caddy_config

   networks:
     caddy_net:
       external: true
   ```
   > ⚠️ **Placeholder Reminder:** Replace `YOUR_DUCKDNS_TOKEN_HERE` with your actual secret token copied from DuckDNS.

4. Create your base `Caddyfile`:
   ```caddyfile
   {
       email your_email@example.com
   }

   *.yourname.duckdns.org {
       tls {
           dns duckdns {env.DUCKDNS_TOKEN}
       }

       handle {
           abort
       }
   }
   ```
   > ⚠️ **Placeholder Reminder:**
   > * Replace `your_email@example.com` with your real email address (for Let's Encrypt renewal alerts).
   > * Replace `yourname` with your registered DuckDNS subdomain prefix (e.g., `*.myserver.duckdns.org`).

5. Build and run Caddy:
   ```bash
   docker compose up -d --build
   ```

---

## 🧩 Services Deployment

### 📦 Portainer

```bash
mkdir -p ~/docker/portainer && cd ~/docker/portainer
```

**`docker-compose.yml`**:
```yaml
services:
  portainer:
    container_name: portainer
    image: portainer/portainer-ce:lts
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    networks:
      - caddy_net

volumes:
  portainer_data:
    name: portainer_data

networks:
  caddy_net:
    external: true
```

Start Portainer:
```bash
docker compose up -d
```

**Caddy snippet** (add inside `*.yourname.duckdns.org` in `~/docker/caddy/Caddyfile`):
```caddyfile
    @portainer host portainer.yourname.duckdns.org
    handle @portainer {
        reverse_proxy portainer:9443 {
            transport http {
                tls_insecure_skip_verify
            }
        }
    }
```
> ⚠️ **Placeholder Reminder:** Replace `yourname` with your actual DuckDNS subdomain prefix.

Reload Caddy:
```bash
docker exec -w /etc/caddy caddy caddy reload
```

---

### 🚫 Pi-hole

#### 1. Free Port 53 from systemd-resolved
Check if port 53 is occupied:
```bash
sudo ss -tulpn | grep :53
```
If occupied by `systemd-resolved`, disable its stub listener:
```bash
sudo sed -r -i.orig 's/#?DNSStubListener=yes/DNSStubListener=no/g' /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
```

#### 2. Deploy Pi-hole
```bash
mkdir -p ~/docker/pihole && cd ~/docker/pihole
```

**`docker-compose.yml`**:
```yaml
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      - "53:53/tcp"
      - "53:53/udp"
    environment:
      TZ: 'Asia/Kolkata'
      FTLCONF_dns_listeningMode: 'ALL' # Accepts queries over Tailscale subnet
    volumes:
      - './etc-pihole:/etc/pihole'
    cap_add:
      - NET_ADMIN
      - SYS_NICE
    restart: unless-stopped
    networks:
      - caddy_net

networks:
  caddy_net:
    external: true
```

Start and set the admin password:
```bash
docker compose up -d
docker exec -it pihole pihole setpassword
```

**Caddy snippet**:
```caddyfile
    @pihole host pihole.yourname.duckdns.org
    handle @pihole {
        reverse_proxy pihole:80
    }
```
> ⚠️ **Placeholder Reminder:** Replace `yourname` with your actual DuckDNS subdomain prefix.

Reload Caddy:
```bash
docker exec -w /etc/caddy caddy caddy reload
```

#### 3. Route Tailscale DNS to Pi-hole
1. Go to [Tailscale Admin Console > DNS](https://login.tailscale.com/admin/dns).
2. Under **Nameservers**, add a **Custom** nameserver and enter your Pi's `100.x.y.z` Tailscale IP.
3. Enable **Override local DNS**.

---

### 🔐 Vaultwarden

```bash
mkdir -p ~/docker/vaultwarden/data && cd ~/docker/vaultwarden
```

**`docker-compose.yml`**:
```yaml
services:
  vaultwarden:
    container_name: vaultwarden
    image: vaultwarden/server:latest
    restart: unless-stopped
    environment:
      - DOMAIN=https://vault.yourname.duckdns.org
      - SIGNUPS_ALLOWED=true
    volumes:
      - ./data:/data
    networks:
      - caddy_net

networks:
  caddy_net:
    external: true
```
> ⚠️ **Placeholder Reminder:** In `DOMAIN=https://vault.yourname.duckdns.org`, replace `yourname` with your actual DuckDNS subdomain prefix.

Start Vaultwarden:
```bash
docker compose up -d
```

**Caddy snippet**:
```caddyfile
    @vault host vault.yourname.duckdns.org
    handle @vault {
        reverse_proxy vaultwarden:80
    }
```
> ⚠️ **Placeholder Reminder:** Replace `yourname` with your actual DuckDNS subdomain prefix.

Reload Caddy:
```bash
docker exec -w /etc/caddy caddy caddy reload
```

> **Security Note:** Once your primary account is created, edit `docker-compose.yml`, change `SIGNUPS_ALLOWED=false`, and run `docker compose up -d` to lock registration.

---

### 🗂️ FileBrowser Quantum

Configured with an in-memory `tmpfs` cache to prevent microSD wear caused by image thumbnail creation.

```bash
mkdir -p ~/docker/filebrowser/data && cd ~/docker/filebrowser
```

**`data/config.yaml`**:
```yaml
server:
  cacheDir: /home/filebrowser/data/tmp
  sources:
    - path: /host_home
      name: "Home Directory"
      config:
        defaultEnabled: true
    - path: /storage
      name: "External Storage"
      config:
        defaultEnabled: true
```

**`docker-compose.yml`**:
```yaml
services:
  filebrowser:
    container_name: filebrowser
    image: gtstef/filebrowser:stable
    restart: unless-stopped
    user: "1000:1000"
    volumes:
      - ${HOME}:/host_home
      - /mnt/hdd:/storage
      - ./data:/home/filebrowser/data
    tmpfs:
      - /home/filebrowser/data/tmp:size=256M,uid=1000,gid=1000
    networks:
      - caddy_net

networks:
  caddy_net:
    external: true
```

Start FileBrowser:
```bash
docker compose up -d
```

**Caddy snippet**:
```caddyfile
    @files host files.yourname.duckdns.org
    handle @files {
        reverse_proxy filebrowser:80
    }
```
> ⚠️ **Placeholder Reminder:** Replace `yourname` with your actual DuckDNS subdomain prefix.

Reload Caddy:
```bash
docker exec -w /etc/caddy caddy caddy reload
```
*(Default credentials: `admin` / `admin`. Change immediately in Settings > Users).*

---

### 🧭 Glance Dashboard

Glance provides a clean, unified homelab dashboard featuring local system stats, Pi-hole telemetry, service uptime monitors, RSS/Reddit feeds, and GitHub release notifications.

#### 1. Setup Project & Environment Variables
```bash
mkdir -p ~/docker/glance/config && cd ~/docker/glance
```

Create a secure `.env` file to hold your sensitive tokens:
```bash
nano .env
```
Add the following keys:
```env
GITHUB_TOKEN=your_github_personal_access_token_here
PIHOLE_PASSWORD=your_pihole_v6_api_password_here
```
> ⚠️ **Placeholder Reminder:**
> * Replace `your_github_personal_access_token_here` with your GitHub PAT to enable software update checks without rate-limiting.
> * Replace `your_pihole_v6_api_password_here` with your Pi-hole v6 App Password (configured in Pi-hole Web UI > Settings > API).

#### 2. Create Docker Compose File
Create `docker-compose.yml`:
```yaml
services:
  glance:
    container_name: glance
    image: glanceapp/glance:latest
    restart: unless-stopped
    volumes:
      - ./config:/app/config
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /sys:/sys:ro
      - /:/host:ro
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    environment:
      - TZ=Asia/Kolkata
      - GITHUB_TOKEN=${GITHUB_TOKEN}
      - PIHOLE_PASSWORD=${PIHOLE_PASSWORD}
    networks:
      - caddy_net

networks:
  caddy_net:
    external: true
```

#### 3. Create Dashboard Configuration
Create `config/glance.yml`:
```yaml
server:
  host: 0.0.0.0
  port: 8080
  proxied: true

theme:
  background-color: 50 1 6
  primary-color: 24 97 58
  negative-color: 209 88 54

pages:
  - name: Homelab
    columns:
      # COLUMN 1 (LEFT)
      - size: small
        widgets:
          - type: clock
            hour-format: 12h
            hide-header: true

          - type: calendar
            first-day-of-week: sunday
            hide-header: true

          - type: weather
            location: Kochi, India
            units: metric
            hour-format: 12h
            hide-location: true
            hide-header: true

          - type: to-do

      # COLUMN 2 (MIDDLE)
      - size: full
        widgets:
          - type: monitor
            title: Services & Uptime
            cache: 1m
            sites:
              - title: Portainer
                url: https://portainer.yourname.duckdns.org
                icon: di:portainer
                timeout: 3s
              - title: Pi-hole
                url: https://pihole.yourname.duckdns.org/admin/
                icon: di:pi-hole
                timeout: 3s
              - title: FileBrowser
                url: https://files.yourname.duckdns.org
                icon: si:files
                timeout: 3s
              - title: Vaultwarden
                url: https://vault.yourname.duckdns.org
                icon: di:bitwarden
                timeout: 3s

          - type: group
            widgets:
              - type: rss
                title: Selfh.st
                style: detailed-list
                limit: 10
                collapse-after: 5
                feeds:
                  - url: https://selfh.st/rss/
                    title: Selfh.st

              - type: rss
                title: Hacker News
                style: detailed-list
                limit: 10
                collapse-after: 5
                feeds:
                  - url: https://news.ycombinator.com/rss
                    title: Hacker News

              - type: reddit
                title: Self-Hosted
                subreddit: selfhosted
                show-thumbnails: true
                limit: 10
                collapse-after: 5

              - type: reddit
                title: Homelab
                subreddit: homelab
                show-thumbnails: true
                limit: 10
                collapse-after: 5

      # COLUMN 3 (RIGHT)
      - size: small
        widgets:
          - type: server-stats
            hide-header: true
            servers:
              - type: local
                name: Raspi
                cpu-temp-sensor: cpu_thermal
                mountpoints:
                  "/host":
                    name: SD Card

          - type: dns-stats
            hide-header: true
            service: pihole-v6
            url: https://pihole.yourname.duckdns.org
            password: ${PIHOLE_PASSWORD}

          - type: releases
            title: Software Updates
            token: ${GITHUB_TOKEN}
            show-source-icon: true
            collapse-after: 4
            repositories:
              - glanceapp/glance
              - caddyserver/caddy
              - portainer/portainer
              - filebrowser/filebrowser
              - dani-garcia/vaultwarden
              - pi-hole/pi-hole
```
> ⚠️ **Placeholder Reminder:** Replace every occurrence of `yourname` in `config/glance.yml` with your actual registered DuckDNS subdomain.

#### 4. Launch Glance
```bash
docker compose up -d
```

#### 5. Add Caddy Route
Add inside `*.yourname.duckdns.org` in `~/docker/caddy/Caddyfile`:
```caddyfile
    @glance host glance.yourname.duckdns.org
    handle @glance {
        reverse_proxy glance:8080
    }
```
> ⚠️ **Placeholder Reminder:** Replace `yourname` with your actual DuckDNS subdomain prefix.

Reload Caddy:
```bash
docker exec -w /etc/caddy caddy caddy reload
```

---

## 📜 Central Caddyfile Reference

Save this consolidated configuration to `~/docker/caddy/Caddyfile`:

> ⚠️ **Important Placeholders to Replace:**
> 1. Replace `your_email@example.com` with your actual email address.
> 2. Replace every occurrence of `yourname` with your registered DuckDNS subdomain prefix.

```caddyfile
{
    email your_email@example.com
}

*.yourname.duckdns.org {
    tls {
        dns duckdns {env.DUCKDNS_TOKEN}
    }

    # Portainer
    @portainer host portainer.yourname.duckdns.org
    handle @portainer {
        reverse_proxy portainer:9443 {
            transport http {
                tls_insecure_skip_verify
            }
        }
    }

    # Pi-hole
    @pihole host pihole.yourname.duckdns.org
    handle @pihole {
        reverse_proxy pihole:80
    }

    # Vaultwarden
    @vault host vault.yourname.duckdns.org
    handle @vault {
        reverse_proxy vaultwarden:80
    }

    # FileBrowser
    @files host files.yourname.duckdns.org
    handle @files {
        reverse_proxy filebrowser:80
    }

    # Glance Dashboard
    @glance host glance.yourname.duckdns.org
    handle @glance {
        reverse_proxy glance:8080
    }

    # Drop any unknown host queries
    handle {
        abort
    }
}
```

---

## 🔍 Troubleshooting & Verification

* **Inspect Caddy SSL and Routing Logs:**
  ```bash
  docker logs -f caddy
  ```
* **Verify Caddyfile Syntax Before Reloading:**
  ```bash
  docker exec -w /etc/caddy caddy caddy validate
  ```
* **Inspect Glance Logs:**
  ```bash
  docker logs -f glance
  ```
* **Verify Port 53 Listening State:**
  ```bash
  sudo lsof -i :53
  ```
* **Check Memory Usage & Container Load:**
  ```bash
  docker stats --no-stream
  ```
* **Restart the Entire Stack Cleanly:**
  ```bash
  for dir in ~/docker/*/; do (cd "$dir" && docker compose restart); done
  ```

---

## ⚖️ License
This repository is published under the [MIT License](LICENSE).
