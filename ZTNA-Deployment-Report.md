# Infrastructure Security and Zero Trust Network Access (ZTNA) Deployment
**Author:** Supawat H. (uriel0byte) | **Environment:** Google Cloud Platform | **Status:** Complete

---

## Executive Summary

The purpose of this project is to secure (harden) a publicly available GCP Linux server to a Zero Trust model. I removed password-based authentication with Authentication by cryptographic key pairs, dropped (silently) all inbound reconnaissance traffic at the kernel level, and restricted SSH access to an encrypted Private Mesh Network. Automated nightly backups are regularly performed to prevent the loss of the configuration due to erroneous remote commands. An internal audit confirmed that the server actively drops all unauthorized traffic and has no attack surface accessible from the Public Internet.

---

## Objective

1. Build a cloud server that is resilient to routine hostile internet traffic with zero manual intervention.
2. Transition from an open-perimeter model (Traditional) to the Zero Trust model in which no connection will be accepted without being verified, regardless of where the connection was initiated.
3. Document every stage of this deployment and audit process for future reference as a replicable Tier 1 Administrative reference.

---

## Core Technology Stack

| Layer | Tool |
|---|---|
| Cloud Provider | Google Cloud Platform (GCP) Compute Engine |
| Operating System | Ubuntu 26.04 LTS "Resolute" (Minimal) |
| Cryptographic Auth | OpenSSH (ed25519) |
| Intrusion Prevention | Fail2ban |
| Host Firewall | UFW + iptables |
| Zero Trust Routing | Tailscale (WireGuard mesh VPN) |
| Backup Automation | Bash + rsync + cron |
| Mobile Access | Termius (iPad) |

---

## Prerequisites

Before starting, the following were in place. Replicating this project requires the same baseline.

