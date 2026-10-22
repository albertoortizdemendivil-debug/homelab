# Remote access with Tailscale

[← Back to index](../README.md) · [← Encryption](encryption.md) · [Samba →](samba-documentation.md)

Tailscale builds a private network (a WireGuard-based mesh VPN) between the server and my devices. It is the **only way in from outside**: no port is open on the router.

## Why Tailscale instead of opening ports

There are two ways to reach a home server from outside:

| | Open ports on the router | Tailscale |
|---|---|---|
| Exposure | SSH and Samba visible on the internet, scanned by bots around the clock | Nothing visible from the internet |
| Works with CGNAT | No: the router has no public IP of its own | Yes |
| Setup | Port forwarding, static IP or dynamic DNS | Install the agent and sign in |
| Encryption | Depends on the service | Always, end to end |

**CGNAT** (Carrier-Grade NAT) is when the ISP shares one public IP between many customers. In that case opening ports is impossible even if you want to. Tailscale works through it: devices connect outwards and find each other through coordination servers. If a direct connection is not possible, traffic goes through relay servers (DERP).

To check whether your connection uses CGNAT, compare the WAN IP shown on the router's status page with the IP the internet sees (any "what is my IP" website). If they differ, or the WAN IP starts with `100.64.` to `100.127.` or is a private address, you are behind CGNAT.

## Install the agent on the server

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

`tailscale up` prints a link: open it in a browser and sign in to attach the server to your account.

Check:

```bash
tailscale status          # devices and whether they are connected
tailscale ip -4           # the server's Tailscale IP (100.x.y.z range)
systemctl is-enabled tailscaled
```

## Add the other devices

Install the Tailscale app on each device and sign in with **the same account**:

- Windows (daily laptop)
- Android (phone)

Every device gets **its own Tailscale IP** in the `100.x.y.z` range, separate from its normal network address. They all appear in `tailscale status` and can reach each other by name or by that IP.

### Note for phones

On Android, Tailscale runs as a VPN, so the phone shows the VPN (key) icon while it is connected. Two consequences:

- **Only one VPN can be active at a time on Android.** Turning on another VPN app disconnects Tailscale, and the other way round.
- **Only tailnet traffic goes through the tunnel.** By default, normal internet traffic keeps using the phone's own connection. Everything else is sent through the tunnel only if an exit node is configured, which is not the case here.

## Use it

```bash
# SSH (non-default port, key-based)
ssh -p <ssh-port> <user>@<tailscale-name>

# Samba
\\<tailscale-name>\zen-ssd
```

Samba setup: [Samba documentation](samba-documentation.md). Share overview: [NAS service](../services/nas/README.md).

To see whether a connection is direct or relayed:

```bash
tailscale ping <device-name>
```

If it answers "via DERP(...)", traffic goes through a relay (it works, with more latency). If it answers with an IP, the connection is direct.

## Firewall policy

Tailscale does not replace the firewall: the server only accepts inbound traffic on its Tailscale interface (`tailscale0`):

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0 to any port <ssh-port> proto tcp
sudo ufw allow in on tailscale0 to any port 445 proto tcp
sudo ufw enable
sudo ufw status verbose
```

Result:

```
<ssh-port>/tcp  on tailscale0   ALLOW IN   Anywhere
445/tcp         on tailscale0   ALLOW IN   Anywhere
```

Any new service (DNS, monitoring...) gets added the same way: only on `tailscale0`.

Restrict a service with the firewall and, if it supports it, by source address (`100.64.0.0/10`). Do not rely on binding the service to the `tailscale0` interface by name: Samba could not bind to it and ended up listening on localhost only. See the [Samba incident report](../docs/runbooks/incident-report-samba.md), and [Samba documentation, firewall](samba-documentation.md#7-firewall-ufw) for the working setup.

## Tips and limits

- **Key expiry.** By default each device's key expires after a while (180 days at the time of writing). It can be disabled for the server in the Tailscale admin console, so it does not disconnect on its own.
- **Dependency.** If `tailscaled` goes down, the only way in is the physical console. That is worth watching once monitoring is in place.
- **After a reboot**, the server waits for the main disk passphrase at the console before `tailscaled` can start. Until then, it cannot be reached over the network. Once it is up, unlock the external SSD as in the [daily routine](encryption.md#daily-routine).
- **Access control.** With a handful of personal devices, ACLs are not needed, but Tailscale can restrict which device may talk to which.
