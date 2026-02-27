# Logbook: Nginx Proxy Manager

## 2026-02-26 VLAN Migration & Startup Timeout Fix
**Objective:** Move NPM to VLAN 30 (DMZ), establish static IP assignment, and fix service freeze upon system reboot.
**Context:** DHCP assignment in the new VLAN 30 failed dynamically, causing the service to be unreachable. OpenResty requires immediate DNS resolution at startup, otherwise, the start sequence hangs and times out.

**Actions Taken:**
1. Changed Proxmox network config (`net0`) to static IPv4: `10.10.30.51/24`.
2. Assigned custom DNS server `192.168.142.45` (AdGuard) in Proxmox LXC settings.
3. Created Systemd drop-in file `/etc/systemd/system/openresty.service.d/override.conf`:
    * Added `TimeoutStartSec=600`
    * Added `Restart=on-failure`
    * Added `RestartSec=5`
4. Reset service auto-start bindings: `systemctl disable openresty` followed by `systemctl enable openresty`.
5. Adjusted Home Assistant `configuration.yaml` to accept `10.10.30.51` as a trusted proxy.

**Errors & Troubleshooting:**
* **Error:** System freeze and `inactive (dead)` status for OpenResty service after reboot. Ping tests showed immediate 0ms response to AdGuard, but `systemd` strict limits aborted the start.
* **Fix:** Applied the override config to give OpenResty more time to resolve domains during boot.

**Outcome:**
* Status: ⚠️ Partial (Service operates flawlessly, but auto-start is inconsistent; `systemctl start openresty` via Proxmox console may be required after a host reboot).
* **Next Steps:** Monitor auto-start behavior on next full system maintenance window.