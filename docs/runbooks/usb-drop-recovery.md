# Runbook: recovery after a USB drop of the encrypted SSD

[← Back to index](../../README.md) · Related: [Disk encryption](../../server/encryption.md) · [Samba documentation](../../server/samba-documentation.md) · [NAS service](../../services/nas/README.md)

Use this when the external SSD disconnects and the mount stops working.

**Symptoms:** `Input/output error` on writes, applications saying `unable to open database file`, and `umount` answering "not mounted" while `mount` says the opposite. `lsblk` no longer shows the LUKS mapping under the SSD. Writes from Samba clients (the [NAS share](../../services/nas/README.md)) fail as well.

**Cause:** a momentary USB disconnect leaves the `dm-crypt` mapping orphaned. Writes that were only in cache are lost.

**Trap:** `ls` keeps listing the files with their size and date because it reads from cache. The real test is writing:

```bash
touch /mnt/zen-ssd/test && rm /mnt/zen-ssd/test
```

**Steps, in order (stop as soon as one works):**

```bash
# 1. Stop everything that uses the disk (containers, Samba...)
docker stop $(docker ps -q) 2>/dev/null
sudo systemctl stop smbd

# 2. Release the mount and close the mapping
sudo umount -l /mnt/zen-ssd
sudo cryptsetup luksClose zen_ssd

# 3. If luksClose says "still in use": check "Open count", then force it
sudo dmsetup info /dev/mapper/zen_ssd
sudo dmsetup remove --force zen_ssd

# 4. Open and mount again (letter and UUID from lsblk -f)
sudo cryptsetup luksOpen /dev/disk/by-uuid/<LUKS-UUID> zen_ssd
sudo mount /dev/mapper/zen_ssd /mnt/zen-ssd

# 5. Check the marker file and a real write, then start Samba again
ls -la /mnt/zen-ssd/.ssd_ok
touch /mnt/zen-ssd/test && rm /mnt/zen-ssd/test
sudo systemctl start smbd
```

Step 5 matters because Samba was stopped in step 1, and it refuses connections without the `.ssd_ok` marker ([NAS service, safeguard](../../services/nas/README.md#safeguard-refuse-connections-while-the-ssd-is-not-mounted)). Start again the containers you stopped.

If nothing releases the mapping, **rebooting** is the clean and safe way out: after boot, unlock the main disk and repeat the mount.

Afterwards, run `sync` and check that the important data is still there. If the drop happens again, check the cable and the USB port first. To confirm that Samba clients are back, see [NAS service, verify](../../services/nas/README.md#verify). If Samba is running but unreachable over Tailscale, that is a different problem: see the [Samba incident report](incident-report-samba.md).
