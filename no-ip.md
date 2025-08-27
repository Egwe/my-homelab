# Proxmox Guide: Dynamic DNS for Remote Access with No-IP

## 1. Objective

This document details the foundational step for enabling remote access to my homelab: configuring a Dynamic DNS (DDNS) service. My home internet connection has a dynamic public IP address that changes periodically. DDNS provides a stable, unchanging hostname that always points to my home network's current IP.

This guide is the result of a real-world installation and represents the critical link between my public domain and my home server. The steps below cover the creation of a free DDNS hostname, the automation of IP updates via my router, and the final connection to my public domain using a `CNAME` record. This configuration is a **prerequisite** for Nginx Proxy Manager to function correctly.

---

## 2. System Specifications & Prerequisites

*   **Router:** A consumer router with a built-in DDNS client (this guide uses a FRITZ!Box as the example).
*   **Domain Registrar:** Access to the DNS management panel for my public domain.
*   **External Service:** A free account at [No-IP.com](https://www.noip.com/).

---

## 3. Phase 1: No-IP Account and Hostname Creation

The first step is to create a stable hostname with the DDNS provider that my router can update.

1.  **Create a Free Account:** I signed up for a free account at [No-IP.com](https://www.noip.com/).

2.  **Create a DDNS Hostname:** After logging in, I navigated to the Dynamic DNS section and created a new hostname.

3.  **Configure the Hostname:**
    *   **Hostname:** I chose a unique and memorable name (e.g., `my-homelab-dns`).
    *   **Domain:** I selected one of the free domain suffixes provided by No-IP (e.g., `ddns.net`).
    *   **Record Type:** The default `A` record was correct.
    *   This resulted in a full DDNS hostname like `my-homelab-dns.ddns.net`.

4.  **Save the Hostname.** This name now serves as the stable target that my dynamic home IP will be associated with.

---

## 4. Phase 2: Router Configuration (DDNS Client)

This phase automates the IP update process, making the setup resilient to IP changes.

1.  **Log in to my Router's Web UI.**

2.  **Navigate to the DynDNS Settings.** On my FRITZ!Box, this is located under `Internet` -> `Permit Access` -> `DynDNS`.

3.  **Configure the DDNS Client:**
    *   **DynDNS Provider:** I selected `No-IP.com` from the list.
    *   **Domain Name:** I entered the full No-IP hostname created in Phase 1 (e.g., `my-homelab-dns.ddns.net`).
    *   **User Name:** My No-IP account username (or email).
    *   **Password:** My No-IP account password.

4.  **Apply the Settings.** My router is now responsible for automatically notifying No-IP whenever my public IP address changes.

---

## 5. Phase 3: Public Domain Configuration (CNAME Record)

The final step is to link my easy-to-remember public subdomains to the functional DDNS hostname. This is done at my domain registrar.

1.  **Log in to my Domain Registrar's control panel.**

2.  **Navigate to the DNS settings** for my parent domain (e.g., `your-domain.com`).

3.  **Create a New `CNAME` Record** for each service I want to expose. For example, for n8n:
    *   **Type:** `CNAME`
    *   **Hostname:** `n8n` (This represents `n8n.your-domain.com`).
    *   **Value / Target / Points to:** The full No-IP hostname (`my-homelab-dns.ddns.net`).

4.  **Save the Record.** I repeated this process for any other subdomains I needed. This creates an alias, effectively telling the internet "for the IP of `n8n.your-domain.com`, go ask `my-homelab-dns.ddns.net`."

---

## 6. Phase 4: Final Verification

1.  **Use a Public DNS Checking Tool,** such as [dnschecker.org](https://dnschecker.org/).

2.  **Primary Verification (CNAME Lookup):**
    I performed a `CNAME` record lookup for one of my service subdomains (e.g., `n8n.your-domain.com`). The result correctly showed that it points to my No-IP hostname (`my-homelab-dns.ddns.net`).

3.  **Secondary Verification (A Record Lookup):**
    I then performed an `A` record lookup for my No-IP hostname (`my-homelab-dns.ddns.net`). The result correctly showed my home's current public IP address.

When both of these checks pass, the entire DNS chain is confirmed to be working correctly, and traffic intended for my public subdomains will be successfully directed to my home network.
