# Hardening Validation
Running the attack again to show the hardening steps are effective
## V1 — Re-test: Network Scanning

**Command:**

```bash
nmap -sS -sV -O -p- 192.168.10.160 -oN retry.txt
```

**Expected:** Suricata IPS detects and drops the scan traffic.

**Result:**

```
Note: Host seems down. If it is really up, but blocking our probes, try -Pn
Nmap done: 1 IP address (0 hosts up) scanned in 4.00 seconds
```

![Nmap blocked by Suricata](../screenshots/validation/nmapfail.png)

Suricata's scan detection rules trigger on the SYN probe pattern and drop the traffic. The host appears offline to the attacker. OPNsense syslog forwards the Suricata alert to Wazuh.

---

## V2 — Re-test: Password Spraying

**Command:**

```bash
crackmapexec smb 192.168.10.5 -u users.txt -p 'Welcome123' --continue-on-success
```

**Expected:** Account lockout policy activates after 5 failures; no successful logins.

**Result:** Every account returns `STATUS_ACCOUNT_LOCKED_OUT` after the 5-attempt threshold is hit.

![All accounts locked](../screenshots/validation/crackmapfail.png)

**Wazuh Event 4740 — Account locked out:**

The lockout event fires for `svc_backup` on the DC, confirming the policy is working.

![Event 4740](../screenshots/validation/4740lockout.png)

**Event 4625 — Logon failure (account locked):**

Post-lockout, even correct credentials are rejected.

![Logon failure after lockout](../screenshots/hardening/accountlogonfail.png)

---

## V3 — Re-test: Kerberoasting

**Command (using jsmith's new password):**

```bash
impacket-GetUserSPNs northport.local/jsmith:N3wStr0ngP@ss2026! \
  -dc-ip 192.168.10.5 \
  -request \
  -outputfile kerbertry.txt
```

**Expected:** Ticket is still obtainable (SPN still exists) but the hash is now AES-256 encrypted with a 30-character random password — highly unlikely to crack.

**Result:** Ticket is returned. The `PasswordLastSet` field shows the recent reset (2026-05-11).

![Kerberoast re-test](../screenshots/validation/kerberfail.png)

Running hashcat against the new hash with rockyou.txt fails — the 30-character random password (`Nls#UEZ2J8nq%670&R$HPtTdubxAgF`) is not in any wordlist and would take decades to brute force at GPU speed.

> **Key lesson:** Kerberoasting cannot be fully prevented while an SPN exists (Kerberos is working as designed). The defence is making the hash worthless — AES-only encryption with a high-complexity long password removes the practical cracking path.

---

## V4 — Re-test: PsExec Lateral Movement

**Command:**

```bash
impacket-psexec northport.local/svc_backup:NEWSTRONGPASSWORD@192.168.10.20
```

**Expected:** Windows Firewall blocks the SMB connection attempt.

**Result:**

```
[-] [Errno Connection error (192.168.10.20:445)] timed out
```

![PsExec connection refused](../screenshots/validation/connectionrefuse.png)

The Windows Firewall rule blocking inbound TCP 445 causes the connection to time out. PsExec never reaches the point of uploading its service binary, even if the correct password would be used.

---

## V5 — Re-test: SMB Data Exfiltration

**Command:**

```bash
smbclient //192.168.10.123/shared -N
```

**Expected:** Authentication required; guest access denied.

**Result:**

```
tree connect failed: NT_STATUS_ACCESS_DENIED
```

![SMB access denied](../screenshots/validation/smbclientdeny.png)

`guest ok = no` in the Samba configuration rejects the unauthenticated connection immediately. The company files are no longer accessible without valid `NORTHPORT\Operations` credentials.

---

## Validation Summary Table

| Attack | Hardening Applied | Re-test Command | Result |
|---|---|---|---|
| Network scanning | Suricata IPS on ATTACKER interface | `nmap -sS -p- 192.168.10.160` | Host appears down — 0 ports found |
| Password spraying | Account lockout (5 attempts, 30 min) + passwords reset | crackmapexec spray | STATUS_ACCOUNT_LOCKED_OUT — Event 4740 fired |
| Kerberoasting | AES-256 only + 30-char random password | impacket-GetUserSPNs | Ticket obtained but uncrackable |
| PsExec lateral movement | Windows Firewall blocking TCP 445 | impacket-psexec | Connection timeout |
| Data exfiltration | Samba guest access disabled + ACLs | smbclient -N | NT_STATUS_ACCESS_DENIED |
| Backdoor account | Deleted via Remove-ADUser | N/A — confirmed deleted | Account no longer exists in AD |
