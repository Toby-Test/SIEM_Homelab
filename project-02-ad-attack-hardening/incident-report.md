# Incident Report — Active Directory Domain Compromise

| Field | Value |
|---|---|
| Report date | May 13, 2026 |
| Analyst | SOC Tier 1 |
| Severity | Critical |
| Status | Remediated and validated |
| Domain affected | northport.local |
| Duration of compromise window | May 9–11, 2026 |

---

## 1. Executive Summary

Between May 9 and May 11, 2026, an attacker operating from 192.168.20.50 executed a six-stage attack against the `northport.local` Active Directory domain, progressing from unauthenticated network reconnaissance to full domain administrator control. The attacker compromised two user accounts via password spraying, cracked a Domain Admin service account hash via Kerberoasting, achieved SYSTEM-level remote code execution on a domain workstation, created a persistent backdoor Domain Admin account, and exfiltrated company files from both a file server and the domain's SYSVOL share.

All stages were detected by Wazuh through a combination of Windows event logs, Sysmon events, and OPNsense syslog forwarding. Hardening was applied on May 11 and validated on May 11–12.

---

## 2. Timeline

| Date / Time | Event |
|---|---|
| May 9 @ 13:41 | Attacker performs host discovery scan (`nmap -sn 192.168.10.0/24`) — 4 hosts discovered |
| May 9 @ 13:43 | Full port scan of DC01 (192.168.10.5) — services fingerprinted |
| May 9 @ ~13:45 | enum4linux enumerates domain name `northport.local`, 14 user accounts, password policy |
| May 9 @ 18:49 | Password spray via CrackMapExec — `Welcome123` succeeds for jsmith and mwilliams |
| May 9 @ ~18:50 | Kerberoasting: jsmith credentials used to capture svc_backup TGS ticket |
| May 9–10 | Hashcat offline cracking — svc_backup password recovered |
| May 10 | impacket-psexec to windows-endpoint — NT AUTHORITY\SYSTEM obtained |
| May 10 | impacket-secretsdump against workstation and DC — NTLM hashes extracted |
| May 10 | impacket-psexec to DC01 — `svc_print` backdoor account created and added to Domain Admins |
| May 10 | smbclient to ubuntu file server — company files exfiltrated |
| May 10 | smbclient to DC SYSVOL — Group Policy files exfiltrated |
| May 11 | Hardening applied — passwords reset, service account secured, firewall rules added |
| May 11–12 | All six attacks re-tested — all blocked or rendered ineffective |

---

## 3. Attack Path

```
[Recon] enum4linux → 14 usernames + weak password policy
    ↓
[Initial Access] Password spray → jsmith:Welcome123
    ↓
[Credential Access] Kerberoasting → svc_backup hash → cracked → Domain Admin
    ↓
[Lateral Movement] PsExec → windows-endpoint SYSTEM
    ↓
[Credential Dumping] secretsdump → all NTLM hashes
    ↓
[Persistence] svc_print backdoor account → Domain Admins
    ↓
[Exfiltration] SMB → company files + SYSVOL
```

---

## 4. Affected Assets

| Asset | IP | Impact |
|---|---|---|
| DC01 | 192.168.10.5 | Fully compromised — Domain Admin shell obtained, all hashes dumped, SYSVOL accessed |
| windows-endpoint | 192.168.10.160 | NT AUTHORITY\SYSTEM obtained, local hashes dumped |
| ubuntu-endpoint | 192.168.10.123 | All shared files exfiltrated |
| jsmith | — | Credentials compromised via password spray |
| mwilliams | — | Credentials compromised via password spray |
| svc_backup | — | Hash cracked, full Domain Admin access |
| Domain (northport.local) | — | Full domain compromise via backdoor account |

---

## 5. MITRE ATT&CK Mapping

