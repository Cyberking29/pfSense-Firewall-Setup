# 🛡️ Windows Hardening Notes — SOC Lab

> **Purpose:** Step-by-step hardening notes for a Windows endpoint based on CIS L1 benchmarks, suitable for SOC environments. Includes policy hardening, service reduction, and log monitoring setup.

---

## 📋 Table of Contents

1. [Password Policy](#1-password-policy)
2. [Account Lockout Policy](#2-account-lockout-policy)
3. [UAC (User Account Control)](#3-uac-user-account-control)
4. [Force Group Policy Update](#4-force-group-policy-update)
5. [Disable Unnecessary Services](#5-disable-unnecessary-services)
6. [Sysmon Installation & Configuration](#6-sysmon-installation--configuration)
7. [Event Log Configuration](#7-event-log-configuration)
8. [Advanced Audit Policy](#8-advanced-audit-policy)

---

## 1. Password Policy

**Path:** `Run → gpedit.msc → Computer Configuration → Windows Settings → Security Settings → Account Policies → Password Policy`

Open the Run dialog and launch the Group Policy Editor:

![Open gpedit.msc](1.png)

Navigate to Password Policy:

![Password Policy path](2.png)

After applying CIS L1 benchmark settings:

![Password Policy after CIS L1](3%20after%20using%20cis%20L1.png)

| Setting | Value |
|---|---|
| Enforce password history | 24 passwords |
| Maximum password age | 365 days |
| Minimum password age | 1 day |
| Minimum password length | 14 characters |
| Password must meet complexity requirements | Enabled |
| Relax minimum password length limits | Enabled |
| Store passwords using reversible encryption | Disabled |

---

## 2. Account Lockout Policy

**Path:** `gpedit.msc → ... → Account Policies → Account Lockout Policy`

![Account Lockout Policy](4%20account%20lockout%20policy.png)

| Setting | Value |
|---|---|
| Account lockout duration | 15 minutes |
| Account lockout threshold | 5 invalid logon attempts |
| Allow Administrator account lockout | Enabled |
| Reset account lockout counter after | 15 minutes |

---

## 3. UAC (User Account Control)

**Path:** `gpedit.msc → Computer Configuration → Windows Settings → Security Settings → Local Policies → Security Options`

Before changes:

![UAC before](5%20before%20UAC.png)

After applying hardened UAC settings:

![UAC after](5%20after%20UAC%20changes.png)

| Setting | Value |
|---|---|
| Admin Approval Mode for the Built-in Administrator | Enabled |
| Allow UIAccess applications to prompt for elevation | Disabled |
| Behavior of elevation prompt for administrators | Prompt for credentials on secure desktop |
| Behavior of elevation prompt for standard users | Automatically deny elevation requests |

---

## 4. Force Group Policy Update

After making policy changes, apply them immediately via PowerShell (run as Administrator):

```powershell
gpupdate /force
```

![Force GPUpdate](6.%20force%20policy%20update.png)

Expected output: `Computer Policy update has completed successfully.`

---

## 5. Disable Unnecessary Services

**Path:** `Run → services.msc`

![Open services.msc](7.%20services%20policy.png)

### Print Spooler

- Open **Print Spooler** properties
- Set **Startup type** → `Disabled`
- Click **Stop** to stop the running service
- Click **Apply → OK**

![Disable Print Spooler](8%20disable%20print%20spooler.png)

> ⚠️ Only disable if no printing is required on this endpoint.

### Xbox Services

Disable all four Xbox-related services (not needed on a SOC workstation):

![Disable Xbox Services](9.%20disable%20all%20xbox%20service.png)

| Service | Action |
|---|---|
| Xbox Accessory Management | Disabled |
| Xbox Live Auth Manager | Disabled |
| Xbox Live Game Save | Disabled |
| Xbox Live Networking Service | Disabled |

---

## 6. Sysmon Installation & Configuration

Sysmon provides detailed process, network, and file activity logging — essential for SOC visibility.

### Download

- Visit: [Sysinternals Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- Current version: **Sysmon v15.21**
- Download the zip (~4.6 MB) and extract to a local folder (e.g., `C:\Users\<user>\Downloads\Sysmon`)

![Download Sysmon](10.%20download%20sysmon.png)

Sysmon folder contents after extraction:

![Sysmon files](10a.sysmon%20file.png)

### Install with Config

```powershell
cd C:\Users\<user>\Downloads\Sysmon
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```

![Installing Sysmon](11.%20installing%20sysmon.png)

Expected output confirms:
- `Sysmon64 installed.`
- `SysmonDrv installed.`
- `Starting SysmonDrv... started.`
- `Starting Sysmon64... started.`

> 💡 Use a community config such as [SwiftOnSecurity/sysmon-config](https://github.com/SwiftOnSecurity/sysmon-config) for the XML.

### Verify in Event Viewer

```
Run → eventvwr.msc → Applications and Services Logs → Microsoft → Windows → Sysmon → Operational
```

![Open Event Viewer](11b.%20check%20event%20log%20by%20sysmon.png)

A process creation event (Event ID 1) will show fields like:
- `UtcTime` — timestamp of the event
- `Image` — full path of the process (e.g., `C:\Windows\System32\PING.EXE`)
- `Description` — human-readable description
- `CommandLine` — full command used (e.g., `PING.EXE 8.8.8.8`)

![Event log captured by Sysmon](12.%20event%20log%20captured.png)

---

## 7. Event Log Configuration

Configure logs to **archive when full** to avoid losing historical data.

### Sysmon Log

**Path:** `Event Viewer → Applications and Services Logs → Microsoft → Windows → Sysmon → Operational → Properties`

![Sysmon log properties](13.%20save%20logs%20without%20losing%20old%20ones.png)

| Setting | Value |
|---|---|
| Maximum log size | 1,048,384 KB (~1 GB) |
| When full | Archive the log when full, do not overwrite events |

### Windows Application Log

**Path:** `Event Viewer → Windows Logs → Application → Properties`

![Windows Application log properties](13a.%20save%20logs%20for%20windows.png)

| Setting | Value |
|---|---|
| Maximum log size | 102,480 KB (~100 MB) |
| When full | Archive the log when full, do not overwrite events |

> Apply the same archiving setting to **Security**, **System**, and **Setup** logs.

---

## 8. Advanced Audit Policy

**Path:** `Local Security Policy (secpol.msc) → Advanced Audit Policy Configuration → System Audit Policies → Account Logon`

Before configuration:

![Before audit policy edit](15.before%20edit%20sec%20pol%20.png)

Enable **both Success and Failure** auditing for each subcategory:

![Audit Credential Validation settings](15a.png)

| Subcategory | Audit Events |
|---|---|
| Audit Credential Validation | Success + Failure |
| Audit Kerberos Authentication Service | Success + Failure |
| Audit Kerberos Service Ticket Operations | Success + Failure |
| Audit Other Account Logon Events | Success + Failure |

> This ensures failed login attempts and credential abuse attempts are captured in the Security log.

---

## 🔧 Tools Referenced

| Tool | Purpose |
|---|---|
| `gpedit.msc` | Local Group Policy Editor |
| `services.msc` | Windows Services Manager |
| `eventvwr.msc` | Event Viewer |
| `secpol.msc` | Local Security Policy |
| `gpupdate /force` | Force apply group policies |
| Sysmon v15.21 | Advanced system activity monitoring |

---

## 📚 References

- [CIS Microsoft Windows Benchmarks](https://www.cisecurity.org/cis-benchmarks/)
- [Microsoft Sysinternals – Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)

---

*Maintained by [@Cyberking29](https://github.com/Cyberking29?tab=repositories) | SOC Lab Project*
