# Variant — Ubuntu on an external SSD

If you would rather not touch the internal disk at all, install Ubuntu to an external SSD. Same
runbook, three differences.

## 1. The ESP goes on the external disk

In the installer's manual partitioning, create **both** the ext4 `/` **and** an EFI System Partition
**on the external disk** (e.g. 512 MB, FAT32, mount `/boot/efi`), then set **"Device for boot loader
installation"** to the **external** disk. The installer will otherwise reuse the internal ESP.

After install, verify (from Ubuntu or Windows) that the **internal** ESP has no `EFI/ubuntu` folder:

```bash
ls /boot/efi/EFI            # on the internal ESP you should see Windows/, not ubuntu/
```

If `EFI/ubuntu` landed on the internal ESP, unplugging the SSD turns the Windows boot into a
`grub rescue>` prompt. Windows is undamaged, but the boot flow is broken — recreate the external ESP
and reinstall.

## 2. Performance caveats

- USB 3.x is fast enough for daily use, but external enclosures can drop off the bus under load.
- Do **not** keep a conda env or dataset hot-path on the external disk if it is USB-attached — latency
  and power management bite. Put heavy data on the shared internal NTFS volume instead.
- Thunderbolt/USB4 enclosures behave far better than plain USB 3.0 ones.

## 3. Reversibility is cleaner

Nothing on the internal disk changed: no shrink, no new partition, no GRUB written to the internal ESP.
Unplug the SSD and Windows boots exactly as before. The trade-off is you now depend on the external
disk being plugged in for Linux to appear at all.

## Checklist differences

- [ ] External ESP created **on the external disk**; bootloader device set to it.
- [ ] Internal ESP has **no** `EFI/ubuntu` folder.
- [ ] Windows boots normally with the SSD **unplugged**.
- [ ] Linux boots with the SSD plugged in.

Back to [README](../README.md).