| Technique ID | Name | Stage |
|---|---|---|
| T1046 | Network Service Discovery | Reconnaissance |
| T1110.003 | Password Spraying | Initial Access |
| T1558.003 | Kerberoasting | Credential Access |
| T1003.001 | LSASS Memory (secretsdump) | Credential Dumping |
| T1021.002 | SMB/Windows Admin Shares (PsExec) | Lateral Movement |
| T1569.002 | Service Execution | Execution |
| T1136.002 | Create Domain Account | Persistence |
| T1048 | Exfiltration Over Alt Protocol (SMB) | Exfiltration |
| T1078.002 | Valid Accounts: Domain Accounts | Defence Evasion |

---

## 6. Wazuh Detection Summary

| Attack Stage | Event IDs Fired | Wazuh Rule(s) |
|---|---|---|
| Reconnaissance | Event 5156, anonymous logon | Custom rules 100015–100016, OPNsense syslog |
| Password spray | Event 4625 (×many), Event 4624 | Rule 60122 (logon failure), Rule 92652 (success) |
| Kerberoasting | Event 4769 (TGS request) | Custom rule 100012 (if RC4) |
| PsExec lateral movement | Event 7045, Event 4672, Event 4624 | Rule 92650 (service from root path) |
| Backdoor creation | Event 4720, Event 4728 | Custom rules 100010, 100011 |
| Exfiltration | Event 5140 (share access) | Rule 60175 (file share access) |

---

## 7. Root Cause Analysis

The attack chain succeeded because of compounding misconfigurations, each of which independently enabled the next stage:

| Root Cause | Impact | Fix Applied |
|---|---|---|
| No account lockout policy | Password spray could run unchecked | Lockout: 5 attempts / 30 minutes |
| Weak default passwords (`Welcome123`) | Two accounts compromised instantly | Passwords reset; policy enforced |
| Service account in Domain Admins | Kerberoast gave immediate DA access | Removed from Domain Admins |
| RC4 encryption + short password on SPN account | Hash was crackable in minutes | AES-256 only + 30-char password |
| No SMB restrictions on workstation | PsExec lateral movement trivial | Firewall rule blocking TCP 445 |
| SMB guest access enabled on file server | Files stolen without credentials | Guest access disabled; ACLs applied |

---

## 8. Hardening Actions Taken

1. Password policy enforced: 14-character minimum, complexity required, 90-day expiry
2. Account lockout policy: 5 attempts → 30-minute lockout
3. Compromised accounts (jsmith, mwilliams) reset to strong passwords
4. svc_backup removed from Domain Admins
5. svc_backup: 30-character random password, AES-256 encryption enforced, logon restricted to DC01 only
6. svc_print backdoor account deleted
7. Windows Firewall inbound rule: block TCP 445 on workstation
8. Samba: guest access disabled, read-only, restricted to Operations department
9. OPNsense Suricata IPS enabled in Netmap mode on ATTACKER interface
10. Anonymous LDAP binding disabled via ADSI Edit

---

## 9. Validation

All hardening measures were validated by re-running the original attacks:

| Attack | Post-Hardening Result |
|---|---|
| Network scan | Host appears offline (Suricata IPS blocking) |
| Password spray | STATUS_ACCOUNT_LOCKED_OUT — Event 4740 fired |
| Kerberoasting | Ticket obtained but AES-256 + 30-char password — uncrackable |
| PsExec lateral movement | Connection timed out — port 445 blocked |
| SMB exfiltration | NT_STATUS_ACCESS_DENIED |
| Backdoor account | Confirmed deleted |

---

## 10. Analyst Notes

This exercise illustrates a core principle of Active Directory security: misconfigurations are cumulative. No single weakness here was catastrophic in isolation — but together they formed an unbroken chain from unauthenticated reconnaissance to full domain compromise in under 24 hours.

The most instructive finding is the Kerberoasting detection gap. Once a service ticket leaves the DC, the cracking is entirely offline. The only prevention is making the hash useless: AES encryption means the attacker gets a harder problem, and a 30-character random password means no wordlist will ever solve it. The detection window is Event 4769 — which is why the advanced audit policy enabling it is not optional.

The backdoor account detection (Events 4720 + 4728 in rapid succession) is a strong example of why custom rules matter. The default Wazuh rules would fire on each event individually. The combination — new account immediately added to Domain Admins — is a nearly unambiguous attacker signature that deserves a level-15 alert and immediate escalation.
