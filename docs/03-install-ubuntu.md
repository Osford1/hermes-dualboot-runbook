# 03 — Install Ubuntu 24.04

Boot the USB you made in [02-windows-preparation.md](02-windows-preparation.md). Choose **Try Ubuntu**
first if you want to check Wi-Fi and the disk map before committing; then **Install Ubuntu**.

The one dangerous screen is manual partitioning. Everything else is a Next button. Read the size
labels before you confirm — this is the step that can wipe Windows.

## 3.1 Installer, up to partitioning

1. Language, keyboard layout.
2. Connect to Wi-Fi or plug in ethernet. Tick **Install third-party software for graphics and Wi-Fi**
   (you want the NVIDIA driver and media codecs; without it you install them later by hand).
3. **Installation type:** choose **Something else** (a.k.a. *Manual partitioning*). Do **not** pick
   "Erase disk and install Ubuntu" — that erases everything.

## 3.2 Manual partitioning (the dangerous screen)

You should see the existing NTFS partitions and the **free space** you carved. Select the free space
and click **+** to add a partition:

| Setting | Value |
| --- | --- |
| Size | the whole free space (≈ 100 GB) |
| Type for the new partition | Primary |
| Use as | **Ext4 journaling file system** |
| Mount point | `/` |

Then look at the EFI System Partition (labelled *efi* or `FAT32`, a few hundred MB at the start of
the disk):

- **Mount point: leave it as `/boot/efi`.**
- **Format: NO.** Ticking "Format" on the ESP destroys Windows' own boot files. The Ubuntu installer
  adds its loader to the existing ESP; it does not need to format it.

Nothing else is touched. In particular, the two NTFS partitions (Windows C: and data D:) must have
**no** action — no format, no mount point, no "use as". If you see them in the change summary, stop.

> **Swap:** the reference install used a swapfile, not a swap partition, so there is no separate
> swap row to create. The default "no swap partition" is fine; Ubuntu 24.04 creates a swapfile on `/`.

Before you click **Install now**, read the confirmation summary out loud. It lists exactly what will
be formatted. You should see **one** `Format` row: your new ext4 `/`. Anything else → **Go Back**.

## 3.3 Finish the installer

- Timezone: pick yours (see [clock-and-rtc.md](clock-and-rtc.md) for the clock decision — with a
  no-DST zone such as `Asia/Shanghai`, DST is irrelevant and either RTC branch works).
- Create your user. Note the username — it becomes `/home/<you>/`.
- Let the install run (~10–20 min with third-party software). Reboot when asked.
- Remove the USB stick on reboot.

## 3.4 First Windows boot — re-enable BitLocker

GRUB now owns the boot menu and lists both **Ubuntu** and **Windows Boot Manager**. Boot Windows
once and turn protection back on:

```powershell
# Boot Windows from the GRUB menu, then:
manage-bde -protectors -enable C:
manage-bde -status C: | Select-String "Protection|Conversion"
```

Expect `Protection On`. Reboot once more and confirm the key is again required/available — a suspend
that was never re-enabled leaves the volume effectively unencrypted.

You can also set the Windows partition as the default GRUB entry if you use it more:

```bash
# in Ubuntu, after first boot
sudo grub-set-default "Windows Boot Manager"   # name is whatever appears in the menu
```

## Checkpoint 3

- [ ] GRUB menu lists both **Ubuntu** and **Windows Boot Manager**.
- [ ] Windows boots normally, and `manage-bde -status C:` shows `Protection On`.
- [ ] Ubuntu boots to a login screen.
- [ ] `lsblk -f` in Ubuntu shows: an ext4 `/` of ~100 GB, a `vfat` ESP with `/boot/efi`, and **two
      NTFS volumes with no mount point** — check the sizes match C: (~449 GB) and D: (~355 GB):

  ```bash
  lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL
  sudo parted -l
  ```

- [ ] Nothing on C: or D: was formatted (their contents are intact when you boot Windows).

Next: [04-first-boot.md](04-first-boot.md) — drivers, mounts, and the encrypted-C: reality.
