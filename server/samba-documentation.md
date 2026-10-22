# Documentation: Samba Server Accessible Only Through Tailscale

[← Back to index](../README.md) · Related: [Disk encryption](encryption.md) · [Remote access](remote-access.md) · [NAS service](../services/nas/README.md)

**Server:** laptop running Ubuntu 24 LTS, managed over SSH
**Shared resource:** `zen-ssd` (encrypted external drive mounted at `/mnt/zen-ssd`)
**Access:** only through Tailscale, using the `<user>` Samba user

This is the full, step-by-step setup. The [NAS service page](../services/nas/README.md) gives the overview and how to verify a connection. If something breaks, start at [Maintenance and common problems](#11-maintenance-and-common-problems), or read the [incident report](../docs/runbooks/incident-report-samba.md) behind the source restriction used here.

## Goal

Share the server's drive as a network folder with these conditions:

- Reachable only from devices on my Tailscale network.
- If the drive is not mounted, the folder is not shared (so nothing gets written by mistake to the system disk).
- Only the `<user>` user can log in, and anything created stays private.

## 1. Requirements

- Ubuntu with Tailscale installed and connected (`tailscale status`). Setup: [Remote access](remote-access.md).
- External SSD unlocked and mounted at `/mnt/zen-ssd`. Procedure: [Disk encryption, daily routine](encryption.md#daily-routine).
- A Linux user (`<user>`) who owns that folder.

## 2. Installation

```bash
sudo apt update
sudo apt install samba
```

## 3. Drive marker file

The share checks that a file named `.ssd_ok` exists inside the drive. If the drive is not mounted, that file is missing and Samba rejects the connection.

```bash
touch /mnt/zen-ssd/.ssd_ok
```

The file lives inside the encrypted volume, so create it with the SSD unlocked and mounted. Do not delete it. It only needs to be created once. Why this safeguard exists: [NAS service, safeguard](../services/nas/README.md#safeguard-refuse-connections-while-the-ssd-is-not-mounted).

## 4. Samba user

Samba has its own password, separate from the Linux one. The user must already exist on the system.

```bash
sudo smbpasswd -a <user>
```

Use a strong password, different from the SSH one.

## 5. Configuration (`/etc/samba/smb.conf`)

Back up the file before editing:

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
sudo nano /etc/samba/smb.conf
```

### `[global]` section

These lines are added on top of the Ubuntu package defaults described in the [NAS service page](../services/nas/README.md#share-definition):

```ini
[global]
    smb ports = 445
    hosts allow = 127.0.0.1 100.64.0.0/10
    hosts deny = 0.0.0.0/0
```

| Option | Purpose |
|---|---|
| `smb ports = 445` | Uses only port 445 and disables port 139 (NetBIOS, old and unnecessary). |
| `hosts allow` | Accepts connections only from the server itself and the Tailscale range (`100.64.0.0/10`). |
| `hosts deny = 0.0.0.0/0` | Rejects any other source. |

> `interfaces = lo tailscale0` with `bind interfaces only = yes` is deliberately **not** used. Samba did not bind to the Tailscale interface and only listened on localhost. See the [incident report](../docs/runbooks/incident-report-samba.md).

### Share definition

```ini
[zen-ssd]
    path = /mnt/zen-ssd
    read only = No
    valid users = <user>
    create mask = 0600
    directory mask = 0700
    root preexec = /usr/bin/test -f /mnt/zen-ssd/.ssd_ok
    root preexec close = Yes
```

| Option | Purpose |
|---|---|
| `path` | Folder being shared. |
| `read only = No` | Allows writing (read and write access). |
| `valid users` | Only that user can access the share. |
| `create mask = 0600` | New files can only be read and written by their owner. |
| `directory mask = 0700` | New folders are accessible only by their owner. |
| `root preexec` | Runs a command as root on connection. Here it checks that `.ssd_ok` exists. |
| `root preexec close = Yes` | If the command fails (drive not mounted), the connection is closed. |

## 6. Validate and restart

```bash
testparm -s
sudo systemctl restart smbd
```

`testparm -s` should print the configuration without errors.

## 7. Firewall (ufw)

Allow port 445 only from the Tailscale interface:

```bash
sudo ufw allow in on tailscale0 to any port 445 proto tcp
sudo ufw status verbose
```

The rule must show `on tailscale0`. If it says `Anywhere` with no interface, the port is open to the whole local network. The full policy, SSH included, is in [Remote access, firewall policy](remote-access.md#firewall-policy).

## 8. Connecting from a client

With Tailscale active on the device:

- **Windows:** in File Explorer, type `\\<tailscale-name>\zen-ssd` (or map a network drive).
- **User:** `<user>`, with the Samba password.

Connect using the Tailscale name or IP (`tailscale ip -4` on the server), not the local network address.

## 9. Verification

```bash
ss -tlnp | grep 445
systemctl status smbd
```

- Port 445 must appear as listening.
- `smbd` must be `active (running)`.
- From a device without Tailscale (for example, on mobile data) the folder must not be reachable.

To see a real connection, use `sudo smbstatus` ([NAS service, verify](../services/nas/README.md#verify)).

## 10. Security layers

1. **Router:** no ports open to the internet.
2. **ufw:** allows incoming traffic only on `tailscale0`.
3. **Samba (`hosts allow/deny`):** rejects any source that is not Tailscale or the server itself.
4. **User (`valid users`):** only `<user>`, with its own password.
5. **Permissions (`create mask` / `directory mask`):** anything created stays private.
6. **`.ssd_ok` marker:** nothing is shared if the drive is not mounted.
7. **Disk encryption:** the data is ciphertext when the SSD is locked ([Disk encryption](encryption.md)).

## 11. Maintenance and common problems

| Symptom | Likely cause | What to do |
|---|---|---|
| Connection refused or the share closes on entry | SSD not unlocked or mounted, or `.ssd_ok` missing | `findmnt /mnt/zen-ssd` and `ls -la /mnt/zen-ssd/.ssd_ok`. After a reboot, [unlock and mount the SSD](encryption.md#daily-routine). No need to restart `smbd`. |
| Reads work but writes fail with `Input/output error`, or files seem to vanish | The USB SSD dropped and the mapping is orphaned | Follow the [USB drop recovery runbook](../docs/runbooks/usb-drop-recovery.md). |
| `smbd` is `active (running)` but port 445 only appears on `127.0.0.1` / `[::1]`, and clients cannot connect | `interfaces = lo tailscale0` and `bind interfaces only = yes` in `smb.conf` | Remove those lines and use `hosts allow` / `hosts deny`. Full story: [incident report](../docs/runbooks/incident-report-samba.md). |
| Cannot connect from outside home | Tailscale inactive on the client or the server, or the node key expired | `tailscale status` on both. See [Remote access, tips and limits](remote-access.md#tips-and-limits). |
| Cannot connect from the local network without Tailscale | Expected: only Tailscale is allowed | Use Tailscale and the Tailscale name or IP. |
| Not working after a reboot | The main disk waits for its passphrase at the console, so `tailscaled` is not running yet, and the SSD is unlocked by hand | Type the main disk passphrase at the console, then [unlock and mount the SSD](encryption.md#daily-routine). |
| `testparm -s` reports errors, or Samba does not start | Broken configuration | Fix the reported line, or [revert](#12-reverting) to the backup. |

### Notes

- **Do not add the SSD to `/etc/fstab`.** It is unlocked and mounted by hand on purpose: [why](encryption.md#external-ssd-luks2).
- **Samba does not need to start after Tailscale.** It no longer binds to a Tailscale address (that was the cause of the [incident](../docs/runbooks/incident-report-samba.md)), so the start order does not matter.
- **Test reboot** after any network or security change: reboot, type the main disk passphrase at the console, reach the server over Tailscale, unlock the SSD, then connect to the share.
- **Hardening not applied yet** (`server min protocol`, `server smb encrypt`): see [NAS service, possible improvements](../services/nas/README.md#possible-improvements).

## 12. Reverting

```bash
sudo cp /etc/samba/smb.conf.bak /etc/samba/smb.conf
sudo systemctl restart smbd
```
