
# 🏠 Home Network + Pi-hole Lab Guide

A complete, copy-paste for setting up a home network using a **Google Fiber router/modem**, an **unmanaged Ethernet switch**, a **Raspberry Pi 3B+ running Pi-hole**, plus **Mac** and **Windows PC** clients.

---

## Table of Contents
1. [Network Topology (Text Overview)](#1-network-topology-text-overview)  
2. [Raspberry Pi Setup](#2-raspberry-pi-setup)  
3. [Pi-hole Installation](#3-pi-hole-installation)  
4. [Router Configuration (Google Fiber)](#4-router-configuration-google-fiber)  
5. [Client Verification (Windows & macOS)](#5-client-verification-windows--macos)  
6. [Service Reliability (Restart + Watchdog)](#6-service-reliability-restart--watchdog)  
7. [Monitoring with Glances](#7-monitoring-with-glances)  
8. [Security Hardening (SSH + Firewall)](#8-security-hardening-ssh--firewall)  
9. [Enhancements (Blocklists, Local DNS, HTTPS, Logging)](#9-enhancements-blocklists-local-dns-https-logging)  
10. [Troubleshooting Workflow](#10-troubleshooting-workflow)  
11. [Notes & Lessons Learned](#11-notes--lessons-learned)

---

## 1. Network Topology (Text Overview)

- **Google Fiber Router/Modem** → internet gateway + DHCP server.  
- **Unmanaged Switch** → simple Ethernet fan-out; no configuration.  
- **Peers on the LAN (all cabled to the switch):**  
  - **Windows PC**  
  - **MacBook Air (M1)**  
  - **Raspberry Pi 3B+** running **Pi-hole** (DNS at `192.168.1.103`).  
- **DNS Flow** → Router forwards all LAN DNS queries to the Pi-hole. Pi-hole resolves/blocks per your lists and forwards upstream as configured.

---

## 2. Raspberry Pi Setup

1. **Flash OS**
   - Write **Ubuntu Server (ARM64 Lite)** to the microSD (via Raspberry Pi Imager or balenaEtcher).

2. **Boot & Network**
   - Insert card, connect the Pi to the **unmanaged switch** via Ethernet, then power on.

3. **Headless SSH**
   ```bash
   sudo systemctl enable ssh
   sudo systemctl start ssh
   ```

4. **Update the System**
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

---

## 3. Pi-hole Installation

1. **Run the Official Installer**
   ```bash
   curl -sSL https://install.pi-hole.net | bash
   ```

2. **Static IP**
   - Ensure the Pi has a static IP (**`192.168.1.103`**). You can set this during install or (preferably) via DHCP reservation on the router (next section).

3. **Web UI (Pi-hole v6+)**
   - Pi-hole v6+ serves the web UI directly from **`pihole-FTL`** (no Lighttpd needed).
   - Access the dashboard at:
     ```
     http://192.168.1.103/admin
     ```

---

## 4. Router Configuration (Google Fiber)

- Open the **Google Home / Google Fiber** app.  
- Navigate to **Networking → Advanced → DNS** and set:
  - **Primary DNS:** `192.168.1.103` (Pi-hole)
  - **Secondary DNS (optional fallback):** `1.1.1.1` or `8.8.8.8`
- Reserve the Pi’s IP:
  - **Device settings → Networking → Advanced → DHCP reservation** for the Pi’s MAC → **`192.168.1.103`**.

> **Tip:** If Pi-hole becomes unavailable and you didn’t set a secondary DNS, clients may stall on lookups. Temporarily revert router DNS to defaults or add a fallback until Pi-hole is back.

---

## 5. Client Verification (Windows & macOS)

### Windows (PowerShell)
```powershell
Set-DnsClientServerAddress -InterfaceIndex 12 -ServerAddresses 192.168.1.103
netsh interface ipv6 delete dnsservers name="Ethernet 8" all
nslookup google.com
```

### macOS (Terminal)
```bash
scutil --dns | grep nameserver
dig google.com
```

✅ Expect to see the resolver using **`pi.hole` / `192.168.1.103`**.

---

## 6. Service Reliability (Restart + Watchdog)

```bash
sudo systemctl restart pihole-FTL
```

Create `/etc/systemd/system/pihole-watchdog.service`:
```ini
[Unit]
Description=Pi-hole watchdog
After=network.target

[Service]
ExecStart=/usr/bin/bash -c 'while true; do systemctl is-active --quiet pihole-FTL || systemctl restart pihole-FTL; sleep 60; done'
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable pihole-watchdog
sudo systemctl start pihole-watchdog
```

---

## 7. Monitoring with Glances

```bash
sudo apt install python3-venv -y
python3 -m venv /opt/glances
source /opt/glances/bin/activate
pip install glances[bottle]
```

Systemd service `/etc/systemd/system/glances.service`:
```ini
[Unit]
Description=Glances
After=network.target

[Service]
ExecStart=/opt/glances/bin/glances -w --disable-plugin cloud -p 61209
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now glances
```
Web UI: `http://192.168.1.103:61209`

---

## 8. Security Hardening (SSH + Firewall)

```bash
sudo nano /etc/ssh/sshd_config
# Set:
# PermitRootLogin no
# PasswordAuthentication no

sudo systemctl restart ssh
```

```bash
sudo apt install ufw -y
sudo ufw allow 22/tcp
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
sudo ufw allow 80/tcp
sudo ufw enable
```

---

## 9. Enhancements (Blocklists, Local DNS, HTTPS, Logging)

- Add blocklists via UI → *Group Management → Adlists*  
  Example: `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts`

- Add local DNS entries: UI → *Local DNS → DNS Records*

- Optional HTTPS: use Nginx or Caddy reverse proxy + Let's Encrypt

- Set long-term stats in `/etc/pihole/pihole-FTL.conf`:  
  ```
  MAXDBDAYS=365
  ```

---

## 10. Troubleshooting Workflow

```bash
nslookup google.com 192.168.1.103
sudo systemctl restart pihole-FTL
sudo ss -ltnp | grep :53
sudo ss -ltnp | grep :80
sudo systemctl status pihole-FTL
```

---

## 11. Notes & Lessons Learned

- Pi-hole v6+ doesn’t need Lighttpd. `pihole-FTL` serves the UI.  
- Use watchdog to auto-restart failed DNS service.  
- Always test with `nslookup` or `dig`.  
- Use DHCP reservations instead of static IPs when possible.  
- Add a secondary DNS to avoid total failures when Pi-hole is down.
