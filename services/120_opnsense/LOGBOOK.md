# Logbook: OPNsense

## 2026-02-26 Firewall Access Recovery & DMZ Routing
**Objective:** Regain access to the Web GUI after changing the default port and configure secure routing for the new DMZ (VLAN 30).
**Context:** The Web GUI port was changed to 8443, but the existing WAN firewall rule still only allowed port 443, locking out the admin. Concurrently, network changes required strict routing between VLAN 30 and VLAN 20.

**Actions Taken:**
1. Regained access by disabling the packet filter via Proxmox Console: `pfctl -d`.
2. Modified WAN rule in Web GUI: Changed allowed destination port from `443` to `8443`.
3. Re-enabled packet filter via Proxmox Console: `pfctl -e`.
4. Resolved FritzBox ARP cache conflict by completely deleting the stale `AE` MAC address entry.
5. Configured DMZ Pinholing (Firewall > Rules > VLAN_30_DMZ) strictly in this order:
    * Passed NPM (`10.10.30.51`) to Home Assistant (`10.10.20.53:8123`)
    * Passed NPM to Jellyfin (`10.10.20.55:8096`)
    * Passed NPM to n8n (`10.10.20.54:5678`)
    * Passed NPM to Paperless (`10.10.20.52:8000`)
    * Blocked all traffic from `VLAN_30_DMZ net` to `VLAN_20_APP net`
    * Passed NPM to `*` (Internet & AdGuard Access)

**Errors & Troubleshooting:**
* **Error:** Web GUI completely unreachable from WAN side.
* **Fix:** Temporary shell override of firewall states to correct the port rule.

**Outcome:**
* Status: ✅ Success
* **Next Steps:** Setup WireGuard VPN on VLAN 40.