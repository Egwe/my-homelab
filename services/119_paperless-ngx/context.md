Of course. Here is a comprehensive report summarizing our entire session, structured professionally in English and formatted in Markdown.

---

### **Comprehensive Report: Configuration and Hardening of Paperless-ngx on Proxmox VE**

**Date:** 2024-10-26
**Project ID:** PNX-PVE-MIG-01
**Status:** Completed

---

### **1.0 Executive Summary**

This report details the successful configuration, security hardening, and data migration of a Paperless-ngx instance running within an unprivileged LXC container (ID 103) on a Proxmox VE host. The primary objective was to transition the application from a default, self-contained state to a robust, portable, and secure setup. All application data was successfully migrated to an external network-mounted storage location. An automated document ingestion workflow was established via a secure Samba share, overcoming complex cross-container permission challenges. The final configuration is stable, secure, and resilient to container rebuilds.

---

### **2.0 Initial State Assessment**

*   **Platform:** Proxmox VE
*   **Application:** Paperless-ngx
*   **Container:** Unprivileged Debian LXC (ID 103)
*   **Data Storage:** All application data (`media`, `data`, `consume` directories) was located within the container's internal root filesystem (`/opt/paperless/`).
*   **Networking:** Default configuration, not hardened for external access.
*   **Ingestion:** Manual document uploads via the web interface.

---

### **3.0 Project Objectives**

1.  **Optimize OCR Settings:** Configure the Optical Character Recognition engine for German-language documents and efficient processing.
2.  **Harden Security:** Implement necessary configurations for secure access via a reverse proxy.
3.  **Externalize Data:** Migrate all user-generated data to an external storage location (`/mnt/file-server/`) to ensure data persistence and simplify backups.
4.  **Automate Ingestion:** Establish a "consume" folder accessible as a Samba network share for automated, drag-and-drop document importing.

---

### **4.0 Implementation Phases**

#### **4.1 Phase 1: OCR Configuration**

The `paperless.conf` file was modified to optimize OCR performance and workflow:
*   `PAPERLESS_OCR_LANGUAGE=deu`: Set the primary OCR language to German.
*   `PAPERLESS_OCR_MODE=skip`: Configured to only process files that do not already have a text layer, saving significant system resources.
*   `PAPERLESS_OCR_SKIP_ARCHIVE_FILE=always`: Instructed the system to write OCR data directly into the original file, preventing the creation of duplicate archived versions.
*   Default settings for image processing (`clean`, `deskew`, `rotate_pages`) were confirmed to be active.

#### **4.2 Phase 2: Security Hardening for Reverse Proxy Access**

To ensure secure operation behind a reverse proxy, the following parameters were set:
*   `PAPERLESS_CSRF_TRUSTED_ORIGINS`: Configured with the public domain name and the local container IP address to mitigate Cross-Site Request Forgery attacks.
*   `PAPERLESS_URL`: Set to the public-facing URL to ensure correct link generation within the application.
*   **Analysis:** It was determined that `PAPERLESS_SECRET_KEY` and `PAPERLESS_ALLOWED_HOSTS` are managed automatically and securely by the application, requiring no manual intervention for this setup.

#### **4.3 Phase 3: Data Migration to External Storage**

This critical phase involved moving all persistent data outside the LXC container.

1.  **Host Preparation:** A directory structure was created on the host at `/mnt/file-server/paperless-ngx/` with `media`, `data`, and `consume` subdirectories.
2.  **Data Migration:** The container was stopped, and all content from its internal `/opt/paperless/media`, `/opt/paperless/data`, and `/opt/paperless/consume` directories was moved to the corresponding new locations on the host.
3.  **Proxmox Bind Mount:** A bind mount was added to the LXC configuration file (`/etc/pve/lxc/103.conf`), mapping the host directory `/mnt/file-server/paperless-ngx` to the `/data` directory inside the container:
    ```conf
    mp0: /mnt/file-server/paperless-ngx,mp=/data
    ```
4.  **Application Reconfiguration:** The `paperless.conf` file inside the container was updated to point to the new paths:
    *   `PAPERLESS_MEDIA_ROOT=/data/media`
    *   `PAPERLESS_DATA_DIR=/data/data`
    *   `PAPERLESS_CONSUMPTION_DIR=/data/consume`

#### **4.4 Phase 4: Automated Import via Samba Share**

The final phase was to enable the automated consumption of files dropped into a network share. This introduced a cross-container permission challenge.

