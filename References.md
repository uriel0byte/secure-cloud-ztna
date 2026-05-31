## Concepts and Key Takeaways

This section explains the core concepts behind what was built. Useful for review, and for any reader unfamiliar with the tooling.

---

## Zero Trust Network Access (ZTNA)

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

## PSAD, Postfix, and Email Alerting

This section covers everything deployed in the PSAD project from the ground up — what each tool is, why it exists, how the pieces connect, and why specific decisions were made during setup. Written for personal review and for anyone replicating this from scratch.

---

### What PSAD Is

PSAD stands for Port Scan Attack Detector. It was written by Michael Rash, the same person behind fwknop and fwsnort, and it has been around since the early 2000s. The core idea is simple: firewalls block traffic but don't analyze it. PSAD sits alongside the firewall, reads what the firewall logs, and turns raw blocked-packet entries into meaningful intelligence — who is scanning, what ports they hit, how aggressive the scan was, and whether the pattern matches a known attack signature.

It is not a next-generation tool. It is not glamorous. But it works, it is lightweight, and it runs on any Linux system with iptables or UFW without requiring kernel modules, additional drivers, or root-level packet capture. For a single hardened server, it is exactly the right size.

---

### What PSAD Is Not

PSAD is not a packet sniffer. It does not sit on the network interface and inspect traffic in real time. Tools like Suricata and Zeek do that — they run in the data path and analyze every packet as it arrives. PSAD reads a log file after the firewall has already processed and dropped the packets. This means PSAD's detection is only as good as what the firewall chooses to log.

PSAD is also not a prevention system by default. In detection-only mode (`ENABLE_AUTO_IDS N`) it observes and reports. It does not modify firewall rules, does not block IPs, and does not interfere with traffic in any way. Auto-IDS mode changes that, but detection-only is the correct starting point.

---

### How the Detection Pipeline Works

Understanding the full chain makes the configuration choices obvious:

```
Internet traffic arrives
        |
        v
UFW evaluates the packet against its rules
        |
        v
Packet is BLOCKED (default-deny inbound)
UFW writes a [UFW BLOCK] line to /var/log/syslog
        |
        v
PSAD tails /var/log/syslog continuously
Reads the [UFW BLOCK] line
Extracts: source IP, destination port, protocol, flags
        |
        v
PSAD increments the packet counter for that source IP
Checks the packet against its signature database
        |
        v
If packet count crosses DANGER_LEVEL threshold:
PSAD escalates the danger level for that IP
        |
        v
If danger level crosses EMAIL_ALERT_DANGER_LEVEL:
PSAD calls sendmail to fire an alert email
        |
        v
Postfix receives the mail request
Authenticates to Gmail SMTP via SASL
Relays the email to urielbyte.alerts@gmail.com
```

Every step in this chain has a config setting. If any step breaks, the chain stops there.

---

### The psad.conf Settings Explained

**`EMAIL_ADDRESSES`**

Where alert emails go. Multiple addresses can be separated by spaces. In this deployment it points to the dedicated alerts inbox rather than a personal address — keeps security alerts isolated and easy to filter.

**`HOSTNAME`**

The server's name as it appears in alert emails. Useful when you have multiple servers reporting to the same inbox. In this deployment it resolves to `uriel-soc-ztna-lab-01.tailf09853.ts.net` via Tailscale's hostname assignment.

**`ENABLE_AUTO_IDS`**

The most important setting. `N` means detection-only — PSAD watches and reports but never touches the firewall. `Y` enables auto-blocking, where PSAD writes iptables rules to ban flagged IPs automatically. The reason to start with `N`: on a remote server with no physical or out-of-band console access, an aggressive false positive in auto-IDS mode can lock out legitimate traffic — including your own SSH connection. Detection-only first, auto-IDS later once you understand what normal traffic looks like on your server.

**`IPT_SYSLOG_FILE`**

Tells PSAD which log file to read. This must match where UFW actually writes its log entries. On Ubuntu 26.04, that is `/var/log/syslog`. Getting this wrong is the most common reason PSAD appears to run fine but detects nothing — it is reading the right format from the wrong file.

**`DANGER_LEVEL1` through `DANGER_LEVEL5`**

