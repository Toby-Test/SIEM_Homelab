# Custom Wazuh Detection Rules

Default Wazuh rules catch many of the AD attacks out of the box. These custom rules add targeted, high-confidence detection specifically tuned to the attack patterns observed in this lab.

---

## Why Custom Rules Matter

Default rules operate on broad patterns — "any service account logon" or "any failed authentication." Custom rules narrow the detection to specific behaviours that are highly anomalous in a given environment. A rule that fires on `svc_` account names authenticating to workstations, or on the exact PSEXESVC service name, produces far fewer false positives than a generic equivalent.

For a portfolio, custom rules demonstrate understanding of detection engineering: reading source events, understanding the attacker's footprint, and translating that into logic.

---

## Rule File Location

Custom rules are written to the Wazuh Manager:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

---

## Rules

```xml
<group name="custom_ad_rules,">

  <!-- Detect account added to Domain Admins -->
  <rule id="100010" level="15">
    <if_sid>60144</if_sid>
    <field name="win.eventdata.targetUserName">Domain Admins</field>
    <description>CRITICAL: Account added to Domain Admins group</description>
    <mitre>
      <id>T1098</id>
    </mitre>
  </rule>

  <!-- Detect new domain account created on DC -->
  <rule id="100011" level="12">
    <if_sid>60137</if_sid>
    <description>HIGH: New user account created on domain controller</description>
    <mitre>
      <id>T1136.002</id>
    </mitre>
  </rule>

  <!-- Detect Kerberoasting (RC4 ticket request) -->
  <rule id="100012" level="13">
    <if_sid>60006</if_sid>
    <field name="win.eventdata.ticketEncryptionType">0x17</field>
    <description>HIGH: Kerberos ticket requested with RC4 encryption (possible Kerberoasting)</description>
    <mitre>
      <id>T1558.003</id>
    </mitre>
  </rule>

  <!-- Detect PsExec service installation -->
  <rule id="100013" level="14">
    <if_sid>60045</if_sid>
    <field name="win.eventdata.serviceName">PSEXESVC</field>
    <description>CRITICAL: PsExec service installed (possible lateral movement)</description>
    <mitre>
      <id>T1021.002</id>
    </mitre>
  </rule>

  <!-- Detect service account logon to workstation -->
  <rule id="100014" level="12">
    <if_sid>60106</if_sid>
    <field name="win.eventdata.targetUserName" type="pcre2">^svc_</field>
    <description>HIGH: Service account authenticated to endpoint (abnormal behaviour)</description>
    <mitre>
      <id>T1078.002</id>
    </mitre>
  </rule>

</group>
```

![Custom rules in Wazuh](../screenshots/hardening/customwazuhrule.png)

---

## Rule Breakdown

### Rule 100010 — Domain Admins membership change (level 15)

**Source event:** Windows Event ID 4728 (member added to security-enabled global group)  
**Parent SID:** 60144 (Wazuh's rule for Event 4728)  
**Filter:** `targetUserName` contains "Domain Admins"  
**Why level 15:** Adding any account to Domain Admins is one of the highest-risk AD changes possible. A false positive here is less dangerous than missing a real event. Wazuh level 15 is the maximum — this should wake someone up.  
**MITRE:** T1098 — Account Manipulation

### Rule 100011 — New domain account on DC (level 12)

**Source event:** Windows Event ID 4720 (user account created)  
**Why useful:** Accounts should be created through a controlled HR/IT process. An account creation event from an unusual source (a service account, or after hours) warrants review.  
**MITRE:** T1136.002 — Create Account: Domain Account

### Rule 100012 — Kerberoasting indicator (level 13)

**Source event:** Windows Event ID 4769 (Kerberos service ticket request)  
**Filter:** `ticketEncryptionType` = `0x17` (RC4-HMAC)  
**Why RC4 is the indicator:** Modern environments use AES-256 (`0x12`) for Kerberos. An RC4 ticket request is an explicit downgrade — tools like Impacket request RC4 specifically because it produces hashes that crack faster. After AES enforcement on `svc_backup`, this rule would fire on any remaining RC4 requests as a residual risk indicator.  
**MITRE:** T1558.003 — Kerberoasting

### Rule 100013 — PsExec service (level 14)

**Source event:** Windows Event ID 7045 (new service installed)  
**Filter:** `serviceName` = "PSEXESVC"  
**Why specific:** PsExec always installs a service named PSEXESVC. This is an exact-match indicator with virtually no legitimate explanation outside of authorised admin work. Level 14 reflects the near-certainty of malicious intent.  
**MITRE:** T1021.002 — SMB/Windows Admin Shares

### Rule 100014 — Service account on endpoint (level 12)

**Source event:** Windows Event ID 4624 (successful logon)  
**Filter:** `targetUserName` matches regex `^svc_` (any account starting with "svc_")  
**Why useful:** Service accounts should authenticate to the services they're configured for — not interactively to workstations. A successful network logon from any `svc_` account to a user workstation is abnormal and warrants investigation.  
**MITRE:** T1078.002 — Valid Accounts: Domain Accounts

---

## Additional Custom Rules (Enumeration Detection)

A second custom rule group targets the anonymous LDAP/SMB enumeration behaviour observed during Attack 1:

![Enumeration detection rules](../screenshots/hardening/customrule.png)

These rules fire on anonymous NTLM logons from external IPs and on bursts of anonymous session activity — patterns characteristic of enum4linux or ldapsearch reconnaissance rather than legitimate internal traffic.
