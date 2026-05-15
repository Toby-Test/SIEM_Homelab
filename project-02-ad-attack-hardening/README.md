# Active Directory Attack & Hardening Lab

A hands-on Active Directory security lab built on Proxmox VE, simulating six MITRE ATT&CK-mapped attack scenarios against a Windows Server 2022 domain, with full evidence-based hardening validation.

---

## Overview

This project extends the existing SIEM detection lab by adding a Windows Server 2022 Domain Controller running Active Directory for the fictional company **Northport Freight**. Six distinct attack techniques are executed from an isolated Kali Linux VM — from initial reconnaissance through to domain compromise and data exfiltration. Every attack is detected by Wazuh, then every hardening measure is validated by re-running the same attacks and confirming failure.

---

## Monitored Assets

Three Wazuh agents active prior to attack simulation:

| Agent ID | Name              | IP              | OS                               |
|----------|-------------------|-----------------|----------------------------------|
| 001      | ubuntu-endpoint   | 192.168.10.123  | Ubuntu 24.04.4 LTS               |
| 002      | windows-endpoint  | 192.168.10.160  | Windows 10 Pro 10.0.19045        |
| 003      | DC01              | 192.168.10.5    | Windows Server 2022 Standard     |

![Wazuh agents](screenshots/setup/nodes.png)

---

## Tools Used

| Category        | Tool                                  |
|-----------------|---------------------------------------|
| Hypervisor      | Proxmox VE                            |
| Firewall / IPS  | OPNsense + Suricata (Netmap IPS mode) |
| SIEM            | Wazuh 4.x                             |
| Domain          | Active Directory (Windows Server 2022)|
| Attack tools    | Nmap, enum4linux, CrackMapExec        |
| Attack tools    | Impacket (GetUserSPNs, PsExec, secretsdump) |
| Attack tools    | Hashcat, smbclient                    |
| Detection       | Custom Wazuh rules, Sysmon, GPO audit |

---

## Project Sections

| Document | Contents |
|---|---|
| [Domain Setup](docs/01-domain-setup.md) | DC deployment, AD structure, user accounts, monitoring config |
| [Attack Scenarios](docs/02-attack-scenarios.md) | Six MITRE ATT&CK-mapped attacks with evidence |
| [Hardening](docs/03-hardening.md) | All hardening measures applied |
| [Validation](docs/04-validation.md) | Re-test results proving each fix works |
| [Custom Wazuh Rules](docs/05-custom-rules.md) | Detection engineering for AD-specific attacks |
| [Incident Report](incident-report.md) | Full incident write-up from analyst perspective |

---

## Attack Scenarios — Summary

Six attacks executed in sequence, simulating a realistic kill chain from unauthenticated recon to full domain compromise.

| # | Attack | MITRE ID | Detection | Outcome |
|---|--------|----------|-----------|---------|
| 1 | Network Reconnaissance | T1046 | Event 5156, OPNsense logs, Wazuh anonymous logon rules | 14 users enumerated, all services fingerprinted |
| 2 | Password Spraying | T1110.003 | Event 4625 (failures), Event 4624 (success) | jsmith + mwilliams compromised via `Welcome123` |
| 3 | Kerberoasting | T1558.003 | Event 4769 (TGS request) | svc_backup hash captured and cracked |
| 4 | PsExec Lateral Movement | T1021.002 / T1569.002 | Event 7045 (service install), Event 4672, Rule 92650 | NT AUTHORITY\SYSTEM on windows-endpoint |
| 5 | Backdoor Domain Account | T1136.002 | Event 4720, Event 4728 | svc_print created and added to Domain Admins |
| 6 | Data Exfiltration (SMB) | T1048 | Event 5140 (share access) | Company files and SYSVOL contents downloaded |

---

## Hardening — Summary

