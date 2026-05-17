# secure-cloud-ztna

A hands-on deployment of a Zero Trust Network Access (ZTNA) architecture on a Google Cloud Platform Linux server — built, hardened, and audited from scratch.

This is not a tutorial follow-along. Every phase was executed live on a real cloud instance, troubleshot in real time, and verified against actual system telemetry. The full deployment report documents every decision, every command, and every mistake made along the way.

---

## What Was Built

A GCP `e2-micro` instance running Ubuntu 26.04 LTS (Minimal) was transformed from a default, publicly exposed server into a Zero Trust environment:

- Password authentication stripped and replaced with ed25519 cryptographic key pairs
- Host firewall set to default-deny with ICMP echo-requests silently dropped at the kernel level — the server returns no response to ping sweeps
- SSH access locked exclusively to a private WireGuard mesh network via Tailscale — no publicly reachable ports remain
- Multi-device access provisioned with per-device key pairs (VMware workstation + iPad via Termius)
- Nightly automated configuration backups via rsync and cron
- Full internal audit across socket state, authentication logs, and the kernel ring buffer to verify every layer

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
- Complete session command reference including troubleshooting missteps
- Concept explanations for every technology used
- Project retrospective — what worked, what didn't, what comes next

→ **[Read the full report](./ZTNA-Deployment-Report.md)**

---

## Context

This project is part of an active portfolio built toward a SOC Tier 1 / Blue Team role. The focus is on documentation quality, verifiable evidence of execution, and honest post-mortems — not just getting things to work.

Other projects: [Cybersecurity-Writeups](https://github.com/uriel0byte/Cybersecurity-Writeups)

---

**Supawat H. (uriel0byte)**
[LinkedIn](https://www.linkedin.com/in/supawat-h-145371392) · [YouTube](https://www.youtube.com/@urielbyte)
