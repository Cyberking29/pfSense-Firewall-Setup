# 🛡️ pfSense Firewall — SOC Lab Setup 

> **Project:** pfSense Installation + GeoIP Blocking + Malicious IP Rules  
> **Environment:** VirtualBox / Bare-metal | pfSense 2.8.1-RELEASE  
> **Purpose:** SOC home lab — perimeter firewall with threat intelligence blocking

---

## 🔍 What is pfSense?

**pfSense** is a free, open-source firewall and router operating system based on **FreeBSD**. It is developed and maintained by Netgate and is widely used in enterprise networks, home labs, and SOC environments as a powerful, cost-effective alternative to commercial firewall appliances (e.g., Cisco ASA, Fortinet, Palo Alto).

Unlike a simple consumer router, pfSense gives you **full control** over your network perimeter — from traffic filtering and NAT to VPN, intrusion detection, and threat intelligence blocking — all through a clean web-based GUI or command line.

---

## 🤔 What Does pfSense Do?

At its core, pfSense acts as the **gatekeeper between your internal network and the outside world**. Here's what it handles:

| Function | Description |
|---|---|
| **Stateful Firewall** | Inspects packets and tracks connection state to allow or deny traffic based on rules you define |
| **Router / NAT** | Routes traffic between interfaces (WAN ↔ LAN) and translates private IPs to a public IP |
| **DHCP Server** | Automatically assigns IP addresses to devices on your internal network |
| **DNS Resolver** | Resolves domain names locally, can block malicious domains |
| **VPN Gateway** | Supports OpenVPN, WireGuard, and IPsec for secure remote access |
| **Traffic Shaping** | Prioritizes or limits bandwidth by application or host |
| **Logging & Monitoring** | Records firewall events, blocked traffic, and connection states for review |
| **Package Ecosystem** | Extensible via packages like pfBlockerNG (threat intel), Suricata (IDS/IPS), Snort, and more |

---

## 🏢 Why is pfSense Important for SOC / Blue Teams?

In a Security Operations Center context, pfSense is valuable because it gives analysts **visibility and control at the network perimeter** — which is where most threats first appear.

**1. Perimeter Defense**  
pfSense is the first line of defense. Properly configured rules stop known-bad traffic before it ever reaches internal systems — reducing alert volume and analyst fatigue.

**2. Threat Intelligence Integration**  
Via **pfBlockerNG**, pfSense can ingest IP blocklists and GeoIP data from public and commercial threat feeds, automatically blocking millions of known malicious addresses with no manual intervention.

**3. GeoIP Blocking**  
Restricting inbound and outbound traffic from high-risk countries (e.g., North Korea, Russia) is a common and effective control that reduces attack surface, especially for organizations with no legitimate business in those regions.

**4. Logging for SIEM Ingestion**  
pfSense can forward firewall logs to a SIEM (e.g., Splunk, Elastic) via syslog. Every blocked connection, matched rule, and state change becomes a data point for detection and investigation.

**5. Network Segmentation**  
Using multiple interfaces and VLANs, pfSense can segment sensitive assets (servers, OT devices) from general user traffic — a critical control for limiting lateral movement after a breach.

**6. Low Cost, High Capability**  
pfSense runs on commodity hardware or as a VM (VirtualBox, VMware, Proxmox), making it ideal for SOC home labs, training environments, and small-to-medium organizations with limited budgets.

**7. Hands-On Skill Development**  
Working with pfSense directly builds practical skills in firewall rule logic, packet filtering, network architecture, and threat blocking — all directly transferable to enterprise tools.

---

