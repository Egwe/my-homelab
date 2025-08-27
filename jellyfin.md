# Proxmox Guide: Jellyfin in an Unprivileged LXC with NVIDIA GPU Passthrough

## 1. Objective

This document details the complete, step-by-step process for deploying Jellyfin Media Server in a secure, **unprivileged** LXC container on Proxmox VE, with full NVIDIA GPU acceleration for hardware transcoding.

This guide provides a stable, repeatable configuration for achieving high-performance transcoding, which offloads CPU-intensive tasks to the dedicated GPU. The steps below are designed to correctly handle driver installation, device passthrough, and the specific permissions required for an unprivileged container, ensuring a robust and secure media server.

---

## 2. System Specifications

*   **Server:** Dell Optiplex 7010
*   **CPU:** 3rd Gen Intel Core i5-3470 (with integrated HD2500 Graphics)
*   **Dedicated GPU:** NVIDIA Quadro K1200
*   **Proxmox VE Version:** 8.x

---

## 3. Phase 1: Proxmox Host Preparation

The host system must be correctly prepared to handle the NVIDIA driver and authorize the passthrough to a secure container.

### 3.1. BIOS Configuration

1.  Reboot the Proxmox host and enter the BIOS setup.
2.  Enable **VT-d** (Intel Virtualization Technology for Directed I/O) or **AMD-Vi**.
3.  Disable **Secure Boot**.

### 3.2. Host NVIDIA Driver Installation

The host requires the official NVIDIA driver to be installed directly. The driver version on the host and in the container **must match**.

1.  **Find the Driver:** Go to the [NVIDIA Driver Downloads](https://www.nvidia.com/Download/index.aspx) page. Select your GPU and copy the link address for the download.
2.  **Install Prerequisites:** From the Proxmox host shell, install the necessary build tools.
    ```bash
    apt update
    apt install -y build-essential pve-headers-$(uname -r)
    ```
3.  **Download and Install the Driver:**
    ```bash
    # Example using a specific driver version. Always get the correct link for your GPU.
    wget https://us.download.nvidia.com/XFree86/Linux-x86_64/580.76.05/NVIDIA-Linux-x86_64-580.76.05.run
    
    chmod +x NVIDIA-Linux-x86_64-*.run
    ./NVIDIA-Linux-x86_64-*.run
    ```
4.  Follow the on-screen installer prompts. After completion, **reboot the Proxmox host**.

### 3.3. Host Verification and Permissions Setup

1.  **Verify Driver:** After rebooting, confirm the driver is loaded correctly on the host. This is a critical checkpoint.
    ```bash
    nvidia-smi
    ```
    The command must succeed and display your GPU details.

2.  **Get Group IDs (GIDs):** We need the GIDs for the `video` and `render` groups to map them into the container.
    ```bash
    getent group video   # Note the number (e.g., 44)
    getent group render  # Note the number (e.g., 104)
    ```

3.  **Authorize GID Mapping:** To prevent a container startup error, explicitly authorize the mapping on the host. Edit `/etc/subgid`:
    ```bash
    nano /etc/subgid
    ```
    Add the following lines, using the GIDs you found above. This allows the `root` user to map these groups for containers.
    ```
    root:44:1
    root:104:1
    ```

---

## 4. Phase 2: LXC Container Setup

This setup uses the Proxmox VE Helper Scripts for initial creation, followed by manual configuration for the GPU passthrough.

### 4.1. Initial Container Creation

1.  Run the Jellyfin helper script from the Proxmox host shell:
    ```bash
    bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/jellyfin.sh)"
    ```
2.  Follow the prompts. In the advanced settings, ensure the container is **Unprivileged**.
3.  Complete the setup to create a standard Jellyfin container. **Do not start it yet.**

### 4.2. Manual GPU Passthrough Configuration

1.  Make sure the container is stopped (`pct stop <CT_ID>`).
2.  **Edit the container's configuration file** (e.g., `/etc/pve/lxc/105.conf`).
3.  Add the following complete block to the end of the file. This is the core configuration that enables the passthrough and permissions.

    > **Note:** Replace `44` and `104` if your system GIDs were different.

    ```conf
    # --- NVIDIA Passthrough & ID Mapping ---

    # 1. Allow the container to access NVIDIA character devices (major number 195)
    lxc.cgroup2.devices.allow: c 195:* rwm

    # 2. Mount the essential NVIDIA device nodes into the container
    lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
    lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
    lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file

    # 3. Map host GIDs to the container for permissions in an unprivileged environment
    lxc.idmap: u 0 100000 65536
    lxc.idmap: g 0 100000 44
    lxc.idmap: g 44 44 1
    lxc.idmap: g 45 100045 59
    lxc.idmap: g 104 104 1
    lxc.idmap: g 105 100105 65431
    ```

---

## 5. Phase 3: Software Configuration Inside the Container

1.  **Start the container** (`pct start <CT_ID>`) and **enter its shell** (`pct enter <CT_ID>`).

2.  **Create Matching Groups:**
    ```bash
    # From INSIDE the container's shell
    groupadd --gid 44 video
    groupadd --gid 104 render
    # A 'group already exists' error is harmless and can be ignored.
    ```

3.  **Add Jellyfin User to Groups:**
    ```bash
    usermod -aG video,render jellyfin
    ```

4.  **Download and Install the Matching NVIDIA Driver Libraries:**
    To avoid a `Driver/library version mismatch` error, we must install the **exact same driver version** as the host, but *without* the kernel module.

    ```bash
    # Inside the container, download the same .run file used on the host
    wget https://us.download.nvidia.com/XFree86/Linux-x86_64/580.76.05/NVIDIA-Linux-x86_64-580.76.05.run
    
    chmod +x NVIDIA-Linux-x86_64-*.run
    
    # Run the installer with the crucial --no-kernel-module flag
    ./NVIDIA-Linux-x86_64-*.run --no-kernel-module
    ```
    Follow the on-screen prompts to complete the installation of the user-space libraries.

5.  **Reboot the container** to ensure all changes are applied (`reboot`).

---

## 6. Phase 4: Final Verification and Configuration

1.  Enter the container's shell again (`pct enter <CT_ID>`).
2.  **Primary Verification (`nvidia-smi`):**
    ```bash
    nvidia-smi
    ```
    This command should now succeed and display your GPU's status, with matching driver versions and no errors.

3.  **Jellyfin Dashboard Configuration:**
    *   Open your Jellyfin web interface (`http://<CT_IP>:8096`).
    *   Navigate to **Dashboard** -> **Playback**.
    *   Set the **Hardware Acceleration** dropdown to **NVIDIA NVENC**.
    *   Enable the hardware encoding options you need (e.g., H.264, HEVC).
    *   Click **Save**.

4.  **Real-time Transcoding Verification:**
    To confirm everything is working, play a video file from a client that forces a transcode (e.g., by setting the playback quality lower than the source).
    
    While the video is playing, run `watch -n 1 nvidia-smi` in the container's terminal. You will see the `GPU-Util` percentage and `Memory-Usage` increase, providing definitive proof that Jellyfin is using the GPU for transcoding. The "Processes" section should also show a `jellyfin-ffmpeg` process.
