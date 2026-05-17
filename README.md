# secure-cloud-ztna

A hands-on deployment of a Zero Trust Network Access (ZTNA) architecture on a Google Cloud Platform Linux server, built, hardened and audited from scratch.

This is not a tutorial follow along. Each phase was run live on a real cloud instance, fixed in real-time and checked against real system telemetry. The full deployment report documents every choice, each command, and each mistake along the way.

---

## What Was Built

I took a default, publicly exposed GCP `e2-micro` instance running Ubuntu 26.04 LTS (Minimal) and converted it into a Zero Trust environment:

- Password authentication replaced with ed25519 cryptographic key pairs
- Host firewall configured to default-deny, ICMP echo-requests dropped silently at the kernel level, i.e. server returns no response to ping sweeps
- Lockdown SSH access to a private WireGuard mesh network over Tailscale – no open, publicly accessible ports
- Multi device access with per device key pairs (VMware workstation + iPad using Termius)
- Automated daily backups of the configuration via rsync and cron
- Run a full internal audit over socket state, authentication logs and kernel ring buffer to check all layers

---

## Architecture

```
Public Internet
      │
      ▼
 GCP e2-micro (us-central1-f)
 ┌─────────────────────────────┐
 │  UFW — default deny         │
 │  iptables — DROP ICMP       │
 │  Fail2ban — auth monitoring │
 │  SSH — tailscale0 only      │
 └──────────────┬──────────────┘
                │ Tailscale VPN (WireGuard)
       ┌────────┴────────┐
       │                 │
 VMware Ubuntu       iPad (Termius)
 (ed25519 key)       (ed25519 key)
```

---

## Tech Stack

| Layer | Tool |
|---|---|
| Cloud | GCP Compute Engine |
| OS | Ubuntu 26.04 LTS "Resolute" (Minimal) |
| Cryptographic Auth | OpenSSH (ed25519) |
| Intrusion Prevention | Fail2ban |
| Host Firewall | UFW + iptables |
| Zero Trust Routing | Tailscale (WireGuard) |
| Backup Automation | Bash + rsync + cron |
| Mobile Access | Termius (iPad) |

---

## Repo Contents

```
secure-cloud-ztna/
├── README.md
└── ZTNA-Deployment-Report.md   ← Full deployment report
```

---

## Report

The full deployment report covers:

- Phase-by-phase deployment log with inline terminal recordings
- Complete list of session commands, including troubleshooting errors
- Explanation of concepts for each technology used
- Project Retrospective – what worked, what didn’t and what’s next

→ **[Read the full report](./ZTNA-Deployment-Report.md)**

---

## Background

This project is part of an active portfolio built towards a SOC Tier 1 / Blue Team role. I care about documentation quality, verifiable evidence of execution, and honest post-mortems, not just making things work.

Other projects: [Cybersecurity-Writeups](https://github.com/uriel0byte/Cybersecurity-Writeups)

---

**Supawat H. (uriel0byte)**
[LinkedIn](https://www.linkedin.com/in/supawat-h-145371392) · [YouTube](https://www.youtube.com/@urielbyte)
