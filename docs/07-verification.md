# 07 — Verification

Every phase ends with a checkpoint. This file puts them in one place so you can re-run the whole set
after a change. Copy-paste each block; the pass condition is on the line above it.

## Windows-side, before the install

```powershell
manage-bde -status C: | Select-String "Protection|Conversion"     # Protection Off (suspended)
powercfg /a                                                       # hibernate/fast startup unavailable
Get-Partition | Select-Object PartitionNumber, DriveLetter, @{n='SizeGB';e={[math]::Round($_.Size/1GB)}}
```
Pass: an ~100 GB gap with no drive letter next to D: (~355 GB), C: unchanged (~449–450 GB).

## After the install

```bash
lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL
sudo parted -l
```
Pass: an ext4 `/` of ~100 GB, a `vfat` ESP mounted `/boot/efi`, and **two NTFS volumes with no mount
point**, sized ~449 GB (C:) and ~355 GB (D:). Nothing else changed.

```powershell
# from Windows, after the first Windows boot
manage-bde -status C: | Select-String "Protection"                # Protection On (re-enabled)
```

## Ubuntu baseline

```bash
curl -sI https://github.com | head -1                             # answers (mirror or proxy working)
nvidia-smi                                                        # lists the GPU + driver version
ls /mnt/data && ls /mnt/win 2>/dev/null                           # /mnt/data works; /mnt/win FAILS (encrypted C:)
df -h / | tail -1                                                 # root has room (install wants ~12 GB)
```

## Clock agreement (the twin-OS check)

```bash
timedatectl | grep -E 'Local time|Universal time|RTC in local TZ'
sudo hwclock --show --utc                                          # matches `date -u`
```
Pass: `RTC in local TZ:` is **no** (UTC branch) and `Universal time` equals the RTC. Then boot Windows
and confirm its clock reads the true local time. See [clock-and-rtc.md](clock-and-rtc.md).

## Hermes port

```bash
PROFILE=~/.local/share/hermes          # or ~/.hermes, whichever the installer used
hermes --version
hermes doctor
find "$PROFILE"/skills -name SKILL.md | wc -l                      # equals your Windows skill count
grep -rInF -e 'C:\' -e 'AppData' -e 'OneDrive' "$PROFILE"/memories "$PROFILE"/cron 2>/dev/null   # expect nothing
hermes cron list                                                   # job count matches; paths are Linux
hermes chat -q "Search our old sessions for a topic only you know and summarize it"
```
Pass: doctor clean, skill count matches, no Windows paths remain, old sessions are searchable (proves
`state.db` + memory + skills all landed), and a test prompt returns a real model response.

## Robotics (if you did phase 06)

```bash
ros2 pkg list | grep -Ei 'so_arm|sts_hardware'
ros2 control list_controllers                                      # both 'active'
ls -l /dev/ttySERVO; groups | grep dialout
python -c "import torch; print(torch.cuda.is_available())"         # True
```

## The single end-to-end check

Boot Ubuntu, open the vault, run a Hermes prompt that recalls memory, and command the arm from
software with the sim mirroring it. If all four work from one cold boot, the port is done.

Back to [README](../README.md).