| Hardening Measure | Attack Mitigated |
|---|---|
| Password policy: 14 chars + complexity + lockout (5/30min) | Password spraying |
| Compromised account passwords reset | Password spraying |
| svc_backup: removed from Domain Admins | Privilege escalation |
| svc_backup: 30-char random password, AES-256 only, DC01-restricted logon | Kerberoasting |
| Backdoor account svc_print deleted | Persistence |
| Windows Firewall: block inbound TCP 445 on workstation | PsExec lateral movement |
| Samba: guest access disabled, read-only, valid users only | SMB exfiltration |
| OPNsense Suricata IPS on ATTACKER interface | Network scanning |

---

## Validation Results

| Attack | Re-test Result | Evidence |
|---|---|---|
| Network scanning | Host appears down — Suricata blocked | nmap returns 0 hosts |
| Password spraying | All accounts locked (Event 4740) | STATUS_ACCOUNT_LOCKED_OUT for all |
| Kerberoasting | Ticket obtained — AES-256, 30-char password, uncrackable | hashcat fails |
| PsExec lateral movement | Connection timeout — SMB blocked | Errno timeout on port 445 |
| Data exfiltration | NT_STATUS_ACCESS_DENIED | Guest access disabled |

---

## Key Screenshots

### Reconnaissance — User Enumeration
14 domain users extracted without credentials via enum4linux.

![Users enumerated](screenshots/recon/Users.png)

### Password Spray — Credential Compromise
CrackMapExec identifies valid credentials from the attacker segment.

![Password spray success](screenshots/attacks/pwcrack.png)

### Kerberoasting
svc_backup service ticket captured via compromised jsmith credentials.

![Kerberoast SPN](screenshots/attacks/kerberoast.png)
![Hash cracked](screenshots/attacks/hashcrackfin.png)

### PsExec — Domain Admin → SYSTEM
Full lateral movement to windows-endpoint with Domain Admin credentials.

![PsExec SYSTEM shell](screenshots/attacks/impacketwin.png)
![PsExec service install alert](screenshots/attacks/winprogram2.png)

### Backdoor Account — Wazuh Detection
Events 4720 and 4728 fire immediately when svc_print is created and added to Domain Admins.

![Account created](screenshots/attacks/svcprintwaz.png)
![Added to Domain Admins](screenshots/attacks/svcprintdomainwaz.png)

### Hardening — Password Policy + Service Account
Group Policy and PowerShell hardening applied and confirmed.

![Password policy](screenshots/hardening/pwpolicy.png)
![Service account hardened](screenshots/hardening/svcfix.png)

### Validation — Spray Blocked
All accounts locked out after 5 attempts — Event 4740 fires.

![Accounts locked](screenshots/validation/crackmapfail.png)
![Event 4740](screenshots/validation/4740lockout.png)

### Validation — PsExec Blocked + SMB Denied
Both lateral movement paths confirmed closed.

![PsExec refused](screenshots/validation/connectionrefuse.png)

---

## Skills Demonstrated

- Active Directory deployment, OU structure, and user account management
- Group Policy configuration for security hardening
- Multi-stage attack simulation across a realistic corporate AD environment
- SIEM alert triage and MITRE ATT&CK mapping across six attack techniques
- Service account security (SPN management, AES enforcement, logon restrictions)
- Detection engineering — custom Wazuh rule writing and validation
- Evidence-based hardening: every fix proven by re-testing the attack

---

## What's Next

- [ ] Active Directory Certificate Services (ADCS) attack scenarios (ESC1/ESC8)
- [ ] Pass-the-Hash and Pass-the-Ticket attacks
- [ ] Kerberos delegation abuse (unconstrained/constrained delegation)
- [ ] BloodHound AD attack path visualisation
- [ ] Insider threat simulation (Project 3)
- [ ] Microsoft Sentinel integration as a second SIEM

---

*All activity in this lab is conducted in a fully isolated self-hosted virtual environment. The domain `northport.local` and all user accounts are fictional. No external systems, public networks, or real organisations are involved.*
