# Infrastructure Setup

## Proxmox VE — Hypervisor

Proxmox VE is installed bare-metal on the lab host and serves as the foundation for all virtual machines

### Virtual Network Bridges

Four Linux bridges are configured to create isolated network segments:

| Bridge | Role      | Purpose                                |
| ------ | --------- | -------------------------------------- |
| vmbr0  | WAN       | OPNsense WAN interface                 |
| vmbr1  | LAN       | OPNsense LAN, Wazuh Manager, endpoints |
| vmbr2  | Attacker  | OPNsense OPT1 (ATTACKER), Kali Linux   |
| vmbr3  | Tailscale | Remote management access               |

![Network bridges](../screenshots/setup/pvenet.png)

vmbr2 is intentionally isolated — Kali Linux can only reach the LAN through the OPNsense firewall rule, which logs all crossing traffic.

---

## OPNsense — Perimeter Firewall

OPNsense sits between all network segments and enforces traffic policy. It exposes three interfaces:

| Interface | Identifier | Device  | Subnet            |
|-----------|------------|---------|-------------------|
| WAN       | wan        | vtnet0  | DHCP (home LAN)   |
| LAN       | lan        | vtnet1  | 192.168.10.1/24   |
| ATTACKER  | opt1       | vtnet2  | 192.168.20.1/24   |

![Interface assignments](../screenshots/setup/opnsense-assignments.png)

### Firewall Rules — Attacker Segment

A single rule permits Kali to reach Lab LAN hosts. The rule is explicitly logged so that every attack connection generates a firewall event visible in the SIEM.

![Firewall rule](../screenshots/setup/firewallrules.png)

### Syslog Forwarding to Wazuh

OPNsense is configured to forward all syslog output to the Wazuh Manager over UDP port 514. This provides network-layer visibility in the SIEM — firewall connection logs appear alongside endpoint authentication events for cross-source correlation.

![Syslog config](../screenshots/setup/opnsense-syslog-wazuh.png)

