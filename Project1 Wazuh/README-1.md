# Project 1 — SIEM Detection Lab

Wazuh SIEM deployed on Proxmox VE with OPNsense network segmentation, monitoring an Ubuntu/Windows endpoint pair against a Kali attacker VM.

---

## What it is

A self-contained virtual lab built to learn the practical mechanics of SIEM operation: log ingestion, parsing, rule correlation, dashboard triage, and alert investigation. The environment simulates a minimal enterprise: a firewall, a Linux server, a Windows endpoint, and an attacker. Attacks are run manually from Kali, telemetry flows into Wazuh, and alerts are investigated as a Tier 1 analyst would handle them.

## Why it matters

The single most common gap on a junior SOC analyst's CV is hands-on SIEM exposure. This project demonstrates that I can stand up a SIEM end-to-end, instrument endpoints with agents, generate realistic attack telemetry, and triage the resulting alerts using the same workflow used in a real SOC — without enterprise tooling. The skills transfer directly to Splunk, Microsoft Sentinel, and QRadar.

## What's in this folder

| File                                     | Purpose                                                                                |
| ---------------------------------------- | -------------------------------------------------------------------------------------- |
| `README.md`                              | This file — entry point and reading order                                              |
| `portfolio-writeup.md`                   | The main project document: architecture, attack scenarios, detections, lessons learned |
| `screenshots/`                           | Annotated evidence of alerts firing in the Wazuh dashboard                             |
| `configs/wazuh-agent-ossec.conf.example` | Sanitised endpoint agent configuration                                                 |

## Skills demonstrated by this project

- Deploying a SIEM (Wazuh manager + indexer + dashboard) on Ubuntu Server
- Network segmentation using OPNsense (LAN / attacker zones via Proxmox Linux bridges)
- Installing and configuring Wazuh agents on Linux and Windows endpoints
- Enabling Windows Event Log forwarding (Security, System, Application channels)
- Manual simulation of SSH brute force (`hydra`), port scanning (`nmap`), and failed Windows logon attempts
- Reading and interpreting Wazuh's default ruleset (5710, 5712, 18152, etc.)
- Writing short-form incident summaries from raw alert data

## Attack scenarios covered

| Scenario | MITRE ATT&CK | Primary detection source |
|---|---|---|
| SSH brute force against Ubuntu host | T1110.001 — Brute Force: Password Guessing | `sshd` auth log → Wazuh rule 5712 |
| `nmap` port scan from attacker segment | T1046 — Network Service Discovery | OPNsense Suricata syslog → Wazuh decoder |
| Failed Windows logon attempts | T1110.003 — Password Spraying (manual) | Windows Event 4625 → Wazuh rule 60122 |


## Limitations specific to this project

- Single-host lab — no high-availability, no clustering, no realistic log volume
- Default Wazuh ruleset only; custom rule authoring is demonstrated in Project 2
- Attacks are manually triggered, not automated via tooling like Atomic Red Team
- No log retention strategy or hot/warm/cold storage tiering tested
