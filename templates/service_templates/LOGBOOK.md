# Logbook: [Service Name]

## [YYYY-MM-DD] Entry Title (e.g., Initial Setup)
**Objective:** [What were you trying to do?]
**Context:** [e.g., "Following the official documentation" or "Migrating from old host"]

**Actions Taken:**
1.  Created LXC container using [Source/Script].
2.  Executed command: `apt update && apt upgrade -y`
3.  Modified config file `/etc/config/app.conf`:
    * Changed `listen_port` to `8080`.
    * Enabled `ssl = true`.

**Errors & Troubleshooting:**
* **Error:** "Permission denied on /mnt/media"
* **Fix:** Ran `chmod -R 775` on the host directory and mapped user ID 1000.

**Outcome:**
* Status: ✅ Success / ⚠️ Partial / ❌ Failed
* **Next Steps:** Configure Backup job in Proxmox.

---

## [YYYY-MM-DD] [Short Description of Change]
**Objective:**
**Actions Taken:**
* ...
* ...
**Outcome:**

---