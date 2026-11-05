# 01 — Risk gate and backup

**Read this before any command that changes a disk.** The two places this job destroys data are the
installer's manual partitioner and a BitLocker-protected C: that Windows will not let Linux mount.
Get either wrong and you lose a filesystem, not a setting.

There is no "undo" for a formatted partition. Everything below is written so that the worst case is
"restore from backup", never "reinstall Windows from scratch".

## What can be lost, and what prevents it

| Step | What is at risk | Reversal | Prevention |
| --- | --- | --- | --- |
| Shrinking D: | Nothing (a shrink is non-destructive) — but an interrupted shrink can corrupt the volume | Re-extend D: into the freed space in `diskmgmt.msc` | Fully update Windows first; suspend BitLocker; take the backup |
| Bootloader change during install | Booting Windows (GRUB takes over the EFI boot order) | GRUB's menu still lists Windows; `efibootmgr` can restore order | Install Ubuntu **after** the shrink, into unallocated space, and never format the ESP |
| Manual partitioning | **Any** filesystem you tick "Format" on | None | Read the size labels before confirming; the new ext4 row is the *only* row you format |
| Reading C: from Linux | Nothing (it just fails) — but people then try dangerous workarounds | n/a | Accept that an encrypted C: is unreadable from Linux; use the staged payload instead |
| Post-install hibernation | Ubuntu waking to a corrupt NTFS if Windows hibernated mid-write | `ntfsfix` (data loss possible) | Keep Windows "Fast Startup" off and always shut Windows down fully before dual-booting |

## The backup, and how to verify it

Do not trust "it copied". Verify.

1. **Back up the data you cannot recreate** to a disk you will physically unplug, or to an
   off-machine location. On the reference machine the irreplaceable set was: the Hermes profile
   (`%LOCALAPPDATA%\hermes`), the vault slice, and the staging payload itself.
2. **Verify by count and by size**, not by a green progress bar:

   ```powershell
   # Count and total the source, then compare to the copy
   $src = 'C:\Users\<you>\AppData\Local\hermes'
   $dst = 'E:\hermes-backup\hermes'
   (Get-ChildItem $src -Recurse -File | Measure-Object -Property Length -Sum) |
     Select-Object Count, @{n='GB';e={[math]::Round($_.Sum/1GB,2)}}
   ```

   If the copy's file count and total bytes do not match the source, the backup is not done — redo it.
3. **Unplug the backup drive** before you boot the installer. A backup that is plugged in during a
   manual-partitioning mistake is not a backup.

The reference machine's shrink freed ~100 GB in a few minutes and reported success; the guide's own
count check (`Get-ChildItem -Recurse | Measure-Object`) is the pattern to copy.

## Space cost (state it before every irreversible action)

- Shrink of D:: **−100 GB from D:, +100 GB unallocated.** D: went 455 GB → 355 GB.
- Ubuntu install: ~10–12 GB used at first boot; the rest is free for the port and the robotics stack.
- Swap: the reference install used a **2 GB swapfile** (`.swapfile`), not a swap partition, so no
  partition math and no risk of a swap partition landing on the wrong device.
- The robotics stack (ROS 2 Jazzy + LeRobot + CUDA wheels) can consume **15–25 GB**; budget for it
  before you commit the partition size.

## Three rules that keep it safe

1. **One writer per asset.** Two installers never write to the same disk at once. Windows owns the
   shrink; the Ubuntu installer owns only the unallocated space.
2. **Copy, never move.** When you stage a payload or a profile, copy it. A moved file that the copy
   step then fails on is gone.
3. **Never format a partition Windows calls "Unknown" and never `chkdsk` a partition Linux made.**
   The Linux partition has no drive letter in Windows for a reason; leave it alone.

## Credential rule

The Hermes payload contains `.env` with **live provider keys** and full session history. Handle it as
a secret:

- Keep the payload and the profile **off cloud storage and off any shippable USB** that leaves your
  control.
- When you copy the profile, the `.env` comes with it — that is intended, but it means the payload
  file is now a secret file. `chmod 600` it on Linux:

  ```bash
  chmod 600 ~/hermes-port/hermes-port-payload.tar.gz
  ```

- Never paste a key value into a README, a note, a commit, or a chat. If you must document a provider,
  name the **variable** (`DEEPSEEK_API_KEY`), never the value.

## Done means

- [ ] Backup copied, verified by file count **and** total bytes, and physically unplugged.
- [ ] BitLocker recovery key readable and stored somewhere other than the machine.
- [ ] You have run `manage-bde -status C:` and know whether C: is encrypted.
- [ ] You know which volume you are shrinking, and that it is **not** C:.

Next: [02-windows-preparation.md](02-windows-preparation.md).
