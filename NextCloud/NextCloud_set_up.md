# Self-Hosted Nextcloud Server

> A production-grade, security-hardened personal cloud storage server built on a bare-metal Proxmox hypervisor, accessible from anywhere via Cloudflare Tunnel with zero open inbound ports.

---

## Table of Contents

- [Overview](#overview)
- [Hardware](#hardware)
- [Architecture](#architecture)
- [Infrastructure & Hosting](#infrastructure--hosting)
- [Remote Access & Networking](#remote-access--networking)
- [Firewall Configuration](#firewall-configuration)
- [Apache Hardening](#apache-hardening)
- [PHP Configuration](#php-configuration)
- [Nextcloud Application Security](#nextcloud-application-security)
- [Caching & Performance](#caching--performance)
- [SSH Hardening](#ssh-hardening)
- [Security Monitoring](#security-monitoring)
- [Antivirus & Intrusion Detection](#antivirus--intrusion-detection)
- [User Management](#user-management)
- [Key Takeaways](#key-takeaways)

---

## Overview

This project documents the full design, deployment, and hardening of a self-hosted Nextcloud instance running on a repurposed enterprise workstation. The goal was to build a private, secure, and performant alternative to commercial cloud storage services like Google Photos and iCloud — with full ownership of the data and infrastructure.

The server runs 24/7 as a home lab appliance, serving multiple family members with photo backup, file storage, and media access from anywhere in the world — without exposing a single inbound port to the public internet.

---

## Hardware

| Component | Details |
|-----------|---------|
| **Host Machine** | HP Z820 Workstation |
| **CPUs** | Dual Intel Xeon (16 cores / 32 threads) |
| **Hypervisor** | Proxmox VE |
| **Storage** | ZFS pool (`tank`) — 3.6TB usable |
| **Disk Write Speed** | 771 MB/s (verified via `dd`) |
| **Idle Power Draw** | ~80–120W (optimised) |

---

## Architecture

```
Internet
    │
    ▼
Cloudflare (DNS + CDN + DDoS Protection)
    │  No open inbound ports on router
    ▼
Cloudflare Tunnel (cloudflared)
    │  Outbound-only encrypted connection
    ▼
Nextcloud LXC Container (192.168.1.224)
    │  Apache + PHP 8.2 + MariaDB + Redis
    ▼
ZFS Storage Pool (3.6TB — /mnt/tank)
```

```
Remote Admin Access:
Developer Machine ──► Tailscale WireGuard VPN ──► Proxmox / Nextcloud SSH
```

---

## Infrastructure & Hosting

- Deployed Nextcloud as a **lightweight LXC container** on **Proxmox VE** rather than a full VM, significantly reducing memory and CPU overhead
- Data stored on a dedicated **ZFS pool** providing data integrity through checksumming, copy-on-write, and self-healing
- OS and data disks are **separated** — Nextcloud application runs on the system disk while all user data lives on the ZFS pool
- Server configured for **24/7 operation** with CPU frequency scaling (`powersave` governor) and deep C-state sleep enabled, keeping idle power draw minimal

---

## Remote Access & Networking

### Cloudflare Tunnel
- Configured **Cloudflare Tunnel** (`cloudflared`) for public access — the tunnel makes an **outbound-only** connection to Cloudflare, meaning **zero inbound ports** are open on the router
- Real home IP address is **completely hidden** behind Cloudflare's network
- Automatic **SSL/TLS certificate** management via Cloudflare — no Let's Encrypt configuration required
- **Bot Fight Mode** enabled on Cloudflare to block known scanners (Shodan, Censys, etc.) before they reach the origin

### Tailscale VPN
- **Tailscale** WireGuard VPN installed on all servers and admin devices for encrypted remote administration
- All SSH access routed exclusively through the Tailscale interface — SSH is **invisible to the public internet**
- Wazuh security monitoring traffic also routed through Tailscale for end-to-end encryption

---

## Firewall Configuration

UFW configured following the **principle of least privilege** — every rule is locked to the minimum required source, destination, and protocol.

```
Default: deny (incoming) | allow (outgoing)
```

| Rule | Port/Protocol | Source | Reason |
|------|--------------|--------|--------|
| Nextcloud HTTPS | 443/tcp | Cloudflare IPs only | Public web access via Cloudflare only |
| Nextcloud HTTP/HTTPS | 80,443/tcp | 192.168.1.0/24 | Local network direct access |
| SSH | 22/tcp | tailscale0 interface | Admin access via encrypted VPN only |
| Tailscale WireGuard | 41641/udp | Anywhere | VPN peer-to-peer connections |
| Wazuh agent (out) | 1514,1515/tcp | → Wazuh Tailscale IP | Security event shipping via VPN |
| Redis | 6379/tcp | 127.0.0.1 only | Internal caching — never exposed |
| Notify Push | 7867/tcp | 192.168.1.0/24 | Real-time sync notifications |
| Cloudflare tunnel (out) | 443/tcp, 7844/udp | Anywhere | Outbound tunnel connectivity |

All **Cloudflare IP ranges** are explicitly enumerated in UFW rules — traffic from any other source on port 443 is silently dropped.

---

## Apache Hardening

- Stripped server version and OS information from all HTTP response headers:
  ```apache
  ServerTokens Prod
  ServerSignature Off
  ```
- Configured Apache to **only accept connections from Cloudflare's published IP ranges** — direct origin access is blocked at the web server level as a secondary defence
- Enforced HTTPS for all Nextcloud-generated URLs via `overwriteprotocol`
- Extended `TimeOut` to 3600 seconds to support large file uploads without interruption

---

## PHP Configuration

Default PHP settings are far too conservative for a media server. The following limits were increased in `/etc/php/8.2/apache2/php.ini`:

| Setting | Default | Configured |
|---------|---------|------------|
| `upload_max_filesize` | 2MB | 10GB |
| `post_max_size` | 8MB | 10GB |
| `memory_limit` | 128MB | 512MB |
| `max_execution_time` | 30s | 3600s |
| `max_input_time` | 60s | 3600s |
| `output_buffering` | On | Off |

These changes increased large file upload speeds from **~76KB/s** to significantly higher throughput, resolving a major performance bottleneck.

---

## Nextcloud Application Security

Configuration managed in `/var/www/nextcloud/config/config.php`:

- **`trusted_domains`** — explicitly enumerated list of allowed domains; requests from unlisted domains are rejected
- **`trusted_proxies`** — all Cloudflare IP ranges registered so visitor IPs are correctly resolved rather than showing Cloudflare's IPs in logs
- **`forwarded_for_headers`** — set to `HTTP_CF_CONNECTING_IP` to use Cloudflare's real IP header for accurate logging and rate limiting
- **`overwriteprotocol`** — forces HTTPS for all internally generated URLs
- **`version_hide`** — prevents Nextcloud version fingerprinting by automated scanners
- **`preview_max_x/Y`** — limited to 2048px to prevent excessive storage consumption from thumbnail generation

---

## Caching & Performance

### Redis
- **Redis** configured for both memory caching (`memcache.local`) and distributed locking (`memcache.locking`) via Unix socket for maximum performance
- Fixed socket permissions by adding `www-data` to the `redis` group — resolved recurring `RedisException` errors in logs

### Preview Generation
- **Preview Generator** app installed and configured to pre-generate photo thumbnails
- Pre-generation cron job runs every 10 minutes, ensuring thumbnails exist before users browse their photos — eliminates on-demand generation lag
- Nextcloud background jobs run via system cron every 5 minutes for reliable task execution

### Notify Push
- **High Performance Backend** (`notify_push`) running on port 7867 for instant file change notifications to desktop and mobile clients — eliminates polling delays

---

## SSH Hardening

Applied to both Proxmox host and Nextcloud LXC via `/etc/ssh/sshd_config`:

```bash
PermitRootLogin prohibit-password    # Key authentication only
PasswordAuthentication no            # Password login completely disabled
MaxAuthTries 3                       # Limit brute force attempts
LoginGraceTime 30                    # Reduce authentication window
X11Forwarding no                     # Disable unnecessary features
AllowTcpForwarding no                # Reduce attack surface
```

- Authentication uses **ED25519 SSH keys** — mathematically stronger than RSA and shorter key length
- Private keys protected with a passphrase and backed up to encrypted offline storage
- SSH port restricted to **Tailscale interface only** via UFW — the port does not respond to connections from any other network interface

---

## Security Monitoring

### Wazuh SIEM
- **Wazuh agent** deployed on both Proxmox host and Nextcloud LXC, reporting to a dedicated Wazuh manager LXC
- All agent-to-manager communication routed through **Tailscale** — encrypted WireGuard tunnel end to end
- Wazuh provides:
  - Real-time log analysis and alerting
  - File integrity monitoring (FIM) — detects unauthorised changes to system files
  - Rootkit detection
  - Security event correlation across all monitored hosts
  - Vulnerability assessment

### Fail2ban
- **Fail2ban** installed on Proxmox host monitoring SSH and Proxmox web UI login attempts
- Custom filter for Proxmox authentication failures
- Bans after **3 failed attempts** for **1 hour**
- Local network (`192.168.1.0/24`) whitelisted to prevent self-lockout

---

## Antivirus & Intrusion Detection

| Tool | Purpose | Schedule |
|------|---------|----------|
| **ClamAV** | Malware scanning of web and home directories | Daily at 2am |
| **Freshclam** | Automatic virus definition updates | Continuous |
| **rkhunter** | Rootkit detection with baseline comparison | Weekly Sunday 3am |
| **chkrootkit** | Secondary rootkit verification | Weekly Sunday 4am |

All scan results are logged and can be forwarded to Wazuh for centralised alerting.

---

## User Management

- Multi-user deployment serving multiple family members from a single instance
- User accounts managed via **Nextcloud OCC command line tool**:
  ```bash
  # Create user
  sudo -u www-data php /var/www/nextcloud/occ user:add username

  # Reset password
  sudo -u www-data php /var/www/nextcloud/occ user:resetpassword username

  # List users
  sudo -u www-data php /var/www/nextcloud/occ user:list
  ```
- Per-user **storage quotas** configurable via web UI
- Admin account separated from standard user accounts

---

## Key Takeaways

This project demonstrates practical application of:

| Skill Area | What Was Applied |
|------------|-----------------|
| **Linux Administration** | Proxmox, LXC, ZFS, systemd, cron, package management |
| **Network Security** | UFW firewall design, principle of least privilege, network segmentation |
| **Reverse Proxy & Tunnelling** | Cloudflare Tunnel, zero open port architecture |
| **VPN & Encrypted Access** | Tailscale WireGuard, interface-level firewall rules |
| **Web Server Hardening** | Apache security headers, IP allowlisting, PHP tuning |
| **SIEM & Monitoring** | Wazuh agent deployment, log shipping over encrypted tunnel |
| **Intrusion Detection** | rkhunter, chkrootkit, ClamAV, Fail2ban |
| **SSH Hardening** | Key-only auth, ED25519, interface restriction |
| **Application Security** | Trusted proxies, version hiding, header stripping |
| **Performance Tuning** | Redis caching, preview generation, PHP limits |

---

*Built and documented by Ryan Erasmus — home lab project running 24/7 on repurposed enterprise hardware.*
