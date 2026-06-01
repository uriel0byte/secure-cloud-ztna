# Port Scan Attack Detector (PSAD) Deployment

**Author:** Supawat H. (uriel0byte)  
**Environment:** Google Cloud Platform  
**Status:** Complete

---

## Summary

This project adds active scan detection to the ZTNA environment. PSAD monitors UFW/iptables log entries in real time, identifies port scan patterns, and fires email alerts when a threshold is crossed. It runs in detection-only mode — no automatic blocking. The alert pipeline routes through a Postfix SMTP relay to a dedicated Gmail inbox.

This is the detection layer for what the port scan baseline measures. The baseline answers "what does the attack surface look like from a known vantage point?" PSAD answers "is anyone unknown actively probing it right now?"

---

## Objective

1. Deploy PSAD in detection-only mode on the GCP server, reading UFW log entries from syslog.
2. Configure a working email alert pipeline via Postfix and Gmail SMTP so detections reach an actual inbox rather than sitting silently in a log file.
3. Tune detection thresholds to filter background internet noise and only alert on meaningful scan activity.
4. Verify the full pipeline with a live Nmap scan from the local VM against the public IP, confirming detection and email delivery end to end.

---

## Tech Stack

| Layer | Tool |
|---|---|
| Scan Detection | PSAD 2.4.6 |
| Log Source | UFW via `/var/log/syslog` |
| Mail Relay | Postfix 3.10.6 |
| SMTP Provider | Gmail (port 587, STARTTLS) |
| Auth | SASL with Gmail App Password |
| Alert Destination | Dedicated Gmail inbox (`urielbyte.alerts@gmail.com`) |
| Platform | GCP e2-micro, Ubuntu 26.04 LTS (Minimal) |

---

## Relationship to Port Scan Baseline

The port scan baseline project established what the server's attack surface looks like from two fixed vantage points — external (public internet) and internal (Tailnet). It runs on a schedule and diffs against a clean reference to catch changes over time.

PSAD covers a different gap. It watches for unknown parties probing the server in real time, from any source, at any time. Where the baseline is scheduled and perspective-limited, PSAD is continuous and passive. Together they answer the full picture: the baseline confirms the surface hasn't changed, PSAD watches who is scanning it.

One limitation in the port scan baseline was the absence of alerting — changes sat in `diff_log.txt` until manually reviewed. PSAD closes that gap for active reconnaissance attempts. The baseline limitation for scheduled external scanning remains open.

---

## Pre-Setup

A dedicated Gmail account was created for security alerts before any server-side work began: `urielbyte.alerts@gmail.com`. Using a separate inbox rather than a personal address keeps alert traffic isolated and makes it easier to spot detections without digging through unrelated email.

A Gmail App Password was generated for SMTP authentication. Google blocks direct password authentication for third-party SMTP clients, so an App Password is required. It was created under the account's security settings with 2-Step Verification enabled, labeled `psad-server`. The password is stored in `/etc/postfix/sasl_passwd` on the server with permissions set to `600`.

---

## Deployment Log

### Phase 1 - Install PSAD

PSAD was not available on the server — Ubuntu 26.04 Minimal ships with a minimal package set. During installation, apt flagged `postfix` as a suggested dependency for mail delivery. PSAD installed cleanly and the service started immediately with systemd preset enabled, meaning it will survive a reboot without manual intervention.

One thing to note: the install prompt asked for a mail server configuration type. "No configuration" was selected at this stage — postfix was not yet configured for Gmail relay, so leaving it unconfigured was intentional. This gets corrected in Phase 3.

