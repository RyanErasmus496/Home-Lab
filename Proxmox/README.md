# 🔒 Proxmox Security

Properly securing Proxmox was absolutely essential — it is the master hypervisor controlling all VMs and LXCs on the machine. Here is how I did that.

---

## UFW Firewall

```
Status: active
Logging: on (medium)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip
```

> Default deny on all incoming traffic is standard practice — nothing gets in unless explicitly allowed.

### Inbound Rules

| Port / Interface         | Action   | From            | Purpose |
|--------------------------|----------|-----------------|---------|
| `8006/tcp`               | ALLOW IN | x.x.x.x/x       | Proxmox web management UI — restricted to home network |
| `22/tcp` on `tailscale0` | ALLOW IN | Anywhere         | SSH — restricted to Tailscale VPN only |
| `5900:5999/tcp`          | ALLOW IN | x.x.x.x/x       | VNC for Proxmox web UI terminal — home network only |
| `3128/tcp`               | ALLOW IN | x.x.x.x/x       | SPICE proxy (better than VNC for graphical VMs) — home network only |
| `41641/udp`              | ALLOW IN | Anywhere         | Tailscale WireGuard port (see Tailscale section) |
| `1514/tcp` on `tailscale0` | ALLOW IN | Anywhere       | Wazuh agent communication (see Wazuh doc) |
| `1515/tcp` on `tailscale0` | ALLOW IN | Anywhere       | Wazuh agent communication |
| `5404/udp`               | ALLOW IN | x.x.x.x/x       | Corosync cluster sync — home network only |
| `5405/udp`               | ALLOW IN | x.x.x.x/x       | Corosync cluster sync — home network only |
| `vmbr0`                  | ALLOW IN | Anywhere         | Virtual network bridge (see explanation below) |

> All IPv6 equivalents of the above rules are also applied.

### Outbound Rules

| Port / Interface | Action    | To              | Purpose |
|------------------|-----------|-----------------|---------|
| `vmbr0`          | ALLOW OUT | Anywhere         | Virtual bridge outbound traffic |
| `443/tcp`        | ALLOW OUT | Anywhere         | HTTPS — package retrieval and Cloudflare |
| `7844/tcp`       | ALLOW OUT | Anywhere         | Cloudflare QUIC tunnel protocol (see Nextcloud doc) |

> All IPv6 equivalents of the above rules are also applied.

### vmbr0 — Virtual Network Bridge

`vmbr0` is the virtual network bridge Proxmox uses to link all VMs and LXCs to the network. Allowing all traffic on this bridge is required — without it, tools like Nextcloud and Wazuh cannot communicate with the Proxmox host and will break.

---

## Tailscale

Tailscale is a zero-trust, peer-to-peer mesh VPN I use to securely SSH into my server from outside my home network.

**How it works:** Install Tailscale on any VM or LXC you want to access externally, then connect it to your private Tailscale network via the provided link. This lets me lock port `22` down to Tailscale devices only — meaning only machines I've manually added to the network can SSH in.

Ports `1514` and `1515` are also restricted to the Tailscale interface so I can access the Wazuh dashboard remotely.

### Why WireGuard / Why Tailscale?

Tailscale uses **WireGuard** under the hood on port `41641/udp`. WireGuard is one of the fastest and most secure VPN protocols available, using:

- **ChaCha20** — fast symmetric encryption
- **Curve25519** — efficient elliptic-curve key exchange

I specifically chose Tailscale over OpenVPN or IPsec because my hardware is older — those protocols are too resource-heavy to run efficiently. Tailscale gives me the security I need without the overhead.

---

## Malware & Rootkit Scanning

### Rootkit Scanning — chkrootkit + rkhunter

I run both tools weekly in tandem rather than picking one:

| Tool        | Strength |
|-------------|----------|
| `chkrootkit` | Better at detecting well-known rootkits and scanning system binaries |
| `rkhunter`   | Better at finding obscure or harder-to-detect rootkits |

Together they cover each other's blind spots, giving a more comprehensive scan that's less likely to miss anything.

### Antivirus — ClamAV

A weekly `clamscan` runs to cover all bases. ClamAV is lightweight while still being thorough — important given my hardware constraints.

> These lightweight tools were chosen deliberately. My server is in my bedroom, and heavier scanning tools would push fan RPM up noticeably. Scans are scheduled in the early hours of the morning to avoid resource contention while the server is in active use.

### Crontab

```cron
# Rkhunter — every Sunday at 3am
0 3 * * 0 rkhunter --check --skip-keypress --report-warnings-only | mail -s "rkhunter report" root

# Chkrootkit — every Sunday at 4am
0 4 * * 0 chkrootkit | grep -i infected | mail -s "chkrootkit report" root

# ClamAV — every Sunday at 5am
0 5 * * 0 clamscan -r --infected --log=/var/log/clamav/weekly-scan.log /home /var/www
```

---

## Security Philosophy

> *Only allow what is strictly needed, from exactly where it needs to come from, on exactly the interface it should arrive on.*
