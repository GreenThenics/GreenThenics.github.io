---
title: "Part 2: The Target — Hosting DVWA behind the Shield"
date: 2026-04-07 19:15:10 +0530
categories: [Home Lab, Cybersecurity]
tags: [dvwa, docker, nat, port-forwarding, networking, linux]
---

## Introduction

In the previous part, we laid the foundation by setting up our virtual network and pfSense firewall. Now, it's time to bring in our first target: **DVWA (Damn Vulnerable Web Application)**. In this post, we'll focus on hosting this application behind our pfSense shield using Docker and configuring port forwarding to make it accessible.

---

## 4. Internet Access for Ubuntu

Since Ubuntu's network settings restrict it from accessing the internet directly, pfSense must be configured to receive requests on its LAN interface, apply firewall rules, and forward traffic out via the WAN.

### Why Ubuntu Can't Reach the Internet by Default

pfSense is designed for the real internet, where private IPs (`192.168.x.x`, `10.x.x.x`) are considered spoofed. In a VirtualBox lab:

- **WAN is Bridged** — gets an IP from your home router (e.g., `192.168.1.10`).
- **Kali is also Bridged** — also has a private IP (e.g., `192.168.1.15`).
- **Conflict** — Kali "attacking" pfSense WAN from a private IP gets blocked before any rules run.

### Fix 1: Ubuntu DNS

If Ubuntu can't resolve domain names, force it to use a public DNS server:

```bash
sudo nano /etc/resolv.conf
```

Add this line at the top:

```
nameserver 8.8.8.8
```

Save with `Ctrl+O`, `Enter`, then exit with `Ctrl+X`.

### Fix 2: pfSense DNS Settings

If `ping 8.8.8.8` fails, pfSense isn't routing traffic correctly.

1. Go to **Services > DHCP Server**.
2. Under **DNS Servers**, enter your home router's IP address.
![pfSense DNS](/assets/images/pfsense-dns.png)
3. Save.

Test connectivity from Ubuntu:

```bash
ping google.com
```

---

## 5. Redirecting pfSense to DVWA

### 5a. NAT Port Forwarding

1. Navigate to **Firewall > NAT** in the top menu.
2. On the **Port Forward** tab, click **Add** (up arrow).
3. Configure as follows:

| Field | Value |
|---|---|
| Interface | `WAN` |
| Protocol | `TCP` |
| Source | `Any` |
| Destination | `WAN Address` |
| Destination Port Range | `HTTP` to `HTTP` |
| Redirect Target IP | `192.168.2.10` (Ubuntu VM) |
| Redirect Target Port | `HTTP` |
| Filter Rule Association | `Add associated filter rule` |

### 5b. Fix ICMP (Ping) Access from Kali

To allow your Kali machine to ping the pfSense WAN IP:

1. Go to **Firewall > Rules > WAN**.
2. Click **Add** (up arrow to place the rule at the top).
3. Configure as follows:

| Field | Value |
|---|---|
| Protocol | `ICMP` |
| ICMP Subtypes | `Echo Request` (or `Any`) |
| Source | `Any` (or your Kali IP) |
| Destination | `WAN Address` |

4. **Save** and **Apply Changes**.

Test from Kali:

```bash
ping 192.168.1.10
```

### 5c. Move pfSense WebGUI to a Different Port

pfSense uses Port 80/443 for its own admin panel. Move it so DVWA can use Port 80.

1. Go to **System > Advanced > Admin Access**.
2. Find **TCP Port** and change it to `8090`.
3. **Save.**

> **Note:** Your browser will lose connection. The GUI is now at:
> - From Ubuntu: `https://192.168.2.1:8090`
> - From Host: `https://192.168.1.10:8090`

### 5d. Disable WebGUI Port 80 Redirect

1. Log into the pfSense GUI at `https://192.168.2.1:8090`.
2. Go to **System > Advanced > Admin Access**.
3. Check the box: **"Disable webConfigurator login redirect"**.
4. **Save.**

---

## 6. DVWA Setup

Install and run DVWA (Damn Vulnerable Web Application) using Docker on your Ubuntu VM.

### Step 1 — Install Docker

```bash
sudo apt update
sudo apt install -y docker.io

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Verify installation
docker --version
```

### Step 2 — Fix Docker Permissions (Recommended)

```bash
sudo usermod -aG docker $USER
```

> Log out and back in for this to take effect.

### Step 3 — Pull and Run DVWA

```bash
# Pull the image
docker pull vulnerables/web-dvwa

# Run the container
sudo docker run --rm -d -p 80:80 vulnerables/web-dvwa
```

### Step 4 — Access DVWA

Open a browser and go to `http://localhost` (or `http://192.168.2.10`).

- **Username:** `admin`
- **Password:** `password`

### Step 5 — Initialize the Database

> **Do not skip this step.** After login, click **"Create / Reset Database"**.

### Step 6 — Verify & Manage

```bash
# Check running containers
docker ps

# Stop a container
docker stop <container_id>

# Remove a container
docker rm <container_id>
```

### Verification from Kali

```bash
curl -I http://192.168.1.10
```

A successful response looks like:

```
HTTP/1.1 302 Found
Server: Apache
Set-Cookie: security=low
```

This confirms the attacker has a direct path through pfSense to the vulnerable DVWA target.

---