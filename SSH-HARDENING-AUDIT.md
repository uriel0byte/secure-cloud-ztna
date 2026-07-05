# SSH Hardening Audit

**Author:** Supawat H. (uriel0byte)  |  **Environment:** Google Cloud Platform  |  **Status:** Complete

---

## Summary

The ZTNA deployment locked SSH to the Tailscale interface and replaced password authentication with ed25519 key pairs. That covered access control. It did not cover what OpenSSH was advertising to clients during the handshake, and the defaults included algorithms that have no business being offered in 2026.

This project ran `ssh-audit` against the server, identified every failure and warning in the output, made targeted changes to `sshd_config` to remove the weak algorithms, and re-audited to confirm the results. Zero failures and zero warnings on the second scan.

---

## Objective

1. Install `ssh-audit` and capture the pre-hardening baseline from both `localhost` and the Tailscale interface.
2. Identify every flagged algorithm and understand why it was flagged.
3. Add explicit algorithm allow-lists to `sshd_config` to remove the weak options.
4. Re-audit and confirm a clean result.
5. Verify that a live SSH connection from the local VM still works after the changes.

---

## Tech Stack

| Layer | Tool |
|---|---|
| Audit Tool | ssh-audit 3.3.0 |
| Target Daemon | OpenSSH 10.2p1 |
| Config File | `/etc/ssh/sshd_config` |
| Platform | GCP e2-micro, Ubuntu 26.04 LTS (Minimal) |

---

## Relationship to Previous Projects

The ZTNA deployment report acknowledged that `sshd_config` was only partially hardened. Password authentication was disabled and root login was blocked, but the cipher and algorithm configuration was left at OpenSSH defaults. This project closes that gap.

The backup script already covers `/etc/ssh` nightly. A manual backup was run at the start of this session before touching the config, so the pre-hardening state is preserved in `~/infrastructure_backups/config_20260703_104859`.

---

## Pre-Execution

```bash
sudo ~/infrastructure_backups/backup_configs.sh
# Backup completed successfully at /home/uriel/infrastructure_backups/config_20260703_104859

sudo apt install ssh-audit
# ssh-audit 3.3.0-2build1 installed
```

---

## Deployment Log

### Phase 1 - Baseline Audit

