# Alert Analysis

## Overview

The Wazuh SIEM generated **1,119 alerts** against `ubuntu-endpoint` during a 30-minute window. The alerts escalate in severity as Wazuh's correlation engine detects the pattern of repeated failures crossing defined thresholds.

![Alert feed](../screenshots/alerts/wazuhalert.png)

---

## Alert Escalation Chain

Wazuh processes individual log events and applies frequency-based correlation rules on top of them. This produces a layered alert chain that distinguishes a single failed login from a sustained attack.

### Level 5 — Individual Failures

**Rule 5760 — sshd: authentication failed**  
Fired on every individual SSH authentication failure. This is the base-level event sourced directly from `sshd` via journald.

- Agent: `ubuntu-endpoint`
- Source IP: `192.168.20.50` (Kali)
- Destination user: `root`
- Decoder: `sshd`
- Fired: 641 times
- MITRE: T1110.001 (Password Guessing), T1021.004 (Remote Services: SSH)

![Rule 5760 detail](../screenshots/alerts/documentwazuh1.png)
![Rule 5760 MITRE mapping](../screenshots/alerts/documentwazuh2.png)


---

### Level 8–10 — Threshold and Correlation Rules

Once the individual failure count crosses Wazuh's internal frequency thresholds, higher-severity composite rules fire.

**Rule 5758 — Maximum authentication attempts exceeded** (level 8)  
SSH server's own per-connection limit has been hit. Indicates the attacker is cycling through multiple connection attempts.

**Rule 2502 — User missed the password more than one time** (level 10)  
PAM-based composite rule. Triggered when a user fails authentication multiple times within a short window.

**Rule 5551 — PAM: Multiple failed logins in a small period of time** (level 10)  
The primary brute force detection rule. Aggregates PAM failures and fires when frequency crosses threshold.

- rule.firedtimes: 15
- rule.frequency: 8 (requires 8 correlated events)
- MITRE: T1110 (Brute Force) — Credential Access

![Rule 5551 detail](../screenshots/alerts/doc2wazuhaert.png)
![Rule 5551 MITRE and compliance](../screenshots/alerts/doc2part3.png)

**Rule 40111 — Multiple authentication failures** (level 10)  
Correlation rule that aggregates cross-decoder failure events. Provides a single high-severity indicator spanning both sshd and PAM sources.

![Escalation alert view](../screenshots/alerts/wazuhalert2.png)

---

## MITRE ATT&CK Mapping

| Technique ID | Name             | Tactic              | Observed Behaviour                              |
|--------------|------------------|---------------------|-------------------------------------------------|
| T1110        | Brute Force      | Credential Access   | Automated password cycling via Hydra            |
| T1110.001    | Password Guessing| Credential Access   | Single username, large wordlist                 |
| T1021.004    | Remote Services: SSH | Lateral Movement | SSH as the access vector post-compromise       |

---

## Full Alert Field Reference

The table below documents all fields parsed from a representative Rule 5760 event:

| Field                    | Value                                                              |
|--------------------------|--------------------------------------------------------------------|
| `_index`                 | wazuh-alerts-4.x-2026.05.07                                        |
| `agent.id`               | 001                                                                |
| `agent.ip`               | 192.168.10.123                                                     |
| `agent.name`             | ubuntu-endpoint                                                    |
| `data.dstuser`           | root                                                               |
| `data.srcip`             | 192.168.20.50                                                      |
| `data.srcport`           | 40854                                                              |
| `decoder.name`           | sshd                                                               |
| `full_log`               | May 07 14:21:14 ubuntu-endpoint sshd[3020]: Failed password for root from 192.168.20.50 port 40854 ssh2 |
| `location`               | journald                                                           |
| `manager.name`           | wazuh-man                                                          |
| `rule.description`       | sshd: authentication failed.                                       |
| `rule.firedtimes`        | 641                                                                |
| `rule.groups`            | syslog, sshd, authentication_failed                                |
| `rule.id`                | 5760                                                               |
| `rule.level`             | 5                                                                  |
| `rule.mitre.id`          | T1110.001, T1021.004                                               |
| `rule.mitre.tactic`      | Credential Access, Lateral Movement                                |
| `rule.mitre.technique`   | Password Guessing, SSH                                             |
| `timestamp`              | May 7, 2026 @ 07:21:14                                             |

---

## Key Observations

1. **Dual-source coverage** — both `sshd` and `pam` decoders independently logged each authentication failure, providing corroboration between two separate kernel subsystems.

2. **Threshold escalation** — Wazuh's correlation rules produced a clear severity ramp from level 5 individual events to level 10 composite brute-force indicators without any custom rule writing.

3. **Network-layer corroboration** — OPNsense syslog forwarding means every Hydra connection also generated a firewall log entry at `192.168.10.10`, providing a second independent data source confirming the source IP and timing.

4. **MITRE coverage** — Default Wazuh rules correctly mapped the attack to T1110/T1110.001 (Brute Force / Password Guessing) and T1021.004 (Remote Services: SSH) without any additional configuration.