Packet count thresholds per source IP. Each level is a number of packets — when a source IP's blocked packet count crosses a threshold, PSAD escalates it to that danger level. The levels are not automatic severity scores — they are configurable thresholds you set based on your environment.

Default values in PSAD 2.4.6:
- Level 1: 5 packets
- Level 2: 15 packets
- Level 3: 150 packets (changed from default 1500 in this deployment)
- Level 4: 1500 packets
- Level 5: 10000 packets

Why level 3 was changed from 1500 to 150: a default Nmap scan of 1000 ports sends roughly 1000+ SYN packets. At 1500, a complete Nmap scan would not reach the email alert threshold. At 150, any full port scan triggers an alert while casual single-port background probes (5-10 packets) stay below it.

**`EMAIL_ALERT_DANGER_LEVEL`**

The minimum danger level that triggers an email. Set to 1 in this deployment, meaning any detection fires an email. This is correct for initial setup — you want to see everything while learning what your server's normal traffic looks like. In production, raising this to 2 or 3 reduces inbox noise significantly.

**`MIN_DANGER_LEVEL`**

The minimum danger level PSAD tracks internally. At 1, PSAD starts counting from the very first blocked packet. Raising this tells PSAD to ignore sources below the threshold entirely — they never appear in `psad -S` output at all.

---

### The Danger Level Threshold — Thinking It Through

The threshold question comes down to: what does background internet noise look like vs what does a real scan look like?

Background noise: automated bots probing specific ports they care about — 3389 for RDP, 22 for SSH, 80/443 for web vulnerabilities. These typically send 1-5 packets per IP and move on. They are not scanning your full port range. They are running a list and checking each entry once.

A real port scan: Nmap's default scan covers 1000 ports. A SYN scan sends one packet per port. That is 1000 packets minimum from a single source in a short window. An aggressive scan (`-T4` or higher) sends them faster. An exhaustive scan (`-p-`) hits all 65535 ports.

Setting the alert threshold at 150 means: a bot that hits 3 specific ports never triggers an alert. An operator running a quick targeted scan of 20 ports never triggers an alert. Anyone scanning more than 150 ports from a single IP within PSAD's tracking window does trigger an alert. That is a reasonable line.

---

### Signatures

PSAD ships with a signature file at `/etc/psad/signatures` derived from Snort rules. These are pattern matches on specific combinations of port, protocol, flags, and payload that correspond to known attack behavior.

When a blocked packet matches a signature, PSAD names it in the output — "MISC MS Terminal Server communication attempt" for RDP probes, "Nmap -sT or -sS scan" for SYN scans, and so on. A packet can trigger a detection both by crossing the packet count threshold and by matching a signature — either alone is enough.

`sudo psad --sig-update` downloads the latest signature file from cipherdyne.org and replaces the current one. Running this periodically keeps detection current. After updating, restart PSAD or send it a HUP signal (`sudo psad -H`) to reload the new signatures without a full restart.

---

### What `psad -S` Shows

`psad -S` prints the current detection state — everything PSAD has accumulated since it last reset. Key fields:

**Top 50 signature matches** — which named attack patterns have been seen and how many times.

**Top 25 attackers** — source IPs ranked by packet count, with danger level, total packets, and signature match count. This is the primary triage view.

**Top 20 scanned ports** — which destination ports are being probed most. Seeing port 3389 at the top means RDP scanners. Port 22 means SSH brute-force preparation. Port 80/443 means web vulnerability scanning.

**iptables log prefix counters** — how many `[UFW BLOCK]`, `[UFW ALLOW]`, and `[UFW AUDIT]` entries PSAD has processed. This shows PSAD is actually reading the log file.

**IP Status Detail** — per-IP breakdown: danger level, destination, scanned ports, packet count, matched signatures, and how many alert emails have been sent for that IP.

---

### Why `/etc/psad/auto_dl` Exists

`auto_dl` stands for automatic danger level. It is a file where you assign a fixed danger level to specific IPs or networks, overriding PSAD's normal packet-count logic.

Format:
```
IP_ADDRESS    DANGER_LEVEL;
```

Setting an IP to danger level `0` means PSAD will never escalate it regardless of how many packets it sees from that source. Setting it to `5` means PSAD immediately treats any traffic from that IP as a severe threat.

**Why `10.128.0.2` was whitelisted:**

