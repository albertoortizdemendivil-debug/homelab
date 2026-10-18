# Disk encryption (LUKS)

[← Back to index](../README.md) · [Remote access →](remote-access.md)

The server has two independent encryption layers, each with its own passphrase:

| Disk | Encryption | How it is unlocked |
|---|---|---|
| Main disk (NVMe) | LUKS + LVM, created by the installer | Typed at the physical console on every boot |
| External SSD (USB, 1 TB) | LUKS2 + ext4 | By hand over SSH, after the server has booted |

## Main disk

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

## External SSD (LUKS2)

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

## Daily routine

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

## Recovery after a USB drop

If the SSD disconnects for a moment, the mount can be left orphaned and every write fails with `Input/output error`. The step-by-step procedure is in the [USB drop recovery runbook](../docs/runbooks/usb-drop-recovery.md).

## Pending improvements

- **LUKS header backup.** If the header is corrupted, the data is unrecoverable even with the right passphrase:

  ```bash
  sudo cryptsetup luksHeaderBackup /dev/sdX2 --header-backup-file zen_ssd-header.img
  ```

  Store it off the server and off the SSD. Anyone holding that file plus an old passphrase could open the volume, so protect it like a password.
- **A second copy of the data** on another disk or location. A single SSD is a single point of failure.
