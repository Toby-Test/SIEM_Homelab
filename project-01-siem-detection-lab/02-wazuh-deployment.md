# Wazuh Deployment

## Wazuh Manager

Wazuh is installed on a dedicated Ubuntu Server VM using the all-in-one installer, which deploys the Wazuh manager, indexer (OpenSearch), and dashboard in a single operation.

**VM specifications:**

| Setting  | Value              |
|----------|--------------------|
| VM ID    | 101                |
| Name     | wazuh-man          |
| OS       | Ubuntu Server LTS  |
| vCPUs    | 2                  |
| RAM      | 4 GB               |
| Disk     | 50 GB              |
| Network  | vmbr1 (LAN)        |
| IP       | 192.168.10.10      |

The Wazuh dashboard is accessible at `https://192.168.10.10` after installation.

---

## Agent Enrolment

Two endpoints are enrolled as Wazuh agents. Both show **active** status prior to the attack simulation.

![Agent dashboard](../screenshots/alerts/wazuhdashboard.png)

| Agent ID | Name             | IP              | OS                          | Version |
|----------|------------------|-----------------|-----------------------------|---------|
| 001      | ubuntu-endpoint  | 192.168.10.124  | Ubuntu 24.04.4 LTS          | v4.14.5 |
| 002      | windows-endpoint | 192.168.10.160  | Windows 10 Pro 10.0.19045   | v4.14.5 |

### Ubuntu Endpoint Agent

The Wazuh agent is installed from the official Wazuh APT repository and pointed at the manager's static IP. The `WAZUH_MANAGER` environment variable during installation sets the manager address in `ossec.conf` automatically.

```bash
sudo WAZUH_MANAGER="192.168.10.10" apt install wazuh-agent -y
sudo systemctl enable --now wazuh-agent
```

### Windows Endpoint Agent

The agent is deployed via the Wazuh dashboard's **Deploy new agent** workflow, which generates a PowerShell one-liner pre-populated with the manager address.

**Sysmon** is additionally installed on the Windows endpoint using the SwiftOnSecurity configuration. This dramatically expands log coverage beyond default Windows Event Log entries, capturing process creation (Event ID 1), network connections (Event ID 3), and file modifications. The Wazuh agent `ossec.conf` is updated to include the Sysmon event channel:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

---

## Log Sources

| Source                     | Forwarding method        | Data captured                              |
|----------------------------|--------------------------|--------------------------------------------|
| ubuntu-endpoint (sshd/PAM) | Wazuh agent (journald)   | Auth events, SSH sessions, PAM failures    |
| windows-endpoint           | Wazuh agent (eventchannel)| Windows Event Log, Sysmon events          |
| OPNsense firewall          | Syslog UDP 514           | Connection logs, firewall allow/block rules |
