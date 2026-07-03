# SSH Hardening Audit

**Author:** Supawat H. (uriel0byte)  |  **Environment:** Google Cloud Platform  |  **Status:** Complete

---

## Summary

The ZTNA deployment locked SSH to the Tailscale interface and replaced password authentication with ed25519 key pairs. That covered access control. It did not cover what OpenSSH was advertising to clients during the handshake  and the defaults included several algorithms that have no business being offered in 2026.

This project ran `ssh-audit` against the server to get a full picture of what was being offered, identified every failure and warning in the output, made targeted changes to `sshd_config` to strip the weak algorithms, and confirmed the results with a second scan. Zero failures and zero warnings on the re-audit.

---

## Objective

1. Install `ssh-audit` and capture the pre-hardening baseline from both `localhost` and the Tailscale interface.
2. Identify and understand every flagged algorithm in the output.
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

The ZTNA deployment report noted that `sshd_config` was only partially hardened  password authentication was disabled and root login was blocked, but the cipher and algorithm configuration was left at OpenSSH defaults. Those defaults are conservative by consumer standards but not by security audit standards. This project finishes what the ZTNA deployment started.

The backup script already covers `/etc/ssh` nightly. A manual backup was run at the start of this session before touching any config, so the pre-hardening state is preserved in `~/infrastructure_backups/config_20260703_104859`.

---

## Pre-Execution

A manual backup was run before any changes:

```bash
sudo ~/infrastructure_backups/backup_configs.sh
# Backup completed successfully at /home/uriel/infrastructure_backups/config_20260703_104859
```

Then `ssh-audit` was installed from the Ubuntu package manager:

```bash
sudo apt install ssh-audit
# ssh-audit 3.3.0-2build1 installed
```

---

## Deployment Log

### Phase 1 - Baseline Audit