`10.128.0.2` is the GCP internal network interface address (`ens4`) — the source IP on every outbound packet the server sends. When UFW logging is set to `medium`, it logs outbound traffic as well as inbound. PSAD reads those outbound log entries and sees a single IP (`10.128.0.2`) sending packets to many different destinations on many different ports. That pattern — one source, many destinations, many ports — is exactly what a scanner looks like. Without the whitelist, PSAD flags the server's own normal traffic as an attack.

The fix is one line in `auto_dl`:
```
10.128.0.2    0;
```

After adding the line, restart PSAD. On startup you will see in the service log:
```
imported auto_dl file, got 1 IP addrs/networks
```

That confirms the whitelist loaded. Any other IP generating noise in your environment — a monitoring service, a known good scanner, a Tailscale relay node — can be added here the same way.

---

### UFW Logging Levels and Their Effect on PSAD

UFW has four logging levels: `off`, `low`, `medium`, `high`, and `full`. The choice of level directly determines what PSAD can see.

**`low`** (production setting): Logs packets that are blocked by the default policy and packets that match explicit logging rules. Does not log every blocked packet — samples. This means fast scans that complete before UFW logs them may not trigger PSAD detection. For day-to-day operation this is the right choice — the noise-to-signal ratio is manageable.

**`medium`**: Logs all packets that do not match a rule, plus rate-limited logging for packets that do match rules. Also logs outbound traffic. This gives PSAD full visibility into inbound scans but introduces the internal IP problem described above. Useful for investigation and testing, not for production.

**`high`** and **`full`**: Log everything including rate-limited packets. Generate significant volume. Rarely appropriate for a single server deployment.