> 📹 **Recording:** [![Asciinema - Phase 1: Install PSAD](https://asciinema.org/a/1168616.svg)](https://asciinema.org/a/1168616)

---

### Phase 2 - Configure PSAD

Six values were changed in `/etc/psad/psad.conf`:

**Email address** set to the dedicated Gmail inbox. This is where all alert emails go.

**`ENABLE_AUTO_IDS`** confirmed as `N`. Detection-only mode — PSAD logs and alerts but never modifies firewall rules automatically.

**`IPT_SYSLOG_FILE`** set to `/var/log/syslog`. This points PSAD at the file where UFW writes its blocked packet entries. Without this, PSAD reads the wrong log source and detects nothing.

**Danger level thresholds** tuned away from defaults. The default `DANGER_LEVEL3` was 1500 packets — reasonable for some environments but the full threshold picture needed review first. Final values:

| Level | Packets | Purpose |
|---|---|---|
| 1 | 5 | Internal tracking starts |
| 2 | 50 | Low suspicion, logging begins |
| 3 | 150 | Alert threshold - email fires here |
| 4 | 1500 | Elevated sustained scan |
| 5 | 10000 | Heavy attack |

Level 3 at 150 packets is the practical alert threshold. A default Nmap scan sends 1000+ packets, so any real scan will cross it. Casual background probes hitting one or two ports won't.

Signatures were updated from the upstream source before restarting:

```bash
sudo psad --sig-update
```

PSAD restarted cleanly. The sendmail errors visible in the startup log at this stage were expected — postfix wasn't configured yet. PSAD runs regardless, it just can't send email until the relay is in place.

> 📹 **Recording:** [![Asciinema - Phase 2: Configure PSAD](https://asciinema.org/a/1168624.svg)](https://asciinema.org/a/1168624)

---

### Phase 3 - Postfix and Gmail SMTP Relay

Postfix was already partially installed as a PSAD dependency but had no `main.cf` — the service was inactive with the condition `ConditionPathExists=/etc/postfix/main.cf was not met`. Running `dpkg-reconfigure postfix` with "Internet Site" selected generated the base configuration and brought the service up.

`libsasl2-modules` was installed separately to enable SASL authentication, which is required for Gmail SMTP.

The Gmail relay configuration was added to `/etc/postfix/main.cf`:

```
relayhost = [smtp.gmail.com]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_security_level = encrypt
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
```

**First error hit:** duplicate entries. The `dpkg-reconfigure` step had already written a `relayhost =` line and an `smtp_tls_security_level` line to `main.cf`. Adding the relay config at the bottom created conflicts — postfix warned about overriding earlier entries on startup. Fixed by removing the original empty `relayhost =` and the conflicting `smtp_tls_security_level` line, keeping only the Gmail relay values.

**Second error:** typo in the test email address — `urielbytet` instead of `urielbyte`. Caught immediately, corrected on the next send.

The credentials file `/etc/postfix/sasl_passwd` was locked to `600` permissions and converted to a postfix hash database with `postmap`. Postfix cannot read the credentials without this step.

Test email confirmed delivered. Three emails arrived in the inbox — the failed attempts from before the duplicate fix were queued and delivered once postfix was correctly configured.

> 📹 **Recording:** [![Asciinema - Phase 3: Postfix and Gmail SMTP Relay](https://asciinema.org/a/1168739.svg)](https://asciinema.org/a/1168739) 

---

### Phase 4 - Integration and Clean Startup

PSAD was restarted after postfix was confirmed working. The sendmail errors present in earlier startups were gone. Startup log showed all signature imports completing cleanly and no mail errors.

```
imported p0f-based passive OS fingerprinting signatures
imported TOS-based passive OS fingerprinting signatures
imported auto_dl file, got 1 IP addrs/networks
imported original Snort rules in /etc/psad/snort_rules/ for reference info
imported 206 psad Snort signatures from /etc/psad/signatures
imported valid icmp types and codes
imported valid icmp6 types and codes
starting up psad version 2.4.6
```

> 📹 **Recording:** [![Asciinema - Phase 4: Integration and Clean Startup](https://asciinema.org/a/1168741.svg)](https://asciinema.org/a/1168741)

---

### Phase 5 - Live Trigger Test

Before the deliberate test ran, PSAD had already detected real external sources probing the server. Within minutes of the service starting, two IPs appeared in the attacker list — an AWS EC2 instance out of Ohio and a separate scanner — both targeting port 3389 (RDP) and port 22. Both matched the "MISC MS Terminal Server communication attempt" signature. Alert emails were sent. This was not part of the test. It was live background internet noise hitting the server during setup, and it confirmed the full pipeline was working before the Nmap scan even ran.

The deliberate test: Nmap SYN scan from the local VMware workstation against the public IP `35.206.67.167`. The local VM's public IP (`223.206.32.53`) is a shared Thai ISP address, so the scan traffic merged with existing detections from that IP rather than appearing as a separate source in PSAD output. UFW logging at `low` level captures only a sample of blocked packets, which meant PSAD's packet count for that IP climbed but attribution to the specific scan was indirect.

UFW logging was temporarily raised to `medium` to get full packet visibility for the test. At `medium`, PSAD detected the scan and fired the alert email — subject line confirmed "TCP flags: Nmap -sT or -sS", which is PSAD's signature match for a SYN stealth scan. Detection confirmed.

**Problem surfaced by medium logging:** PSAD flagged `10.128.0.2` — the server's own internal GCP network interface — as a scanner. At `medium` logging level, UFW logs outbound traffic too, including Tailscale keepalive packets and DNS queries going out through `ens4`. PSAD has no way to distinguish legitimate outbound traffic from scan traffic without explicit whitelisting. This produced 16 alert emails from the server's own normal activity.

Two fixes applied:

1. UFW logging reverted to `low` — only blocked inbound packets logged, outbound noise eliminated.
2. `10.128.0.2` added to `/etc/psad/auto_dl` with danger level `0` — PSAD will never alert on that IP regardless of logging level.

PSAD restarted to a clean state after both fixes.

> 📹 **Recording:** [![Asciinema - Phase 5: Live Trigger Test (Server)](https://asciinema.org/a/1168743.svg)](https://asciinema.org/a/1168743)
> 📹 **Recording:** [![Asciinema - Phase 5: Live Trigger Test (Attacker View - Local VM)](https://asciinema.org/a/1168744.svg)](https://asciinema.org/a/1168744)

---

### Phase 5.5 - Final Status Verification

UFW confirmed at `low` logging, default-deny inbound, SSH allowed only on `tailscale0`. PSAD running clean with zero accumulated detections — fresh state ready for production monitoring.

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
22 on tailscale0           ALLOW IN    Anywhere
22 (v6) on tailscale0      ALLOW IN    Anywhere (v6)
```

> 📹 **Recording:** [![Asciinema - Phase 5.5: Final Status](https://asciinema.org/a/1168745.svg)](https://asciinema.org/a/1168745)

---

## Real-World Detections

PSAD caught unsolicited reconnaissance from three separate sources within hours of going live. None of these were part of the controlled test. They arrived on their own.

| Source IP | Origin | Organization | Ports Probed | Action |
|---|---|---|---|---|
| `18.221.18.194` | Columbus, Ohio, US | Amazon AWS EC2 | 22, 3389 | Dropped by UFW |
| `103.1.210.25` | Hanoi, Vietnam | Viettel | 3389 | Dropped by UFW |
| `45.142.193.164` | Amsterdam, North Holland | SKYNET NETWORK LTD | 3389 | Dropped by UFW |

The AWS EC2 instance is a common pattern — rented cloud infrastructure running automated port sweeps. The Viettel IP out of Hanoi arrived after PSAD was fully configured and fired an alert with no manual trigger. Three countries, two major cloud and ISP networks, all blocked and logged automatically.

This is what the internet looks like from any public IP. The server has been running since the ZTNA deployment. The difference now is that every probe gets named, logged, and reported.

```json
// 18.221.18.194
{
  "ip": "18.221.18.194",
  "hostname": "ec2-18-221-18-194.us-east-2.compute.amazonaws.com",
  "city": "Columbus",
  "region": "Ohio",
  "country": "US",
  "org": "AS16509 Amazon.com, Inc."
}

// 103.1.210.25
{
  "ip": "103.1.210.25",
  "city": "Hanoi",
  "region": "Hanoi",
  "country": "VN",
  "org": "AS38731 Vietel - CHT Company Ltd"
}


// 45.142.193.164
{
  "ip": "45.142.193.164",
  "city": "Amsterdam",
  "region": "North Holland",
  "country": "NL",
  "org": "AS214295 SKYNET NETWORK LTD",
  "postal": "1012",
  "timezone": "Europe/Amsterdam",
}
```

> 📧 **Alert email — Postfix relay test:** [screenshot]

> 📧 **Alert email — PSAD Nmap detection:** [screenshot]

> 📧 **Alert email — Real-world detection (Viettel/Hanoi):** [screenshot]

---

## Command Reference (Full Session Log)

Commands marked `# [ERROR]` failed and are included because the fix that followed is instructive.

---

### Phase 1 - Install PSAD

```bash
# Install PSAD — pulls libnetaddr-ip-perl and libnet-ipv6addr-perl as dependencies
sudo apt install psad

# Verify service started and is enabled at boot
sudo systemctl status psad

# Confirm config directory structure
ls /etc/psad/
# Expected: archive  icmp6_types  icmp_types  ip_options  posf  protocols
#           psad.conf  signatures  snort_rule_dl  snort_rules  auto_dl  pf.os
```

---

### Phase 2 - Configure PSAD

```bash
# Open main config file
sudo vim /etc/psad/psad.conf

# Key values to set (use Ctrl+W in nano to search, or / in vim):
# EMAIL_ADDRESSES       urielbyte.alerts@gmail.com;
# ENABLE_AUTO_IDS       N;
# IPT_SYSLOG_FILE       /var/log/syslog;
# DANGER_LEVEL2         50;
# DANGER_LEVEL3         150;
# Leave DANGER_LEVEL1 (5), DANGER_LEVEL4 (1500), DANGER_LEVEL5 (10000) at defaults

# Verify all critical values saved correctly
sudo grep -E "EMAIL_ADDRESS|ENABLE_AUTO_IDS|IPT_SYSLOG_FILE|DANGER_LEVEL" /etc/psad/psad.conf | grep -v "^#"

# Update signatures from upstream
sudo psad --sig-update

# Restart to load new config and signatures
sudo systemctl restart psad
sudo systemctl status psad
```

---

### Phase 3 - Postfix and Gmail SMTP Relay

```bash
# Confirm postfix location
whereis postfix

# Install postfix (already present as PSAD dependency) and SASL auth modules
sudo apt install postfix libsasl2-modules

# postfix had no main.cf — run the configuration wizard
sudo dpkg-reconfigure postfix
# Select: 2 (Internet Site)
# System mail name: uriel-soc-ztna-lab-01
# All other prompts: accept defaults or leave blank

# Verify postfix started
sudo systemctl status postfix

# Create Gmail credentials file — enter address:app-password on one line
sudo nano /etc/postfix/sasl_passwd
# [smtp.gmail.com]:587    urielbyte.alerts@gmail.com:app-password-here

# Lock permissions — postfix refuses to read world-readable credential files
sudo chmod 600 /etc/postfix/sasl_passwd

# Convert to postfix hash database — required for lookup
sudo postmap /etc/postfix/sasl_passwd

# Add Gmail relay config to main.cf
sudo vim /etc/postfix/main.cf
# Add at bottom (remove any existing empty relayhost= and smtp_tls_security_level= lines first):
# relayhost = [smtp.gmail.com]:587
# smtp_sasl_auth_enable = yes
# smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
# smtp_sasl_security_options = noanonymous
# smtp_tls_security_level = encrypt
# smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt

# Reload postfix to apply changes
sudo systemctl reload postfix

# [ERROR] Typo in email address — 'urielbytet' not 'urielbyte'
echo "PSAD relay test" | sudo mail -s "Postfix test" urielbytet.alerts@gmail.com

# [ERROR] Duplicate entries in main.cf — warnings about overriding relayhost and tls level
# Fix: remove original empty relayhost= and smtp_tls_security_level=may lines from main.cf
sudo vim /etc/postfix/main.cf
sudo systemctl reload postfix

# Correct test send
echo "PSAD relay test from uriel-soc-ztna-lab-01" | sudo mail -s "Postfix Test" urielbyte.alerts@gmail.com
# Confirm email arrives in Gmail inbox before proceeding
```

---

### Phase 4 - Integration and Clean Startup

```bash
# Restart PSAD with postfix now working — sendmail errors should be gone
sudo systemctl restart psad
sudo systemctl status psad
```

---

### Phase 5 - Live Trigger Test

```bash
# --- On SERVER ---
# Watch syslog for UFW BLOCK entries in real time during the scan
sudo tail -f /var/log/syslog | grep UFW

# --- On LOCAL VM ---
# Run SYN scan against public IP to trigger PSAD
sudo nmap -Pn -sS 35.206.67.167

# Check local VM public IP — needed to identify scan source in PSAD output
curl ifconfig.me

# --- On SERVER ---
# Check PSAD detection status after scan completes
sudo psad -S

# UFW logging at 'low' only samples blocked packets — raise to medium for full visibility
sudo ufw logging medium
sudo systemctl restart psad

# Run second Nmap scan from local VM with medium logging active
# sudo nmap -Pn -sS 35.206.67.167  (run on local VM)

# Check updated PSAD status — scan source and email alert count should be visible
sudo psad -S

# Problem: medium logging causes PSAD to flag server's own outbound traffic (10.128.0.2)
# Fix 1: revert UFW logging
sudo ufw logging low

# Fix 2: whitelist server's internal GCP interface IP
sudo nano /etc/psad/auto_dl
# Add: 10.128.0.2    0;

# Restart PSAD to apply whitelist and reset detection state
sudo systemctl restart psad

# Verify clean state
sudo psad -S

# Lookup real-world scanner IPs for attribution
curl https://ipinfo.io/18.221.18.194
curl https://ipinfo.io/103.1.210.25
```

---

### Phase 5.5 - Final Status Verification

```bash
# Confirm UFW is active at low logging with correct rules
sudo ufw status verbose

# Confirm PSAD is running clean with no stale detections
sudo psad -S

# Confirm PSAD service status and auto_dl whitelist was loaded
sudo systemctl status psad
# Look for: "imported auto_dl file, got 1 IP addrs/networks"
```

---

## What PSAD Is and How It Works

### The Problem It Solves

A firewall blocks traffic. That is its entire job. When UFW drops a packet from an unknown IP, it writes a log entry and moves on — it does not try to understand whether that packet was part of a scan, a single probe, or a sustained attack. PSAD reads those same log entries and does the analysis the firewall doesn't.

The result is a detection layer sitting on top of the firewall. The firewall keeps blocking. PSAD keeps watching what the firewall blocks and building a picture of who is doing what.

---

### How It Reads the Firewall

PSAD does not inspect live network traffic. It reads log files. Specifically, it tails `/var/log/syslog` and looks for lines tagged `[UFW BLOCK]`. Every blocked packet UFW logs becomes input to PSAD's analysis.

This is why `IPT_SYSLOG_FILE` matters in the config — point it at the wrong file and PSAD reads nothing. It is also why UFW's logging level affects PSAD's visibility. At `low`, UFW logs a sample of blocked packets. At `medium`, it logs every blocked packet plus outbound traffic. PSAD's detection accuracy scales directly with how much UFW gives it to read.

---

### Danger Levels

PSAD assigns a danger level (1-5) to each source IP based on how many packets it has sent and whether those packets match known attack signatures. The levels are thresholds, not scores — once an IP crosses a threshold, it stays at that level until PSAD resets.

The alert email fires when an IP reaches `EMAIL_ALERT_DANGER_LEVEL`, which is set to 1 in this deployment. That means any detection triggers an email. In a higher-traffic environment you would raise this to reduce noise.

---

### Signatures

PSAD ships with a signature set derived from Snort rules. When a blocked packet matches a known pattern — RDP probes, Nmap scan fingerprints, specific protocol anomalies — PSAD names the match in its output. The "MISC MS Terminal Server communication attempt" detections in Phase 5 are real-world background internet noise matching that signature. The "Nmap -sT or -sS scan" match in the alert email is PSAD recognising the SYN scan flag pattern.

Signatures are updated with `psad --sig-update`. Running this periodically keeps detection current.

---

### Detection-Only vs Active Blocking

`ENABLE_AUTO_IDS N` keeps PSAD in detection-only mode. With auto-IDS enabled, PSAD would write iptables rules to block flagged IPs automatically. For a production environment that is powerful but risky — a false positive blocks legitimate traffic, and on a remote server with no out-of-band access that can mean locking yourself out. Detection-only is the right starting point: understand what PSAD flags before trusting it to act on those flags automatically.

---

### The Internal IP Problem

At `medium` logging level, UFW logs outbound packets too. PSAD sees those and has no built-in way to distinguish "server sending a DNS query" from "something scanning outbound targets." The `auto_dl` file is how you tell PSAD to ignore specific IPs. Setting an IP's danger level to `0` means PSAD will never escalate it regardless of packet count.

`10.128.0.2` is the GCP internal network interface address — the source IP on all outbound packets leaving through `ens4`. Whitelisting it stops PSAD from alerting on the server's own normal traffic.

**Further reading:** [PSAD Documentation](http://www.cipherdyne.org/psad/docs/) | [Postfix SASL Authentication](https://www.postfix.org/SASL_README.html)

---

## What I Actually Learned

This was my first time working with an intrusion detection tool of any kind. A few things that genuinely confused me during the process and how I now understand them.

**Why PSAD needs UFW logging turned on.**
PSAD doesn't sniff packets off the wire — it reads a text file. If UFW isn't writing blocked packets to syslog, PSAD has nothing to read. The firewall and the detector are completely separate processes. The firewall blocks, writes a log line, and forgets about it. PSAD reads that log line and remembers it. They only connect through the log file.

**Why the danger levels matter.**
At first the threshold numbers felt arbitrary. What clicked was thinking about it as a counter per IP. Every blocked packet from a single source increments that IP's counter. Background internet noise — random bots probing port 22 or 3389 — might send 3 or 4 packets and move on. A real port scanner sends hundreds or thousands. Setting the alert threshold at 150 means the random bot never triggers an alert but any actual scan does. The threshold is the line between "someone knocked on the door" and "someone is trying every window."

**Why the internal IP got flagged.**
When I raised UFW to medium logging to improve PSAD's visibility, the server started alerting on itself. That was confusing until I understood that `10.128.0.2` is the source address on every packet the server sends outbound — DNS queries, Tailscale keepalives, package update checks. At medium logging level UFW logs outbound traffic too, and PSAD sees all those outbound packets as a single IP sending a lot of traffic to many destinations. From PSAD's perspective that looks like scanning behavior. The `auto_dl` whitelist is how you tell PSAD "this IP is trusted, ignore it regardless of packet count."

**What detection-only actually means in practice.**
Detection-only sounds like a limitation but it's the right starting point. The alternative — auto-blocking — means PSAD writes iptables rules to ban flagged IPs automatically. On a remote server with no physical access, an aggressive false positive could lock out legitimate traffic including your own connection. Understanding what PSAD flags before trusting it to act on those flags is the correct order of operations. You earn the right to enable auto-blocking by first knowing what normal looks like.

**The shared IP problem.**
The local VM shares a public IP with other users on the same Thai ISP. When the deliberate Nmap test ran, PSAD couldn't tell that scan apart from background probes already coming from that address. This isn't a flaw in PSAD — it's a network reality. In a professional environment each source has a unique IP and attribution is clean. In a home lab on a shared ISP address, the signal gets mixed with noise. The same tool behaves differently depending on the network topology around it.

**What the real-world detections meant.**
The AWS EC2 scanner and the Viettel IP from Hanoi arriving unsolicited within hours of deployment was not surprising once you understand how the internet works — but it was the first time I'd seen it happen live on a server I own. Every public IP gets scanned constantly. The ZTNA stack was already blocking all of it. PSAD made it visible.

---

## Limitations

**Shared public IP.** The local VM and unrelated background scanners share the same Thai ISP public IP (`223.206.32.53`). PSAD cannot distinguish the deliberate test scan from background noise — it sees packet counts from that IP climbing but cannot attribute them to a specific source machine. This is a lab constraint, not a PSAD limitation.

**Detection depends on UFW logging level.** At `low`, UFW samples blocked packets. If a scan is fast enough or the logging window narrow enough, PSAD may not see enough packets to cross the threshold. `medium` gives full visibility but introduces outbound noise. `low` is the production choice here, with the understanding that very fast scans might slip under the threshold.

**Detection-only mode.** PSAD logs and alerts but takes no action. A confirmed scan source continues probing until it gives up or the operator manually intervenes. Active blocking would close that gap but introduces false-positive risk on a remote server.

**No log rotation for PSAD data.** PSAD writes to `/var/log/psad/` continuously. A logrotate configuration for PSAD's output files would keep disk usage in check over time.

**Email alert volume.** With `EMAIL_ALERT_DANGER_LEVEL` set to 1, every detection fires an email. On a public IP that sees constant background probing, this floods the inbox quickly — as demonstrated during the medium logging test. Raising the alert level to 2 or 3 would reduce volume while still catching meaningful scans.

---

## Project Retrospective

### What Worked

**Live detection before the test even ran.** PSAD flagged real external scanners targeting RDP and SSH within minutes of being configured. The alert pipeline was confirmed working against actual internet noise, not just the controlled test. That's more honest evidence than a clean lab environment would produce.

**Catching the medium logging problem during setup.** Temporarily raising UFW logging to test PSAD visibility surfaced the internal IP false-positive issue. Discovering it during setup rather than after deployment meant it got fixed and documented rather than becoming an unexplained alert flood later.

**auto_dl whitelist.** Adding `10.128.0.2` to the whitelist was a clean fix — one line, no firewall rule changes, no service restarts beyond PSAD itself. Worth knowing the mechanism exists for any future IP that generates noise.

---

### What Could Have Been Done Better

**No unique external test IP.** Because the local VM shares a public IP with background scanners, the live trigger test couldn't cleanly isolate the deliberate scan in PSAD output. A separate VPS with a dedicated IP would have produced cleaner evidence — a single source IP appearing in PSAD's attacker list with a packet count matching exactly one Nmap scan.

**Alert threshold needs tuning for production.** `EMAIL_ALERT_DANGER_LEVEL 1` is correct for initial setup and testing but will flood the inbox on a public IP. Raising it to `2` or `3` after a baseline observation period would keep alerts meaningful.

**No PSAD log rotation.** `/var/log/psad/` will grow without bound. A logrotate config should be added.

**Active blocking not implemented.** Detection-only is the right starting point, but the logical next step is testing auto-IDS in a controlled environment to understand its behavior before enabling it on a live server. That remains future work.

**Credential exposure in terminal recording.** The Gmail App Password was briefly visible in the Asciinema recording for Phase 3 during the `sasl_passwd` file creation. The password was immediately revoked and replaced. For future recordings involving credentials, write the file contents before starting the recording, or use a placeholder value during recording and substitute the real credential after the camera is off.

**Gmail suspension** The initial email alert volume from testing and the medium logging experiment triggered Gmail's abuse detection and temporarily suspended the account.

        *  *Note: The email evidence screenshots are unavailable due to account suspension during testing*

---

## Future Work

- [ ] Raise `EMAIL_ALERT_DANGER_LEVEL` to 2 or 3 after a baseline observation period to reduce alert noise
- [ ] Add logrotate configuration for `/var/log/psad/`
- [ ] Test and document auto-IDS (active blocking) mode in a controlled environment
- [ ] Explore wiring PSAD alerts into a SIEM rather than email for persistent correlation

---

## Contact

**LinkedIn:** [Supawat H.](https://www.linkedin.com/in/supawat-h-145371392)
**YouTube:** [@urielbyte](https://www.youtube.com/@urielbyte)

---

*Report authored by Supawat H. (uriel0byte).*