> 📹 **Recording:** [![asciicast](https://asciinema.org/a/1260063.svg)](https://asciinema.org/a/1260063)

Both interfaces were scanned  `localhost` for the loopback view and `100.114.172.16` for the Tailscale interface. Both returned identical results, confirming the same sshd process and config applies regardless of which interface you connect through.

```bash
ssh-audit localhost
ssh-audit 100.114.172.16
```

The first scan of `100.114.172.16` timed out, then succeeded on a second attempt. The Tailscale interface takes a moment to respond when it hasn't been used recently. This is not an sshd issue.

**Pre-hardening output summary:**

Key exchange failures (3): `ecdh-sha2-nistp256`, `ecdh-sha2-nistp384`, `ecdh-sha2-nistp521`

Host key failure (1): `ecdsa-sha2-nistp256`

MAC failures (2): `hmac-sha1-etm@openssh.com`, `hmac-sha1`

MAC warnings (6): `umac-64-etm@openssh.com`, `umac-64@openssh.com`, `umac-128@openssh.com`, `hmac-sha2-256`, `hmac-sha2-512` (encrypt-and-MAC mode), `umac-64` (small tag size)

Everything else  ciphers, the post-quantum kex algorithms, the Curve25519 algorithms  was clean.

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

**Post-hardening output summary:**

Key exchange: 5 algorithms, all `[info]`  no failures, no warnings.

Host keys: 3 algorithms (`rsa-sha2-512`, `rsa-sha2-256`, `ssh-ed25519`), all `[info]`.

Ciphers: unchanged, all `[info]`.

MACs: reduced from 10 entries to 3, all `[info]`. Every failure and warning is gone.

Zero failures. Zero warnings.

Access from the local VMware VM was confirmed immediately after the re-audit. The session connected normally, which means the algorithm changes are compatible with the client.

---

## What the Audit Found and Why It Mattered

### Key Exchange  The NIST Curve Failures

**What was flagged:** `ecdh-sha2-nistp256`, `ecdh-sha2-nistp384`, `ecdh-sha2-nistp521`

**What key exchange actually is:** At the start of every SSH connection, before any authentication happens, the client and server run a key exchange protocol to derive a shared secret that neither side sent over the wire. That shared secret becomes the encryption key for the session. The algorithm used here determines how strong that negotiation is.

**Why these curves are a problem:** NIST P-256, P-384, and P-521 are elliptic curves defined by parameters that the NSA generated and published in the late 1990s. The issue is that nobody outside the NSA knows how those specific curve parameters were chosen. With well-designed elliptic curves, the constants are generated from something publicly verifiable  like the digits of pi  so anyone can confirm there's no hidden structure that could weaken them. The NIST curves have no such verifiable origin. Edward Snowden's disclosures in 2013 made the cryptography community significantly less comfortable trusting NSA-designed primitives. Whether there is actually a backdoor is unknown and probably unknowable. But when you have solid alternatives with transparent math, there's no reason to keep these.

**What replaced them:** The allow-list keeps the post-quantum hybrid algorithms (`mlkem768x25519`, `sntrup761x25519`) and `curve25519-sha256`. Curve25519 was designed by Daniel Bernstein with fully transparent, publicly verifiable constants. The post-quantum algorithms wrap Curve25519 with a second layer designed to resist attacks from a quantum computer. These are not experimental  they've been the default in OpenSSH since version 9.9.

**In sshd_config:**

```
KexAlgorithms mlkem768x25519-sha256,sntrup761x25519-sha512,sntrup761x25519-sha512@openssh.com,curve25519-sha256,curve25519-sha256@libssh.org
```

OpenSSH does not have a "remove this one algorithm" option. You specify exactly what you want offered, and anything not on the list is dropped automatically. The NIST curves were removed simply by not including them.

---

### Host Key  The ECDSA Failure

**What was flagged:** `ecdsa-sha2-nistp256` with two complaints  the NIST curve problem again, plus a secondary warning about weak random number generation potentially leaking the key.

**What host key algorithms are:** When a client connects to the server for the first time, the server sends its host public key. The client checks it against `~/.ssh/known_hosts` to confirm it's talking to the right machine and not an imposter. The host key algorithm controls the cryptographic signature used for that proof of identity.

**Why ecdsa-sha2-nistp256 is doubly problematic:** First, it uses the same P-256 NIST curve with the same trust concerns as above. Second, ECDSA signatures require a random number called a nonce during signing. If that nonce is ever repeated or predictable, the private key can be mathematically recovered from just two signatures. This has happened in practice  Sony's PlayStation 3 was broken this way in 2010. The algorithm is fundamentally brittle in a way that ed25519 is not, because ed25519 derives its nonce deterministically from the message and key rather than relying on a random number generator.

**What replaced it:** `ssh-ed25519` was already present and is the correct modern choice. RSA at 3072-bit remains in the list as a compatibility option for older clients. The ECDSA host key was removed by simply not including it in the allow-list.

**In sshd_config:**

```
HostKeyAlgorithms rsa-sha2-512,rsa-sha2-256,ssh-ed25519
```

Note that this directive controls which host key *types* the server advertises to clients during negotiation. The actual host key files still exist on disk  `/etc/ssh/ssh_host_ecdsa_key` is still there. It just never gets offered to clients anymore. You could delete the file, but leaving it doesn't cause any harm since it's never used.

---

### MACs  The SHA-1 Failures and ETM Warnings

**What was flagged:**

Failures: `hmac-sha1-etm@openssh.com`, `hmac-sha1`  both use the SHA-1 hash algorithm.

Warnings: `umac-64-etm@openssh.com`, `umac-64@openssh.com`  64-bit tag size. Several others  `umac-128@openssh.com`, `hmac-sha2-256`, `hmac-sha2-512`  flagged for using encrypt-and-MAC mode rather than encrypt-then-MAC.

**What MACs are:** A Message Authentication Code is a short tag appended to each packet that proves the packet wasn't modified in transit. Without it, an attacker positioned between you and the server could flip bits in the ciphertext and you'd have no way to detect it. The MAC is what provides integrity  it's separate from encryption, which only provides confidentiality.

**Why SHA-1 is a failure:** SHA-1 has been practically broken since 2017 when Google published the first real collision  two different inputs that produce the same SHA-1 hash. In the context of a MAC, a collision attack means an attacker could potentially forge authenticated packets. There is no reason to keep SHA-1 in a MAC in 2026.

**Why the 64-bit tag warning matters:** The tag size determines how hard it is to forge a valid MAC by guessing. A 64-bit tag means an attacker needs roughly 2^64 attempts to forge one by brute force. That sounds large, but with high-speed connections and long-lived sessions, it's within the range of concern. 128-bit tags double the exponent and push forgery well out of reach.

**What encrypt-and-MAC mode means and why ETM is better:** The non-ETM variants  `hmac-sha2-256`, `hmac-sha2-512`, `umac-128@openssh.com`  compute the MAC over the *plaintext* before encryption, then send both the ciphertext and the MAC together. The problem is that the server has to decrypt the packet before it can verify the MAC. An attacker who can send crafted ciphertext and observe timing differences in the decryption can extract information about the plaintext  this class of attack is called a padding oracle. ETM (Encrypt-then-MAC) fixes this by computing the MAC over the *ciphertext*. The server checks the MAC first. If it fails, the packet is dropped immediately without decryption, which eliminates the oracle entirely.

**What replaced the full list:** Three MACs, all ETM, all with 128-bit or larger tags:

```
MACs umac-128-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
```

The SHA-1 variants were dropped entirely. The non-ETM variants were dropped. The 64-bit tag variants were dropped. Ten entries became three.

---

### The Additional Info Note  Terrapin and chacha20-poly1305

The re-audit still shows an informational note about `chacha20-poly1305@openssh.com` and the Terrapin vulnerability (CVE-2023-48795). This is not a finding that requires action.

Terrapin was a 2023 attack against the SSH protocol that could allow a man-in-the-middle attacker to strip certain extension negotiation messages from the handshake. The fix was a protocol-level countermeasure called strict key exchange, which your server supports  confirmed by `kex-strict-s-v00@openssh.com` appearing in the output. The note is saying that if a connecting *client* is old enough to not support strict kex, the vulnerability could still exist on that channel. Both your VMware client and Termius on iPad are recent enough to support it. This is an informational notice, not an open vulnerability on this server.

---

### Hardening Parameters

These four settings were added alongside the algorithm changes. They don't affect cryptography but they close some standard gaps.

**`LoginGraceTime 30`**  After a client connects, it has 30 seconds to complete authentication before the server drops the connection. The default is 120 seconds. A shorter window reduces the exposure window for connection-flooding attacks.

**`MaxAuthTries 3`**  The server will drop the connection after 3 failed authentication attempts. Irrelevant here since password authentication is disabled, but it's defense in depth. If key authentication somehow failed and someone tried to brute-force in, they'd get three attempts per connection instead of the default six.

**`MaxSessions 3`**  Caps the number of concurrent sessions per connection. Default is 10. Three is more than enough for normal use on a single-user server.

**`AllowUsers uriel`**  Explicitly whitelist the only account that should be able to log in. Without this, any user account on the server could theoretically authenticate via SSH if they had a valid key. With it, even if another account existed, it would be blocked at the SSH level regardless of keys.

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

# Test config syntax  no output means no errors
sudo sshd -t

# Reload the daemon (existing sessions survive this)
sudo systemctl reload ssh

# Re-audit to confirm clean results
ssh-audit localhost
ssh-audit 100.114.172.16
```

---

## What I Actually Learned

**OpenSSH defaults are not the same as secure defaults.** The server was running OpenSSH 10.2p1, which is current. The default config still offered three NIST elliptic curve algorithms, ECDSA host keys, SHA-1 MACs, and non-ETM MACs. None of those are catastrophically broken in most threat models, but none of them belong on a server that has cleaner alternatives available. "Recent version" and "well-configured" are different things.

**Algorithm allow-lists work by exclusion, not subtraction.** There's no directive like `RemoveAlgorithm ecdh-sha2-nistp256`. You specify exactly what you want, and OpenSSH drops everything not on the list. Understanding this changes how you approach the config  instead of hunting down each bad option to remove, you define the target state and the rest disappears automatically.

**The ETM ordering matters more than the hash function.** Both `hmac-sha2-256` and `hmac-sha2-256-etm@openssh.com` use SHA-256. The hash is fine in both cases. The vulnerability in the non-ETM version comes from the order of operations, not the hash. Knowing *why* ETM is better makes it easier to recognize bad MAC configurations in other contexts.

**The NIST curve question is about trust, not proof.** There's no published exploit for P-256 or P-384. The concern is structural  the curve parameters were generated by an agency with a documented history of weakening cryptographic standards, without any verifiable process. You can't prove there's a backdoor, and you can't prove there isn't one. When you have alternatives like Curve25519 with fully transparent math, the question becomes why you'd keep the untransparent ones.

**`sudo sshd -t` before every reload.** A syntax error in `sshd_config` doesn't crash the running daemon  it just means the next connection will fail because a new daemon process can't start. Your existing session survives, but anyone trying to connect after a bad reload hits a dead socket. Testing the config before applying it takes five seconds and eliminates that failure mode.

---

## Limitations

**The ECDSA host key file still exists on disk.** `/etc/ssh/ssh_host_ecdsa_key` is not offered to clients anymore because of the `HostKeyAlgorithms` directive, but the file is still there. It could be deleted for cleanliness. It's not a security issue in its current state since it's never used, but it's worth noting.

**Client compatibility was only tested against two clients.** The VMware VM and the iPad both connected successfully after the hardening. Older SSH clients that don't support the post-quantum algorithms or Curve25519 would fail to connect now. That's not a problem for this setup  both clients are recent  but it would matter in a multi-user environment.

**The Terrapin informational note will always be present with chacha20-poly1305.** This is a known limitation of the strict kex countermeasure. The note is not actionable on the server side. Both connected clients support strict kex, so the practical exposure is zero.

---

## Project Retrospective

### What Worked

**Running the manual backup first.** The pre-hardening `sshd_config` is preserved in a timestamped backup directory. If the changes had broken something, restoring the previous state would have been a single `rsync` command. This was already habit from the ZTNA deployment, but it's especially important when touching SSH config on a remote server with no out-of-band console access.

**`sudo sshd -t` before reloading.** No errors on the syntax test, clean reload, no disruption to the existing session. The test step is worth the five seconds every time.

**Scanning both interfaces.** Running `ssh-audit` on both `localhost` and `100.114.172.16` confirmed that the same config applies to both  there's nothing in a drop-in `/etc/ssh/sshd_config.d/` file that could be silently overriding settings on one interface but not the other. The timeout on the first Tailscale scan was a distraction but the second attempt confirmed the result.

---

### What Could Have Been Done Better

**No drop-in file was used.** The algorithm directives were added directly to `/etc/ssh/sshd_config` rather than creating a clean `/etc/ssh/sshd_config.d/ztna-hardening.conf` drop-in file. The drop-in approach would have kept the hardening changes separate from the base config, making it clearer what was modified and easier to review later. The current state works correctly but is slightly messier to audit.

**The ECDSA host key file was left on disk.** Removing `/etc/ssh/ssh_host_ecdsa_key` and `/etc/ssh/ssh_host_ecdsa_key.pub` would be the complete version of this cleanup. Since the key is no longer advertised, leaving the files is not a security concern it's just incomplete.

---

## Future Work

- [ ] Remove the ECDSA host key files from disk since they're no longer offered to clients
- [ ] Move the algorithm directives to a dedicated drop-in file at `/etc/ssh/sshd_config.d/ztna-hardening.conf`
- [ ] Fail2ban audit and weekly summary export
- [ ] Unattended-upgrades configuration

---

## Contact

**LinkedIn:** [Supawat H.](https://www.linkedin.com/in/supawat-h-145371392)
**YouTube:** [@urielbyte](https://www.youtube.com/@urielbyte)

---

*Report authored by Supawat H. (uriel0byte).*
