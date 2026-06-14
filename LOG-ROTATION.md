# Log Rotation

**Author:** Supawat H. (uriel0byte)  |  **Environment:** Google Cloud Platform  |  **Status:** Complete

---

## Summary

Three months of nightly automation had no cleanup logic behind it. `~/port_scan_logs/` was accumulating a scan file pair every night with no upper bound. `/var/log/psad/` was building a new subdirectory per detected IP - over 150 entries within two weeks of PSAD going live. `~/infrastructure_backups/` was holding every config snapshot since May 30th.

This project closes that gap. `diff_log.txt` and `cron_debug.log` get rotated by logrotate at 1MB with four archived copies before deletion. The timestamped scan file pairs, PSAD IP subdirectories, and backup directories get cleaned by `find`-based cron jobs on a 30-day retention window.

---

## Objective

1. Configure logrotate to rotate `diff_log.txt` and `cron_debug.log` in `~/port_scan_logs/` at 1MB, keeping four copies before deletion.
2. Add nightly `find` cron jobs to delete timestamped scan file pairs and config backup directories older than 30 days.
3. Handle `/var/log/psad/` per-IP subdirectories with the same `find` approach since logrotate is not suited to directory trees.

---

## Tech Stack

| Layer | Tool |
|---|---|
| Log File Rotation | logrotate |
| Directory Cleanup | GNU find + cron |
| Config Location | `/etc/logrotate.d/ztna_log` |
| Platform | GCP e2-micro, Ubuntu 26.04 LTS (Minimal) |

---

## Relationship to Previous Projects

Both prior projects noted the cleanup gap explicitly. PORT-SCAN-BASELINE.md listed it under Limitations: the diff log and timestamped scan files accumulate without bound. PSAD-DEPLOYMENT.md listed it twice - once under Limitations, once under What Could Have Been Done Better. This project closes both.

The PSAD limitation is worth noting separately: `/var/log/psad/` doesn't contain log files in the usual sense. PSAD creates a subdirectory per attacker IP and writes detection data inside it. That structure rules out logrotate entirely. A `find` job targeting one level of subdirectories is the right tool for it.

---

## Pre-Execution Recon

Before writing anything, the state of all three targets was checked.

`/etc/logrotate.d/psad` didn't exist. PSAD's package doesn't ship a logrotate config on Ubuntu 26.04 Minimal, so there was nothing to patch or conflict with.

`~/port_scan_logs/` had 15 external and 15 internal timestamped scan file pairs dating back to May 31st, `diff_log.txt` at 975 bytes across 52 lines, and `cron_debug.log` at 10KB - both owned by root after being written by the root crontab. The ownership mattered later.

`~/infrastructure_backups/` had 16 config backup directories from May 30th onward, all 4KB each, totalling 72K.

`/var/log/psad/` had over 150 per-IP subdirectories totalling 1.9MB, oldest dating to June 3rd.

No recording for the recon phase - read-only commands, nothing to build.

---

## Deployment Log

### Phase 1 - logrotate Config

