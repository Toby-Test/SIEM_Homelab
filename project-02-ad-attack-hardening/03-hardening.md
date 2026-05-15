# Hardening

Hardening is applied on **May 11, 2026** after all six attack scenarios are documented. Every measure directly addresses a specific vulnerability that was exploited.

---

## H1 — Password Policy (Stops Password Spraying)

### Problem

No account lockout policy existed. The default password complexity was insufficient. `Welcome123` was a valid credential for multiple accounts.

### Fix

Group Policy updated on the Default Domain Policy:

**Password Policy:**

| Setting | Before | After |
|---|---|---|
| Minimum password length | 7 characters | **14 characters** |
| Password complexity | Not enforced | **Enabled** |
| Maximum password age | Not set | **90 days** |
| Password history | Not enforced | **24 passwords** |

**Account Lockout Policy:**

| Setting | Before | After |
|---|---|---|
| Lockout threshold | None | **5 invalid attempts** |
| Lockout duration | N/A | **30 minutes** |
| Reset counter after | N/A | **15 minutes** |

![Password policy GPO](../screenshots/hardening/pwpolicy.png)
![Account lockout policy GPO](../screenshots/hardening/accountlockoutpolicy.png)

### Compromised Account Password Resets

The two accounts confirmed compromised during the spray are reset to strong passwords via PowerShell:

```powershell
Set-ADAccountPassword -Identity jsmith -Reset `
  -NewPassword (ConvertTo-SecureString "N3wStr0ngP@ss2026!" -AsPlainText -Force)

Set-ADAccountPassword -Identity mwilliams -Reset `
  -NewPassword (ConvertTo-SecureString "D1fferentP@ss2026!" -AsPlainText -Force)
```

![Password resets](../screenshots/hardening/newpass.png)

---

## H2 — Service Account Hardening (Stops Kerberoasting)

### Problem

`svc_backup` had three major weaknesses:
1. Member of Domain Admins — unnecessary privilege
2. Crackable password — short enough to fall to a wordlist attack
3. RC4 encryption allowed — enabled offline cracking without AES resistance

### Fix

A complete hardening chain is applied in PowerShell:

**Step 1 — Remove from Domain Admins:**

```powershell
Remove-ADGroupMember -Identity "Domain Admins" -Members svc_backup -Confirm:$false
```

**Step 2 — Generate and set a 30-character random password:**

```powershell
$newPass = -join ((65..90) + (97..122) + (48..57) + (33..38) | `
  Get-Random -Count 30 | ForEach-Object {[char]$_})
Set-ADAccountPassword -Identity svc_backup -Reset `
  -NewPassword (ConvertTo-SecureString $newPass -AsPlainText -Force)
Write-Host "New Password: $newPass"
```

Generated password: `Nls#UEZ2J8nq%670&R$HPtTdubxAgF`

**Step 3 — Force AES-256 encryption, disable RC4:**

```powershell
Set-ADUser -Identity svc_backup -KerberosEncryptionType AES256
```

**Step 4 — Restrict logon to DC01 only:**

```powershell
Set-ADUser -Identity svc_backup -LogonWorkstations "DC01"
```

![Full service account hardening chain](../screenshots/hardening/svcfix.png)

---

## H3 — Delete the Backdoor Account

```powershell
Remove-ADUser -Identity svc_print -Confirm:$false
```

Confirmed deleted. No persistence remains.

---

## H4 — Windows Firewall — Block Inbound SMB (Stops PsExec)

### Problem

Windows Firewall on the workstation had no inbound rule blocking SMB (port 445). PsExec connects over SMB to upload its service binary and execute commands.

### Fix

A new inbound firewall rule is created:

- Rule name: **Block SMB from untrusted**
- Protocol: TCP
- Local port: **445**
- Action: Block
- Profiles: Domain, Private, Public

![Windows Firewall rule](../screenshots/hardening/winfirewall.png)

---

## H5 — Samba Hardening (Stops SMB Exfiltration)

### Problem

The Ubuntu file server's Samba share allowed guest (unauthenticated) access and write permissions to all network users.

### Fix

`/etc/samba/smb.conf` updated:

```ini
[shared]
   path = /srv/shared
   browsable = yes
   read only = yes
   guest ok = no
   valid users = @"NORTHPORT\Operations"
   comment = Company Shared Files
```

![Samba hardened config](../screenshots/hardening/smblocked.png)

Changes made:
- `guest ok = no` — requires authentication
- `read only = yes` — prevents modification
- `valid users = @"NORTHPORT\Operations"` — restricts access to Operations department only

fail2ban is also installed to rate-limit any brute force attempts against SMB.

---

## H6 — OPNsense Suricata IPS (Stops Network Scanning)

### Problem

Port scans from the attacker segment produced no network-level blocking — only endpoint log entries. The attacker could fully enumerate all services without disruption.

### Fix

Suricata is enabled in **IPS mode** (Netmap) on the ATTACKER interface in OPNsense. This actively blocks matching traffic, not just detects it.

- Capture mode: **Netmap (IPS)** — drops matching packets inline
- Interface: **ATTACKER** — inspects all traffic entering from 192.168.20.0/24
- Rulesets enabled: `emerging-scan.rules`, `emerging-policy.rules`

![Suricata IPS config](../screenshots/hardening/ipsopnsense.png)

---

## H7 — Disable Anonymous LDAP / SMB Enumeration

To stop enum4linux from extracting user lists without credentials:

ADSI Edit (`adsiedit.msc`) is used to set the `dsHeuristics` attribute on the Directory Service configuration object to disable anonymous LDAP binding. Combined with the password policy lockout, unauthenticated enumeration is no longer possible.