## 📋 Table of Contents
1. [Network Overview](#network-overview)
2. [Installation](#installation)
3. [Interface Assignment](#interface-assignment)
4. [LAN IP & DHCP Configuration](#lan-ip--dhcp-configuration)
5. [Web GUI Initial Setup](#web-gui-initial-setup)
6. [Malicious IP Blocking (Aliases + Rules)](#malicious-ip-blocking-aliases--rules)
7. [GeoIP Blocking (pfBlockerNG)](#geoip-blocking-pfblockerng)
8. [LAN Firewall Rule (DNS Pass)](#lan-firewall-rule-dns-pass)
9. [Final Rule Summary](#final-rule-summary)

---

## 🌐 Network Overview

```
[Internet / WAN]
       |
  em0 (WAN) — DHCP from host: 10.0.2.15/24
       |
  [ pfSense Firewall ]
       |
  em1 (LAN) — Static: 192.168.56.254/24
       |
  [Internal Network / SOC Machines]
  DHCP Pool: 192.168.56.100 – 192.168.56.200
```

---

## 1. Installation

- Boot from pfSense ISO (Netgate Installer v1.2-RELEASE)
- At the Welcome screen, select **Install → Install pfSense**
- Accept defaults for partition and disk layout
- Reboot after installation completes

---

## 2. Interface Assignment

**WAN Interface:**
- Select `em0` (MAC: `08:00:27:1c:da:da`) — connected to external/NAT network

**LAN Interface:**
- Select `em1` (MAC: `08:00:27:8d:01:00`) — connected to internal host-only network

---

## 3. LAN IP & DHCP Configuration

In the installer's **LAN (em1) Network Mode Setup**:

| Setting | Value |
|---|---|
| Interface Mode | STATIC |
| IP Address | `192.168.56.254/24` |
| DHCPD Enabled | `true` |
| DHCP Range Start | `192.168.56.100` |
| DHCP Range End | `192.168.56.200` |

> **Why `.254`?** Using the last usable host address as the gateway keeps the bottom of the range clean for clients and is easy to remember.

---

## 4. Web GUI Initial Setup

Access the GUI from an internal machine: `http://192.168.56.254`  
Default credentials: `admin / pfsense`

**General Information (Setup Wizard):**

| Field | Value |
|---|---|
| Hostname | `pfSense` |
| Domain | `cdm.local` |
| Primary DNS | `8.8.8.8` |
| Secondary DNS | `8.8.4.4` |
| Override DNS | ✅ Enabled |

Confirm the LAN interface shows `192.168.56.254 /24` — click **Next** to continue through the wizard with remaining defaults.

---

## 5. Malicious IP Blocking (Aliases + Rules)

### Step 5a — Create an Alias

Navigate to: **Firewall → Aliases → Add**

| Field | Value |
|---|---|
| Name | `Malicious_IPs` |
| Description | `Known malicious addresses` |
| Type | `Host(s)` |

Add known bad IP ranges under **IP or FQDN**:
```
203.0.113.0/24
198.51.100.0/24
192.0.2.0/24
```
> Replace with your real threat intel feeds (e.g., Emerging Threats, CISA advisories).

Click **Save** and **Apply Changes**.

### Step 5b — Create a Block Rule on WAN

Navigate to: **Firewall → Rules → WAN → Add**

| Field | Value |
|---|---|
| Action | `Block` |
| Interface | `WAN` |
| Address Family | `IPv4` |
| Protocol | `Any` |
| Source | `Address or Alias` → `Malicious_IPs` |
| Destination | `Any` |
| Log | ✅ Log packets |
| Description | `Block known Malicious IPs` |

Click **Save** and **Apply Changes**.

---

## 6. GeoIP Blocking (pfBlockerNG)

### Install pfBlockerNG

Navigate to: **System → Package Manager → Available Packages**  
Search for `pfBlockerNG-devel` and install.

### Configure GeoIP Block List

Navigate to: **Firewall → pfBlockerNG → IP → IPv4**

| Field | Value |
|---|---|
| Name | `Block_Countries` |
| Description | `High-risk regions` |

Add GeoIP sources (set each to `GeoIP` format, state `ON`):

| Country | Code |
|---|---|
| North Korea | `KP` |
| Russia | `RU` |
| China | `CN` |
| Nigeria | `NG` |

**Settings:**

| Field | Value |
|---|---|
| Action | `Deny Both` |
| Update Frequency | `Weekly` |

Click **Save** → then go to **pfBlockerNG → Update** and run a force update.

> **Note:** `Deny Both` blocks inbound AND outbound traffic to/from those regions.

---

## 7. LAN Firewall Rule (DNS Pass)

Navigate to: **Firewall → Rules → LAN → Add**

This rule allows internal clients to perform DNS lookups through the firewall:

| Field | Value |
|---|---|
| Action | `Pass` |
| Interface | `LAN` |
| Address Family | `IPv4` |
| Protocol | `TCP/UDP` |
| Source | `LAN subnets` |
| Destination | `Any` |
| Destination Port Range | `DNS (53)` → `DNS (53)` |
| Log | ✅ Log packets |

Click **Save** and **Apply Changes**.

---

## 8. Final Rule Summary

**Firewall → Rules → WAN** should show (in order):

| # | Source | Description |
|---|---|---|
| 1 | RFC 1918 networks | Block private networks (default) |
| 2 | Reserved / IANA | Block bogon networks (default) |
| 3 | pfB_Block_Countries_v4 | GeoIP auto rule (pfBlockerNG) |
| 4 | Malicious_IPs | Block known Malicious IPs |

> Rules are processed **top-down**. Ensure block rules appear before any pass rules.

---

## ✅ Verification Checklist

- [ ] pfSense accessible at `192.168.56.254` from internal VM
- [ ] WAN interface receiving DHCP (`10.0.2.15/24`)
- [ ] Internal clients getting IPs in `192.168.56.100–200`
- [ ] Alias `Malicious_IPs` created with threat IPs
- [ ] WAN block rule active and logging
- [ ] pfBlockerNG GeoIP lists updated and active
- [ ] WAN rules show both pfBlockerNG and Malicious IP block entries
- [ ] LAN DNS pass rule active

---

## 📁 Screenshots Reference

| Image | Description |
|---|---|
| `1_install_pfsense.png` | Netgate installer — select Install pfSense |
| `2__wan_interface.png` | WAN interface selection (em0) |
| `2a__lan_interface.png` | LAN interface selection (em1) |
| `3__IP_address_change_to_match_subnet.png` | LAN static IP set to 192.168.56.254/24 |
| `4__dhcp_range.png` | DHCP range start — 192.168.56.100 |
| `4b__dhcp_setup.png` | Full LAN/DHCP config confirmation |
| `7__general_info.png` | Web GUI general info wizard |
| `7a__keep_default.png` | LAN interface confirmation in GUI |
| `8__create_alias_know_mali_ip_address.png` | Creating Malicious_IPs alias |
| `8a__activate_rule_for_block_ip_address.png` | WAN block rule for Malicious_IPs |
| `9__configure_geoIP.png` | pfBlockerNG GeoIP IPv4 config |
| `9a_rules_set.png` | Final WAN rules showing all block entries |
| `9b__LAN_config_internal_network.png` | LAN DNS pass rule |
| `network_config_.png` | pfSense console showing WAN/LAN IPs |
| `pfsense_usinf_gw_addr.png` | Browser login at 192.168.56.254 |

---

## 🔧 Tools & References

- [pfSense Documentation](https://docs.netgate.com/pfsense/en/latest/)
- [pfBlockerNG Wiki](https://docs.netgate.com/pfsense/en/latest/packages/pfblocker.html)
- [Emerging Threats IP Blocklists](https://rules.emergingthreats.net/)
- [CISA Known Exploited Vulnerabilities](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)

---

