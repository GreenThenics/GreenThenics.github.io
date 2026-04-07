---
title: "Part 1: The Foundation - Virtual Networking & pfSense"
date: 2026-04-07 18:24:15 +0530
categories: [Home Lab, Networking]
tags: [pfsense, networking, virtualbox, virtualization, homelab, security]
---

## Introduction

Welcome to the first part of our series on building a secure and scalable virtual laboratory. In this post, we will cover the foundational steps of setting up virtual networking and configuring **pfSense** as our primary firewall and router.

**Network Topology**
![Network Topology](/assets/images/net-diag.png)

> Make sure your to give atleast 4GB of RAM and 2 Processors to both Ubuntu and Kali VM for smooth performance.

---

## Table of Contents

1. [pfSense Installation](#1-pfsense-installation)
2. [WebGUI Access](#2-webgui-access)
3. [LAN Interface Configuration](#3-lan-interface-configuration)

## 1. pfSense Installation

### Boot the ISO

Follow the standard prompts in order:

1. **Accept** the license agreement.
![Accept](/assets/images/pfsense-accept.png)
2. Select **Install**.
![Install](/assets/images/pfsense-install.png)
3. Choose **Auto ZFS** as the partitioning method.
![Auto ZFS](/assets/images/pfsense-auto-zfs.png)

### Select Disk & Pool Type

1. Use your arrow keys to highlight **T Pool Type/Disks** and press Enter.
2. Choose **Stripe** - since you only have one virtual hard drive assigned to this VM, this is your only valid option. Mirroring requires two or more disks.
![Stripe](/assets/images/pfsense-stripe.png)
![Stripe](/assets/images/pfsense-stripe-2.png)

### Mark the Disk (Critical Step)

On the disk selection screen:

- You will see your VirtualBox disk listed (likely labeled `da0` or `ada0`).
- **Do not just press Enter.** Use the **Spacebar** to check the box next to the disk. An asterisk `[*]` should appear beside it.
- Press **Enter** to confirm.
![Mark Disk](/assets/images/pfsense-mark-disk.png)

### Run the Install

1. You will be returned to the previous screen, now showing `stripe: 1 disks` at the top.
2. Highlight **>>> Install** and press Enter.
![Install](/assets/images/pfsense-install-2.png)
3. When prompted to confirm, select **YES** - this wipes the virtual disk, which is fine since it is empty.

### Post-Install

Once installation finishes, pfSense will ask if you want to open a Shell for manual modifications.

- Select **No** → Select **Reboot**.
![Install](/assets/images/pfsense-install-3.png)

---

## 2. WebGUI Access

### The WebGUI Trap

By default, pfSense blocks WebGUI access from the WAN for security. Since your Ubuntu machine has no UI yet, you need to temporarily allow WAN access.

1. In the pfSense console menu, choose **Option 8 (Shell)**.
2. Run the following command to temporarily disable the firewall:

```bash
pfctl -d
```

This disables packet filtering so you can reach the WebGUI from your host browser via the WAN IP.

### Accessing the WebGUI
![Accessing the WebGUI](/assets/images/pfsense-access-webgui.png){: .shadow }

| Machine | URL | Role |
| :--- | :--- | :--- |
| **Ubuntu VM** | `http://192.168.1.1` | Internal access (LAN - Trusted Zone) |
| **Host Machine** | `http://[WAN_IP]` | External access (WAN - Requires override) |

> [!NOTE]
> Accessing the WebGUI from your **Host Machine** (WAN) is blocked by default. You must run `pfctl -d` in the pfSense shell to temporarily disable the firewall before you can reach the interface.
{: .prompt-info }


**Default Credentials:** `admin` / `pfsense`

---

## 3. LAN Interface Configuration

Run through the following prompts in the pfSense console to reconfigure the LAN interface:

```
Enter the number of the interface you wish to configure: 2 (LAN)
Configure IPv4 address LAN interface via DHCP? n
Enter the new LAN IPv4 address: 192.168.2.1
Enter the new LAN IPv4 subnet mask bit count: 24
For a LAN, press ENTER for none: [press Enter]
Configure IPv6 address LAN interface via DHCP6? n
Enter the new LAN IPv6 address: [press Enter]
Do you want to enable the DHCP server on LAN? y
Enter the start address of the IPv4 client address range: 192.168.2.10
Enter the end address of the IPv4 client address range: 192.168.2.100
Do you want to revert to HTTP as the webConfigurator protocol? n
```

### Verification

After completing the above steps, the pfSense main menu should display:

```
WAN: 192.168.1.10
LAN: 192.168.2.1
```

Access the WebGUI from Ubuntu or your host browser and log in with `admin` / `pfsense`. Then set your username and password, click **Reload**, and click **Finish**.

---