The practical implication: at `low` logging, a fast SYN scan that completes in under 10 seconds may only have 50-100 packets logged by UFW before the scan finishes. If that is below your `DANGER_LEVEL3` threshold, no alert fires. Slowing the scan down (Nmap's `-T2` or `-T1` timing) gives UFW more time to log packets and gives PSAD more input to work with.

---

### What Postfix Is and Why It Is Needed

Postfix is a Mail Transfer Agent (MTA) — software that handles sending and receiving email at the system level. When a program on Linux wants to send email, it calls `sendmail` (a command-line interface), which is handled by whatever MTA is installed. Without an MTA, `sendmail` does not exist and any program trying to send email fails immediately.

PSAD calls `sendmail` internally when it fires an alert. Without Postfix (or another MTA), that call fails silently — PSAD thinks it sent the email, the email never goes anywhere, and you never know a detection happened.

Postfix on its own can send and receive email directly if the server has a valid domain and is not blacklisted by major providers. For a GCP instance, direct mail sending is unreliable — GCP's IP ranges are frequently blocked by spam filters. The solution is to route outbound mail through an established provider's SMTP server — in this case Gmail — rather than sending directly. This is called a relay or smarthost configuration.

---

### How the Gmail SMTP Relay Works

Gmail's SMTP server accepts mail from authenticated clients and delivers it on their behalf. The server (`smtp.gmail.com`) listens on port 587 and requires STARTTLS encryption and SASL authentication before accepting any mail.

The configuration in `/etc/postfix/main.cf` tells Postfix:
- Send all outbound mail through `[smtp.gmail.com]:587` rather than delivering directly
- Authenticate using SASL with the credentials in `sasl_passwd`
- Require TLS encryption (`smtp_tls_security_level = encrypt`) — no plaintext connections
- Use the system's CA certificate bundle to verify Gmail's TLS certificate

---

### What SASL Is

SASL stands for Simple Authentication and Security Layer. It is a framework for adding authentication to network protocols — in this case, it lets Postfix prove its identity to Gmail's SMTP server before sending mail.

Without SASL, Postfix would connect to Gmail's SMTP server and Gmail would reject it because it has no way to verify who is sending. With SASL, Postfix presents credentials (the Gmail address and App Password), Gmail verifies them, and the connection is authorized.

`libsasl2-modules` provides the SASL implementation that Postfix uses. Without it installed, Postfix cannot perform SASL authentication even if the config says to.

---

### Why Gmail App Passwords Exist

Google does not allow third-party applications to authenticate with your Gmail account using your regular password. This is a security measure — if a third-party app is compromised, your main account password is not exposed.

App Passwords are a workaround for this restriction. Each App Password is a 16-character credential tied to a specific application and revocable independently. Revoking one does not affect your main account password or any other App Password. If the credential stored in `sasl_passwd` is ever exposed, you revoke that specific App Password and generate a new one — nothing else is compromised.

To generate one: Google Account → Security → 2-Step Verification → App passwords. 2-Step Verification must be enabled first. The password is only shown once at generation time. Store it immediately in `sasl_passwd`.

---

### Why `postmap` Is Required

Postfix does not read plain text files for lookups during mail delivery — the process is too performance-sensitive. Instead it reads binary hash databases built from those plain text files. `postmap` converts a plain text file into the binary format Postfix can query efficiently.

Running `sudo postmap /etc/postfix/sasl_passwd` creates `/etc/postfix/sasl_passwd.db` — the binary hash database. Postfix reads `sasl_passwd.db`, not `sasl_passwd`. If you edit `sasl_passwd` and forget to run `postmap` again, Postfix continues using the old database and your changes have no effect.

```bash
# Edit the credentials file
sudo nano /etc/postfix/sasl_passwd

# Rebuild the database — always required after editing
sudo postmap /etc/postfix/sasl_passwd

# Reload Postfix to apply
sudo systemctl reload postfix
```

---

### Why the File Permissions on `sasl_passwd` Matter

`/etc/postfix/sasl_passwd` contains a Gmail address and App Password in plain text. Setting permissions to `600` (`-rw-------`) means only root can read or write the file. Any other user on the system — including the `uriel` account — cannot read it.

This matters because Postfix runs as root when processing outbound mail, so it can read the file. But if another process or user on the server were ever compromised, they could not read the credentials file and would not be able to extract the App Password.

```bash
sudo chmod 600 /etc/postfix/sasl_passwd
sudo chmod 600 /etc/postfix/sasl_passwd.db
```

Both files should be `600` — the database contains the same credentials as the plain text file.

---

### The `dpkg-reconfigure postfix` Prompts Explained

When Postfix is installed on Ubuntu without a proper configuration, running `dpkg-reconfigure postfix` launches an interactive wizard. Each prompt:

**General mail configuration type:** "Internet Site" means Postfix sends and receives mail directly using SMTP. This is correct for a server that will relay through Gmail — it sets up the base configuration that the Gmail relay settings then modify.

**System mail name:** The domain name that appears in the `From` header of locally-generated mail. Used to identify the source server in alert emails. Set to `uriel-soc-ztna-lab-01` — this appears in the email headers so you know which server sent the alert.

**Root and postmaster mail recipient:** Where system mail for root goes. Left blank here — system mail goes to `/var/mail/nobody` which is fine for a single-server lab.

**Other destinations:** Domains this server considers itself the final destination for. Left blank — this server is not a mail server, it only relays outbound alerts.

**Local networks:** IP ranges allowed to relay mail through this server. Left blank — only localhost can send through this Postfix instance.

**Mailbox size limit:** Set to `0` (unlimited). Not relevant for a relay-only configuration but harmless.

**Local address extension character:** `+` is the standard separator for address extensions (e.g. `user+tag@domain`). Standard setting.

**Internet protocols:** `ipv4` only — the server's external connectivity is IPv4, IPv6 is not needed for this relay.

---

### Real-World Detections — What They Tell You

The three external IPs that appeared unsolicited in PSAD's output within hours of deployment are worth understanding individually.

**`18.221.18.194` — Amazon AWS EC2, Ohio**

An EC2 instance on Amazon's commercial cloud infrastructure. The reverse DNS (`ec2-18-221-18-194.us-east-2.compute.amazonaws.com`) confirms it is a rented compute instance. This is a common pattern for commercial vulnerability scanning services, automated security research tools, and botnets that rent cloud infrastructure to avoid geographic blocking. It probed ports 22 and 3389 — SSH and RDP — which are the two most commonly targeted ports for credential brute-force attacks.

**`103.1.210.25` — Viettel, Hanoi, Vietnam**

Viettel is Vietnam's largest telecommunications provider. This IP arrived after PSAD was fully configured and fired an alert automatically — no manual trigger. It probed port 3389 only. Single-port probes like this are typically automated bots running through IP lists, checking whether RDP is exposed. They are not targeted attacks — they are opportunistic scans looking for any open RDP port to attempt credential stuffing against.

**`45.142.193.164` — Skynet Network Ltd, Amsterdam, Netherlands**

Skynet Network Ltd (AS214295) is a hosting provider that frequently appears in threat intelligence databases. From a defensive Blue Team perspective, you will often spot this IP and its neighboring subnets in firewall and IDS logs triggering automated alerts. OSINT sources confirm this specific IP is highly active as an automated bot, frequently scanning for exposed RDP (Port 3389) and SSH connections. These are not targeted attacks; rather, the host is routinely running through massive IP lists to identify open ports, making it a prime candidate for opportunistic credential stuffing and brute-force attempts.

**What this means in practice:** Public IP addresses are scanned constantly. There is no such thing as a "quiet" period for a public IP. The ZTNA stack was already blocking all of this before PSAD was deployed — UFW's default-deny rule dropped every packet. PSAD's contribution is making the noise visible, named, and reported. The server was never in danger. The detections are evidence of the firewall working, not evidence of a breach.

---

### The Commands You Need to Know

```bash
# Check PSAD's current detection state
sudo psad -S

# Check status for a specific IP
sudo psad --status-ip 18.221.18.194

# Show only detections at or above danger level 2
sudo psad --status-dl 2

# Update signatures from upstream
sudo psad --sig-update

# Reload config and signatures without full restart
sudo psad -H

# Restart PSAD completely (clears accumulated detection state)
sudo systemctl restart psad

# Watch syslog for UFW blocks in real time
sudo tail -f /var/log/syslog | grep "UFW BLOCK"

# Watch for PSAD-specific log entries in real time
sudo tail -f /var/log/syslog | grep -i psad

# Check the raw status output file
sudo cat /var/log/psad/status.out

# Verify config values without opening the file
sudo grep -E "EMAIL_ADDRESS|ENABLE_AUTO_IDS|IPT_SYSLOG_FILE|DANGER_LEVEL" /etc/psad/psad.conf | grep -v "^#"

# Test the Postfix relay manually
echo "Test message" | sudo mail -s "Test subject" urielbyte.alerts@gmail.com

# Check Postfix mail queue (shows pending/stuck emails)
sudo mailq

# Flush the Postfix queue (force immediate delivery attempt)
sudo postfix flush

# Check Postfix logs for delivery status
sudo tail -f /var/log/syslog | grep postfix

# Look up an IP's origin and organization
curl https://ipinfo.io/IP_ADDRESS_HERE
```

---

### Troubleshooting Reference

**PSAD is running but detecting nothing:**
- Check `IPT_SYSLOG_FILE` points to the correct log file
- Verify UFW logging is on: `sudo ufw status verbose` — should show `Logging: on (low)`
- Confirm UFW is actually writing to syslog: `sudo grep "UFW BLOCK" /var/log/syslog | tail -5`
- Check PSAD is reading the log: `sudo psad -S` — look at `iptables log prefix counters`

**Alert emails not arriving:**
- Test Postfix relay directly: `echo "test" | sudo mail -s "test" urielbyte.alerts@gmail.com`
- Check Postfix logs: `sudo tail -20 /var/log/syslog | grep postfix`
- Check mail queue for stuck messages: `sudo mailq`
- Verify `sasl_passwd.db` exists and is up to date: `ls -la /etc/postfix/sasl_passwd*`
- Confirm App Password has not been revoked in Google account settings

**PSAD flagging the server's own traffic:**
- Check if UFW logging is set above `low`: `sudo ufw status verbose`
- If `medium` or higher, revert: `sudo ufw logging low`
- Add the internal GCP interface IP to `auto_dl`: `sudo nano /etc/psad/auto_dl`
- Restart PSAD after changes: `sudo systemctl restart psad`

**Postfix warnings about duplicate config entries:**
- Open `main.cf` and search for duplicate `relayhost` or `smtp_tls_security_level` lines
- Keep only the Gmail relay versions, delete the empty or conflicting originals
- Reload after fixing: `sudo systemctl reload postfix`

**Further reading:** [PSAD Documentation](http://www.cipherdyne.org/psad/docs/) | [Postfix SASL README](https://www.postfix.org/SASL_README.html) | [Postfix Basic Configuration](https://www.postfix.org/BASIC_README.html) | [UFW Documentation](https://help.ubuntu.com/community/UFW)

---
