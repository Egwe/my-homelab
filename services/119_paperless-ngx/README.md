# Service: Paperless-ngx

> **Quick Info**
> * **ID/VMID:** 119
> * **Type:** LXC Container (Privileged/Unprivileged depending on setup, likely Unprivileged)
> * **IP Address:** `192.168.x.119` (Managed via Nginx Proxy Manager)
> * **Status:** `🟢 Active`
> * **OS/Base Image:** Debian 12/13

## 1. Overview
**Purpose:** Document management system for archiving and searching physical documents.
**Dependencies:**
*   **Storage:** External Mount `/mnt/file-server/paperless-ngx`
*   **Database:** PostgreSQL (Internal to container, but data on mount)
*   **Redis:** Internal
*   **Reverse Proxy:** Nginx Proxy Manager

## 2. Configuration & Resources
### Storage & Mounts
*   **Host Path:** `/mnt/file-server/paperless-ngx`
*   **Container Path:** `/data`
*   **Subdirectories:**
    *   `/data/media`: Documents
    *   `/data/data`: Database & App Data
    *   `/data/consume`: Ingestion folder (Samba share)

### Permissions
*   **User/Group:** `paperless`
*   **UID/GID:** `1000`
*   **External Access:** Samba share `consume` folder accessible by user `jonathan` (GID mapped or permission handled).

### Configuration (`paperless.conf`)
*   `PAPERLESS_OCR_LANGUAGE`: `deu` (German)
*   `PAPERLESS_OCR_MODE`: `skip` (Process only files without text layer)
*   `PAPERLESS_OCR_SKIP_ARCHIVE_FILE`: `always` (Modify original, no duplicates)
*   `PAPERLESS_SECRET_KEY`: Migrated from old instance (Preserves secrets/sessions)

## 3. Installation Details
**Method:** Proxmox Community Script (Standard Installation)
**Current Container ID:** 119 (Migrated from legacy 103)

## 4. Backup & Maintenance
*   **Strategy:** Automated DB Dump + External File Storage
*   **Schedule:** Daily at 03:00 via Cron
*   **Location:** SQL Dump saved to `/data` (synced with file server)
*   **Old Container:** ID 103 (Decommissioned/Pending Deletion)

## 5. Migration Notes
*   **Source:** Legacy Container 103 (`/opt/paperless` layout)
*   **Destination:** Container 119 (Standard `/data` layout)
*   **Critical:** `SECRET_KEY` must be preserved to decrypt credentials.
