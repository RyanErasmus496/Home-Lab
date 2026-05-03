[README.md](https://github.com/user-attachments/files/27313682/README.md)

# 🖥️ Home Server

> A personal homelab built on an HP Z820 workstation running Proxmox — documenting my hardware, the reasoning behind key decisions, and how I configure and secure my self-hosted services.

This repository serves as a full written record of my home server project — from the hardware choices and OS selection through to firewall rules, VPN configuration, malware scanning, and self-hosted application setup. The goal is to document not just *what* I did, but *why* I did it.

---

## What's Covered in This Repo

| Section | Summary |
|---------|---------|
| **Hardware & OS** | HP Z820 specs, why I chose Proxmox as a bare-metal hypervisor, and why RAID 1 made sense for this build |
| **Proxmox Security** | UFW firewall rules and philosophy, Tailscale VPN for zero-trust remote access, and weekly rootkit and antivirus scanning with chkrootkit, rkhunter, and ClamAV |
| **Nextcloud** | Why I chose to self-host cloud storage instead of using a commercial provider, and how I expose and secure it safely |

---

## Hardware

| Component    | Details                              |
|--------------|--------------------------------------|
| **Machine**  | HP Z820 Workstation (2012)           |
| **CPU**      | 2x Intel Xeon E5 (8-core each, 16 cores total) |
| **RAM**      | 224GB DDR3                           |
| **Boot**     | 1x 500GB SAS SSD                     |
| **Storage**  | 2x 4TB SAS HDD (RAID 1)             |
| **OS**       | Proxmox VE                           |

### Why RAID 1?

While RAID 1 may seem like an older approach, it is the most practical choice for this build. The HP Z820 only has room for two hard drives, and the primary role of this server is to run Nextcloud as a self-hosted photo and document storage solution. Since the bulk of what is stored are family photos and personal files, having a direct mirror copy on the second drive makes the most sense — if one drive fails, nothing is lost. With only two drive bays available, RAID 1 is the logical and reliable choice.

### Why Proxmox?

From the start, I knew I wanted a Type 1 (bare-metal) hypervisor — an OS that runs directly on the hardware rather than on top of another operating system, giving maximum performance and control over virtualisation.

Proxmox was recommended by a friend and quickly proved to be the right choice for several reasons:

| Factor | Why it mattered |
|--------|----------------|
| **Cost** | 100% free and open source |
| **Community** | Huge community with extensive forums and tutorials |
| **First-time friendly** | As my first server build, the wealth of online help was invaluable |
| **Feature-rich** | Full VM and LXC container support out of the box |

---

## Services

| Service      | Description                              | Status   |
|--------------|------------------------------------------|----------|
| Nextcloud    | Self-hosted cloud storage for family photos and documents | Live |
| Wazuh        | SIEM / security monitoring               | Live  |

---

## Repository Structure

```
Home-Server/
├── proxmox/          # Proxmox host configuration & security hardening
└── nextcloud/        # Nextcloud setup, configuration & security
```

---

## Security Philosophy

> *Only allow what is strictly needed, from exactly where it needs to come from, on exactly the interface it should arrive on.*

Key measures across this homelab:
- UFW firewall with default-deny on all incoming traffic
- Tailscale mesh VPN for zero-trust remote access
- Weekly rootkit scanning (chkrootkit + rkhunter) and ClamAV
- Cloudflare Tunnel for public-facing services — no open ports to the internet

---

## Docs

- [Proxmox Security](./proxmox/README.md)
- [Nextcloud](./nextcloud/README.md)

---

> ⚠️ All IP addresses, subnet ranges, and sensitive values have been redacted.
