# Service: OPNsense (Firewall & Router)

> **Quick Info**
> * **ID/VMID:** VM
> * **Type:** VM
> * **IP Address:** `192.168.142.120` (WAN) / `10.10.30.1` (VLAN 30 DMZ Gateway) / `10.10.20.1` (VLAN 20 APP Gateway)
> * **Status:** 🟢 Active
> * **OS/Base Image:** FreeBSD / OPNsense

## 1. Overview
**Purpose:** Main router, firewall, and VLAN gateway for the entire homelab. Controls traffic flow between the FritzBox (WAN), DMZ, App-Network, and manages Port Forwarding.
**Dependencies:**
* Depends on: FritzBox 7530 AX (Upstream Internet Connection)
* Used by: All Homelab VMs, LXCs, and external connections

## 2. Configuration & Resources
### Hardware/Resources
* **CPU Cores:** 2-4 (typical for homelab routing)
* **RAM:** 4GB - 8GB
* **Storage:** 32GB Boot Disk

### Network & Ports
* **Main Interface:** `https://192.168.142.120:8443`
* **Open Ports:**
    * `8443`: Web GUI (WAN Access)
    * `80` / `443`: Port Forwarding to Nginx Proxy Manager (`10.10.30.51`)

### Storage Mounts (Bind Mounts / Volumes)
* None (Standard VM Disk)

## 3. Installation Details
**Method:** Proxmox VM Installation via OPNsense ISO
**Source/Repo:** Official OPNsense Documentation

## 4. Environment & Secrets (Sanitized)
*Do not store actual passwords here. Reference where they are kept.*
* `ROOT_PASSWORD`: Saved in Bitwarden under "OPNsense Root"
* `GUI_ADMIN`: Saved in Bitwarden under "OPNsense Web GUI"

## 5. Migration Notes (Crucial for Host Switch)
*Specific requirements for moving this to a new host:*
* [x] Requires static IPv4 `192.168.142.120` on FritzBox.
* [x] Needs static IP reservation on Router (FritzBox MAC: `02:86:DB:AD:3D:9C`).
* [x] FritzBox Port Forwarding (80/443) must point to the correct MAC address.