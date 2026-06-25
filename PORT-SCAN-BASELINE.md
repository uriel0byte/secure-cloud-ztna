# Automated Port Scan Baseline

**Author:** Supawat H. (uriel0byte)  |  **Environment:** Google Cloud Platform  |  **Status:** Complete

---

## Summary

This project extends the ZTNA deployment with a scheduled, automated port scanning baseline. Two Nmap scans run nightly from the GCP server itself — one against the public IP to verify zero open ports from the internet, one against the Tailscale IP to verify only port 22 is reachable from inside the Tailnet. Results are timestamped, saved, and diffed against a clean baseline so any change in the attack surface gets flagged automatically.

The local VM scans captured in Phase 1 and Phase 2 serve a separate purpose: they represent the true external attacker perspective, scanning the server from outside with no prior knowledge of the host. Those are the reference baselines. The server-side automation handles continuous change detection over time.

---

## Objective

1. Establish a verified, documented baseline of the server's attack surface from two perspectives — external (public internet) and internal (Tailnet).
2. Automate recurring scans so any deviation from the baseline gets caught without manual intervention.
3. Confirm that the ZTNA controls built in the original deployment are actually holding, not just assumed to be holding.

---

## Tech Stack

| Layer | Tool |
|---|---|
| Port Scanner | Nmap 7.98 |
| Automation | Bash + cron |
| Diff Engine | GNU diff |
| Log Storage | Local filesystem (`~/port_scan_logs/`) |
| Platform | GCP e2-micro, Ubuntu 26.04 LTS (Minimal) |

---

## Scan Architecture

Two perspectives were captured deliberately.

**External (local VMware workstation → public GCP IP `35.206.67.167`):** This is the attacker's view. The scan originates from a machine with no prior relationship to the server, hitting the public IP directly. It has no knowledge of Tailscale and no access to the private mesh. What it sees — or doesn't see — is what any scanner on the internet would see.

**Internal (GCP server → Tailscale IP `100.114.172.16`):** This is the Tailnet view. The scan runs from the server against its own Tailscale interface, confirming what's reachable from inside the private mesh. Only authenticated Tailnet members can reach this address.

The server-side automation runs the internal scan and a self-scan of the public IP on a nightly cron schedule. The local VM scans are one-time reference captures.

---

## Deployment Log

### Phase 1 - External Baseline (Local VM)

Recorded from the local VMware Ubuntu workstation with Tailscale brought down to eliminate any Tailnet influence on the result.

Nmap requires root for SYN scans. Running without sudo produces an immediate error — documented below as it was hit in session.

The `-Pn` flag is required here. Without it, Nmap sends an ICMP ping first to check if the host is alive. The ICMP blackhole built in Phase 2 of the ZTNA deployment drops that ping silently, so Nmap concludes the host is down and skips the scan entirely. `-Pn` forces it to scan regardless.

