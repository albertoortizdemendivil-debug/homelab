# SSH hardening and automatic updates

[← Back to index](../README.md) · [← Remote access](remote-access.md)

SSH is the only way into the server, and it is already limited to the Tailscale interface by the firewall (see [Remote access](remote-access.md)). This guide covers what the SSH service and the host do on top of that, so that a single failed layer does not mean an open door.

## Overview

| Control | Setting | What it protects against |
|---|---|---|
| Key-only authentication | `PasswordAuthentication no`, `KbdInteractiveAuthentication no` | Password guessing |
| No root login | `PermitRootLogin no` | Direct attacks on the most targeted account |
| Non-default port | `Port <ssh-port>` | Noise from automated scans. It reduces log clutter, it is not a security control on its own |
| fail2ban | Temporary bans after repeated failures | Brute-force attempts |
| Idle-session timeout | `TMOUT=900` | A forgotten open session |
| Unattended upgrades | Security patches applied automatically | Known vulnerabilities |
| Firewall | Inbound only on `tailscale0` | Any exposure to the internet |

## Keys

Use **one key per purpose**, so that losing or revoking one does not affect the others. Ed25519 keys are small, fast and the current default choice.

```bash
# On the client (laptop or phone), one key per use
ssh-keygen -t ed25519 -C "<purpose>" -f ~/.ssh/id_ed25519_<purpose>
```

Install the public key on the server **before** turning password login off:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_<purpose>.pub -p <ssh-port> <user>@<tailscale-name>
```

On Windows, where `ssh-copy-id` is not available, append the contents of the `.pub` file to `~/.ssh/authorized_keys` on the server by hand.

In this setup:

- Each client device has its own private key, used to reach the server over SSH (shell and SSHFS).
- A separate key lives on the server and is used only to push to GitHub. Its public half is registered in GitHub.
- No key is reused between services.

A client-side shortcut in `~/.ssh/config` keeps the options in one place:

```
Host homelab
    HostName <tailscale-name>
    Port <ssh-port>
    User <user>
    IdentityFile ~/.ssh/id_ed25519_<purpose>
    IdentitiesOnly yes
```

Then `ssh homelab` is enough.

## SSH daemon configuration

Settings in `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
Port <ssh-port>
```

`KbdInteractiveAuthentication no` closes a second way to type a password (keyboard-interactive through PAM), so that key-only login does not depend on one setting alone.

Two details specific to Ubuntu make it worth checking the **effective** configuration instead of trusting the file:

- `sshd_config` starts with an `Include` of `/etc/ssh/sshd_config.d/*.conf`, and for each option the **first value found wins**. A drop-in file (for example one created by cloud-init) can override your setting.
- Ubuntu 24.04 may start OpenSSH through **socket activation** (`ssh.socket`). In that case the `Port` line can be ignored until the socket is configured, for example with `sudo systemctl edit ssh.socket` and a `ListenStream=<ssh-port>` line (an empty `ListenStream=` first, to clear the default). After editing the socket, run `sudo systemctl daemon-reload && sudo systemctl restart ssh.socket`, because restarting only `ssh` does not change the listening port.

**Apply changes without locking yourself out:**

```bash
# 1. Keep one SSH session open and do the rest from a second one
# 2. Check the syntax
sudo sshd -t

# 3. Apply
sudo systemctl restart ssh

# 4. Open a NEW connection and confirm it works before closing the first
ssh -p <ssh-port> <user>@<tailscale-name>
```

**Verify the effective settings and the real listening port:**

```bash
sudo sshd -T | grep -E '^(port|passwordauthentication|kbdinteractiveauthentication|permitrootlogin|pubkeyauthentication) '
sudo ss -tlnp | grep sshd
```

## fail2ban

fail2ban reads the authentication log and bans an address for a while after repeated failures.

```bash
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
```

Local settings go in `/etc/fail2ban/jail.local`, which survives package updates. With SSH on a non-standard port, the jail must know the port, otherwise bans are applied to port 22 only:

```ini
[sshd]
enabled = true
port    = <ssh-port>
```

The package defaults apply to everything else: a ban of 10 minutes after 5 failures within 10 minutes.

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd              # jail status and banned addresses
sudo fail2ban-client set sshd unbanip <ip>    # lift a ban by hand
```

Because the firewall already restricts SSH to Tailscale, fail2ban is a second layer, not the main one.

## Idle-session timeout

File `/etc/profile.d/autologout.sh`:

```bash
TMOUT=900
readonly TMOUT
export TMOUT
```

- Any bash session, local or over SSH, is closed after **15 minutes** without activity.
- `readonly` stops a user from disabling it from inside the session.
- It only applies to interactive shells. SFTP and SSHFS connections do not use a shell, so file access is not cut.
- For long jobs, run them with `nohup` or in a terminal multiplexer so they survive the logout.

## Unattended upgrades

Security patches are applied without manual intervention.

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

By default only the security pocket of the release is enabled. The configuration is in `/etc/apt/apt.conf.d/50unattended-upgrades` and the schedule in `/etc/apt/apt.conf.d/20auto-upgrades`.

**Automatic reboot stays off** (`Unattended-Upgrade::Automatic-Reboot "false"`). The main disk is encrypted and its passphrase is typed at the console on every boot, so an unattended reboot would leave the server offline until someone typed it. After a kernel update the reboot is done by hand.

```bash
ls /var/run/reboot-required                       # exists when a reboot is pending
apt-config dump | grep -i periodic                # schedule in effect
sudo unattended-upgrade --dry-run --debug         # simulate a run
ls /var/log/unattended-upgrades/                  # logs of past runs
```

## Verification

```bash
# Services enabled at boot
systemctl is-enabled ssh fail2ban unattended-upgrades

# SSH: effective settings and listening port
sudo sshd -T | grep -E '^(port|passwordauthentication|kbdinteractiveauthentication|permitrootlogin|pubkeyauthentication) '
sudo ss -tlnp | grep sshd

# fail2ban
sudo fail2ban-client status sshd

# Idle timeout
cat /etc/profile.d/autologout.sh

# Firewall
sudo ufw status verbose
```

## Possible improvements

None of these is applied yet:

- `AllowUsers <user>`: only that account may log in over SSH.
- `MaxAuthTries 3` and a shorter `LoginGraceTime`: fewer attempts per connection.
- Run [`ssh-audit`](https://github.com/jtesta/ssh-audit) against the server to review key exchange and cipher choices.
