# 🔥 pfSense Firewall Setup — SOC Lab

> **Purpose:** Step-by-step setup and configuration of pfSense as a network firewall/router in a SOC lab environment. Covers installation, interface setup, firewall rules, GeoIP blocking, and LAN configuration.

---

## 📌 What is pfSense?

**pfSense** is a free, open-source firewall and router platform built on FreeBSD. It is deployed at the network perimeter to control and monitor all inbound and outbound traffic.

### What it does:
- Acts as a **firewall** — inspects and filters network packets based on rules
- Functions as a **router** — directs traffic between WAN (internet) and LAN (internal network)
- Provides **DHCP** — automatically assigns IP addresses to devices on the network
- Supports **VPN, DNS filtering, traffic shaping**, and more via packages like pfBlockerNG

### Why it matters for SOC:
- Gives visibility into **who is talking to what** on the network
- Enables **blocking of malicious IPs and countries** at the perimeter
- Logs traffic for **SIEM ingestion and threat hunting**
- Free alternative to expensive commercial firewalls — ideal for home labs and small orgs

---

## 📋 Table of Contents

1. [Installation](#1-installation)
2. [WAN Interface Assignment](#2-wan-interface-assignment)
3. [LAN Interface Assignment](#3-lan-interface-assignment)
4. [LAN IP & DHCP Configuration](#4-lan-ip--dhcp-configuration)
5. [Web GUI Setup](#5-web-gui-setup)
6. [Network Verification](#6-network-verification)
7. [Firewall Aliases — Block Malicious IPs](#7-firewall-aliases--block-malicious-ips)
8. [Firewall Rule — Block Malicious IPs](#8-firewall-rule--block-malicious-ips)
9. [GeoIP Blocking with pfBlockerNG](#9-geoip-blocking-with-pfblockerng)
10. [WAN Firewall Rules Summary](#10-wan-firewall-rules-summary)
11. [LAN Rule — Allow Internal DNS](#11-lan-rule--allow-internal-dns)

---

## 1. Installation

Boot from the pfSense ISO and select **Install pfSense** from the Netgate Installer welcome screen.

![Install pfSense](1_install_pfsense.png)

> Version used: **Netgate Installer v1.2-RELEASE / pfSense 2.8.1-RELEASE**

---

## 2. WAN Interface Assignment

Select the WAN interface. Here `em0` is chosen as the WAN-facing adapter (connected to the internet/NAT).

![WAN Interface](2__wan_interface.png)

| Interface | Adapter | Role |
|---|---|---|
| em0 | 08:00:27:1c:da:da | WAN (Internet) |

---

## 3. LAN Interface Assignment

Select the LAN interface. Here `em1` is assigned as the internal network adapter.

![LAN Interface](2a__lan_interface.png)

| Interface | Adapter | Role |
|---|---|---|
| em1 | 08:00:27:8d:01:00 | LAN (Internal) |

---

## 4. LAN IP & DHCP Configuration

Set a static IP for the LAN interface to match your lab subnet, then configure the DHCP range for clients.

![LAN IP Address](3__IP_address_change_to_match_subnet.png)

Set DHCP pool start address:

![DHCP Range Start](4__dhcp_range.png)

Final LAN Network Mode Setup showing all confirmed values:

![DHCP Setup Complete](4b__dhcp_setup.png)

| Setting | Value |
|---|---|
| LAN IP Address | 192.168.56.254/24 |
| Interface Mode | STATIC |
| DHCP Enabled | true |
| DHCP Range Start | 192.168.56.100 |
| DHCP Range End | 192.168.56.200 |

---

## 5. Web GUI Setup

Access the pfSense web interface by navigating to `192.168.56.254` from a machine on the LAN. Sign in with default credentials (`admin` / `pfsense`).

![pfSense Login](pfsense_usinf_gw_addr.png)

### General Information

Configure the hostname, domain, and DNS servers during the setup wizard:

![General Info](7__general_info.png)

| Setting | Value |
|---|---|
| Hostname | pfSense |
| Domain | cdm.local |
| Primary DNS | 8.8.8.8 |
| Secondary DNS | 8.8.4.4 |
| Override DNS | Enabled |

Keep the LAN interface defaults confirmed via the wizard:

![Keep Defaults](7a__keep_default.png)

| Setting | Value |
|---|---|
| LAN IP Address | 192.168.56.254 |
| Subnet Mask | /24 |

---

## 6. Network Verification

After installation, the pfSense console confirms the interface assignments and IP addresses:

![Network Config](network_config_.png)

| Interface | Adapter | IP Address |
|---|---|---|
| WAN (wan) | em0 | 10.0.2.15/24 (DHCP) |
| LAN (lan) | em1 | 192.168.56.254/24 (Static) |

---

## 7. Firewall Aliases — Block Malicious IPs

**Path:** `Firewall → Aliases → Add`

Create an alias named `Malicious_IPs` to group known bad IP ranges. This makes firewall rules easier to manage — update the alias instead of editing each rule.

![Create Alias](8__create_alias_know_mali_ip_address.png)

| Field | Value |
|---|---|
| Name | Malicious_IPs |
| Description | Known malicious addresses |
| Type | Host(s) |
| IPs Added | 203.0.113.0/24, 198.51.100.0/24, 192.0.2.0/24 |

> 💡 These are RFC 5737 documentation ranges used as examples. Replace with real threat intel feeds (e.g. from AbuseIPDB, Emerging Threats).

---

## 8. Firewall Rule — Block Malicious IPs

**Path:** `Firewall → Rules → WAN → Add`

Create a rule to block all traffic from the `Malicious_IPs` alias on the WAN interface.

![Block Rule](8a__activate_rule_for_block_ip_address.png)

| Field | Value |
|---|---|
| Action | Block |
| Interface | WAN |
| Address Family | IPv4 |
| Protocol | Any |
| Source | Malicious_IPs (alias) |
| Destination | Any |
| Log | Enabled |
| Description | Block known Malicious IPs |

---

## 9. GeoIP Blocking with pfBlockerNG

**Path:** `Firewall → pfBlockerNG → IP → IPv4 → GeoIP`

Install **pfBlockerNG** from the Package Manager, then configure GeoIP blocking to deny traffic from high-risk regions.

![GeoIP Config](9__configure_geoIP.png)

| Field | Value |
|---|---|
| Name | Block_Countries |
| Description | High-risk regions |
| Countries Blocked | KP (North Korea), RU (Russia), CN (China), NG (Nigeria) |
| Action | Deny Both (inbound + outbound) |
| Update Frequency | Weekly |

> ⚠️ GeoIP blocking reduces noise but is not a substitute for full threat detection. Some legitimate traffic may originate from these regions.

---

## 10. WAN Firewall Rules Summary

**Path:** `Firewall → Rules → WAN`

The WAN rules tab shows all active blocking rules in order:

![WAN Rules](9a_rules_set.png)

| Rule | Source | Description |
|---|---|---|
| Block | RFC 1918 networks | Block private networks |
| Block | Reserved (IANA) | Block bogon networks |
| Block | pfB_Block_Countries_v4 | GeoIP country block (auto rule) |
| Block | Malicious_IPs | Block known malicious IPs |

> Rules are evaluated top-down. pfBlockerNG auto-rules are inserted above manual rules.

---

## 11. LAN Rule — Allow Internal DNS

**Path:** `Firewall → Rules → LAN → Add`

Allow LAN clients to resolve DNS through the firewall. This ensures internal machines can query DNS on port 53.

![LAN DNS Rule](9b__LAN_config_internal_network.png)

| Field | Value |
|---|---|
| Action | Pass |
| Interface | LAN |
| Address Family | IPv4 |
| Protocol | TCP/UDP |
| Source | LAN subnets |
| Destination | Any |
| Destination Port | DNS (53) |
| Log | Enabled |

---

## 🔧 Tools & Packages Used

| Tool | Purpose |
|---|---|
| pfSense 2.8.1 | Firewall / Router OS |
| pfBlockerNG | GeoIP and IP reputation blocking |
| Firewall Aliases | Group IPs for reusable rules |
| DHCP Server | Auto-assign IPs to LAN clients |
| DNS Resolver | Internal DNS resolution |

---

## 📚 References

- [pfSense Official Docs](https://docs.netgate.com/pfsense/en/latest/)
- [pfBlockerNG Guide](https://docs.netgate.com/pfsense/en/latest/packages/pfblocker.html)
- [AbuseIPDB — Threat Intel](https://www.abuseipdb.com/)
- [Emerging Threats Ruleset](https://rules.emergingthreats.net/)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)

---

*Maintained by [@Cyberking29](https://github.com/Cyberking29?tab=repositories) | SOC Lab Project*