> 📹 **Recording:** [![Asciinema - Phase 1: External Baseline Scan (Local VM)](https://asciinema.org/a/1164927.svg)](https://asciinema.org/a/1164927)

> 📹 **Recording:** [![Asciinema - Phase 1: External Baseline Scan (GCP Server)](https://asciinema.org/a/1164964.svg)](https://asciinema.org/a/1164964)

---

### Phase 2 - Internal Baseline (GCP Server)

SSH into the server via Tailscale, then scan the Tailscale IP from the server itself.

The internal scan completed in 8.46 seconds compared to ~201 seconds for the external scan. The difference is latency — the external scan is probing across the public internet, waiting for timeouts on 1000 filtered ports. The internal scan hits the loopback-adjacent Tailscale interface with sub-millisecond response times.

One result difference worth noting between the local VM internal scan and the GCP server internal scan: the local VM reported `999 closed tcp ports (reset)`, while the server reported `999 filtered tcp ports (no-response)`. Both are correct. From the local VM scanning over Tailscale, the firewall drops the packets silently — no response. From the server scanning itself, closed ports respond with a TCP RST because the traffic never hits the firewall. Different perspectives, different responses, same actual state.

> 📹 **Recording:** [![Asciinema - Phase 2: Internal Baseline Scan](https://asciinema.org/a/1164965.svg)](https://asciinema.org/a/1164965)

---

### Phase 3 - Automation Script

The script runs both scans, saves timestamped output files, and diffs them against the baselines. A first version was written and tested manually before cron was touched.

**First error hit:** running the script without sudo. SYN scans require raw socket access, which requires root. The script needs to run as root either via sudo or the root crontab — this is why the cron job is scheduled in the root crontab rather than the user crontab.

**Second problem caught during testing:** the initial diff output was noisy. Every run produced diff entries for the timestamp in the comment header, the output file path in the command line, scan duration, and latency values. None of those are meaningful — a port opening or closing is what matters. The fix was to strip those lines before diffing using `grep -v` to exclude lines matching `^#`, `latency`, and `scanned in`. After that, a clean run with no port changes produces an empty diff, which is exactly the right behavior.

The diff log is append-only. Each run adds a timestamped block. If the diff section is empty, nothing changed. If lines appear, something did.

> 📹 **Recording:** [![Asciinema - Phase 3: Automation Script](https://asciinema.org/a/1164967.svg)](https://asciinema.org/a/1164967)

> 📹 **Recording:** [![Asciinema - Phase 3: Update Automation Script](https://asciinema.org/a/1164971.svg)](https://asciinema.org/a/1164971)

---

### Phase 4 - Cron Scheduling and Live Verification

The script was added to the root crontab alongside the existing backup job:

```
0 2 * * * /home/uriel/infrastructure_backups/backup_configs.sh >> /home/uriel/infrastructure_backups/cron_debug.log 2>&1
0 3 * * * /home/uriel/port_scan_logs/scan_and_diff.sh >> /home/uriel/port_scan_logs/cron_debug.log 2>&1
```

The scan fires one hour after the backup. Both jobs log stderr and stdout to their own debug logs so silent failures don't go unnoticed.

Rather than waiting until 03:00 to confirm the job fired, the crontab was temporarily set to fire 2 minutes ahead of the current server time as a live test. The result confirmed cron executed the script on schedule — `cron_debug.log` was written to and a new timestamped scan file appeared in `~/port_scan_logs/`. The crontab was then reset to `0 3 * * *`.

> 📹 **Recording:** [![Asciinema - Phase 4: Cron Scheduling](https://asciinema.org/a/1164972.svg)](https://asciinema.org/a/1164972)

> 📹 **Recording:** [![Asciinema - Phase 4: Update Cron Scheduling](https://asciinema.org/a/1165006.svg)](https://asciinema.org/a/1165006)

---

## Command Reference (Full Session Log)

Commands marked `# [ERROR]` failed and are included because the fix that followed is instructive.

---

### Phase 1 - External Scan (Local VM)

```bash
# Verify Nmap is installed
nmap --version

# Bring Tailscale down to remove mesh network influence from the external scan
sudo tailscale down

# Create the log directory
mkdir -p ~/port_scan_logs

# [ERROR] SYN scan requires root — running without sudo fails immediately
nmap -Pn -sS -oN ~/port_scan_logs/external_baseline.txt 35.206.67.167

# Correct: run with sudo
# -Pn: skip host discovery — required because ICMP blackhole makes host appear down
# -sS: SYN stealth scan — sends SYN, reads response, never completes the handshake
# -oN: save output in normal format for diff comparison
sudo nmap -Pn -sS -oN ~/port_scan_logs/external_baseline.txt 35.206.67.167

# Bring Tailscale back up before Phase 2
sudo tailscale up
```

---

### Phase 1 - External Scan (GCP Server)

```bash
# SSH into the server via Tailscale
ssh 100.114.172.16

# Create the log directory on the server
mkdir ~/port_scan_logs
cd port_scan_logs/

# Nmap not installed on Ubuntu Minimal
nmap --version
# -bash: nmap: command not found

# Install Nmap
sudo apt install nmap

# Verify installation
nmap --version

# Run external baseline scan from the server against its own public IP
sudo nmap -Pn -sS -oN external_baseline.txt 35.206.67.167

# Confirm output was saved
cat external_baseline.txt
```

---

### Phase 2 - Internal Scan (GCP Server)

```bash
# Run internal baseline scan against the Tailscale IP
sudo nmap -Pn -sS -oN internal_baseline.txt 100.114.172.16

# Confirm both baseline files exist
ls
cat internal_baseline.txt
```

---

### Phase 3 - Automation Script

```bash
# Write the scan and diff script
nano ~/port_scan_logs/scan_and_diff.sh
```

```bash
#!/bin/bash

LOGDIR="/home/uriel/port_scan_logs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

EXTERNAL_NEW="$LOGDIR/external_$TIMESTAMP.txt"
INTERNAL_NEW="$LOGDIR/internal_$TIMESTAMP.txt"

EXTERNAL_BASELINE="$LOGDIR/external_baseline.txt"
INTERNAL_BASELINE="$LOGDIR/internal_baseline.txt"

nmap -Pn -sS -oN "$EXTERNAL_NEW" 35.206.67.167
nmap -Pn -sS -oN "$INTERNAL_NEW" 100.114.172.16

echo "=== EXTERNAL DIFF ===" >> "$LOGDIR/diff_log.txt"
echo "Timestamp: $TIMESTAMP" >> "$LOGDIR/diff_log.txt"
grep -v "^#\|latency\|scanned in" "$EXTERNAL_BASELINE" > /tmp/ext_clean_base.txt
grep -v "^#\|latency\|scanned in" "$EXTERNAL_NEW" > /tmp/ext_clean_new.txt
diff /tmp/ext_clean_base.txt /tmp/ext_clean_new.txt >> "$LOGDIR/diff_log.txt"

echo "=== INTERNAL DIFF ===" >> "$LOGDIR/diff_log.txt"
grep -v "^#\|latency\|scanned in" "$INTERNAL_BASELINE" > /tmp/int_clean_base.txt
grep -v "^#\|latency\|scanned in" "$INTERNAL_NEW" > /tmp/int_clean_new.txt
diff /tmp/int_clean_base.txt /tmp/int_clean_new.txt >> "$LOGDIR/diff_log.txt"

echo "---" >> "$LOGDIR/diff_log.txt"
```

```bash
# Make the script executable
chmod +x ~/port_scan_logs/scan_and_diff.sh

# [ERROR] Running without sudo — SYN scan fails, grep has no files to compare
./scan_and_diff.sh

# Correct: run with sudo
sudo ~/port_scan_logs/scan_and_diff.sh

# Inspect the diff log — clean run should show only timestamp headers, no port changes
cat ~/port_scan_logs/diff_log.txt

# Clear the log before production use — the failed run left a noisy entry
> ~/port_scan_logs/diff_log.txt
```

---

### Phase 4 - Cron Scheduling

```bash
# Check current server time to set the test cron time correctly
date

# Open root crontab
sudo crontab -e
# Add the scan job with output logging:
# 0 3 * * * /home/uriel/port_scan_logs/scan_and_diff.sh >> /home/uriel/port_scan_logs/cron_debug.log 2>&1

# Verify both jobs are registered
sudo crontab -l

# For live test: temporarily change the scan job to fire 2 minutes from now
# then watch for new files to appear
watch ls -lt ~/port_scan_logs/

# After confirming cron fired, reset to nightly schedule
sudo crontab -e
# Restore: 0 3 * * * /home/uriel/port_scan_logs/scan_and_diff.sh >> /home/uriel/port_scan_logs/cron_debug.log 2>&1

# Confirm the final crontab state
sudo crontab -l
```

---

## Baseline Results

### External Scan - Public IP (35.206.67.167)

Scanned from both the local VMware workstation and the GCP server. Results identical.

```
All 1000 scanned ports on 167.67.206.35.bc.googleusercontent.com (35.206.67.167) are in ignored states.
Not shown: 1000 filtered tcp ports (no-response)
```

1000 ports scanned, zero open, zero response. The UFW default-deny and ICMP blackhole are holding. From the internet, this server does not exist.

---

### Internal Scan - Tailscale IP (100.114.172.16)

```
Not shown: 999 filtered tcp ports (no-response)
PORT   STATE SERVICE
22/tcp open  ssh
```

Port 22 open, 999 filtered. SSH is reachable from inside the Tailnet and nothing else is exposed. The Tailscale hostname `uriel-soc-ztna-lab-01.tailf09853.ts.net` resolved correctly, confirming the mesh routing is working.

---

### Diff Log - Clean State

A clean run with no port changes produces this in `diff_log.txt`:

```
=== EXTERNAL DIFF ===
Timestamp: 20260530_172654
=== INTERNAL DIFF ===
---
```

Empty diff sections mean the current scan matches the baseline exactly. If a port appeared or disappeared, the changed lines would show up between the headers.

---

## Post-Deployment Detection

On June 1st at 03:00, the nightly scan caught a real change. Port 25 (SMTP) had opened on the Tailscale interface, introduced by Postfix during the PSAD deployment two days prior. The diff log flagged it automatically:

```
=== EXTERNAL DIFF ===                                                                                               
Timestamp: 20260601_030001                                                                                          
=== INTERNAL DIFF ===                                                                                               
2c2                                                                                                                 
< Not shown: 999 filtered tcp ports (no-response)                                                                   
---                                                                                                                 
> Not shown: 998 filtered tcp ports (no-response)                                                                   
4a5                                                                                                                 
> 25/tcp open  smtp   
```

Postfix listens on port 25 by default to accept inbound mail. For a relay-only configuration — where the server only sends outbound alerts — this is unnecessary exposure. The fix was setting `inet_interfaces = loopback-only` in `/etc/postfix/main.cf`, restricting Postfix to localhost only. A follow-up scan confirmed port 25 closed. The internal baseline was updated to reflect the corrected state.

This is the two projects working together exactly as intended — the PSAD deployment introduced an unintended service, and the port scan baseline caught it the same night.

> 📹 **Recording:** [![Asciinema - Post-Deployment: Port 25 Detection and Fix](https://asciinema.org/a/1172668.svg)](https://asciinema.org/a/1172668)

---

## Script Breakdown

This section explains what the automation script does, line by line. Skip it if you already know Bash. Come back to it if the diff output ever looks wrong and you need to trace why.

---

### Variables

```bash
LOGDIR="/home/uriel/port_scan_logs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
```

`LOGDIR` is just a shorthand so the path doesn't get typed out on every line. `TIMESTAMP` captures the current date and time the moment the script runs — format is `YYYYMMDD_HHMMSS`, so a run at 03:00 on June 1st produces `20260601_030000`. Every run gets its own timestamp, which is how the output files stay separate and sortable.

```bash
EXTERNAL_NEW="$LOGDIR/external_$TIMESTAMP.txt"
INTERNAL_NEW="$LOGDIR/internal_$TIMESTAMP.txt"
```

These are the paths for the new scan files. They don't exist yet — the script creates them when Nmap runs.

```bash
EXTERNAL_BASELINE="$LOGDIR/external_baseline.txt"
INTERNAL_BASELINE="$LOGDIR/internal_baseline.txt"
```

These point to the fixed baseline files captured manually in Phase 1 and Phase 2. They never change unless you intentionally update them.

---

### Scans

```bash
nmap -Pn -sS -oN "$EXTERNAL_NEW" 35.206.67.167
nmap -Pn -sS -oN "$INTERNAL_NEW" 100.114.172.16
```

Two Nmap scans, one after the other. The external scan runs first and takes around 200 seconds because it's waiting for timeouts on 1000 filtered ports over the public internet. The internal scan runs after and finishes in under 10 seconds. `-oN` writes the output to the timestamped file paths defined above.

---

### Diff Logic

```bash
echo "=== EXTERNAL DIFF ===" >> "$LOGDIR/diff_log.txt"
echo "Timestamp: $TIMESTAMP" >> "$LOGDIR/diff_log.txt"
```

These two lines write a header into the diff log before each comparison. The `>>` operator appends to the file rather than overwriting it, so the log builds up a history of every run.

```bash
grep -v "^#\|latency\|scanned in" "$EXTERNAL_BASELINE" > /tmp/ext_clean_base.txt
grep -v "^#\|latency\|scanned in" "$EXTERNAL_NEW" > /tmp/ext_clean_new.txt
```

This is the noise filter. Raw Nmap output includes lines that change on every run regardless of whether the network state changed — the timestamp in the comment header, the full command path, scan duration, and latency. Diffing the raw files would flag those every time and make the log useless.

`grep -v` inverts the match — it prints every line that does NOT match the pattern. The pattern `^#\|latency\|scanned in` breaks down as: lines starting with `#` (the comment lines), or lines containing `latency`, or lines containing `scanned in`. Those get stripped. What's left is the actual port data.

The cleaned output goes to `/tmp/` as temporary files. They only exist long enough to run the diff.

```bash
diff /tmp/ext_clean_base.txt /tmp/ext_clean_new.txt >> "$LOGDIR/diff_log.txt"
```

`diff` compares the two cleaned files line by line. If they're identical, it outputs nothing — the diff section in the log stays empty, which means no change. If a line differs, `diff` shows it with `<` for the baseline and `>` for the new scan. A port appearing would show as a `>` line. A port disappearing would show as a `<` line.

The same block runs again for the internal scan with the `int_` prefixed files.

```bash
echo "---" >> "$LOGDIR/diff_log.txt"
```

A separator line at the end of each run. Makes the log readable when multiple runs stack up.

---

### Why /tmp/ and Not the Log Directory

The cleaned files are written to `/tmp/` rather than `~/port_scan_logs/` because they're throwaway intermediates. They serve no purpose after the diff runs. Writing them to the log directory would fill it with files that aren't scan results and aren't worth keeping. `/tmp/` is cleared on reboot automatically.

---

### What a Detected Change Looks Like

If port 80 opened on the external IP, the diff section would look like this:

```
=== EXTERNAL DIFF ===
Timestamp: 20260601_030000
2a3
> 80/tcp open  http
=== INTERNAL DIFF ===
---
```

The `>` means that line exists in the new scan but not the baseline. That's the signal to investigate.

---

## Limitations

**Self-scan perspective.** The automated scans run from the server against itself. This is not the same as an external attacker scanning from the internet. The server has Tailscale running and knows its own interfaces — the scan result reflects internal visibility, not external exposure. The local VM baseline captures the true external perspective, but it doesn't run on a schedule.

**Diff log grows indefinitely.** The log is append-only with no rotation. Over months of daily runs it will get large. Addressed in the log rotation project, logrotate now handles `diff_log.txt` at a 1MB threshold. See [LOG-ROTATION.md](./LOG-ROTATION.md).

**No alerting.** The script detects changes and logs them. It does not send an alert. A port change at 03:00 sits in `diff_log.txt` until someone manually checks it. Wiring this to an email notification or a SIEM would close that gap.

**Scan files accumulate.** Every run creates two new timestamped files. Addressed in the log rotation project a `find`-based cron job now deletes scan file pairs older than 30 days. See [LOG-ROTATION.md](./LOG-ROTATION.md).

---

## Project Retrospective

### What Worked

**Baseline separation.** Keeping the local VM scans as the external attacker-perspective reference and running the server-side automation separately was the right call. They answer different questions. The local VM scan answers "what does an attacker on the internet see?" The server automation answers "has anything changed since yesterday?"

**Noise filtering.** Catching the noisy diff output during manual testing rather than after cron started running saved a log full of false positives. The `grep -v` filter makes the diff output actually actionable.

**Live cron verification.** Testing the cron job by temporarily setting it 2 minutes ahead rather than waiting until 03:00 confirmed the automation was working before the session ended. A scheduled job that silently fails is worse than no scheduled job — at least with no job you know you have a gap.

**Cross-project detection.** The nightly scan caught port 25 opening on the Tailscale interface after Postfix was deployed for PSAD. The exposure was introduced and detected within 24 hours — without manual intervention. That is the whole point of automated baseline monitoring.

---

### What Could Have Been Done Better

**No external scheduled scan.** The automated scans run from the server against itself. A proper external baseline would require a second machine — a separate VPS or a service like a scheduled scan from an external IP — to scan the GCP server from the internet on a schedule. That would give a continuous external perspective, not just a one-time reference capture.

**No alerting.** Changes sit in `diff_log.txt` until manually reviewed. For a production environment, this would need to pipe into a notification system.

**No log rotation.** The diff log and the timestamped scan files will accumulate without bound. A cron job to prune files older than 30 days would keep the directory manageable.

---

## Contact

**LinkedIn:** [Supawat H.](https://www.linkedin.com/in/supawat-h-145371392)  
**YouTube:** [@urielbyte](https://www.youtube.com/@urielbyte)

---

*By Supawat H. (uriel0byte).*