| Requirement | Detail |
|---|---|
| GCP Account | Free trial active — $300 credit, 90 days. No billing required to start. |
| Local Workstation | Ubuntu running in VMware. Used for key generation and initial SSH access. |
| Mobile Device | iPad with Termius installed. Used for mobile administration over Tailscale. |
| Asciinema | Installed on the server for terminal session recording. Install: `sudo apt install asciinema` |
| Tailscale Account | Free personal account at [tailscale.com](https://tailscale.com). Required to join devices to the same Tailnet. |
| Basic Linux CLI | Comfortable with navigation, file editing with `nano`, and reading command output. |

> **If you haven't done any remote Linux administration yet,** try going through the [linuxupskillchallenge](https://github.com/livialima/linuxupskillchallenge) first since it teaches you the exact skills needed for this project (Remote Server Access, User Management, File Permissions, Basic Services, and Basic Security Hardening), over 20 days and completely free.

---

## Deployment Log

### Phase 0 — Infrastructure Provisioning

I deployed an `e2-micro (2 vCPUs, 1 GB memory)` instance (`uriel-soc-ztna-lab-01`) in `us-central1-f` (zone). The boot disk was set to a 30GB Standard Persistent Disk and routed through Standard Network Tier to stay within GCP's free tier. Cost control here was intentional, not incidental — cloud billing is an operational security concern.

---

### Phase 1 — Identity Provisioning and Perimeter Hardening

The cloud-provisioned default user had no `sudo` access. Instead of rebuilding the instance, I used the GCP Metadata Agent to inject a public SSH key and create a new user (`uriel`) with `google-sudoers` membership. This gave me immediate administrative access from my local VMware Ubuntu workstation.

**First problem hit:** connecting through a local terminal emulator crashed server-side text editors due to a `$TERM` variable mismatch. Fixed first by exporting `xterm-256color` in the current shell session, then written to the server's `.bashrc` for persistence before moving to Phase 2.

With access stable, I disabled password authentication and root login in sshd_config, then installed and enabled Fail2ban to monitor authentication logs and automatically jail IPs that exceed the failed login threshold.

> 📹 **Recording:** [![Asciinema — Phase 1: Identity & Perimeter Hardening](https://asciinema.org/a/1073068.svg)](https://asciinema.org/a/1073068)
> 📹 **Recording:** [![Asciinema — Phase 1: Enable Fail2Ban](https://asciinema.org/a/1073069.svg)](https://asciinema.org/a/1073069)


---

### Phase 2 — Network Defense and Stealth Mode

UFW was set to default-deny on all inbound traffic. That covers most of the surface, but a standard firewall still tells a scanner "port closed" via a TCP RST packet — confirming the host exists. To avoid that, I edited the underlying `iptables` rules directly in `before.rules` to silently DROP ICMP echo-requests instead of rejecting them. Ping sweeps against the public GCP IP now return 100% packet loss. The server looks dead to anyone scanning it.

> 📹 **Recording:** [![Asciinema — Phase 2: UFW & ICMP Drop](https://asciinema.org/a/1073335.svg)](https://asciinema.org/a/1073335)

---

### Phase 3 — Zero Trust Pivot and Mobile Access

I deployed Tailscale, which issued the server a private Tailnet IP (`100.114.172.16`). SSH was re-bound to the `tailscale0` interface only, then the public SSH rule was deleted. At that point, the server had no publicly reachable ports.

For mobile access, I added an iPad to the Tailnet. A separate ed25519 key pair was generated inside Termius and the public key was appended to `~/.ssh/authorized_keys` on the server. Each device holds its own private key — if the iPad is compromised, that key gets revoked without affecting anything else.

> 📹 **Recording:** [![Asciinema — Phase 3: Zero Trust Pivot](https://asciinema.org/a/1073350.svg)](https://asciinema.org/a/1073350)

---

### Phase 4 — Automated Configuration Backup

Editing SSH or firewall rules remotely without a fallback is how you lock yourself out of a cloud instance with no console access. I wrote a Bash script using `rsync` to create timestamped backups of `/etc/ssh` and `/etc/ufw` nightly. The Ubuntu Minimal image did not ship with `cron`, so I diagnosed the missing daemon, installed it, and scheduled the script at 02:00 in the root crontab.

> 📹 **Recording:** [![Asciinema — Phase 4: Backup Automation](https://asciinema.org/a/1073382.svg)](https://asciinema.org/a/1073382)

---

### Phase 5 — Internal Security Audit

I ran a full internal audit to confirm the stack was working, not just assumed to be working.

- `ss -tulpn` confirmed the SSH daemon was listening on the correct interface.
- `journalctl -u ssh.service` showed the clear before/after: early auth events from the public external IP, recent events exclusively from the `100.x.x.x` Tailscale range.
- `dmesg | grep UFW` returned live `[UFW BLOCK]` entries — the firewall was actively dropping background internet noise in real time.

The audit verified behavior from the inside out, which is the only way to know the defense is actually functioning.

> 📹 **Recording:** [![Asciinema — Phase 5: Internal Audit](https://asciinema.org/a/1073472.svg)](https://asciinema.org/a/1073472)

---

## Command Reference (Full Session Log)

Complete chronological record of every command executed, including troubleshooting missteps. Commands marked `# [ERROR]` failed and are included because the fix that followed is instructive.

---

### Phase 1 — Environment and Identity

```bash
# Fix terminal crash caused by $TERM variable mismatch between local emulator and server
export TERM=xterm-256color
# Then add to ~/.bashrc for persistence:
# echo 'export TERM=xterm-256color' >> ~/.bashrc

# --- Run on LOCAL machine ---
# Generate ed25519 key pair — -C flag labels the key to its source device
ssh-keygen -t ed25519 -C "uriel@vmware-local"

# --- Run on SERVER ---
# Harden the SSH daemon: disable password auth and block direct root login
sudo nano /etc/ssh/sshd_config
# Set: PasswordAuthentication no
# Set: PermitRootLogin no

# Reload the daemon to apply sshd_config changes — edits do not take effect until this runs
sudo systemctl reload ssh

# Install brute-force and credential-stuffing protection
sudo apt install fail2ban

# Enable fail2ban at boot and start it immediately — single command does both
sudo systemctl enable --now fail2ban

# Confirm the sshd jail is active and display any currently banned IPs
sudo fail2ban-client status sshd
```

---

### Phase 2 — Firewall and Stealth

```bash
# [ERROR] Typo — 'uwf' is not a command
sudo uwf default allow outgoing

# Allow outbound traffic (UFW default is already permissive outbound, but explicit is better)
sudo ufw default allow outgoing

# Set default-deny on all inbound traffic
sudo ufw default deny incoming

# Temporarily open SSH before enabling the firewall — skipping this locks you out
sudo ufw allow OpenSSH

# Check rules before enabling
sudo ufw status
sudo ufw status numbered

# Activate the firewall
sudo ufw enable

# Verify UFW is running (UFW runs as a oneshot systemd service — Active: exited is normal)
sudo systemctl status ufw

# Enable kernel-level logging for dropped and blocked packets
sudo ufw logging low

# [ERROR] Wrong filename — 'before' does not exist
sudo nano /etc/ufw/before

# Correct file — edit iptables rules to silently DROP ICMP echo-requests
sudo nano /etc/ufw/before.rules
# In the ICMP rules section, change:
#   -A ufw-before-input -p icmp --icmp-type echo-request -j ACCEPT
# to:
#   -A ufw-before-input -p icmp --icmp-type echo-request -j DROP

# Apply changes
sudo ufw reload

# Verify the full ruleset including default policies
sudo ufw status verbose

# Deep kernel verification — confirm the DROP rule is loaded at the iptables level
sudo iptables -L ufw-before-input -n -v
```

---

### Phase 3 — Zero Trust Pivot

```bash
# Install Tailscale VPN agent (run on both server and local VM)
curl -fsSL https://tailscale.com/install.sh | sh

# Authenticate to your private Tailnet control plane
sudo tailscale up

# Confirm the Tailscale-assigned private IP
tailscale ip -4

# Lock SSH access to the Tailscale interface only
sudo ufw allow in on tailscale0 to any port 22

# Remove the public SSH rule — server now has no reachable public ports
sudo ufw delete allow OpenSSH

sudo ufw reload

# Confirm rule state after pivot
sudo ufw status verbose

# Disconnect from public SSH session to test Zero Trust access
exit

# External ping test — verify ICMP blackhole is active (expect 100% packet loss)
ping 35.206.67.167

# Reconnect via Tailscale — two methods, both valid
tailscale ssh uriel@100.114.172.16   # Tailscale's built-in SSH wrapper (prompts a warning on first use)
ssh uriel@100.114.172.16             # Standard SSH over the private Tailnet IP
```

**Mobile Access (iPad via Termius):**

```bash
# After generating the ed25519 key pair in Termius, append the public key on the server
nano ~/.ssh/authorized_keys
# Add the iPad public key on a new line below the existing VMware key
# Each device holds its own private key — revoke per-device without affecting others
```

> The same pattern applies to any device on the Tailnet — generate a key pair on the device, append the public key to ~/.ssh/authorized_keys, connect via the Tailscale IP. Windows users can use PuTTYgen for key generation and PuTTY as the SSH client.

---

### Phase 4 — Automated Backups

```bash
# Create the backup target directory
mkdir -p ~/infrastructure_backups

# Write the rsync backup script
nano ~/backup_configs.sh
#Scripts Content:
-----------------------------------------
#!/bin/bash

# Define the destination directory with a timestamp
DEST="/home/uriel/infrastructure_backups/config_$(date +%Y%m%d_%H%M%S)"

# Create the timestamped directory
mkdir -p $DEST

# Sync the critical directories
rsync -a /etc/ssh $DEST/
rsync -a /etc/ufw $DEST/

echo "Backup completed successfully at $DEST"
-----------------------------------------

# Verify the file was created
ls -l

# Grant execution permission
chmod +x ~/backup_configs.sh

# [ERROR] Running without sudo — rsync cannot read /etc/ssh or /etc/ufw as a regular user
./backup_configs.sh

# Correct: run with elevated privileges for a manual test
sudo ./backup_configs.sh

# Verify the timestamped backup directories were created
sudo ls -la ~/infrastructure_backups/

# [ERROR] cron not installed on Ubuntu Minimal — 'command not found'
sudo crontab -e

# [ERROR] Wrong package name
sudo apt install crontab

# Correct package name
sudo apt install cron

# Schedule the script to run nightly at 02:00
sudo crontab -e
# Add this line:
# 0 2 * * * /home/uriel/backup_configs.sh

# Check directory state after scheduling
ls -la
```

---

### Phase 5 — Telemetry Audit

```bash
# Start Asciinema recordings before running audit commands
asciinema rec phase4_backups.cast
asciinema rec phase5_audit.cast

# Verify listening sockets — confirm SSH daemon is bound to the correct interface
sudo ss -tulpn

# Verify cron is running and the scheduled job was registered
sudo journalctl -u cron --since "24 hours ago"
sudo grep CRON /var/log/syslog

# Audit SSH auth logs — verify Tailscale IP is the only accepted source
sudo journalctl -u ssh.service --since "2 hours ago"
sudo journalctl -u ssh.service | grep "Accepted"
sudo journalctl -u ssh.service | grep -i "accept*"
# Note: the * here is regex, not a shell wildcard. In regex, * means "zero or more of the
# preceding character (t)" — not "match anything". The shell glob wildcard you know from
# commands like 'grep *.txt' is expanded by the shell before the command runs, so grep never
# sees it as a pattern. Here the quotes prevent shell expansion, so grep treats it as regex.
# 'accept' alone would return the same results. This was the exact command run in session.

# Verify active UFW BLOCK entries via two methods
sudo dmesg | grep UFW | tail -n 20
sudo grep UFW /var/log/syslog | tail -n 20

# Test higher logging verbosity to see more packet detail
sudo ufw logging medium
sudo ufw status verbose

# Revert to standard logging level after audit is complete
sudo ufw logging low

# Upload recordings
asciinema upload phase4_backups.cast
asciinema upload phase5_audit.cast
```

---

## Concepts and Key Takeaways

This section explains the core concepts behind what was built. Useful for review, and for any reader unfamiliar with the tooling.

### Zero Trust Network Access (ZTNA)

Traditional perimeter security assumes that anything inside the network is trustworthy. Zero Trust starts from the opposite assumption: the network is already hostile. Every connection — regardless of where it comes from — must prove identity and authorization before being granted access. Location on the network means nothing. Cryptographic proof of identity means everything.

In this project: by locking SSH to the `tailscale0` interface, the server refuses all connections that haven't already authenticated to the private Tailnet. There is no way in from the public internet, even if an attacker knows the IP.

**Further reading:** [NIST SP 800-207: Zero Trust Architecture](https://csrc.nist.gov/publications/detail/sp/800-207/final)

---

### ICMP Blackholing

ICMP (Internet Control Message Protocol) is the protocol behind `ping`. Attackers use ping sweeps to quickly map which IPs in a range have live hosts. A standard firewall rule `REJECT`s these packets, which still sends a message back ("host unreachable") — confirming the IP is occupied. Changing the rule to `DROP` means no response is sent at all. The scanner sees nothing and moves on.

**Further reading:** [Nmap Host Discovery Documentation](https://nmap.org/book/man-host-discovery.html)

---

### Cryptographic Authentication (ed25519)

Password authentication is vulnerable to brute force, credential stuffing, and database leaks. Public key authentication works differently: the server holds a public key, and the connecting machine holds the corresponding private key. During authentication, the server issues a challenge that only the correct private key can solve mathematically — the private key never leaves the client machine. ed25519 is the current recommended algorithm: smaller key size than RSA with stronger security guarantees.

**Further reading:** [OpenSSH Manual — Key Types](https://www.openssh.com/manual.html)

---

### Configuration Drift

Configuration drift is what happens when a system is managed manually over time — sshd_config gets edited here, a firewall rule gets added there — and there is no record of what the original clean state looked like. When something breaks, recovery becomes guesswork. The rsync backup in this project captures the exact state of `/etc/ssh` and `/etc/ufw` at 02:00 every night, including file ownership and permissions. This matters because `sshd` will refuse to start if the ownership or permission bits on its config files are wrong.

**Further reading:** [rsync Manual](https://linux.die.net/man/1/rsync)

---

### VPN Types, Mesh Networks, and Related Protocols

A lot of terms get used interchangeably in security conversations when they describe fundamentally different things. This covers where each one sits and how they relate to what was built in this project.

---

**Normal traffic (unencrypted)**

Plain internet traffic travels unencrypted between your device and its destination. Your ISP sees it. Anyone positioned between you and the server can read it. There is no tunnel, no authentication requirement, and no privacy guarantee. This is the default state of any server before hardening.

---

**Traditional VPN — hub-and-spoke**

A traditional VPN routes all client traffic through a central server. Every device connects to that server — not to each other. Your traffic leaves your device encrypted, arrives at the VPN server, gets decrypted, and continues onward. If the server goes down, everyone loses access. This is the model behind corporate VPNs and consumer services like NordVPN or ExpressVPN.

The structural problem: every packet travels to the central server and back, even if two devices are on the same local network. The server is a bottleneck and a single point of failure.

---

**IPSec — the protocol underneath most traditional VPNs**

IPSec (Internet Protocol Security) is not a VPN product — it is a suite of protocols that encrypts and authenticates traffic at the IP layer. Most corporate VPNs and site-to-site tunnels run on top of it. When someone says "we use a VPN" in an enterprise context, there is a good chance IPSec is doing the actual encryption work underneath.

It operates in two modes. Transport mode encrypts only the payload of each packet, leaving the header visible. Tunnel mode wraps the entire original packet inside a new encrypted one — this is what VPN tunnels use. IPSec also typically pairs with IKE (Internet Key Exchange) to handle the negotiation of encryption keys between two endpoints before the tunnel comes up.

The reason WireGuard exists is largely a reaction to IPSec's complexity. IPSec is powerful but the configuration surface is large, the attack surface follows, and getting it wrong is easy. WireGuard achieves similar security with a codebase that is roughly 100 times smaller. For this project, WireGuard via Tailscale was the right call — IPSec would have been significant overhead for a single-server personal lab.

---

**Mesh VPN — what Tailscale is**

In a mesh VPN, every device connects directly to every other device — peer-to-peer, no central relay for actual traffic. There is still a coordination server (Tailscale's control plane), but it only handles one job: brokering the initial introduction between devices and managing key exchange. After that handshake, traffic flows directly between peers without touching Tailscale's infrastructure at all.

Each device gets a stable private IP in the `100.x.x.x` range. It does not matter if you are at home, on a coffee shop WiFi, or on mobile data — the Tailscale IP stays the same. This is exactly why locking SSH to the `tailscale0` interface works in this project: the server only accepts connections from inside that private mesh, regardless of where the connecting device physically is.

Tailscale is built on WireGuard. WireGuard is the underlying protocol that handles the encryption between peers. Tailscale is the management layer on top — it handles device authentication, key rotation, and the coordination plane so you never have to configure WireGuard manually. You get WireGuard's performance and security with none of the manual configuration.

One additional capability worth knowing: Tailscale supports exit nodes, where device A routes its internet traffic through device B. For example, while traveling abroad you can route through your home server to access region-locked content. This project did not use that feature, but it is part of the same architecture.

---

**Tor — anonymity network, not a VPN**

Tor routes traffic through a chain of volunteer-run relays, typically three. Each relay only knows the previous hop and the next hop — never the full path. This makes it very difficult to trace traffic back to its origin.

The tradeoff is performance. The multi-hop routing adds significant latency by design. Tor solves an anonymity problem, not a connectivity or access control problem. Using it for persistent SSH administration would be impractical. It is the wrong tool for what this project does.

---

**DoH and DoT — DNS encryption only**

DNS over HTTPS (DoH) and DNS over TLS (DoT) are not VPNs. They only encrypt DNS queries — the lookups your device makes when translating a domain name like `tailscale.com` into an IP address.

Normal DNS is unencrypted. Your ISP and anyone monitoring the network can see every domain you look up, even when the connection itself is encrypted via HTTPS. DoH and DoT wrap those queries in encryption so the lookups are not visible to network observers. It is a useful privacy layer, but a narrow one — it only protects the name resolution step, not the traffic itself.

---

**Quick reference**

| | Purpose | Encrypts traffic | Hides identity | Needs central server |
|---|---|---|---|---|
| Normal traffic | Connectivity | No | No | No |
| Traditional VPN | Privacy / access control | Yes | Partially | Yes — always |
| IPSec | Encryption protocol underlying most VPNs | Yes | Partially | Yes — or peer-to-peer |
| Mesh VPN (Tailscale) | Secure P2P connectivity | Yes | No | Coordination only |
| Tor | Anonymity | Yes | Yes | No (relay network) |
| DoH / DoT | DNS query privacy only | DNS only | No | Depends on resolver |

**Further reading:** [Cloudflare — What is IPSec](https://www.cloudflare.com/learning/network-layer/what-is-ipsec/) | [Tailscale How It Works](https://tailscale.com/blog/how-tailscale-works) | [WireGuard Technical Whitepaper](https://www.wireguard.com/papers/wireguard.pdf) | [Tor Project Overview](https://www.torproject.org/about/history/) | [Cloudflare — What is DoH](https://www.cloudflare.com/learning/dns/dns-over-https/)

---

### Fail2ban

Fail2ban watches log files (in this case `/var/log/auth.log`) for patterns that indicate an attack — repeated failed login attempts from the same IP, for example. When a threshold is crossed, it fires an iptables rule to block that IP for a configurable duration. It does not prevent a first attempt, but it makes large-scale brute-force and credential-stuffing attacks impractical by rate-limiting the attacker at the kernel level.

**Further reading:** [Fail2ban Documentation](https://www.fail2ban.org/wiki/index.php/Main_Page)

---

## Project Retrospective

### What Worked

**Proactive hardening.** Fail2ban was active and monitoring before the Zero Trust pivot, meaning the authentication layer was protected from the moment the public port was live — not patched in after the fact.

**Stealth verification.** The ICMP DROP rule was confirmed from outside, not just assumed. External ping sweeps returned 100% packet loss. The difference between "I think the firewall blocks pings" and "I ran the test and confirmed it" matters.

**Audit discipline.** Phase 5 used three separate data sources — socket state, auth logs, and the kernel ring buffer — to verify the architecture rather than relying on the firewall status page. Inside-out verification is more reliable than trusting a UI.

---

### What Could Have Been Done Better

**No infrastructure-as-code.** This entire environment was built manually, command by command. That is fine for learning and for understanding what every setting does. But if the GCP instance needs to be rebuilt from scratch, it currently requires manual reproduction of every step. There is no single file that defines the desired state.

**Backups are on the same disk.** The rsync script copies configuration files to `~/infrastructure_backups` on the same GCP persistent disk. If the disk fails or the instance is deleted, the backups go with it. They are only useful for recovery from accidental misconfiguration, not hardware failure.

---

### Future Work

**Infrastructure Metrics Dashboard.** Deploying a Prometheus node exporter on the server with a Grafana frontend would provide real-time visibility into system performance — CPU, memory, disk, and network throughput. While not a security tool by default, anomalous spikes in these metrics can surface signs of active attacks or misconfiguration that logs alone might miss.

**Network Traffic Monitoring.** Deploying Suricata or Zeek on the server to inspect inbound traffic at the packet level would add a detection layer below the firewall. Where UFW blocks by rule, an IDS can identify suspicious patterns in traffic that the firewall technically allows — port scans, protocol anomalies, or known malicious signatures.

**Host-based intrusion detection.** Wazuh (open source) could be deployed as an agent on this server to monitor file integrity, watch for privilege escalation attempts, and generate alerts on suspicious system calls — adding active detection on top of the current passive firewall logging.

**Infrastructure as Code.** Rewriting these manual steps as Terraform (to provision the GCP instance) and Ansible (to configure it) would make the environment fully reproducible in minutes. This is the standard for production infrastructure.

**Off-site backup storage.** Modifying the backup script to push archives to a GCP Cloud Storage bucket separates backup storage from the host — the most basic requirement for backups to be useful in a real failure scenario.

**Centralized log forwarding.** Shipping authentication, UFW, and system logs to an external SIEM (Splunk, Elastic, or Wazuh's built-in stack) would allow for persistent threat tracking, correlation across time, and alerting — which is the actual SOC use case this lab is preparing for.

**Automated port scanning baseline.** Running scheduled Nmap scans from an external host against the GCP IP and logging the results would create a continuous record of the server's external attack surface over time.

---

## Contact

**LinkedIn:** [Supawat H.](https://www.linkedin.com/in/supawat-h-145371392)  
**YouTube:** [@urielbyte](https://www.youtube.com/@urielbyte) — published 
technical walkthroughs demonstrating tool usage and methodology

---

*Report authored by Supawat H. (uriel0byte).*
