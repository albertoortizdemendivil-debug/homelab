# Documentation: Samba Server Accessible Only Through Tailscale

**Server:** laptop running Ubuntu 24 LTS, managed over SSH
**Shared resource:** `zen-ssd` (external drive mounted at `/mnt/zen-ssd`)
**Access:** only through Tailscale, using the `zen` user

## Goal

Share the server's drive as a network folder with these conditions:

- Reachable only from devices on my Tailscale network.
- If the drive is not mounted, the folder is not shared (so nothing gets written by mistake to the system disk).
- Only the `zen` user can log in, and anything created stays private.

## 1. Requirements

- Ubuntu with Tailscale installed and connected (`tailscale status`).
- Drive mounted at `/mnt/zen-ssd`.
- A Linux user (`zen`) who owns that folder.

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

Do not delete this file. With the drive mounted, it only needs to be created once.

## 4. Samba user

Samba has its own password, separate from the Linux one. The user must already exist on the system.

```bash
sudo smbpasswd -a zen
```

Use a strong password, different from the SSH one.

## 5. Configuration (`/etc/samba/smb.conf`)

Back up the file before editing:

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
sudo nano /etc/samba/smb.conf
```

### `[global]` section

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

> `interfaces = lo tailscale0` with `bind interfaces only = yes` is not used. Samba did not bind to the Tailscale interface and only listened on localhost (see `informe-incidencia-samba.md`).

### Share definition

```ini
[zen-ssd]
    path = /mnt/zen-ssd
    read only = No
    valid users = zen
    create mask = 0600
    directory mask = 0700
    root preexec = /usr/bin/test -f /mnt/zen-ssd/.ssd_ok
    root preexec close = Yes
```

| Option | Purpose |
|---|---|
| `path` | Folder being shared. |
| `read only = No` | Allows writing (read and write access). |
| `valid users = zen` | Only `zen` can access the share. |
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

The rule must show `on tailscale0`. If it says `Anywhere` with no interface, the port is open to the whole local network.

## 8. Connecting from a client

With Tailscale active on the device:

- **Windows:** in File Explorer, type `\\100.120.51.42\zen-ssd` (or map a network drive).
- **User:** `zen`, with the Samba password.

Connect using the Tailscale IP, not the local network IP.

## 9. Verification

```bash
ss -tlnp | grep 445
systemctl status smbd
```

- Port 445 must appear as listening.
- `smbd` must be `active (running)`.
- From a device without Tailscale (for example, on mobile data) the folder must not be reachable.

## 10. Security layers

1. **Router:** no ports open to the internet.
2. **ufw:** allows incoming traffic only on `tailscale0`.
3. **Samba (`hosts allow/deny`):** rejects any source that is not Tailscale or the server itself.
4. **User (`valid users`):** only `zen`, with its own password.
5. **Permissions (`create mask` / `directory mask`):** anything created stays private.
6. **`.ssd_ok` marker:** nothing is shared if the drive is not mounted.

## 11. Maintenance and common problems

| Symptom | Likely cause | What to do |
|---|---|---|
| Connects but the folder closes on entry | Drive not mounted or `.ssd_ok` missing | `ls -la /mnt/zen-ssd/.ssd_ok`, and check the mount with `df -h`. |
| Cannot connect from outside home | Tailscale inactive on the client or server | `tailscale status` on both. |
| Cannot connect from the local network without Tailscale | Expected: only Tailscale is allowed | Use Tailscale or the `100.x.x.x` IP. |
| Samba active but 445 not listed | Broken configuration | `testparm -s` and review `smb.conf`. |
| Not working after a reboot | Samba started before the drive or Tailscale | Restart `smbd`; see the next section. |

### Recommended improvements

- **Mount with `nofail` in `/etc/fstab`**, so the laptop boots even if the drive does not respond.
- **Make Samba wait for Tailscale** (`sudo systemctl edit smbd`):
  ```ini
  [Unit]
  After=tailscaled.service
  Wants=tailscaled.service
  ```
- **Test reboot** after any network or security change, to confirm everything starts on its own.

## 12. Reverting

```bash
sudo cp /etc/samba/smb.conf.bak /etc/samba/smb.conf
sudo systemctl restart smbd
```
