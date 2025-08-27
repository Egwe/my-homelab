# My Proxmox Home Lab

Welcome to my home lab repository! This is a living document of my personal server setup, running on Proxmox VE. The purpose of this repository is to store configurations, notes, and detailed guides for the services I run, both for my own reference and to share with the home lab community.

My lab is focused on self-hosting essential services, IoT automation, and experimenting with local AI/LLM technologies.

---

## Hardware

The entire lab runs on a single, power-efficient machine:

*   **Server:** Dell Optiplex 7010
*   **CPU:** 3rd Gen Intel Core i5-3470 (with integrated HD2500 Graphics)
*   **GPU:** NVIDIA Quadro K1200 (Primarily for AI/ML workloads)
*   **Memory:** 16 GB RAM
*   **Storage:** LVM-Thin on local NVMe storage

---

## Virtualization Platform

*   **Hypervisor:** Proxmox VE
*   **Version:** 8.4.6

---

## Services Overview

All services are deployed as lightweight, unprivileged LXC containers unless otherwise specified for compatibility reasons.

| ID  | Service Name          | Type           | Description                                                               |
|-----|-----------------------|----------------|---------------------------------------------------------------------------|
| 100 | **n8n**               | Unprivileged LXC | A workflow automation tool to connect various apps and services.          |
| 101 | **NGINX Proxy Manager** | Unprivileged LXC | Manages reverse proxying, SSL certificates, and public access to services.|
| 102 | **MQTT**              | Unprivileged LXC | An MQTT broker for managing IoT device communication and smart home data. |
| 103 | **Nextcloud-VM**      | **VM**         | A full-featured private cloud for file hosting, contacts, and calendars.  |
| 104 | **AdGuard Home**      | Unprivileged LXC | Network-wide ad and tracker blocking via DNS.                             |
| 105 | **Ollama**            | Unprivileged LXC | The backend server for running local large language models (LLMs).        |
| 106 | **Open WebUI**        | Unprivileged LXC | A user-friendly, ChatGPT-style web interface for interacting with Ollama. |

---

## Configuration & Documentation

This repository contains detailed, step-by-step guides for some of the more complex setups in this lab.

*   **[Dynamic DNS (No-IP) Setup Guide](./no-ip.md)**: Walks through the process of setting up a Dynamic DNS service using No-IP to handle a non-static home IP address. Covers creating the hostname, configuring a router to automate IP updates, and linking a custom domain via a `CNAME` record.

*   **[Nginx Proxy Manager Guide](./nginx-proxy-manager.md)**: A guide to setting up Nginx Proxy Manager as the central gateway for secure remote access. Covers the initial LXC installation, security setup, and the process of configuring a Proxy Host with automated SSL for a backend service.

*   **[n8n Self-Hosting Guide](./n8n.md)**: Details the deployment of n8n in an LXC and the critical post-installation steps. Focuses on configuring the environment for remote access (Webhook URL), ensuring privacy (disabling telemetry), and resolving the "Connection Lost" UI bug via WebSocket support.

*   **[Ollama NVIDIA GPU Passthrough Guide](./ollama.md)**: A complete walkthrough of the process for passing an NVIDIA GPU to an unprivileged LXC container, including driver installation, troubleshooting, and final verification for the Ollama service.

*(More guides will be added as the lab evolves.)*

---

## Future Goals

*   Expand IoT automation using MQTT and n8n.
*   Experiment with different LLMs and fine-tuning models on the Ollama instance.
*   Deploy a dedicated monitoring stack (e.g., Grafana + Prometheus).
