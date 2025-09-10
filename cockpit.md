# Proxmox Guide: Unprivileged NAS with Cockpit and Samba

## 1. Objective

This document details the complete, step-by-step process for deploying a high-performance Network Attached Storage (NAS) server in a secure, **unprivileged** LXC container on Proxmox VE.

This guide utilizes Cockpit for web-based management, the 45Drives Cockpit modules for simplified file sharing, and Samba for robust, cross-platform network access. The configuration leverages a Proxmox bind mount, allowing the container to securely access a storage directory directly from the host. The final result is a stable, easily managed, and secure home lab NAS.

---

## 2. System Specifications

*   **Proxmox VE Version:** 8.x
*   **Container OS:** Debian 12 (Bookworm)
*   **Management UI:** Cockpit with 45Drives Modules (File Sharing, Identities, Navigator)
*   **File Sharing Protocol:** Samba
*   **Storage Path (Host):** `/mnt/file-server` (Example path, adjust as needed)

---

## 3. Phase 1: Proxmox Host Preparation

The host system must be prepared to provide the storage directory and grant the unprivileged container the necessary permissions to manage it.

### 3.1. Prepare Host Storage Directory

Ensure the directory you intend to share exists on the Proxmox host. If not, create it.

```bash
# This example uses /mnt/file-server. Change this path to match your setup.
mkdir -p /mnt/file-server
```

### 3.2. Set Host Permissions for Unprivileged Container

To allow an unprivileged container to manage files, the directory ownership on the host must be mapped to the `nobody` user and group (UID/GID `100000`). This is a critical security step.

1.  **Change Ownership and Permissions:** From the Proxmox host shell, run the following commands. This gives the container's root user full control over the directory.
    ```bash
    chown -R 100000:100000 /mnt/file-server
    chmod -R 775 /mnt/file-server
    ```
2.  **Verification:** Confirm the ownership has been set correctly.
    ```bash
    ls -ld /mnt/file-server
    ```
    The output must show `100000 100000` as the owner and group.

---

## 4. Phase 2: LXC Container Setup

This setup uses a Proxmox Community Helper Script for initial creation, followed by a manual configuration for the storage bind mount.

### 4.1. Initial Container Creation

1.  From the Proxmox host shell, run the helper script to create a Debian container with Cockpit pre-installed.
    ```bash
    bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/cockpit.sh)"
    ```
2.  Follow the prompts. In the advanced settings, it is crucial to select **Unprivileged**.
3.  Complete the setup. **Do not start the container yet.**

### 4.2. Configure Storage Bind Mount

1.  Ensure the container is stopped (`pct stop <CT_ID>`).
2.  **Edit the container's configuration file** (e.g., `/etc/pve/lxc/100.conf`). Replace `100` with your container's ID.
3.  Add the following line to the end of the file. This makes the host's directory available inside the container.

    ```conf
    # --- Storage Bind Mount ---
    mp0: /mnt/file-server,mp=/mnt/file-server
    ```

---

## 5. Phase 3: Software Configuration Inside the Container

1.  **Start the container** (`pct start <CT_ID>`) and **enter its shell** (`pct enter <CT_ID>`).

2.  **Clean Up and Update:**
    ```bash
    # From INSIDE the container's shell
    apt update
    rm /root/*.deb 2>/dev/null || true # Clean up any previous downloads
    ```

3.  **Download and Install 45Drives Cockpit Modules:**
    These modules provide the user-friendly interface for managing users and shares. The following links are the validated, correct versions for a Debian 12 (Bookworm) system.

    ```bash
    # Download Cockpit File Sharing (Bookworm version)
    wget https://github.com/45Drives/cockpit-file-sharing/releases/download/v4.3.1-2/cockpit-file-sharing_4.3.1-2bookworm_all.deb

    # Download Cockpit Navigator (Focal version is compatible)
    wget https://github.com/45Drives/cockpit-navigator/releases/download/v0.5.10/cockpit-navigator_0.5.10-1focal_all.deb

    # Download Cockpit Identities (Focal version is compatible)
    wget https://github.com/45Drives/cockpit-identities/releases/download/v0.1.12/cockpit-identities_0.1.12-1focal_all.deb
    ```

4.  **Install the Modules and Dependencies:**
    This command installs all three `.deb` packages and automatically pulls in all required dependencies, including the Samba server itself.

    ```bash
    apt install /root/*.deb
    ```
5.  **Reboot the container** to ensure all services start correctly (`reboot`).

---

## 6. Phase 4: User and Share Configuration (Cockpit UI)

1.  Open your Cockpit web interface (`http://<CT_IP>:9090`).

2.  **Create a Samba User:**
    *   Navigate to the **Identities** tab.
    *   Click "Create new user". Enter the username (e.g., `jonathan`) and a strong password.
    *   After creating the user, click on their name in the list.
    *   Scroll down to the **Samba Password** section, set the password (it can be the same as the user password), and click "Change".

3.  **Create the Samba Share:**
    *   Navigate to the **File Sharing** tab.
    *   Scroll to the "Samba Shares" section and click the `+` button.
    *   Configure the share with the following settings:
        *   **Share Name:** A simple name for the network (e.g., `file-server`).
        *   **Path:** The path to the bind mount: `/mnt/file-server`.
        *   **Guest OK:** **OFF** (to enforce user login).
        *   **Read Only:** **OFF** (to allow write access).
        *   **Browseable:** **ON** (to make the share visible on the network).
        *   **Inherit Permissions:** **ON**. This is the modern equivalent of "Windows ACLs" and is critical for correct permission handling.
    *   Click **Apply**.

---

## 7. Phase 5: Final Permissions and Verification

This final step links the authenticated Samba user to the Linux file system, granting write permissions.

1.  **Enter the container's shell again** (`pct enter <CT_ID>`).

2.  **Set Directory Ownership:**
    Change the ownership of the storage directory to the user you created. This grants the user full control over all files and folders within the share. Replace `jonathan` with your actual username.

    ```bash
    # From INSIDE the container's shell
    chown -R jonathan:jonathan /mnt/file-server
    ```

3.  **Connect to the Network Share:**
    *   From a client machine (e.g., Windows), open File Explorer.
    *   In the address bar, type `\\<container_ip_address>\file-server`.
    *   When prompted, enter the username (`jonathan`) and the **Samba password** you set.

4.  **Final Verification:**
    To confirm everything is working, create a new folder or text file within the mapped network drive. The operation should succeed without any permission errors. Your NAS is now fully configured and operational.
