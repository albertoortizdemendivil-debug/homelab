# 1. Disk encryption (LUKS)

[← Back to index](README.md) · [2. Samba →](2-Samba.md)

The server has two independent encryption layers, each with its own passphrase:

| Disk | Encryption | How it is unlocked |
|---|---|---|
| Main disk (NVMe) | LUKS + LVM, created by the installer | Typed at the physical console on every boot |
| External SSD (USB, 1 TB) | LUKS2 + ext4 | By hand over SSH, after the server has booted |

## 1.1 Main disk

Configured during the Ubuntu Server 24.04 installation, with no commands:

1. *Storage configuration* → **Use an entire disk**.
2. Tick **Set up this disk as an LVM group**.
3. Tick **Encrypt the LVM group with LUKS** and choose the passphrase.

Result (`lsblk`, trimmed):

```
nvme0n1
├─nvme0n1p1   1G     /boot/efi        unencrypted (needed to boot)
├─nvme0n1p2   2G     /boot            unencrypted
└─nvme0n1p3   474G   crypto_LUKS
  └─dm_crypt-0
    └─ubuntu--vg-ubuntu--lv   100G   /
```

Only `/boot` and the EFI partition are unencrypted. Everything else lives inside the LUKS container.

**Decisions:**

- The passphrase is typed at the console on every boot. I did not set up remote unlock (`dropbear-initramfs`): the server is rarely far from me, and it would add attack surface.
- This passphrase is **not in any password manager**. It is memorised and written on paper. It is needed before anything boots, so it cannot depend on a file stored on the server itself.
- Choose a passphrase you can type on **any keyboard layout**. Avoid symbols that move between layouts (`~`, `}`, `\`, `|`). On one occasion I could not type mine on the server's console for exactly that reason.

Checks:

```bash
lsblk
sudo cryptsetup status dm_crypt-0
sudo cryptsetup luksDump /dev/nvme0n1p3 | head -20
```

## 1.2 External SSD (LUKS2)

> ⚠️ `luksFormat` and `mkfs` **destroy everything** on the partition. Check the device with `lsblk` before running anything.

The SSD has two partitions: a small 16 MB one and a large one of about 931 GB. The encryption goes on the large one (`/dev/sdX2` in these commands).

```bash
# 1. Identify the right device
lsblk -f

# 2. Encrypt the partition with LUKS2 (asks for the passphrase twice)
sudo cryptsetup luksFormat --type luks2 /dev/sdX2

# 3. Open it: it appears as /dev/mapper/zen_ssd
sudo cryptsetup luksOpen /dev/sdX2 zen_ssd

# 4. Create the filesystem inside the encrypted volume
sudo mkfs.ext4 /dev/mapper/zen_ssd

# 5. Mount it
sudo mkdir -p /mnt/zen-ssd
sudo mount /dev/mapper/zen_ssd /mnt/zen-ssd
```

Checks:

```bash
sudo cryptsetup status zen_ssd         # shows LUKS2, the cipher and the device
findmnt /mnt/zen-ssd                   # confirms the mount
```

**Passphrase:** different from the main disk's. This one is kept in KeePassXC (a `.kdbx` file with AES-256 and Argon2id on my daily laptop), because it is typed over SSH and not before boot.

**Why it is not in `crypttab` or `fstab`:** mounting it automatically would mean storing a key file on the server or typing a second passphrase at the console on every boot. I chose manual unlocking: it is more secure, and the server is almost always reachable over SSH.

## 1.3 Daily routine

The device letter changes between boots (`sdb` became `sda`), so the disk is always opened **by UUID**.

After every reboot:

```bash
lsblk -f                                          # find the LUKS partition and its UUID
sudo cryptsetup luksOpen /dev/disk/by-uuid/<LUKS-UUID> zen_ssd
sudo mount /dev/mapper/zen_ssd /mnt/zen-ssd
```

Before unplugging the SSD:

```bash
sudo umount /mnt/zen-ssd
sudo cryptsetup luksClose zen_ssd
```

Windows cannot read LUKS. If the SSD is plugged in there and Windows offers to format it, the answer is **no**.

## 1.4 Recovery after a USB drop

**Symptoms:** `Input/output error` on writes, applications saying `unable to open database file`, and `umount` answering "not mounted" while `mount` says the opposite. `lsblk` no longer shows the LUKS mapping under the SSD.

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
```

If nothing releases the mapping, **rebooting** is the clean and safe way out: after boot, unlock the main disk and repeat the mount.

Afterwards, run `sync` and check that the important data is still there. If the drop happens again, check the cable and the USB port first.

## 1.5 Pending improvements

- **LUKS header backup.** If the header is corrupted, the data is unrecoverable even with the right passphrase:

  ```bash
  sudo cryptsetup luksHeaderBackup /dev/sdX2 --header-backup-file zen_ssd-header.img
  ```

  Store it off the server and off the SSD. Anyone holding that file plus an old passphrase could open the volume, so protect it like a password.
- **A second copy of the data** on another disk or location. A single SSD is a single point of failure.
