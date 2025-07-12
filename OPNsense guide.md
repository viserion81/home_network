# Home Network Core Setup with **OPNsense**
A comprehensive, step‑by‑step guide for designing, configuring, and hardening a modern home network core using OPNsense, VLANs, WireGuard, and managed switches & access points.

> **Hardware assumed**  
> * ODROID H4+ (or similar) running OPNsense  
> * XikeStor 2.5 Gb managed switch  
> * Netgear Orbi RBR750 in Access‑Point mode  
> * Vodafone cable modem in bridge/DMZ mode  

---

## Table of Contents
1. [Verify Public IPv4 & Bridge‑Mode](#1-verify-public-ipv4--enable-bridge-mode-on-vodafone-modem)
2. [OPNsense Initial Setup](#2-opnsense-initial-setup-firmware-update-interface-assignments-optional-lagg-and-vlan-creation)  
   2.1. Installation  
   2.2. Wizard & Firmware Update  
   2.3. Interface Assignments  
   2.4. Optional LAGG  
   2.5. VLAN Creation  
3. [Per‑VLAN DHCP, DNS & Firewall](#3-per-vlan-dhcp-scopes-unbound-split-horizon-dns-and-firewall-rules)
4. [Remote Access & Port Forwarding](#4-wireguard-road-warrior-setup-and-nat-80443-to-future-reverse-proxy-host)
5. [Switch Configuration](#5-switch-configuration-xikestor-managed-25-gb-switch)
6. [Orbi AP Configuration](#6-orbi-reconfiguration-to-pure-ap-with-home--guest-ssids-dhcp-disabled)
7. [Hardening Checklist](#hardening-checklist)

---

## 1. Verify Public IPv4 & Enable Bridge‑Mode on Vodafone Modem
*Consult your Vodafone modem manual—the UI differs between models.*

1. **Activate Bridge/DMZ mode**  
   Navigate to the modem’s admin page → *Advanced* → *Bridge / DMZ*.  
   Save, reboot, and ensure the modem passes the public IP to OPNsense.

2. **Connect OPNsense WAN** to a LAN port on the modem.

3. **Validate**  
   ```bash
   ping -c2 9.9.9.9     # should succeed
   ping -c2 google.com  # succeeds if DNS works
   ```

---

## 2. OPNsense Initial Setup, Firmware Update, Interface Assignments, Optional LAGG, and VLAN Creation

### 2.1 Installation
1. **Download** the *vga* image → flash to USB (Etcher).  
2. **Boot** ODROID H4+ → run installer (`username: installer`, `password: opnsense`).  
3. Pick **Install (ZFS)**, destroy disk, set **root password**, reboot.

### 2.2 First‑Boot Wizard & Firmware Update
*Access GUI at* `https://192.168.1.1`.

* Skip wizard (click logo) to learn the layout.  
* **System ▸ Settings ▸ General** – hostname, domain, timezone, DNS.  
* **System ▸ Firmware ▸ Status** – apply all updates, reboot if required.  
* **Plugins** – install Realtek driver if 2.5 Gb NICs need it.

### 2.3 Interface Assignments
* **Disable hardware off‑loading** under *Interfaces ▸ Settings* to avoid IDS/IPS issues.  
* **WAN**: DHCP(v6), block private/bogon if public IP.  
* **LAN**: static `192.168.1.1/24`, IPv6 track interface.

### 2.4 Optional LAGG
Skip unless you truly need >1 Gbps aggregated throughput.

### 2.5 Create Four VLANs
Parent = fastest LAN NIC (e.g., `igc1`).

| VLAN | Purpose   | Gateway IP          |
|------|-----------|---------------------|
| 1    | LAN       | 192.168.1.1/24      |
| 20   | IoT       | 192.168.20.1/24     |
| 40   | Servers   | 192.168.40.1/24     |
| 99   | Management| 192.168.99.1/24     |

*Interfaces ▸ Other Types ▸ VLAN* → add tags, then *Interfaces ▸ Assignments* → map & enable each.

---

## 3. Per‑VLAN DHCP Scopes, Unbound Split‑Horizon DNS, and Firewall Rules

### 3.1 DHCP
*Services ▸ DHCPv4 ▸ VLAN* – enable, set ranges (e.g., `192.168.20.100–200`).  
Reserve static addresses under *Leases* after devices appear.

### 3.2 Unbound DNS
Enable DNSSEC, register DHCP leases, add DoT upstream (Quad9, Cloudflare), enable DNSBL & cron.

### 3.3 Firewall Isolation
Create `PrivateNetworks` alias (`10.0.0.0/8,172.16.0.0/12,192.168.0.0/16`).  
For each VLAN:

1. Allow **DNS** to firewall (`port 53`).  
2. Allow **any** to **!PrivateNetworks** (internet only).  

Add specific rules for LAN ↔ Servers, MGMT → firewall UI as needed.

---

## 4. WireGuard “Road‑Warrior” Setup & NAT 80/443

### 4.1 WireGuard VPN
*VPN ▸ WireGuard* → create instance (`10.10.10.1/24`), add peers (`/32`).  
Assign interface, hybrid NAT for `WG_VPN net`, allow UDP `51820` on WAN.

### 4.2 Port Forward to Reverse‑Proxy
*Firewall ▸ NAT ▸ Port Forward* – forward TCP 80/443 on WAN to static IP of Nginx Proxy Manager in **Servers VLAN**.

---

## 5. Switch Configuration (XikeStor 2.5 Gb)

1. **Change default IP** to `192.168.1.2`, gateway `192.168.1.1`.  
2. **Create VLANs 20, 40, 99**.  
3. **Trunk port** to OPNsense: *Tagged* VLAN 1,20,40,99, PVID 1.  
4. **Access ports**: Untagged, PVID = VLAN ID.  
5. Save to NVRAM!

---

## 6. Orbi RBR750 as Pure AP

* WAN port → trunk on switch (tagged 1,20,40).  
* Set **AP Mode**, disable DHCP.  
* Create SSIDs:  

   | SSID          | VLAN | Notes                    |
   |---------------|------|--------------------------|
   | MyHomeWiFi    | 1    | main LAN                 |
   | MyIoTDevices  | 20   | client isolation on      |
   | GuestWiFi     | 40   | client isolation on      |

---

## Hardening Checklist
* Change **all** default passwords.  
* Create non‑root admin on OPNsense; consider disabling `root` GUI login.  
* SSH key auth only.  
* Cron jobs: update DNSBL & IDS/IPS rules daily.  
* Back‑up config before firmware upgrades.  
* Consider 2FA for GUI if/when supported.  
* Restrict management access via firewall rules or dedicated MGMT VLAN.

---

### 🧪 Validation Quick‑Sheet
* `ping 9.9.9.9`, `ping google.com` from each VLAN.  
* Cross‑VLAN pings must **fail**.  
* WireGuard client: ping LAN, internet.  
* External: hit `https://your-service.domain` (via reverse proxy).  

---

**License:** CC‑BY‑SA 4.0  
**Author:** Adapted for Marc  
