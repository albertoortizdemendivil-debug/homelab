# Homelab

A self-hosted homelab on a repurposed laptop running Ubuntu Server: encrypted storage, private remote access through a Tailscale mesh VPN, and small services I can understand and maintain.

The first service is a private home NAS: a LUKS-encrypted SSD shared over Samba and reachable from anywhere, with no ports opened on the router. I built it to learn how a small, realistic system is secured and operated end to end: disk encryption, remote access, a firewall policy, file sharing and, above all, what to do when something breaks.

> **Background:** the same server originally hosted a personal Telegram assistant. I retired that project and kept the infrastructure, which turned out to be the part worth keeping.

## Architecture

```
 Windows laptop ─┐
 Android phone  ─┼──  Tailscale (WireGuard mesh)  ──►  Ubuntu Server 24.04
 other devices  ─┘                                      │
                                                        ├─ SSH     key-only, non-default port, fail2ban
                                                        ├─ Samba   SMB3, one authenticated user
                                                        └─ ufw     inbound allowed only on tailscale0
                                                              │
                                              /mnt/<ssd>  ◄───┘
                                              LUKS2 + ext4, unlocked by hand
```

## Documentation

### Server

| Guide | What it covers |
|---|---|
| [Disk encryption](server/encryption.md) | LUKS on the main disk and the external SSD, daily unlock routine |
| [Remote access](server/remote-access.md) | Tailscale, working behind NAT or CGNAT, firewall policy |

### Services

| Service | What it does | Status | Docs |
|---|---|---|---|
| NAS | Samba share on the encrypted SSD | Running | [services/nas](services/nas/README.md) |
| AdGuard Home | DNS with ad and tracker filtering, served over Tailscale | Planned | — |
| Uptime Kuma | Service monitoring with alerts to a chat app | Planned | — |
| restic | Versioned, encrypted backups to a second location | Planned | — |

### Runbooks

| Runbook | When to use it |
|---|---|
| [USB drop recovery](docs/runbooks/usb-drop-recovery.md) | The external SSD disconnects and writes fail with `Input/output error` |

## Repository layout

```
homelab/
├── README.md                       overview, index and status
├── server/                         the machine itself
│   ├── encryption.md               LUKS on the main disk and the external SSD
│   └── remote-access.md            Tailscale and firewall policy
├── services/                       what runs on top of it
│   └── nas/
│       └── README.md               Samba share on the encrypted SSD
└── docs/
    └── runbooks/
        └── usb-drop-recovery.md    step-by-step recovery after a USB drop
```

Conventions:

- One folder per service under `services/`, with its own `README.md` and, when it runs in Docker, its `docker-compose.yml`.
- Anything about the machine rather than a service goes in `server/`.
- Step-by-step incident procedures go in `docs/runbooks/`.
- File names are lowercase and hyphenated, and nothing is numbered: the index above defines the reading order.
- No secrets and no real addresses are committed. Values appear as `<placeholders>`.

## Stack

| Layer | Technology |
|---|---|
| OS | Ubuntu Server 24.04 LTS, headless |
| Main disk | LUKS + LVM, configured by the installer |
| Data disk | 1 TB USB SSD, LUKS2 + ext4, separate passphrase |
| File sharing | Samba (SMB3), plus SSHFS from mobile and desktop clients |
| Remote access | Tailscale (WireGuard-based mesh VPN) |
| Hardening | SSH keys only, `fail2ban`, `ufw`, idle-session timeout, unattended security updates |
| Secrets | KeePassXC (AES-256, Argon2id) |
| Containers | Docker and Docker Compose, kept for upcoming services |

## Security model

1. **Nothing is exposed to the internet.** There is no port forwarding. The firewall only accepts SSH and Samba on the Tailscale interface.
2. **Two encryption layers with different passphrases.** The main disk is unlocked at the console on every boot. The data disk is unlocked manually, after boot, over SSH.
3. **Manual unlock on purpose.** No key file sits on the server, so a stolen laptop or a copied disk is just ciphertext. The cost is that after a reboot someone has to type two passphrases.
4. **Least privilege on the share.** One named Samba user, no guest access, restrictive file masks. Samba also refuses connections while the encrypted disk is not mounted, so nothing can land on the wrong disk.
5. **Secrets hygiene.** Any secret that shows up in a chat, a log or a commit is treated as compromised and rotated.

## Hardware

HP Pavilion Gaming laptop (Intel i5-10300H, 8 GB RAM, 500 GB NVMe) with a 1 TB USB SSD.

## What I learned

- **Test a disk by writing to it, not by listing it.** After a USB drop, `ls` kept showing the files while every write failed with `Input/output error`. The [runbook](docs/runbooks/usb-drop-recovery.md) came from that incident.
- **Device letters are not stable.** The SSD changed from `sdb` to `sda` after a reboot, so it is now always referred to by UUID.
- **A service must never write to a mount point that is not mounted.** It silently fills the empty directory on the system disk instead. Samba now checks for a marker file on the SSD before accepting any connection.
- **Logs can leak secrets.** An HTTP client library printed full URLs, API token included, at its default log level.
