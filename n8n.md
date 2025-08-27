# Proxmox Guide: Secure, Self-Hosted n8n with a Reverse Proxy

## 1. Objective

This document details the complete, step-by-step process for deploying n8n (a workflow automation tool) in a secure, **unprivileged** LXC container on Proxmox VE. It also covers the critical post-installation steps required for stable remote access via a reverse proxy.

This guide is the result of a real-world installation and addresses several common challenges that are often overlooked. The steps below represent the final, stable configuration after significant troubleshooting, including:
*   Correctly configuring the application to work behind a reverse proxy.
*   Ensuring data privacy by disabling all telemetry and diagnostic reporting.
*   Resolving the common "Connection Lost" UI error by enabling WebSocket support.

---

## 2. System Specifications

*   **Proxmox VE Version:** 8.x
*   **Core Prerequisite:** A functioning Reverse Proxy (this guide assumes Nginx Proxy Manager).
*   **Core Prerequisite:** A custom domain name configured to point to the reverse proxy.

---

## 3. Phase 1: LXC Container Creation

This setup uses the Proxmox VE Helper Scripts for a quick and standardized initial deployment.

1.  **Run the n8n helper script** from the Proxmox host shell:
    ```bash
    bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/n8n.sh)"
    ```
2.  Follow the prompts to complete the standard, unprivileged container setup.
3.  After installation, note the container's local IP address, as it will be needed for the subsequent configuration steps.

---

## 4. Phase 2: Container Configuration

For n8n to function correctly and privately, several environment variables must be configured. The installation script conveniently places all configuration variables in a single file.

1.  **Enter the container's shell** (`pct enter <CT_ID>`).

2.  **Edit the environment file:**
    ```bash
    sudo nano /opt/n8n.env
    ```

3.  **Add the required configuration.** The file may be empty or contain other variables. Add the following lines:

    ```env
    # --- Webhook & Privacy Configuration ---

    # 1. Sets the public-facing URL for generating correct webhook links.
    #    Replace 'n8n.your-domain.com' with your actual public subdomain.
    WEBHOOK_URL=https://n8n.your-domain.com/

    # 2. Disables all telemetry and diagnostic data reporting to n8n's servers.
    N8N_TELEMETRY_ENABLED=false
    N8N_DIAGNOSTICS_ENABLED=false
    ```

4.  **Save the file** and exit the editor.

5.  **Restart the n8n service** to apply the new environment variables. This step is mandatory.
    ```bash
    sudo systemctl restart n8n.service
    ```

---

## 5. Phase 3: Remote Access Configuration (Nginx Proxy Manager)

With the container properly configured, the final step is to make it securely accessible from the internet.

1.  **In the Nginx Proxy Manager UI**, navigate to `Hosts` -> `Proxy Hosts` and click `Add Proxy Host`.

2.  **Configure the host** with the following settings:

    *   **`Details` Tab:**
        *   **Domain Names:** The full public subdomain (e.g., `n8n.your-domain.com`).
        *   **Scheme:** `http`
        *   **Forward Hostname / IP:** The local IP of the n8n LXC container.
        *   **Forward Port:** `5678`
        *   **Websockets support:** **Enable this toggle.** This is the critical step to prevent the "Connection Lost" UI error.

    *   **`SSL` Tab:**
        *   **SSL Certificate:** Select `Request a new SSL Certificate`.
        *   **Force SSL:** Enable this toggle.

3.  **Save the Proxy Host.** Nginx Proxy Manager will now handle traffic routing and SSL for your n8n instance.

---

## 6. Phase 4: Final Verification

1.  **Primary Verification (UI Access):**
    Navigate to your public domain (`https://n8n.your-domain.com`). The n8n login page should load securely with a valid SSL certificate. You should not experience any "Connection Lost" pop-ups.

2.  **Webhook URL Verification:**
    Log in to n8n, create a new workflow, and add a "Webhook" trigger node. Inspect the "Webhook URLs" section in the node's properties. The URL should correctly display your public domain, confirming that the `WEBHOOK_URL` environment variable was successfully applied. This is the definitive proof of a correct setup.
