# 🏠 Active Directory Home Lab

A hands-on cybersecurity home lab simulating a real-world enterprise environment with Active Directory, Splunk SIEM, Sysmon telemetry, and a Kali Linux attacker machine — all running on VirtualBox.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Lab Architecture](#lab-architecture)
- [Prerequisites](#prerequisites)
- [Setup Guide](#setup-guide)
  - [1. Virtual Machines](#1-virtual-machines)
  - [2. Network Configuration](#2-network-configuration)
  - [3. Splunk SIEM Setup](#3-splunk-siem-setup)
  - [4. Target Machine Setup (Windows 10)](#4-target-machine-setup-windows-10)
  - [5. Splunk Server Configuration](#5-splunk-server-configuration)
  - [6. Windows Server (ADDC01) Setup](#6-windows-server-addc01-setup)
  - [7. Active Directory Configuration](#7-active-directory-configuration)
  - [8. Join Target Machine to Domain](#8-join-target-machine-to-domain)

---

## Overview

This lab replicates a small enterprise network to practice:

- **Active Directory** administration and domain management
- **SIEM** log ingestion and threat detection with **Splunk**
- **Endpoint telemetry** collection with **Sysmon**
- **Adversary simulation** using **Kali Linux**

---

## Lab Architecture

| Machine     | OS                  | IP Address            | Role                                   |
| ----------- | ------------------- | --------------------- | -------------------------------------- |
| `target-PC` | Windows 10 Pro      | `192.168.10.100`      | Domain-joined workstation / log source |
| `ADDC01`    | Windows Server 2022 | `192.168.10.7`        | Active Directory Domain Controller     |
| `Splunk`    | Ubuntu Server 22.04 | `192.168.10.10`       | Splunk SIEM server                     |
| `Kali`      | Kali Linux          | DHCP (`192.168.10.x`) | Attacker machine                       |

**Network:** VirtualBox NAT Network — `AD-Project` — `192.168.10.0/24`

---

## Prerequisites

- [VirtualBox](https://www.virtualbox.org/) installed on your host machine
- ISO images for:
  - Kali Linux
  - Windows 10 Pro
  - Windows Server 2022
  - Ubuntu Server 22.04
- [Splunk Enterprise](https://www.splunk.com/en_us/download/splunk-enterprise.html) `.deb` package (Linux) — downloaded to host machine
- [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) — downloaded to Windows machines
- [Sysmon Olaf configuration](https://github.com/olafhartong/sysmon-modular) — `sysmonconfig.xml`

---

## Setup Guide

### 1. Virtual Machines

Install the following VMs in VirtualBox:

- **Kali Linux**
- **Windows 10 Pro** (will be renamed to `target-PC`)
- **Windows Server 2022** (will be renamed to `ADDC01`)
- **Ubuntu Server 22.04** (will be used as the Splunk server)

---

### 2. Network Configuration

Create a shared NAT Network so all VMs can communicate with each other and reach the internet.

1. In VirtualBox, go to **File → Tools → Network Manager**.
2. Go to the **NAT Networks** tab and click **Create**.
3. Under **General Options**, configure:
   - **Name:** `AD-Project`
   - **IPv4 Prefix:** `192.168.10.0/24`
   - ✅ Enable DHCP
4. Click **Apply**.
5. For **all four VMs** (Kali, Windows 10, ADDC01, Splunk), go to their **Settings → Network** and set:
   - **Attached to:** NAT Network
   - **Name:** `AD-Project`

---

### 3. Splunk SIEM Setup

#### 3.1 Configure a Static IP on the Ubuntu Server

Boot the Splunk (Ubuntu Server) machine and assign it a static IP.

1. Check the current IP:

   ```bash
   ip a
   ```

2. Edit the Netplan configuration:

   ```bash
   sudo nano /etc/netplan/50-cloud-init.yaml
   ```

3. Replace the contents with:

   ```yaml
   network:
     version: 2
     ethernets:
       enp0s3:
         dhcp4: no
         addresses: [192.168.10.10/24]
         nameservers:
           addresses: [8.8.8.8]
         routes:
           - to: default
             via: 192.168.10.1
   ```

   > ⚠️ YAML is indentation-sensitive — ensure spacing is exact.

4. Apply the configuration:

   ```bash
   sudo netplan try   # press Enter if prompted
   # or
   sudo netplan apply
   ```

5. Verify the static IP is applied:
   ```bash
   ip a   # should show 192.168.10.10
   ping google.com
   ```

#### 3.2 Install Splunk Enterprise

1. On your **host machine**, download the Splunk Enterprise `.deb` package from the [Splunk website](https://www.splunk.com).

2. On the Ubuntu Server, install VirtualBox Guest Additions (for shared folder access):

   ```bash
   sudo apt-get install virtualbox-guest-additions-iso -y
   ```

   > If a purple screen appears, press **Enter**.

3. In VirtualBox, add a **Shared Folder** pointing to the directory containing the `.deb` file, then **reboot** the Ubuntu Server.

4. Add your user to the `vboxsf` group:

   ```bash
   sudo adduser <your-username> vboxsf
   ```

5. Create a mount point and mount the shared folder:

   ```bash
   mkdir share
   sudo mount -t vboxsf -o uid=1000,gid=1000 ActiveDirectory_HomeLab share/
   ```

6. Install Splunk from the shared folder:

   ```bash
   sudo dpkg -i share/splunk-*.deb
   ```

7. Start Splunk as the `splunk` user:

   ```bash
   cd /opt/splunk/bin
   sudo -u splunk bash
   ./splunk start
   ```

   - Accept the license agreement and set an **admin username and password**.
   - Splunk will be accessible at: `http://192.168.10.10:8000`

8. Exit the `splunk` user and enable Splunk to start on boot:
   ```bash
   exit
   sudo ./splunk enable boot-start -user splunk
   ```

---

### 4. Target Machine Setup (Windows 10)

#### 4.1 Rename and Set Static IP

1. Rename the PC to `target-PC` via **Settings → System → About → Rename this PC**.
2. Set a static IP via **Network and Internet → Change adapter options → Ethernet → Properties → IPv4**:
   - **IP Address:** `192.168.10.100`
   - **Subnet Mask:** `255.255.255.0`
   - **Default Gateway:** `192.168.10.1`

3. Verify connectivity — open a browser and navigate to `http://192.168.10.10:8000` to confirm you can reach the Splunk web interface.

#### 4.2 Install Splunk Universal Forwarder

1. Download the [Splunk Universal Forwarder](https://www.splunk.com/en_us/download/universal-forwarder.html) for Windows from the Splunk website.
2. Run the installer. When prompted for a **Receiving Indexer**, set:
   - **Host:** `192.168.10.10`
   - **Port:** `9997`

#### 4.3 Install Sysmon

1. Download [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) and the [Olaf Sysmon config](https://github.com/olafhartong/sysmon-modular/blob/master/sysmonconfig.xml).
2. Open **PowerShell as Administrator**, navigate to the Sysmon download directory, and run:
   ```powershell
   .\Sysmon64.exe -i ..\sysmonconfig.xml
   ```

#### 4.4 Configure Splunk Forwarder Inputs

Create or place a file named `inputs.conf` at:

```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
```

with the following content:

```ini
[WinEventLog://Application]
index = endpoint
disabled = false

[WinEventLog://Security]
index = endpoint
disabled = false

[WinEventLog://System]
index = endpoint
disabled = false

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = endpoint
disabled = false
renderXml = true
source = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
```

Restart the Splunk Universal Forwarder service via **Services** or PowerShell:

```powershell
Restart-Service SplunkForwarder
```

---

### 5. Splunk Server Configuration

Using the `target-PC` browser, navigate to `http://192.168.10.10:8000` and log in.

1. **Create an Index:**
   - Go to **Settings → Indexes → New Index**
   - Name it `endpoint` and click **Save**

2. **Configure a Receiving Port:**
   - Go to **Settings → Forwarding and Receiving → Configure Receiving → New Receiving Port**
   - Enter port `9997` and click **Save**

3. **Verify Logs:**
   - Go to **Search & Reporting**
   - Run the search: `index=endpoint`
   - You should see logs streaming in from `target-PC`

---

### 6. Windows Server (ADDC01) Setup

1. Rename the server to `ADDC01` via **Settings → System → About → Rename this PC** and restart.
2. Install the **Splunk Universal Forwarder** and configure `inputs.conf` the same way as the target machine (see [Sections 4.2–4.4](#42-install-splunk-universal-forwarder)).
3. Verify in the Splunk dashboard — under **Search & Reporting**, you should now see **two hosts**: `target-PC` and `ADDC01`.

---

### 7. Active Directory Configuration

#### 7.1 Install AD DS Role

1. Open **Server Manager → Manage → Add Roles and Features**.
2. Click **Next** through the wizard until you reach **Server Roles**.
3. Select **Active Directory Domain Services → Add Features**.
4. Click **Next** through the remaining steps and then **Install**. Close when done.

#### 7.2 Promote to Domain Controller

1. Click the **flag icon** in Server Manager and select **Promote this server to a Domain Controller**.
2. Select **Add a new forest** and enter a domain name:
   ```
   mydfir.local
   ```
3. Set a **DSRM password**, then click **Next** through all prompts and click **Install**.
4. The server will **automatically restart**.

#### 7.3 Create Organizational Units and Users

1. In **Server Manager → Tools → Active Directory Users and Computers**, expand `mydfir.local`.

2. Create an **IT** Organizational Unit:
   - Right-click domain → **New → Organizational Unit** → Name: `IT`
   - Right-click `IT` → **New → User** → fill in user details and set a password.

3. Create an **HR** Organizational Unit:
   - Right-click domain → **New → Organizational Unit** → Name: `HR`
   - Right-click `HR` → **New → User** → fill in user details and set a password.

---

### 8. Join Target Machine to Domain

1. On `target-PC`, change the **DNS server** to point to the Domain Controller:
   - Go to **Settings → Network & Internet → Change adapter options**
   - Right-click **Ethernet → Properties → IPv4 → Properties**
   - Set **Preferred DNS Server** to `192.168.10.7` (ADDC01's IP)

2. Join the domain:
   - Search for **This PC → Properties → Advanced System Settings**
   - Under the **Computer Name** tab, click **Change**
   - Select **Domain** and type `MYDFIR.LOCAL`, then click **OK**
   - Enter domain admin credentials when prompted and **Restart**

3. After reboot, on the lock screen you should see the option to log in as **Other User**, with the domain `MYDFIR` shown — use the `JennySmith` (or any other created user) credentials to log in.

---

## 🎉 Congratulations!

You have successfully built a fully functional Active Directory Home Lab! From here you can:

- **Simulate attacks** from the Kali Linux machine (brute force, pass-the-hash, etc.)
- **Detect and investigate** events in Splunk using `index=endpoint`
- **Practice AD administration** — GPOs, user management, permissions
- **Expand the lab** with additional tools like Atomic Red Team, MITRE ATT&CK mappings, or a SOAR platform

---

## 📁 Project Files

| File                                                     | Description                              |
| -------------------------------------------------------- | ---------------------------------------- |
| [`HomeLab_Project_setup.txt`](HomeLab_Project_setup.txt) | Raw setup notes used to build this guide |

---

## 📚 References

- [Splunk Universal Forwarder](https://www.splunk.com/en_us/download/universal-forwarder.html)
- [Sysmon – Microsoft Sysinternals](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [Sysmon Olaf Configuration (GitHub)](https://github.com/olafhartong/sysmon-modular)
- [VirtualBox NAT Network](https://www.virtualbox.org/manual/ch06.html#network_nat_service)
- [Splunk Enterprise](https://www.splunk.com/en_us/download/splunk-enterprise.html)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [Kali Linux](https://www.kali.org/)
