# NAS: Samba share on the encrypted SSD

[← Back to index](../../README.md) · [← Encryption](../../server/encryption.md) · [Remote access →](../../server/remote-access.md) · [Samba →](../../server/samba-documentation.md) 

A basic Samba share exposes the encrypted SSD as a network folder (SMB3). It is only reachable through Tailscale, with one authenticated user and no guest access. The configuration is deliberately standard.

## Install

```bash
sudo apt update
sudo apt install -y samba
sudo systemctl enable --now smbd
```

## Create the user

Samba **does not use Linux passwords**: it keeps its own user database. A Samba user must first exist as a Linux user, and is then added to Samba with a separate password:

```bash
# The Linux user already exists (it is the server account)
sudo smbpasswd -a <user>        # adds the user to Samba and asks for a Samba password
```

List the Samba users:

```bash
sudo pdbedit -L
```

Change the password later with `sudo smbpasswd <user>`.

**Where the passwords are:** never in `smb.conf`. Samba stores them in its own database (by default under `/var/lib/samba/private/`).

## Share definition

Added at the end of `/etc/samba/smb.conf`:

```ini
[zen-ssd]
   path = /mnt/zen-ssd
   read only = no
   valid users = <user>
   create mask = 0600
   directory mask = 0700
   root preexec = /usr/bin/test -f /mnt/zen-ssd/.ssd_ok
   root preexec close = yes
```

| Parameter | Purpose |
|---|---|
| `path` | The folder that is shared: the SSD, already mounted and decrypted |
| `read only = no` | Allows writing |
| `valid users` | Only that user can connect. With no `guest ok`, **guest access is not possible** |
| `create mask = 0600` | New files are readable only by their owner |
| `directory mask = 0700` | Same for new folders |
| `root preexec`, `root preexec close` | Refuse the connection if the SSD is not mounted (see below) |

The `[global]` section keeps the defaults of the Ubuntu package (`server role = standalone server`, `map to guest = Bad User`, `obey pam restrictions = Yes`...). The `[printers]` and `[print$]` sections are defaults too and are not used.

Validate and apply:

```bash
testparm -s                       # checks the syntax and prints the effective configuration
sudo systemctl reload smbd
```

## Firewall

Port 445 is only opened on the Tailscale interface:

```bash
sudo ufw allow in on tailscale0 to any port 445 proto tcp
sudo ufw status verbose
```

Expected output (next to the SSH rule):

```
445/tcp   on tailscale0   ALLOW IN   Anywhere
```

Nothing reaches Samba from the local network or from the internet.

## Connect from clients

Always use the server's Tailscale name or IP:

```
# Windows: File Explorer → "Map network drive"
\\<tailscale-name>\zen-ssd

# Windows, command line
net use Z: \\<tailscale-name>\zen-ssd /user:<user>
```

Any SMB client works on a phone. The SSD can also be reached over SSHFS (see the index).

## Verify

```bash
sudo smbstatus
```

A real connection, with the IP and PID hidden:

```
PID     Username  Group   Machine                              Protocol Version  Encryption  Signing
<pid>   <user>    <user>  <tailscale-ip> (ipv4:<ip>:<port>)    SMB3_11           -           partial(AES-128-GMAC)

Service      pid     Machine           Connected at
zen-ssd      <pid>   <tailscale-ip>    <date>
```

The connection negotiates **SMB 3.1.1** with message signing. The `Encryption` column is empty because SMB itself is not encrypting here, but the traffic already travels encrypted inside the Tailscale (WireGuard) tunnel.

## Safeguard: refuse connections while the SSD is not mounted

Samba starts with the server, but the SSD is mounted by hand. If a client connects before the SSD is mounted, it sees the empty folder on the internal disk, and anything copied there stays on the internal disk instead of the SSD.

The fix is a marker file that only exists on the SSD, plus a rule that makes Samba refuse the connection when the file is missing. The two `root preexec` lines in the share definition above do the second part. The marker is created once, with the SSD mounted:

```bash
sudo touch /mnt/zen-ssd/.ssd_ok
testparm -s
sudo systemctl reload smbd
```

With the SSD unmounted the file does not exist and the connection is refused. With the SSD mounted, everything works as before.

## Possible improvements

None of these is applied yet:

- `server min protocol = SMB3`: reject older SMB versions.
- `server smb encrypt = required`: SMB encryption on top of the tunnel.
- `interfaces = lo tailscale0` with `bind interfaces only = yes`: make Samba listen only on Tailscale (make sure `tailscaled` starts before `smbd`).
