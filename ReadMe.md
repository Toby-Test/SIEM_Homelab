# Cybersecurity Homelab Portfolio

>Self taught.
>Career-changer building toward a junior SOC Analyst role in Melbourne, Australia.

## At a glance

| Field              | Detail                                                                                     |
| ------------------ | ------------------------------------------------------------------------------------------ |
| **Target role**    | Junior SOC Analyst / Blue Team Analyst                                                     |
| **Location**       | Melbourne, VIC, Australia (remote-eligible roles welcomed)                                 |
| **Certifications** | CompTIA Security+ (in progress, target {July 2026}) · Splunk Core Certified User (planned) |
| **Background**     | Career change from hospitality; 5+ years self-taught Linux & homelab                       |
| **Status**         | Active — all two projects complete and documented                                          |
## Projects

### [Project 1 — SIEM Detection Lab](./project-01-siem-detection-lab)

Wazuh SIEM on Proxmox with OPNsense network segmentation. Detects SSH brute force, port scans, and failed Windows logons against an Ubuntu/Windows endpoint pair, with full alert triage methodology documented.
**Tech:** Wazuh, Proxmox VE, OPNsense, Ubuntu Server, Windows 10, Kali Linux.

### [Project 2 — AD Attack & Hardening Lab](./project-02-ad-attack-hardening)

Full Active Directory environment (`northport.local`) with six attack scenarios — recon, password spraying, Kerberoasting, PsExec lateral movement, backdoor account creation, SMB exfiltration. Each attack hardened via GPO and re-validated. Five custom Wazuh rules mapped to MITRE ATT&CK.
**Tech:** Windows Server 2022, Impacket, CrackMapExec, Hashcat, Suricata IDS/IPS.

## Skills demonstrated

- **SIEM & detection engineering** — Wazuh deployment, custom rule creation, MITRE ATT&CK mapping, alert triage
- **Active Directory** — Domain controller build, attack simulation, hardening via GPO, Kerberos internals
- **Incident response** — NIST SP 800-61 lifecycle, containment, eradication, recovery, IR report writing
- **Network security** — OPNsense firewall, network segmentation, Suricata IDS/IPS, syslog forwarding
- **Endpoint** — Sysmon configuration, Windows Event Log analysis, PowerShell Script Block Logging
- **Linux** — Ubuntu Server administration, Samba, SSH hardening, `fail2ban`, log analysis
- **Frameworks** — MITRE ATT&CK, NIST SP 800-61, basic threat modelling

## About me

I'm changing careers from hospitality (kitchen hand with chef-level responsibilities) into cybersecurity. My technical foundation is self-taught over the last several years — Linux, programming, building PCs, and running a homelab. This portfolio is the practical evidence behind that pivot: 2 end-to-end projects that I designed, built, attacked, detected, hardened, and documented.

## Contact

- **LinkedIn:** [linkedin.com/in/toby-van-berkel](https://www.linkedin.com/in/toby-van-berkel)
- **Email:** [toby.test@protonmail.com]
- **GitHub:** [github.com/Toby-Test](https://github.com/Toby-Test)

---

*All lab work was conducted in an isolated, self-hosted virtual environment. No real systems, networks, or third-party assets were targeted.*
