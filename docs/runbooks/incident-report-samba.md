# Incident Report: Samba Share Unreachable After Restricting It to Tailscale

[← Back to index](../../README.md) · Related: [Samba documentation](../../server/samba-documentation.md) · [NAS service](../../services/nas/README.md) · [Remote access](../../server/remote-access.md)

**Date:** October 21, 2026
**Server:** Ubuntu 24.04 LTS laptop, remote access over SSH and Tailscale
**Affected service:** Samba (shared folder)
**Status:** Resolved

## Summary

After restricting Samba so that it only served clients over Tailscale, the shared folder stopped being reachable. Samba started without errors, but it was not listening on the Tailscale IP address. The fix was to change the restriction method: instead of limiting the network interfaces Samba binds to, I limited which clients are allowed to connect.

## What Happened

1. For security, Samba was meant to be reachable only over Tailscale. Until then it listened on all interfaces (`0.0.0.0:445`).
2. The following lines were added to the `[global]` section of `/etc/samba/smb.conf`:
   ```ini
   interfaces = lo tailscale0
   bind interfaces only = yes
   smb ports = 445
   ```
3. The next day the shared folder no longer worked.

## Diagnosis

- `systemctl status smbd` showed `active (running)` and "ready to serve connections". The service had not failed.
- `ss -tlnp` showed port 445 **only on `127.0.0.1` and `[::1]`**. It did not appear on the Tailscale IP (`<TAILSCALE_IP>`), so no remote device could connect.
- Tailscale was running (its port was in the listing), so this was **not a startup-order problem**, as I first suspected.

## Root Cause

Samba did not bind to the `tailscale0` interface when it was given by name. The most likely reason is that Tailscale creates a virtual point-to-point interface, which Samba does not always handle well. This cause was **not confirmed with a direct test**; it was inferred from the behaviour.

With `bind interfaces only = yes`, Samba listens only on the interfaces it manages to recognise. Because it did not recognise `tailscale0`, it was left with the loopback interface only.

## What Was Tried

| Attempt | Result |
|---|---|
| Restart `smbd` | No effect; still listening only on localhost |
| Use the IP instead of the name: `interfaces = lo <TAILSCALE_IP>` | Port 445 still did not appear on that IP |
| Remove `interfaces` and `bind interfaces only`, and restrict by source address (see below) | **Worked** |

## Fix Applied

The `interfaces` and `bind interfaces only` lines were removed (or commented out), and the `[global]` section was left with:

```ini
smb ports = 445
hosts allow = 127.0.0.1 100.64.0.0/10
hosts deny = 0.0.0.0/0
```

- `smb ports = 445` disables port 139 (NetBIOS, legacy and unnecessary).
- `hosts allow` admits connections only from the server itself and from the Tailscale address range (`100.64.0.0/10`).
- `hosts deny` rejects everything else.

The configuration was then validated with `testparm -s`, Samba was restarted with `sudo systemctl restart smbd`, and the share was accessed from a PC with Tailscale enabled at `\\<TAILSCALE_IP>`.

The resulting setup is documented step by step in the [Samba documentation, section 5](../../server/samba-documentation.md#5-configuration-etcsambasmbconf).

## Security After the Change

Samba is listening on all interfaces again, but access is protected by two layers:

1. **ufw:** must allow incoming connections only on `tailscale0` (check with `sudo ufw status verbose`). See [Samba documentation, section 7](../../server/samba-documentation.md#7-firewall-ufw) and the [firewall policy](../../server/remote-access.md#firewall-policy).
2. **Samba (`hosts allow` / `hosts deny`):** rejects any source that is not Tailscale or the server itself.

The router has no open ports, so nothing is reachable from the internet.

## Lessons Learned

- A service showing `active (running)` does not mean it listens where you expect. Always verify with `ss -tlnp`.
- The column that matters in `ss` is **Local Address**, not Peer Address (which always shows `0.0.0.0:*` for listening sockets).
- With VPN interfaces such as Tailscale, **restricting by source address (`hosts allow`) is more reliable than restricting by interface**. The same applies to any new service: see [Remote access, firewall policy](../../server/remote-access.md#firewall-policy).
- Back up `smb.conf` before editing (`smb.conf.bak`) so a rollback takes a single command.
- After network or security changes, test with a full reboot and check that everything comes up the same way.

## Follow-up

- [ ] Verify `sudo ufw status verbose` (active, with rules `on tailscale0`).
- [ ] Reboot the server as a test and confirm the share is still reachable (after a reboot, unlock the SSD first: [daily routine](../../server/encryption.md#daily-routine)).
- Not needed: with `hosts allow` Samba no longer binds to the Tailscale address, so it does not have to start after `tailscaled`.

## Related Documentation

| Where | What it covers |
|---|---|
| [Samba documentation](../../server/samba-documentation.md) | The full setup that came out of this incident, and the [common problems table](../../server/samba-documentation.md#11-maintenance-and-common-problems) that points back here |
| [NAS service](../../services/nas/README.md) | Overview of the share and the [rejected `interfaces` option](../../services/nas/README.md#possible-improvements) |
| [Remote access](../../server/remote-access.md) | Tailscale and the [firewall policy](../../server/remote-access.md#firewall-policy) |
| [Disk encryption](../../server/encryption.md) | How the SSD is unlocked after a reboot |
| [USB drop recovery](usb-drop-recovery.md) | A different failure with a similar symptom (writes to the share fail), caused by the USB SSD and not by the Samba configuration |
