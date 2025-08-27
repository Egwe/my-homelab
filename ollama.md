# Proxmox Guide: Ollama in an Unprivileged LXC with NVIDIA GPU Passthrough

## 1. Objective

This document details the complete, step-by-step process for deploying Ollama in a secure, **unprivileged** LXC container on Proxmox VE, with full NVIDIA GPU acceleration.

This guide is the result of a real-world installation on a system with a dual-GPU setup (Intel iGPU + NVIDIA dGPU), which presents unique challenges. The steps below represent the final, stable configuration achieved after significant troubleshooting, including fixing incorrect device passthrough, resolving driver version mismatches, and correctly configuring permissions for an unprivileged environment.

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
2.  Enable **VT-d** (Intel Virtualization Technology for Directed I/O).
3.  Disable **Secure Boot**.

### 3.2. Host NVIDIA Driver Installation

The host requires the official NVIDIA driver to be installed directly.

1.  **Find the Latest Driver:** Go to the [NVIDIA Driver Downloads](https://www.nvidia.com/Download/index.aspx) page. Select your GPU and copy the link address for the download.
2.  **Install Prerequisites:** From the Proxmox host shell, install the necessary build tools.
    ```bash
    apt update
    apt install -y build-essential pve-headers-$(uname -r)
    ```
3.  **Download and Install the Driver:**
    ```bash
    # Example using a specific driver version. Always get the latest link.
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

3.  **Authorize GID Mapping:** To avoid a container startup error, explicitly authorize the mapping on the host. Edit `/etc/subgid`:
    ```bash
    nano /etc/subgid
    ```
    Add the following lines, using the GIDs you found above. This allows the root user to map these groups for containers.
    ```
    root:44:1
    root:104:1
    ```

---

## 4. Phase 2: LXC Container Setup

This setup uses the Proxmox VE Helper Scripts for initial creation, followed by manual configuration for the GPU passthrough.

### 4.1. Initial Container Creation

1.  Run the Ollama helper script from the Proxmox host shell:
    ```bash
    bash -c "$(wget -qLO - https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/ollama.sh)"
    ```
2.  Follow the prompts. **Crucially, when the script asks about GPU passthrough, say NO**. The script defaults to the Intel iGPU, which is incorrect for this setup. We will configure the NVIDIA GPU manually.
3.  Complete the setup to create a standard, unprivileged Ollama container.

### 4.2. Manual GPU Passthrough Configuration

1.  **Stop the container** (`pct stop <CT_ID>`).
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

3.  **Add Ollama User to Groups:**
    ```bash
    usermod -aG video,render ollama
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

## 6. Phase 4: Final Verification

1.  Enter the container's shell again (`pct enter <CT_ID>`).
2.  **Primary Verification (`nvidia-smi`):**
    ```bash
    nvidia-smi
    ```
    This command should now succeed and display your GPU's status, with matching driver versions and no errors.

3.  **Ollama Service Verification:**
    Check the Ollama service log to confirm it has detected the GPU.
    ```bash
    journalctl -u ollama
    ```
    Look for a confirmation line similar to this:
    `ollama[...]: Device 0: Quadro K1200, compute capability 5.0, VMM: yes`

4.  **Real-time Monitoring:**
    To watch the GPU in action, run `watch -n 1 nvidia-smi` in one terminal while running `ollama run <model>` in another. You will see the `GPU-Util` and `Memory-Usage` spike, providing definitive proof of acceleration.