> 📹 **Recording:** [![Asciinema - Phase 1: logrotate Config](https://asciinema.org/a/1249626.svg)](https://asciinema.org/a/1249626)

The config was written to `/etc/logrotate.d/ztna_log` covering both `diff_log.txt` and `cron_debug.log`.

**First error:** the dry run failed on both files with an insecure permissions error.

```
error: skipping "/home/uriel/port_scan_logs/diff_log.txt" because parent directory has
insecure permissions (It's world writable or writable by group which is not "root") Set
"su" directive in config file to tell logrotate which user/group should be used for rotation.
```

logrotate runs as root by default. When the target files live in a home directory, it doesn't know which user should be responsible for the rotation and refuses to touch them. The fix was adding `su uriel uriel` as the first directive inside the config block.

The second dry run passed cleanly - logrotate switched euid from 0 to 1002, read both files, and correctly reported neither needed rotating yet since both were well below 1MB.

```
switching euid from 0 to 1002 and egid from 0 to 1003 (pid 263947)
considering log /home/uriel/port_scan_logs/diff_log.txt
  log does not need rotating (log size is below the 'size' threshold)
considering log /home/uriel/port_scan_logs/cron_debug.log
  log does not need rotating (log size is below the 'size' threshold)
```

logrotate is already wired into `/etc/cron.daily/` on Ubuntu - confirmed with `ls /etc/cron.daily/ | grep logrotate`. No additional scheduling needed.

---

### Phase 2 - Cron Cleanup Jobs

> 📹 **Recording:** [![Asciinema - Phase 2: Cron Cleanup Jobs](https://asciinema.org/a/1249630.svg)](https://asciinema.org/a/1249630)

Four `find`-based cleanup jobs were added to the root crontab, all scheduled at 04:00 - one hour after the nightly scan script finishes at 03:00.

```
0 4 * * * find /home/uriel/port_scan_logs -name "external_*.txt" -mtime +30 -delete
0 4 * * * find /home/uriel/port_scan_logs -name "internal_*.txt" -mtime +30 -delete
0 4 * * * find /var/log/psad -mindepth 1 -maxdepth 1 -type d -mtime +30 -exec rm -rf {} +
0 4 * * * find /home/uriel/infrastructure_backups -mindepth 1 -maxdepth 1 -type d -mtime +30 -exec rm -rf {} +
```

---

### Phase 3 - Force Test

> 📹 **Recording:** [![Asciinema - Phase 3: Force Test](https://asciinema.org/a/1249631.svg)](https://asciinema.org/a/1249631)

A forced rotation confirmed the config works outside dry-run mode.

**Error before the force test ran:** `cron_debug.log` was owned by root. The `su uriel uriel` directive switches logrotate to the `uriel` user context - but `uriel` can't write to a root-owned file. The force rotation failed immediately.

```
error: error opening /home/uriel/port_scan_logs/cron_debug.log: Permission denied
```

The fix was a `chown` to transfer ownership:

```bash
sudo chown uriel:uriel port_scan_logs/cron_debug.log
```

The force rotation succeeded after that. Both files truncated to 0 bytes with the original content moved to `.1` archives.

```
-rw-r--r-- 1 uriel uriel    0 Jun 14 05:35 cron_debug.log
-rw-r--r-- 1 uriel uriel  11K Jun 14 05:35 cron_debug.log.1
-rw-rw-r-- 1 uriel uriel    0 Jun 14 05:33 diff_log.txt
-rw-rw-r-- 1 uriel uriel  975 Jun 14 05:33 diff_log.txt.1
```

`sudo cat /var/lib/logrotate/status | grep ztna` returned nothing after the forced run. logrotate only writes to the state file during scheduled runs - `--force` bypasses the state check and doesn't update it afterward. It will populate on the first daily cron execution.

---

## Command Reference (Full Session Log)

Commands marked `# [ERROR]` failed and are included because the fix that followed is instructive.

---

### Pre-Execution Recon

```bash
# Check if PSAD ships a logrotate config
cat /etc/logrotate.d/psad
# cat: /etc/logrotate.d/psad: No such file or directory

# Inspect current scan log directory
ls -ls port_scan_logs/

# Check diff log size and line count
wc -l ~/port_scan_logs/diff_log.txt
wc ~/port_scan_logs/diff_log.txt

# Inspect backup directory
ls -lh infrastructure_backups/
```

---

### Phase 1 - logrotate Config

```bash
# Check existing logrotate configs for reference
ls /etc/logrotate.d/

# Write the config
sudo vim /etc/logrotate.d/ztna_log

# Verify it wrote correctly
cat /etc/logrotate.d/ztna_log

# Dry run - first attempt, before su directive
sudo logrotate --debug /etc/logrotate.d/ztna_log
# [ERROR] insecure permissions error on both files - parent directory not root-owned
# Fix: add 'su uriel uriel' as first directive inside the config block

# Edit to add the su directive
sudo vim /etc/logrotate.d/ztna_log

# Dry run - second attempt after fix
sudo logrotate --debug /etc/logrotate.d/ztna_log
# Passes - euid switches to 1002, both files below threshold, no rotation needed

# Confirm logrotate is in the daily cron pipeline
ls /etc/cron.daily/ | grep logrotate

# Check state file - returns nothing before first scheduled run, expected
sudo cat /var/lib/logrotate/status | grep ztna
```

---

### Phase 2 - Cron Cleanup Jobs

```bash
# Review scan log directory before adding cleanup
ls port_scan_logs/

# Review PSAD directory - shows per-IP subdirectories
ls /var/log/psad

# Open root crontab to add cleanup jobs
sudo crontab -e

# Verify all six jobs are present
sudo crontab -l
```

---

### Phase 3 - Force Test

```bash
# Force rotation to confirm the config works outside dry-run mode
sudo logrotate --force /etc/logrotate.d/ztna_log
# [ERROR] Permission denied on cron_debug.log - file is root-owned, su directive runs as uriel

# Fix: transfer ownership to uriel
sudo chown uriel:uriel port_scan_logs/cron_debug.log

# Re-run force rotation
sudo logrotate --force /etc/logrotate.d/ztna_log

# Verify rotation results - both files 0 bytes, .1 archives present
ls -lh port_scan_logs/

# State file check - returns nothing after --force, expected
sudo cat /var/lib/logrotate/status | grep ztna
```

---

## What Log Rotation Is and How It Works

### The Problem It Solves

Log files and automated outputs grow without bound if nothing cleans them up. `diff_log.txt` gets one new block appended every night. The scan script creates two new files per run. PSAD adds a new subdirectory every time it detects a new source IP. None of those processes have any concept of "this is getting too large" - that's the operator's job.

Log rotation is how you handle that systematically. The two tools used here approach the problem differently because the targets are shaped differently.

---

### logrotate - For Single Growing Files

logrotate reads config files in `/etc/logrotate.d/` and handles rotation on a trigger - file size, time, or both. On Ubuntu it runs daily via a script in `/etc/cron.daily/`. Once the config is written, rotation happens automatically.

When logrotate rotates a file it renames the current file to `filename.1`, starts a fresh empty `filename` in its place, and whatever was writing to that file continues without interruption. On the next rotation, `.1` becomes `.2`, up to the `rotate N` limit - after which the oldest copy gets deleted.

**Why `copytruncate`:**
The nightly scan cron job writes to `diff_log.txt` by redirecting output to that path. The default logrotate behavior - rename the file, create a new one - works correctly when a long-running process holds an open file descriptor and gets signaled to reopen it. A cron job doesn't work that way. `copytruncate` avoids the issue entirely: it copies the file's content to the archive, then truncates the original to zero bytes in place. The file stays at the same path. The cron job keeps writing to the same filename. Nothing needs to know rotation happened.

**Why the `su` directive is required:**
logrotate runs as root. When the target files live in a home directory, the parent directory isn't root-controlled, and logrotate treats that as an unresolved security question - it doesn't know which user should own the rotation. `su uriel uriel` tells it explicitly. Without that directive, logrotate skips both files and logs a permissions error.

---

### `find` with Cleanup Flags - For Directories and File Collections

logrotate works well on a single file that grows over time. It doesn't fit timestamped file pairs or directory trees, where each entry is a discrete artifact rather than an accumulating log.

**`-mtime +30`** - matches files or directories modified more than 30 days ago. The `+` means strictly greater than - `-mtime +30` matches 31 days or older, not exactly 30.

**`-name "external_*.txt"`** - glob pattern match on filename. Only files matching the timestamped pattern get touched. `external_baseline.txt` and `internal_baseline.txt` don't match and are never deleted.

**`-mindepth 1 -maxdepth 1`** - scopes the search to exactly one level below the target directory. `-mindepth 1` excludes the starting directory itself. `-maxdepth 1` stops recursion into subdirectories. For `/var/log/psad/` this means the job only sees IP directories, not the files inside them and not the parent. Without these flags, `find` would eventually try to delete `/var/log/psad/` itself if it aged out.

**`-delete` vs `-exec rm -rf {} +`** - `-delete` removes matching files directly, but it only works on files. Directories need `rm -rf`. The `{}` gets replaced by each matched path, and `+` batches them into a single `rm` call rather than invoking `rm` once per result.

**Why 04:00:**
The backup script runs at 02:00. The scan script runs at 03:00. Cleanup at 04:00 runs after both finish, so no job deletes a file that's still being written.

---

## What I Actually Learned

**Why logrotate refused to touch the files initially.**
The error message said "insecure permissions" but what it actually meant was that logrotate couldn't figure out which user should be responsible for the rotation. The home directory is accessible by multiple users - logrotate won't make assumptions about who should own that operation. The `su` directive is how you resolve it. Without it, logrotate treats the ambiguity as a reason to do nothing.

**Why `cron_debug.log` was root-owned.**
The scan script runs in the root crontab. Any file it creates or writes to inherits root's identity unless the script explicitly changes it. `cron_debug.log` is created by output redirection from the cron job - that write operation runs as root, so the file ends up root-owned even though it lives in my home directory. When logrotate switched to the `uriel` user context via the `su` directive, it hit a wall: `uriel` doesn't have write access to a root-owned file. The `chown` fixed the mismatch, but it also surfaced something worth knowing going forward - a file's location doesn't determine its owner, the process that created it does.

**Why `-mindepth 1 -maxdepth 1` matters.**
Without those flags, `find /var/log/psad -type d -mtime +30 -exec rm -rf {} +` would include `/var/log/psad` itself in the results if it ever matched the age criteria, and `rm -rf` would delete the entire directory. `-mindepth 1` excludes the starting point. `-maxdepth 1` prevents recursing into subdirectories and matching files inside the IP directories as separate deletion targets. Together they scope the operation to exactly the right layer - the IP directories themselves, nothing above or below.

**Why `--force` doesn't update the state file.**
logrotate tracks what it has rotated and when in `/var/lib/logrotate/status`. It uses that to decide whether a file needs rotating on the next run. `--force` bypasses that check - it rotates regardless of what the state file says. The tradeoff is that it also skips writing back to the state file afterward. On the next scheduled run, logrotate will see no prior rotation for the ztna config and check the files against the size threshold normally. Since both files were truncated to zero, they won't trigger again until they grow back to 1MB.

**The difference between logrotate and `find` for this use case.**
logrotate is built for a specific pattern: one file that accumulates content over time and cycles through a set of numbered archives. The timestamped scan files don't fit that pattern - each one is a complete, standalone snapshot, not an older version of the current file. `find` with `-mtime` treats each file as an independent artifact with its own age, which is the right model for outputs that are created once and never modified.

---

## Limitations

**The state file doesn't populate until the first scheduled run.** `sudo cat /var/lib/logrotate/status | grep ztna` returns nothing until logrotate runs via cron for the first time. Not a problem, but there's no written confirmation in the state file until the next daily execution.

**Cleanup cron jobs have no logging.** The four `find` jobs run silently. If a job fails - permissions error, filesystem issue - there's no output anywhere to indicate it. The existing scan and backup jobs both redirect stderr to debug logs. The cleanup jobs don't.

**30-day retention is a lab choice, not a baseline.** In a real environment, retention periods are usually dictated by policy or compliance requirements. 30 days was chosen as a reasonable starting point for a 30GB disk. Both the `find` jobs and the logrotate `rotate` count are easy to adjust.

**Baseline files are not covered by rotation.** `external_baseline.txt` and `internal_baseline.txt` are correctly excluded from the `find` cleanup - the glob patterns only match timestamped filenames. But they also have no backup outside this disk. If the instance is lost, the baselines go with it.

**No real deletion confirmed yet.** The logrotate rotation was force-tested and confirmed working. The `find` cleanup jobs were verified by reading the crontab, but no actual deletion has run - nothing in any of the directories is old enough to match `-mtime +30` yet. The first real deletion will happen on July 14th when the May 31st scan files age out.

---

## Project Retrospective

### What Worked

**Recon before writing.** Checking `/etc/logrotate.d/psad` first confirmed there was nothing to conflict with and revealed the structure of `/var/log/psad/` - directories, not files - which settled the tool choice before any config was written.

**Dry run before force.** Running `logrotate --debug` caught the insecure permissions error without touching any files. By the time the force rotation ran, the only thing left to find was the `cron_debug.log` ownership issue.

**Catching the ownership problem during the force test.** `cron_debug.log` being root-owned is a direct consequence of how cron job output redirection works. It only surfaced here because logrotate needed to write to it as a different user. It's a pattern worth remembering - file location doesn't tell you who owns it, the process that wrote it does.

---

### What Could Have Been Done Better

**Cleanup jobs have no logging.** The four `find` cron jobs run with no output redirection. A failed deletion would go unnoticed until the directory was manually inspected. Adding `>> ~/cleanup.log 2>&1` to each job would catch silent failures the same way the scan and backup jobs do.

**`cron_debug.log` ownership should have been caught earlier.** The scan script has been writing `cron_debug.log` as root since the port scan baseline project. It only became a problem here because logrotate needed to act on it as a different user. Auditing file ownership in managed directories as part of each new project deployment would catch this kind of mismatch before it surfaces as an error.

**No verified deletion test.** The `find` jobs were verified for correctness by reading the crontab but never tested against actual deletable files. The first real-world test will happen automatically in July when the oldest scan files age past 30 days. A more thorough approach would have been to create a dummy file with a backdated timestamp using `touch -d` and confirm the job deletes it before relying on it in production.

---

## Future Work

- [ ] Add output logging to the four `find` cleanup cron jobs
- [ ] Off-site backup storage (GCP Cloud Storage)

---

## Contact

**LinkedIn:** [Supawat H.](https://www.linkedin.com/in/supawat-h-145371392)
**YouTube:** [@urielbyte](https://www.youtube.com/@urielbyte)

---

*Report authored by Supawat H. (uriel0byte).*
