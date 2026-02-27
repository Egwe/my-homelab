# Logbook: Paperless-ngx

## 2026-02-17 Documentation Update
**Objective:** Refresh documentation to reflect current infrastructure.
**Actions Taken:**
1.  Parsed `context.md` migration report.
2.  Updated README to reflect Container 119 configuration.

---

## [Recent] Migration from ID 103 to ID 119
**Objective:** Migrate legacy Paperless-ngx instance (ID 103) to a new, standard-compliant LXC container (ID 119) to resolve update issues and improve maintainability.

**Context:** The old container used a deprecated `/opt/paperless` layout incompatible with modern update scripts.

**Actions Taken:**
1.  **Backup (Old ID 103):**
    *   Created full PostgreSQL dump (`pg_dump`).
    *   Saved `PAPERLESS_SECRET_KEY` and OCR config.
    *   Ensured all media/data was on external mount.

2.  **Setup (New ID 119):**
    *   Installed fresh LXC via Proxmox Community Script.
    *   Configured Bind Mount: `/mnt/file-server/paperless-ngx` (Host) -> `/data` (Container).
    *   Applied `PAPERLESS_SECRET_KEY` from old config.
    *   Restored OCR settings (`deu`, `skip` mode).

3.  **Database Migration:**
    *   *Challenge:* New container created an empty DB, causing import conflicts.
    *   *Fix:* Dropped the auto-created database, then imported the SQL dump from ID 103.
    *   Ran `manage.py migrate` to update schema.

4.  **Permissions:**
    *   recursively updated ownership of data directory to UID 1000 (`chown -R 1000:1000 /data`) to match the new `paperless` user.

5.  **Hardening:**
    *   Created Cronjob (Daily 03:00) to dump DB to `/data` for unified backups.

**Outcome:**
*   **Status:** ✅ Success
*   **Current State:** Container 119 is active. ID 103 is powered off.
*   **Key Improvement:** Updates can now be handled via standard scripts; DB backups are automated.

---

## 2024-10-26 Initial Configuration (Legacy ID 103)
**Objective:** Initial hardening and externalization of data.
**Actions Taken:**
*   Moved data to `/mnt/file-server/paperless-ngx`.
*   Configured Samba share for `consume` folder.
*   Hardened OCR settings for German language.