> 📹 **Recording:** [![asciicast](https://asciinema.org/a/1260063.svg)](https://asciinema.org/a/1260063)

Both interfaces were scanned. `localhost` hits the loopback view; `100.114.172.16` hits the Tailscale interface directly. Both returned identical results, confirming the same sshd process and config applies to both.

```bash
ssh-audit localhost
ssh-audit 100.114.172.16
```

The first scan of `100.114.172.16` timed out and succeeded on a second attempt. The Tailscale interface takes a moment to respond when idle. Not an sshd issue.

**Pre-hardening findings:**

Key exchange failures (3): `ecdh-sha2-nistp256`, `ecdh-sha2-nistp384`, `ecdh-sha2-nistp521`

Host key failure (1): `ecdsa-sha2-nistp256`

MAC failures (2): `hmac-sha1-etm@openssh.com`, `hmac-sha1`

MAC warnings (6): `umac-64-etm@openssh.com`, `umac-64@openssh.com`, `umac-128@openssh.com`, `hmac-sha2-256`, `hmac-sha2-512` (encrypt-and-MAC mode), `umac-64` (small tag size)

Ciphers, the post-quantum kex algorithms, and the Curve25519 algorithms were all clean.

---

### Phase 2 - sshd_config Changes

Three algorithm directives and five hardening parameters were added to `/etc/ssh/sshd_config`.

```bash
sudo nano /etc/ssh/sshd_config
```

**Algorithm allow-lists added at the bottom of the file:**

```
KexAlgorithms mlkem768x25519-sha256,sntrup761x25519-sha512,sntrup761x25519-sha512@openssh.com,curve25519-sha256,curve25519-sha256@libssh.org
HostKeyAlgorithms rsa-sha2-512,rsa-sha2-256,ssh-ed25519
MACs umac-128-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
```

**Hardening parameters added in the Authentication block:**

```
LoginGraceTime 30
MaxAuthTries 3
MaxSessions 3
AllowUsers uriel
```

**Already present, confirmed correct:**

```
X11Forwarding no
KbdInteractiveAuthentication no
```

Config syntax was tested before reloading:

```bash
sudo sshd -t
# No output = no errors

sudo systemctl reload ssh
```

---

### Phase 3 - Re-Audit

> 📹 **Recording:** [![asciicast](https://asciinema.org/a/1260067.svg)](https://asciinema.org/a/1260067)

```bash
ssh-audit localhost
ssh-audit 100.114.172.16
```

**Post-hardening results:**

Key exchange: 5 algorithms, all `[info]`.

Host keys: 3 algorithms (`rsa-sha2-512`, `rsa-sha2-256`, `ssh-ed25519`), all `[info]`.

Ciphers: unchanged, all `[info]`.

MACs: 10 entries reduced to 3, all `[info]`.

Zero failures. Zero warnings.

Access from the local VMware VM was confirmed immediately after. The session connected normally.

---

## What the Audit Found and Why It Mattered

### Key Exchange — The NIST Curve Failures

**What was flagged:** `ecdh-sha2-nistp256`, `ecdh-sha2-nistp384`, `ecdh-sha2-nistp521`

Key exchange is how the client and server derive a shared encryption key at the start of every session, before any authentication happens. It is the foundation the rest of the connection sits on.

NIST P-256, P-384, and P-521 are elliptic curves whose defining parameters were generated by the NSA in the late 1990s. The problem is that nobody outside the NSA knows how those parameters were chosen. With well-designed curves, the constants come from something publicly verifiable, like the digits of pi, so anyone can confirm there is no hidden structure weakening them. The NIST curves have no such verifiable origin. After the Snowden disclosures in 2013, the cryptography community's trust in NSA-designed primitives dropped sharply. Whether there is actually a backdoor is unknown. When cleaner alternatives exist, there is no argument for keeping these.

The allow-list keeps the post-quantum hybrid algorithms (`mlkem768x25519`, `sntrup761x25519`) and `curve25519-sha256`. Curve25519 was designed with fully transparent, publicly verifiable constants. The post-quantum algorithms wrap Curve25519 with a second layer designed to resist attacks from a quantum computer. Both have been default in OpenSSH since version 9.9.

OpenSSH has no "remove this one algorithm" option. You specify exactly what you want offered, and anything not on the list gets dropped. The NIST curves were removed by not including them.

```
KexAlgorithms mlkem768x25519-sha256,sntrup761x25519-sha512,sntrup761x25519-sha512@openssh.com,curve25519-sha256,curve25519-sha256@libssh.org
```

---

### Host Key — The ECDSA Failure

**What was flagged:** `ecdsa-sha2-nistp256` — NIST curve suspicion plus a secondary warning about weak random number generation potentially leaking the key.

When a client connects for the first time, the server sends its host public key. The client checks it against `~/.ssh/known_hosts` to confirm it is talking to the right machine. The host key algorithm controls the cryptographic signature used for that proof.

`ecdsa-sha2-nistp256` has two problems. First, it uses P-256, the same curve with the same trust concerns above. Second, ECDSA signatures require a random number called a nonce during signing. If that nonce is ever repeated or predictable, the private key can be mathematically recovered from just two signatures. This is not theoretical — Sony's PlayStation 3 was broken this way in 2010. ed25519 does not have this problem because it derives its nonce deterministically from the message and the key, so the quality of the random number generator is irrelevant.

`ssh-ed25519` was already present and is the right choice. RSA at 3072-bit stays in the list for compatibility with older clients. ECDSA was removed by not including it.

```
HostKeyAlgorithms rsa-sha2-512,rsa-sha2-256,ssh-ed25519
```

This directive controls which host key types the server advertises during negotiation. The actual key file `/etc/ssh/ssh_host_ecdsa_key` still exists on disk — it just never gets offered to clients. The file can be deleted for cleanliness but leaving it causes no harm since it is never used.

---

### MACs — The SHA-1 Failures and ETM Warnings

**What was flagged:**

Failures: `hmac-sha1-etm@openssh.com`, `hmac-sha1` — both use SHA-1.

Warnings: `umac-64-etm@openssh.com`, `umac-64@openssh.com` — 64-bit tag size. `umac-128@openssh.com`, `hmac-sha2-256`, `hmac-sha2-512` — encrypt-and-MAC mode.

A Message Authentication Code is a tag appended to each packet that proves the packet was not modified in transit. Without it, an attacker between you and the server could flip bits in the ciphertext with no way to detect it. The MAC handles integrity. Encryption handles confidentiality. They are separate concerns.

**SHA-1:** Practically broken since 2017 when Google published the first real collision — two different inputs producing the same hash. In a MAC, that means an attacker could forge authenticated packets. SHA-1 has no place in a current configuration.

**64-bit tags:** The tag size determines how hard MAC forgery is by brute force. 64 bits is too small for high-traffic or long-lived connections. 128 bits is the current acceptable floor.

**Encrypt-and-MAC vs. Encrypt-then-MAC:** The non-ETM variants compute the MAC over the plaintext before encryption, then transmit both the ciphertext and the MAC together. The server has to decrypt the packet before it can verify the MAC. An attacker who sends crafted ciphertext and watches timing differences in the server's decryption response can extract plaintext information — this class of attack is called a padding oracle. ETM flips the order: the MAC is computed over the ciphertext instead. The server checks the MAC first and drops the packet immediately on failure, no decryption attempted. The oracle disappears.

Ten MAC entries became three. SHA-1, non-ETM, and 64-bit tag variants are all gone.

```
MACs umac-128-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
```

---

### The Terrapin Note — Not an Action Item

The re-audit still shows an informational note about `chacha20-poly1305@openssh.com` and CVE-2023-48795 (Terrapin). No action required.

Terrapin was a 2023 attack where a man-in-the-middle could strip certain extension negotiation messages from the SSH handshake. The countermeasure is strict key exchange, which this server supports — confirmed by `kex-strict-s-v00@openssh.com` in the output. The note means that if a connecting client is old enough to not support strict kex, the vulnerability could still exist on that channel. Both the VMware client and Termius on iPad are recent enough. This is informational, not an open finding.

---

### Hardening Parameters

Four settings were added alongside the algorithm changes. None affect cryptography. They close standard configuration gaps.

**`LoginGraceTime 30`** — A connecting client has 30 seconds to complete authentication before the server drops the connection. The default is 120 seconds.

**`MaxAuthTries 3`** — The connection is dropped after 3 failed authentication attempts. Password auth is already disabled so this barely matters here, but the default is 6 and there is no reason to leave it at that.

**`MaxSessions 3`** — Caps concurrent sessions per connection. Default is 10. Three is enough for a single-user server.

**`AllowUsers uriel`** — Explicitly whitelists the only account that should log in via SSH. Without this, any user account on the server could authenticate if it had a valid key. With it, the SSH daemon rejects all accounts not on the list regardless of key validity.

---

## Command Reference

```bash
# Manual backup before making changes
sudo ~/infrastructure_backups/backup_configs.sh

# Install ssh-audit
sudo apt install ssh-audit

# Run baseline audit
ssh-audit localhost
ssh-audit 100.114.172.16

# Edit sshd_config
sudo nano /etc/ssh/sshd_config

# Test config syntax — no output means no errors
sudo sshd -t

# Reload the daemon — existing sessions survive this
sudo systemctl reload ssh

# Re-audit to confirm clean results
ssh-audit localhost
ssh-audit 100.114.172.16
```

---

## What I Actually Learned

**Running OpenSSH 10.2p1 does not mean the config is current.** The version was fine. The defaults still included NIST elliptic curves, ECDSA host keys, SHA-1 MACs, and non-ETM MACs. None of those are individually catastrophic in most threat models, but none of them belong on a server that has better options available. Version and configuration are separate questions.

**Algorithm allow-lists work by defining what you want, not removing what you don't.** There is no `RemoveAlgorithm` directive in OpenSSH. You write out exactly what the server should offer and everything else disappears. Once that clicked, the approach shifted from hunting down individual bad options to writing a clean target state.

**ETM is about ordering, not hash strength.** Both `hmac-sha2-256` and `hmac-sha2-256-etm@openssh.com` use SHA-256. The hash is not the problem. The non-ETM version is vulnerable because of when the MAC is computed relative to encryption, not because of which hash it uses. Knowing the actual reason makes it easier to spot bad MAC configs outside this specific context.

**The NIST curve issue is a trust argument, not a proof.** There is no published exploit for P-256 or P-384. The concern is that their parameters were produced by an agency with a documented history of weakening cryptographic standards, with no verifiable process. Curve25519 has transparent math and is faster. The question is not whether P-256 is broken — it is why you would keep it when Curve25519 is available.

**`sudo sshd -t` before every reload, without exception.** A syntax error in `sshd_config` does not crash the running daemon. It means the next fresh connection attempt fails because a new daemon process cannot start. The existing session stays alive but anyone trying to connect gets a dead socket. The syntax test takes five seconds.

---

## Limitations

**The ECDSA host key file still exists on disk.** `/etc/ssh/ssh_host_ecdsa_key` is not offered to clients because of the `HostKeyAlgorithms` directive, but the file is still there. Not a security issue since it is never used, but worth cleaning up.

**Client compatibility was tested against two clients only.** The VMware VM and the iPad both connected cleanly after the hardening. Older SSH clients that do not support the post-quantum algorithms or Curve25519 would fail to connect now. Both clients here are recent, so this is not a concern for this setup.

**The Terrapin note persists.** The informational note about `chacha20-poly1305@openssh.com` will always appear in ssh-audit output as long as that cipher is offered. It is not actionable on the server side. Both connected clients support strict kex.

---

## Project Retrospective

### What Worked

**Manual backup before touching anything.** The pre-hardening `sshd_config` is in `~/infrastructure_backups/config_20260703_104859`. If the changes had broken access, recovery would have been one rsync command. This habit matters more on a remote server with no console fallback than anywhere else.

**`sudo sshd -t` before reloading.** The syntax test returned nothing, the reload was clean, the existing session was not interrupted.

**Scanning both interfaces.** Running ssh-audit on both `localhost` and `100.114.172.16` confirmed that no drop-in file in `/etc/ssh/sshd_config.d/` was silently overriding settings on one interface but not the other. The first Tailscale scan timing out was a minor distraction — the second attempt confirmed the result matched localhost exactly.

---

### What Could Have Been Done Better

**Changes went directly into sshd_config instead of a drop-in file.** The algorithm directives should have gone into `/etc/ssh/sshd_config.d/ztna-hardening.conf`. That keeps the hardening changes separate from the base config, makes it obvious what was modified, and makes it easier to review or revert later. The current state works correctly but is harder to audit at a glance.

**The ECDSA host key files were left on disk.** Removing `/etc/ssh/ssh_host_ecdsa_key` and `/etc/ssh/ssh_host_ecdsa_key.pub` would be the complete version of this cleanup. The files are unused but present.

---

## Future Work

- [ ] Remove the ECDSA host key files from disk
- [ ] Move the algorithm directives to `/etc/ssh/sshd_config.d/ztna-hardening.conf`
- [ ] Fail2ban audit and weekly summary export
- [ ] Unattended-upgrades configuration

---

## Contact

**LinkedIn:** [Supawat H.](https://www.linkedin.com/in/supawat-h-145371392)
**YouTube:** [@urielbyte](https://www.youtube.com/@urielbyte)

---

*Report authored by Supawat H. (uriel0byte).*
