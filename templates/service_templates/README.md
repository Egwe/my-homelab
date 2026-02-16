# Service: [Service Name, e.g., Plex Media Server]

> **Quick Info**
> * **ID/VMID:** [e.g., 105]
> * **Type:** [LXC / VM / Docker]
> * **IP Address:** `[IP_ADDRESS]` (Sanitized or Local)
> * **Status:** `🟢 Active` / `🔴 Offline` / `🟡 Maintenance`
> * **OS/Base Image:** [e.g., Ubuntu 24.04 / Debian 12]

## 1. Overview
**Purpose:** [Brief description of what this service does.]
**Dependencies:**
* Depends on: [e.g., NAS Mount /mnt/media, PostgresDB]
* Used by: [e.g., Smart TV, Mobile Devices]

## 2. Configuration & Resources
### Hardware/Resources
* **CPU Cores:** [e.g., 4]
* **RAM:** [e.g., 8GB]
* **Storage:** [e.g., 32GB Boot Disk]

### Network & Ports
* **Main Interface:** `http://[IP]:[PORT]`
* **Open Ports:**
    * `32400`: Main Service
    * `22`: SSH

### Storage Mounts (Bind Mounts / Volumes)
* `Host Path` -> `Container Path`
* `/mnt/pve/media` -> `/mnt/media` (Read Only)

## 3. Installation Details
**Method:** [e.g., Proxmox Helper Script / Docker Compose / Manual Install]
**Source/Repo:** [Link to GitHub repo or Guide used]

## 4. Environment & Secrets (Sanitized)
*Do not store actual passwords here. Reference where they are kept.*
* `DB_PASSWORD`: Saved in Bitwarden under "HomeLab DB"
* `API_KEY`: Saved in `.env` file (gitignored)

## 5. Migration Notes (Crucial for Host Switch)
*Specific requirements for moving this to a new host:*
* [ ] Requires hardware passthrough (e.g., Intel iGPU `/dev/dri`)
* [ ] Requires specific USB device (e.g., Zigbee Stick)
* [ ] Needs static IP reservation on Router