1.  **Samba Configuration:** The consume folder was shared via a separate, secure NAS container that uses an authenticated user (`jonathan`).
2.  **Permission Conflict:** Files created in the share by user `jonathan` could not be processed by the Paperless-ngx application, which runs as `root` (UID 0) inside its own container.
3.  **Solution via GID Mapping:** A robust, cross-container permission model was implemented:
    a. The numeric Group ID (GID) of the Samba user `jonathan` was identified in the NAS container (Result: `1000`).
    b. In the Paperless-ngx container, a new group (`paperless-consume`) was created with the **exact same GID**: `groupadd --gid 1000 paperless-consume`.
    c. The `root` user within the Paperless-ngx container was added to this new group: `usermod -aG paperless-consume root`.
    d. The consume directory was assigned to this shared group and given appropriate permissions to ensure all new files inherit the correct group ownership:
       ```bash
       chown root:paperless-consume /data/consume
       chmod g+rws /data/consume
       ```
This solution allows two isolated systems to seamlessly and securely interact with the same set of files based on a shared group identity, without compromising the security of either container.

---

### **5.0 Challenges Encountered and Resolutions**

*   **Challenge 1: Incorrect LXC Filesystem Path.**
    *   **Symptom:** Initial `mv` commands failed with "No such file or directory."
    *   **Resolution:** Analysis of `pct config 103` revealed the container's rootfs was on `local-lvm`. The command `pct mount 103` was used to correctly mount the container's filesystem at `/var/lib/lxc/103/rootfs`, providing the true path for the migration commands.

*   **Challenge 2: Incorrect User Assumption.**
    *   **Symptom:** Initial uncertainty about which user required permissions.
    *   **Resolution:** The `ps aux` command inside the container confirmed all Paperless-ngx processes were running as `root` (UID 0), clarifying that all host-side file ownership should be set to `chown -R 0:0`.

*   **Challenge 3: Cross-Container User Mismatch.**
    *   **Symptom:** The `usermod` command failed because the Samba user `jonathan` did not exist in the Paperless-ngx container.
    *   **Resolution:** The strategy was pivoted from user *names* to numeric group *IDs* (GID), creating a common permission layer that both containers understand without needing to know about each other's local user accounts.

---

### **6.0 Final Configuration Summary**

*   **Data Location (Host):** `/mnt/file-server/paperless-ngx/`
*   **LXC Bind Mount:** Maps host data directory to `/data` inside the container.
*   **Permissions:** Data is owned by `root` (UID 0), with the consume directory accessible via a shared GID of `1000`.
*   **Workflow:** Documents dropped into the Samba share are now automatically and reliably ingested by Paperless-ngx.

The project successfully achieved all objectives, resulting in a significantly more robust, secure, and manageable Paperless-ngx deployment.



Das ist eine hervorragende Idee. Eine Dokumentation sollte immer "leben" und den aktuellen Ist-Zustand widerspiegeln.

Ich habe den alten Bericht genommen und die Geschehnisse der letzten Tage (Migration auf Container 119, Datenbank-Rettung, Backup-Strategie) logisch integriert. Der Bericht ist nun so aufgebaut, dass er die *Historie* kurz erwähnt, aber den Fokus auf die *neue, saubere Architektur* legt.

Hier ist der aktualisierte Abschlussbericht für deine Unterlagen:

---

### **Abschlussbericht: Migration und Modernisierung der Paperless-ngx Infrastruktur (Proxmox LXC)**

#### **1. Projektübersicht & Management Summary**

**Ausgangslage:**
Es bestand eine funktionierende Paperless-ngx Instanz in einem LXC-Container (ID 103, "paperless-ngx-old"). Diese basierte jedoch auf einer veralteten Verzeichnisstruktur (`/opt/paperless`), die von den aktuellen Proxmox-Community-Update-Skripten nicht mehr unterstützt wurde ("Deprecated layout"). Ein einfaches Update war nicht mehr möglich.

**Projektziel:**
Migration der kompletten Instanz auf einen neuen, standardkonformen LXC-Container (ID 119), ohne Datenverlust. Gleichzeitig sollte die Ausfallsicherheit durch eine automatisierte Backup-Strategie der Datenbank erhöht und die Kompatibilität für zukünftige Updates sichergestellt werden.

**Ergebnis:**
Die Migration auf Container 119 ("paperless-ngx") war erfolgreich. Alle Dokumente, Tags, Benutzer und Konfigurationen wurden übernommen. Das System läuft nun auf der aktuellen Version, nutzt eine externe Datenbank-Sicherung und ist vollständig in die bestehende Netzwerkinfrastruktur (Nginx Proxy Manager, Fileserver) integriert.

---

#### **2. Technische Architektur (Ist-Zustand)**

Das System wurde von einer "Legacy"-Installation zu einer wartbaren Standard-Installation transformiert.

* **Container:** ID 119 (Debian 12/13 Basis, Unprivileged).
* **Netzwerk:** Statische Zuordnung via Nginx Proxy Manager auf die neue IP (`.119`).
* **Speicher:** Trennung von Logik (Container) und Daten (Fileserver).
* Bind Mount: `/mnt/file-server/paperless-ngx` (Host) -> `/data` (Container).


* **Berechtigungen:** Der interne User `paperless` (UID/GID 1000) besitzt die Schreibrechte auf die externen Daten.

