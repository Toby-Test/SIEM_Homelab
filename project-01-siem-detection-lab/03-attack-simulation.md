# Attack Simulation

**Attacker:** kali-attacker — 192.168.20.50  
**Target:** ubuntu-endpoint — 192.168.10.123  
**Attack type:** SSH brute force (credential stuffing via rockyou wordlist)  
**Tool:** Hydra v9.6

---

## Phase 1 — Reconnaissance

Before launching the attack, the target is confirmed reachable and the SSH service is fingerprinted using Nmap.

```bash
nmap -sV 192.168.10.123
```
**Results:**
![Nmap scan](../screenshots/attack/kalisshopen.png)

The host is up and SSH is accepting connections. Service version data confirms this is a Linux target running a current OpenSSH build.

---

## Phase 2 — Brute Force Execution

Hydra is used to attempt SSH authentication as `root` against the target, cycling through the `rockyou.txt` password wordlist.

```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt \
  ssh://192.168.10.123 -t 4 -V -f
```

**Flag breakdown:**

| Flag             | Effect                                     |
| ---------------- | ------------------------------------------ |
| `-l root`        | Single target username: root               |
| `-P rockyou.txt` | Password list                              |
| `-t 4`           | 4 parallel threads                         |
| `-V`             | Verbose                                    |
| `-f`             | Stop immediately on first valid credential |

![Hydra launch](../screenshots/attack/HYDRAREAL.png)

At execution, Hydra reports the attack rate and total attempt count:

- **Rate:** ~219 tries/min
- **Total candidates:** 14,344,399
- **Threads:** 3 active (SSH server limits parallel connections)

![Hydra running](../screenshots/attack/hydra2.png)

The live output shows each `[ATTEMPT]` line as Hydra cycles through common passwords from the wordlist.

![Live attempts](../screenshots/attack/HYDRALIVE.png)

---

## Notes on Lab Configuration

To allow the full attack lifecycle to be observed (failed attempts escalating through to a successful login), the Ubuntu endpoint's SSH configuration is intentionally weakened for this exercise:

- `PermitRootLogin yes` — allows root authentication
- `PasswordAuthentication yes` — disables key-only enforcement

> This configuration is strictly isolated to the lab environment. It is not representative of any recommended SSH hardening posture and would never be applied to a production or internet-facing system.
