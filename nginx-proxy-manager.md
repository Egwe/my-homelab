# Proxmox Guide: Nginx Proxy Manager for Secure Remote Access

## 1. Objective

This document details the complete, step-by-step process for deploying **Nginx Proxy Manager (NPM)** in a secure, **unprivileged** LXC container on Proxmox VE. NPM serves as the central gateway for securely exposing multiple internal services to the internet.

This guide is the result of a real-world installation and represents the foundational component of my homelab's remote access strategy. It addresses the core challenges of:
*   Centralizing the management of all incoming web traffic.
*   Automating the creation and renewal of SSL certificates from Let's Encrypt.
*   Routing subdomain requests (e.g., `n8n.your-domain.com`) to the correct internal service without exposing multiple ports on the router.

---

## 2. System Specifications & Prerequisites

*   **Proxmox VE Version:** 8.x

For NPM to function correctly, the following external infrastructure **must** be configured first:

*   **Registered Domain Name:** A public domain to which subdomains will be added.
*   **Dynamic DNS (DDNS) Service:** A configured DDNS hostname (e.g., from No-IP) that points to your home's public IP address.
*   **Router Port Forwarding:** Your router must forward incoming traffic on ports `80` (for HTTP) and `443` (for HTTPS) to the local IP address of the NPM container.
*   **Public DNS Records:** Your domain's DNS must have `CNAME` records for each service subdomain, pointing to your DDNS hostname.

---

## 3. Phase 1: LXC Container Creation

This setup uses the Proxmox VE Helper Scripts for a standardized deployment.

1.  **Run the Nginx Proxy Manager helper script** from the Proxmox host shell:
    ```bash
    bash -c "$(wget -qLO - https://raw.githubusercontent.com/tteck/Proxmox/main/ct/nginx-proxy-manager.sh)"
    ```
2.  Follow the prompts to complete the standard, unprivileged container setup.
3.  After installation, note the container's local IP address. This IP is essential for both the initial setup and your router's port forwarding rules.

---

## 4. Phase 2: Initial Setup & Security

Before using NPM, you must perform the initial login and secure the default account.

1.  **Access the Web UI:** Open a browser on your local network and navigate to the admin portal:
    `http://<npm-ip-address>:81`

2.  **Log In with Default Credentials:**
    *   **Email:** `admin@example.com`
    *   **Password:** `changeme`

3.  **Secure the Admin Account:** Upon first login, you will be prompted to change the user details and password. **This is a mandatory security step.**

---

## 5. Phase 3: Configuration of a Proxy Host

A "Proxy Host" is a rule that tells NPM how to route traffic for a specific subdomain. The following steps use `n8n` as an example.

1.  **Navigate to the Proxy Hosts Page:** In the NPM UI, go to `Hosts` -> `Proxy Hosts`.

2.  **Add a New Proxy Host:** Click the `Add Proxy Host` button.

3.  **Configure the Host Details:** Fill out the form with the following information:

    *   **`Details` Tab:**
        *   **Domain Names:** The full public subdomain (e.g., `n8n.your-domain.com`).
        *   **Scheme:** `http`
        *   **Forward Hostname / IP:** The local IP address of the backend service's container (e.g., the n8n LXC IP).
        *   **Forward Port:** The port the backend service is listening on (e.g., `5678` for n8n).
        *   **Websockets support:** **Enable this toggle.** This is critical for real-time services like n8n to avoid "Connection Lost" errors.

    *   **`SSL` Tab:**
        *   **SSL Certificate:** Select `Request a new SSL Certificate` from the dropdown.
        *   **Force SSL:** Enable this toggle to ensure all connections are automatically upgraded to HTTPS.
        *   **Agree to the Let's Encrypt Terms.**

4.  **Save the Proxy Host.** NPM will now attempt to acquire an SSL certificate. If your DNS and port forwarding are correct, the status will turn green.

---

## 6. Phase 4: Final Verification

1.  **Primary Verification (External Access):**
    From a device outside your local network (e.g., a smartphone on mobile data), navigate to the public URL you configured (`https://n8n.your-domain.com`). The service's web page should load correctly.

2.  **SSL Certificate Verification:**
    Check your browser's address bar. There should be a padlock icon, indicating that the connection is secure and the SSL certificate obtained by NPM is valid and trusted.

3.  **Troubleshooting (Audit Log):**
    If the host shows as "Offline" or you encounter errors, the first place to check is the **Audit Log** in the top-right of the NPM UI. It provides detailed error messages about SSL certificate failures or backend connection issues, which are invaluable for diagnostics.
