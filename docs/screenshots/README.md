# Screenshots

Images referenced by the runbook go here. The runbook currently ships **without** screenshots — the
text is written to stand on its own, and each call-out in `docs/owner-todos.md` says exactly which shot
would improve which section. Names to use when you add them:

| File | Shows | Linked from |
| --- | --- | --- |
| `shrink-disk-management.png` | Disk Management: ~100 GB Unallocated next to D:, C: unchanged | `02-windows-preparation.md` |
| `bitlocker-suspended.png` | `manage-bde -status C:` → Protection **Off** | `02-windows-preparation.md` |
| `installer-partitioning.png` | "Something else" — ext4 `/` + existing ESP, no Windows partition formatted | `03-install-ubuntu.md` |
| `grub-boot-menu.png` | GRUB listing Ubuntu + Windows Boot Manager | `03-install-ubuntu.md` |
| `mount-c-win-fails.png` | Terminal: `mount … /mnt/win` fails (C: encrypted) | `04-first-boot.md` |
| `clocks-agree.png` | Ubuntu `timedatectl` + Windows tray clock, same time | `clock-and-rtc.md` |
| `rviz-mujoco-twin.png` | RViz next to MuJoCo following the same command | `06-robotics-stack.md` |
| `hero-dualboot.png` | GRUB menu (or dual-boot desktop) for the README first screen | `README.md` |

Keep PNGs reasonable (≤ ~300 KB each; scale to ~1600 px wide). No personal data, hostnames, usernames,
serial numbers, or license keys visible in any shot.
