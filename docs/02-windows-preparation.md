# 02 — Windows-side preparation

All of this happens in Windows, **before** you boot the Ubuntu installer. The order matters: the
BitLocker suspend must be the last thing before the install, and the shrink must happen while
protection is suspended.

Throughout, `D:` is the **data volume you are shrinking** — it is not C:. Substitute your own letter.

## 2.1 Take the BitLocker recovery key, then suspend protection

C: is almost certainly encrypted on a modern laptop. You cannot shrink a volume with an active
BitLocker protector from a non-Windows environment, and the installer changes the bootloader, so
protection is suspended for the duration.

```powershell
# 1. Confirm state and find where the key is backed up
manage-bde -status C:

# 2. Backup the key somewhere physical/off-machine, then:
manage-bde -protectors -get C:      # read the Numerical Password out; store it off the machine

# 3. Suspend protection (do this immediately before the install, not days ahead)
manage-bde -protectors -disable C:
```

`manage-bde -protectors -disable C:` drops the TPM protector for the current session and the next
reboot clears it. Re-run this **status** check right before you boot the USB:

```powershell
manage-bde -status C: | Select-String "Protection|Conversion"
```

Expect `Protection Off` once suspended. On a machine without BitLocker (or with it off), these
commands are harmless no-ops.

> **Why suspend, not decrypt.** Decrypting C: takes an hour and rewrites the whole volume — more risk,
> not less. A short suspension window around the bootloader change is the smaller exposure. Re-enable
> it on your first Windows boot after the install (see [03-install-ubuntu.md](03-install-ubuntu.md)).

## 2.2 Turn off Fast Startup and hibernation

Windows "Fast Startup" writes a hibernation file and leaves NTFS in a state Linux cannot safely mount.
Hibernation also blocks a clean shrink in some cases.

```powershell
powercfg /h off
powercfg /a
```

`powercfg /a` should now report that hibernate and fast startup are unavailable. You can turn
hibernation back on after the port if you want — but if you do, always **shut Windows down fully**
before booting Ubuntu, never leave it hibernated.

## 2.3 Shrink D: to carve the Ubuntu partition

Do this in the Windows Disk Management GUI, where you can see the map.

1. `Win + X` → **Disk Management** (`diskmgmt.msc`).
2. Confirm the layout: C: (Windows), D: (data), and no free-space surprises.
3. Right-click **D:** → **Shrink Volume**.
4. Enter the amount in MB: **100 GB = `102400` MB**. Take it from the **tail** (Windows shrinks from
   the end), so the new unallocated block sits to the right of D:.
5. Wait. The reference machine reported success in a few minutes. **If it reports an error, stop** —
   do not force it. Free up space on D:, defragment if it is a spinning disk, or shrink less.
6. You should now see **Unallocated** ≈ 100 GB at the far right of the disk map.

After the shrink, `Get-Partition` shows D: at ~355 GB and an unallocated gap of ~100 GB. That gap is
the Ubuntu partition; the installer will claim it.

> **Do not create the partition here.** Leave it unallocated. The Ubuntu installer creates the ext4
> filesystem; a Windows-created partition just confuses the manual partitioner.

## 2.4 Write the Ubuntu 24.04 installer USB

Two ways; both work.

**Rufus (single-OS USB).**

1. Download the Ubuntu 24.04 LTS desktop ISO from <https://releases.ubuntu.com/24.04/>.
2. Download Rufus from <https://rufus.ie/> (the reference machine used `rufus-4.15.exe`).
3. Rufus → select the USB stick → select the ISO → **Partition scheme: GPT**, **Target: UEFI (non
   CSM)** → **Start**.
4. When Rufus offers the "ISO Image / DD Image" choice, **ISO mode** is correct for Ubuntu; it keeps
   the USB readable from Windows.
5. **This erases the USB stick** — confirm it holds nothing you need.

**Ventoy (multi-ISO USB, if you want to keep other ISOs).**

1. Install Ventoy to the USB stick, then copy the Ubuntu ISO onto the data partition.
2. Boot it and pick the ISO from the Ventoy menu. No re-flashing when you change ISOs.

## 2.5 Prepare the BIOS/UEFI, then boot

1. Update the BIOS and Windows fully. A firmware update after a dual-boot install can re-order boot
   entries and confuse you; do it now.
2. Enter the UEFI setup (usually `F2` on ASUS at power-on).
3. Confirm: **UEFI boot enabled**, **Secure Boot On** (Ubuntu 24.04 is signed and boots with it on) or
   Off if your install needs it off — record which. **Fast Boot** in firmware: turn it off so the USB
   is visible.
4. Set the USB stick first in the boot order, or use the one-time boot menu (`F8`/`Esc` on ASUS).
5. **Suspend BitLocker now** if you have not (`manage-bde -protectors -disable C:`), then reboot into
   the USB.

## Checkpoint 2

- [ ] `manage-bde -status C:` shows `Protection Off`.
- [ ] `powercfg /a` shows fast startup/hibernate unavailable.
- [ ] Disk Management shows ~100 GB **Unallocated** at the right of the disk map.
- [ ] `Get-Partition` shows D: ≈ 355 GB and no new drive letter.
- [ ] Ubuntu 24.04 USB boots and reaches the "Try or Install Ubuntu" screen.
- [ ] You know your Wi-Fi password and your Ubuntu username.

Next: [03-install-ubuntu.md](03-install-ubuntu.md).