---

#### **3. Durchgeführte Maßnahmen & Migrationsschritte**

Der Prozess gliederte sich in vier entscheidende Phasen:

##### **Phase 1: Sicherung des Alt-Systems (Container 103)**

Da die Daten (PDFs) bereits extern lagen, musste primär das "Gedächtnis" von Paperless (die Datenbank) gerettet werden.

* Erstellung eines **Full Dumps** der PostgreSQL-Datenbank (`pg_dump`) aus dem alten Container.
* Ablage des Dumps (`paperless-backup.sql`) auf dem sicheren Fileserver-Mount.
* Sicherung des `PAPERLESS_SECRET_KEY` und der OCR-Konfigurationen aus der alten `paperless.conf`.

##### **Phase 2: Aufbau des Neu-Systems (Container 119)**

* Installation eines frischen LXC-Containers mittels der aktuellen Proxmox-Community-Skripte.
* Einbindung des existierenden Daten-Verzeichnisses via Mount-Point (`mp0`) in der Proxmox-Config (`119.conf`).
* **Wichtig:** Übernahme des alten `SECRET_KEY` in die neue Konfiguration. Dies stellte sicher, dass bestehende User-Sessions und verschlüsselte Credentials (z.B. für E-Mail-Abruf) gültig blieben.
* Wiederherstellung der optimierten **OCR-Einstellungen** (Deutsch, `mode=skip`, keine Archiv-Dateien), um die Performance beizubehalten.

##### **Phase 3: Datenbank-Migration (Der kritische Pfad)**

Hier lag die größte technische Hürde. Der neue Container initialisierte bei der Installation bereits eine leere, aber strukturierte Datenbank. Ein einfaches Einspielen des Backups führte zu Konflikten ("Relation already exists").

* **Lösung:** Die im neuen Container automatisch erstellte Datenbank `paperlessdb` wurde komplett gelöscht (`DROP DATABASE`) und sauber neu angelegt.
* **Import:** Das Backup aus Phase 1 wurde in die nun leere Datenbank importiert.
* **Migration:** Mittels `manage.py migrate` wurde das alte Datenbankschema auf den Stand der neuesten Software-Version aktualisiert.

##### **Phase 4: Automatisierung & Backup**

Um das Risiko eines Datenbank-Verlustes in Zukunft zu minimieren, wurde ein **automatisierter Cronjob** eingerichtet.

* **Zyklus:** Täglich um 03:00 Uhr.
* **Aktion:** Erstellung eines SQL-Dumps.
* **Ziel:** Speicherung direkt auf dem externen Fileserver (`/data`), sodass PDFs und Datenbank-Backups physisch am selben Ort liegen.

---

#### **4. Herausforderungen und Lösungen**

Während der Umstellung traten spezifische Probleme auf, die wie folgt gelöst wurden:

* **Herausforderung: Datenbank-Konflikt beim Import.**
* *Problem:* Der Import brach ab, weil Tabellen schon existierten.
* *Lösung:* Die Datenbank musste mit Administrator-Rechten (`postgres`-User) verworfen und neu erstellt werden, bevor der Import gelang.


* **Herausforderung: Mount-Point Sichtbarkeit.**
* *Problem:* Nach Eintragung des Mount-Points in die Config sah der Container die Daten nicht.
* *Lösung:* Änderungen an der `.conf` erfordern zwingend einen kompletten Neustart (`pct reboot`) des Containers, da Mounts nur beim Boot initialisiert werden.


* **Herausforderung: Berechtigungen (Permissions).**
* *Problem:* Der neue Container nutzt intern den User `paperless` (UID 1000), während der alte teilweise als `root` agierte.
* *Lösung:* Rekursives Setzen der Rechte (`chown -R 1000:1000`) auf dem Datenverzeichnis stellte sicher, dass der neue Container Schreibzugriff auf die alten Daten erhielt.


* **Herausforderung: Python-Environment.**
* *Problem:* Befehle wie `manage.py` funktionierten nicht direkt.
* *Lösung:* Nutzung des spezifischen Pfades zum Virtual Environment (`/opt/paperless/venv/bin/python3`).



---

#### **5. Fazit**

Die Paperless-ngx Instanz wurde erfolgreich modernisiert. Das System befindet sich nun in einem **"Healthy State"**:

1. **Update-Fähigkeit:** Dank der Standard-Installation können Updates künftig wieder einfach per Skript durchgeführt werden.
2. **Datensicherheit:** Dokumente und Datenbank-Backups liegen redundant auf dem Fileserver, getrennt vom Container.
3. **Funktionalität:** Alle Workflows (OCR, E-Mail-Import, Samba-Consume, Web-Zugriff) funktionieren wie gewohnt.

Der alte Container (ID 103) wurde deaktiviert und kann nach einer Karenzzeit (empfohlen: 7 Tage) bedenkenlos gelöscht werden.