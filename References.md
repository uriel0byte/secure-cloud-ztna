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
