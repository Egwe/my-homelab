# Service: Nginx Proxy Manager

> **Quick Info**
> * **ID/VMID:** 101
> * **Type:** LXC
> * **IP Address:** `10.10.30.51` (Static, VLAN 30)
> * **Status:** 🟢 Active / ⚠️ Manual Start Required
> * **OS/Base Image:** Debian GNU/Linux 12 (bookworm)

## 1. Overview
**Purpose:** Reverse Proxy handling all incoming web traffic (Ports 80/443), managing SSL certificates via Let's Encrypt, and routing subdomains securely to internal services in VLAN 20.
**Dependencies:**
* Depends on: OPNsense (Firewall Pinholing), AdGuard Home (`192.168.142.45`) for DNS resolution.
* Used by: All external connections to `.egwuatu.net` domains.

## 2. Configuration & Resources
### Hardware/Resources
* **CPU Cores:** 1-2
* **RAM:** 512MB - 1GB
* **Storage:** 4GB - 8GB

### Network & Ports
* **Main Interface:** `http://10.10.30.51:81`
* **Open Ports:**
    * `81`: Admin Web UI
    * `80`: HTTP Traffic (Proxy)
    * `443`: HTTPS Traffic (Proxy)

### Storage Mounts (Bind Mounts / Volumes)
* None 

## 3. Installation Details
**Method:** Proxmox Helper Script (tteck) -> openresty service
**Source/Repo:** https://github.com/community-scripts/ProxmoxVE

## 4. Environment & Secrets (Sanitized)
*Do not store actual passwords here. Reference where they are kept.*
* `ADMIN_EMAIL`: Saved in Bitwarden
* `ADMIN_PASSWORD`: Saved in Bitwarden under "Nginx Proxy Manager"

## 5. Migration Notes (Crucial for Host Switch)
*Specific requirements for moving this to a new host:*
* [ ] Must reside in VLAN 30 (DMZ).
* [x] Needs static IP reservation directly in Proxmox LXC settings (`10.10.30.51/24`, Gateway: `10.10.30.1`).
* [x] Custom DNS Server must be set to `192.168.142.45` in Proxmox to prevent startup freezes.