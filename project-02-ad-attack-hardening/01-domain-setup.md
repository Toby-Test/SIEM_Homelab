# Domain Setup

## Windows Server 2022 — Domain Controller

A Windows Server 2022 Standard Evaluation VM is added to the existing lab LAN and promoted to a Domain Controller running Active Directory for the `northport.local` domain.

**VM specifications:**

| Setting  | Value                                             |
| -------- | ------------------------------------------------- |
| VM ID    | 105                                               |
| Name     | DC01                                              |
| OS       | Windows Server 2022 Standard (Desktop Experience) |
| Machine  | q35 / OVMF (UEFI)                                 |
| vCPUs    | 2                                                 |
| RAM      | 4 GB                                              |
| Disk     | 60 GB                                             |
| Network  | vmbr1 (Lab LAN)                                   |
| IP       | 192.168.10.5 (static)                             |


Active Directory domain name: `northport.local`  
NetBIOS domain: `NORTHPORT`  
DSRM password set during promotion.

DNS on the DC is set to `127.0.0.1` (loopback) so the DC resolves its own domain name through the AD-integrated DNS service. The Windows 10 workstation DNS is updated to `192.168.10.5` to allow domain join.

---

## Active Directory Structure

The domain simulates a small freight company. Four Organisational Units organise staff by department.

### Organisational Units

| OU | Department |
|---|---|
| IT | IT department |
| Operations | Logistics and operations |
| Finance | Finance and accounting |
| Executives | Senior management |

### User Accounts

Twelve domain user accounts are created across the four OUs. Several accounts are given intentionally weak default passwords — a common real-world misconfiguration.

| OU | Username | Weak credential |
|---|---|---|
| IT | jsmith | `Welcome123` |
| IT | dlee | `Summer2024!` |
| Operations | mwilliams | `Welcome123` |
| Operations | cjones | `Tr@nsport1` |
| Operations | staylor | `Freight99!` |
| Finance | abrown | `Budget2024!` |
| Finance | rchen | `F1nance!23` |
| Finance | lnguyen | `Acc0unts!` |
| Executives | jharris | `Ex3cutiv3!` |
| Executives | kmitchell | `L3adersh1p!` |

> `Welcome123` for jsmith and mwilliams reflects a common IT practice of setting simple default passwords during account creation. This is one of the first things an attacker tests.

### Service Account (Intentionally Misconfigured)

A service account `svc_backup` is created in the IT OU and configured with two deliberate vulnerabilities:

1. **Domain Admin membership** — excessive privilege granted to simulate a common "easy way to make it work" shortcut that might be used early on and forgotten about
2. **SPN registered** — makes the account eligible for Kerberoasting:

```powershell
setspn -A MSSQLSvc/dc01.northport.local:1433 svc_backup
```

This simulates a SQL Server service account, one of the most common SPN targets in real environments.

---

## Windows 10 Workstation — Domain Join

The existing Windows 10 VM is joined to `northport.local`:
- DNS updated to point to the DC (192.168.10.5)
- Domain joined via System Properties → Computer Name → Domain: `northport.local`
- Post-join: login with `NORTHPORT\jsmith` to confirm domain authentication works

---

## Ubuntu File Server — SMB Share (Intentionally Insecure)

The Ubuntu endpoint is configured as a file server using Samba, simulating a common small-business setup where Linux hosts share files to Windows clients.

Initial (deliberately insecure) config:

```ini
[shared]
   path = /srv/shared
   browsable = yes
   read only = no
   guest ok = yes
   comment = Company Shared Files
```

`guest ok = yes` allows unauthenticated access — a surprisingly common misconfiguration that enables data theft without any credentials.

Simulated company files placed in `/srv/shared`:
- `salary_report_2025.csv`
- `client_payments.csv`
- `board_minutes.pdf`
- `route_analysis.xlsx`

---

## Monitoring Configuration

### Wazuh Agent on DC01

The Wazuh agent is deployed to DC01 and Sysmon is installed with the SwiftOnSecurity configuration. The `ossec.conf` is updated to ingest Sysmon's event channel:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

All three agents (ubuntu-endpoint, windows-endpoint, DC01) show **active** status before attack simulation begins.

### Advanced Audit Policy (Group Policy)

Default Windows Server logging is insufficient for detecting AD attacks. The following audit categories are enabled via Group Policy on the Default Domain Controllers Policy:

| Category | Policy | Key Event IDs |
|---|---|---|
| Account Logon | Audit Kerberos Authentication Service | 4768 |
| Account Logon | Audit Kerberos Service Ticket Operations | 4769 — critical for Kerberoasting |
| Account Management | Audit User Account Management | 4720, 4722, 4726 |
| Account Management | Audit Security Group Management | 4728, 4732, 4756 |
| Logon/Logoff | Audit Logon | 4624, 4625 |
| Logon/Logoff | Audit Special Logon | 4672 |
| Object Access | Audit File Share | 5140, 5145 |
| Privilege Use | Audit Sensitive Privilege Use | 4673, 4674 |
| Detailed Tracking | Audit Process Creation | 4688 |

Without these policies enabled, several of the attack scenarios would generate no log entries and would be completely invisible in the SIEM.
