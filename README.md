# hermes-dualboot-runbook

I added a native **Ubuntu 24.04** dual boot beside **Windows 11** on my laptop and moved a working
**Hermes Agent** install onto it. This repository is the runbook I wish I had before I started: every
command, in the order I ran it, with the trap that cost me time sitting next to the fix that worked.
It is written so someone else can do the same on a different machine, not just read what I did.

## State

**Done and running.** Ubuntu 24.04 is installed in a **100 GB ext4 partition** carved from the tail of
the D: drive, Windows 11 still boots from the GRUB menu, and Hermes runs natively on both sides. The
disk and encryption facts below are the ones I measured, and the two phases that are *not* fully
verified (a complete real-arm pass, and a vision-model gate) are labelled as such instead of dressed up.

## Start here

Read in order. Each phase ends with a checkpoint you can run.

1. [docs/00-prerequisites.md](docs/00-prerequisites.md) — hardware, media, and what to gather first.
2. [docs/01-risk-and-backup.md](docs/01-risk-and-backup.md) — **the gate.** Read before any disk change.
3. [docs/02-windows-preparation.md](docs/02-windows-preparation.md) — BitLocker, fast startup, the shrink.
4. [docs/03-install-ubuntu.md](docs/03-install-ubuntu.md) — the installer, manual partitioning, GRUB.
5. [docs/04-first-boot.md](docs/04-first-boot.md) — mirrors, drivers, and the partition you *cannot* mount.
6. [docs/05-hermes-port.md](docs/05-hermes-port.md) — skills, plugins, cron, memory: what moves, what doesn't.
7. [docs/06-robotics-stack.md](docs/06-robotics-stack.md) — optional: ROS 2 + SO-101 arm + LeRobot.
8. [docs/07-verification.md](docs/07-verification.md) — every checkpoint in one place.
9. [docs/08-troubleshooting.md](docs/08-troubleshooting.md) — the real errors and their fixes.

Supporting: [docs/clock-and-rtc.md](docs/clock-and-rtc.md) · [docs/payload-and-checksums.md](docs/payload-and-checksums.md) ·
[docs/portable-external-ssd.md](docs/portable-external-ssd.md) · [docs/owner-todos.md](docs/owner-todos.md).

## The machine this was done on

Measured, not copied:

| | |
| --- | --- |
| Laptop | ASUS ROG Zephyrus G14 (Ryzen 7 7735HS) |
| Graphics | AMD Radeon iGPU + NVIDIA RTX 3050 Laptop GPU (hybrid) |
| Disk | 932 GB NVMe SSD, **GPT / UEFI**, Secure Boot **on** |
| Windows | 11, build 26100; sign-in with a Microsoft account |
| **C:** | ~450 GB, **about 87% full** — holds Windows. **Not touched, not shrunk.** |
| **D:** | NTFS data volume, ~455 GB → **~355 GB** after the shrink (plain NTFS, readable from both OSes) |
| **Encryption** | **BitLocker ON** for C: — XTS-AES, "Used Space Only", TPM + numerical password |
| Ubuntu | 24.04 LTS (noble), ext4 `/` = **100 GB**, partition 8 on the internal disk |
| Payloads | staged at `D:\hermes-port` (the Hermes profile) and `D:\so101-ubuntu` (the robotics stack) |
| Clock | RTC set to **UTC** with `RealTimeIsUniversal=1`, so Windows and Linux agree |

Three facts decide the whole shape of the job: **C: is nearly full and must not be resized**, **C: is
BitLocker-encrypted and therefore unreadable from Linux**, and **the free space has to come from D:,
shrunk from inside Windows.** Get those wrong and this is a data-loss incident, so
[01-risk-and-backup.md](docs/01-risk-and-backup.md) comes before any command.

## The three rules

1. **One writer per asset.** Only one OS touches a shared vault or dataset at a time. NTFS is writable
   from Linux; ext4 is invisible to Windows; neither is a network filesystem.
2. **Copy, don't move.** The Windows install stays intact and bootable the whole time. Every payload
   crosses the boundary as a *copy*, verified by checksum.
3. **Never format a Windows partition, never `chkdsk` an ext4 one, never run `disklabel` on the ESP.**
   The installer's "Something else" screen is the only place this can go wrong, and it goes wrong
   quietly.

## What transfers to Linux, and what does not

| Hermes profile part | Moves? | Note |
| --- | --- | --- |
| `memories/`, `skills/` | ✅ | content only |
| `cron/jobs.json` | ✅ | **re-path every job** after the move |
| `config.yaml`, `SOUL.md`, `auth.json` | ✅ | diff `config.yaml` against the fresh install |
| `sessions/`, `state.db` | ✅ | snapshot the DB with the SQLite backup API |
| `plugins/`, `desktop-plugins/`, `pets/` | ✅ | |
| `.env` | ✅ | **live API keys** — keep off cloud sync |
| venvs, `node_modules`, `bin/` launchers | ❌ | **rebuild**, don't copy |
| `state.db` histories | ❌ | per-machine; they do **not** merge — and only **one** machine may poll a chat bot at a time |

Full detail and the commands: [docs/05-hermes-port.md](docs/05-hermes-port.md).

## Repository layout

```
hermes-dualboot-runbook/
├── README.md
├── CREDITS.md
├── LICENSE
├── docs/
│   ├── 00-prerequisites.md
│   ├── 01-risk-and-backup.md
│   ├── 02-windows-preparation.md
│   ├── 03-install-ubuntu.md
│   ├── 04-first-boot.md
│   ├── 05-hermes-port.md
│   ├── 06-robotics-stack.md
│   ├── 07-verification.md
│   ├── 08-troubleshooting.md
│   ├── clock-and-rtc.md
│   ├── payload-and-checksums.md
│   ├── portable-external-ssd.md
│   ├── owner-todos.md
│   └── screenshots/README.md
└── .gitignore
```

## What is verified and what is not

| Claim | How it is grounded |
| --- | --- |
| Dual boot installed; Windows and Ubuntu both boot from GRUB | ✅ executed and observed |
| 100 GB ext4 `/` carved from D:; C: untouched | ✅ executed and observed |
| BitLocker suspend/re-enable sequence | ✅ executed and observed |
| Hermes profile ported and running natively | ✅ executed and observed |
| Clock/UTC fix (`RealTimeIsUniversal=1`) | ✅ applied on the machine |
| Robotics gates 1–2 (controller stack, RViz, mock) | ✅ pre-verified in simulation |
| Robotics gate 3b (real arm), gate 4 (vision) | ⚠️ **checklist only** — not claimed verified here |
| Screenshots referenced by the docs | ⚠️ **not yet added** — see [docs/owner-todos.md](docs/owner-todos.md) |

## What this is not

Not a script suite — it is a **method**. Machine-specific paths are written as
`C:\Users\<you>\…` / `~/…` placeholders, but every command is left copy-pasteable. The reference
machine's numbers (sizes, free space, counts) are labelled as such; substitute your own.

## Credits

Written by a graduate researcher working on precision motion control, who needed the same laptop to run
Windows engineering tools and a native Linux robotics + agent stack. If you reuse or adapt this, a link
back is appreciated.

**Hermes skills the work actually used** (from the Hermes library):

- **`windows-linux-dual-boot`** — the disk/partition/encryption reconnaissance, install-media
  discipline (ISO internals, DD vs ISO mode, Ventoy limits), installer choices (never format the ESP),
  reversibility, the external-drive variant, and the clock-skew branch handling.
- **`ubuntu-dualboot-hermes-port`** — the pre-flight checks, the finding that a BitLocker `C:` cannot be
  mounted from Linux, the Hermes-profile port (what moves, what is rebuilt, re-pathing cron), the
  "stage the payload twice" rule, and shipping with checksums and a runbook.

Cited by the source notes and followed here: `lerobot-environment-setup` (the pip/torch pin gotcha),
`hermes-profile-migration` (payload mechanics, scoped ports, re-syncing).

**Upstream projects this runbook is built on** — all credit to their authors:
[`TheRobotStudio/SO-ARM100`](https://github.com/TheRobotStudio/SO-ARM100) (the SO-101 arm),
[`adityakamath/so_arm_ros2`](https://github.com/adityakamath/so_arm_ros2) and
`sts_hardware_interface` (the ROS 2 control stack and Feetech driver),
[`huggingface/lerobot`](https://github.com/huggingface/lerobot) (LeRobot / HIL-SERL),
[`google-deepmind/mujoco`](https://github.com/google-deepmind/mujoco) (simulation), and
[hermes-agent.nousresearch.com](https://hermes-agent.nousresearch.com/docs) (Hermes Agent).
Tooling: [Rufus](https://rufus.ie/), the
[Tsinghua TUNA mirrors](https://mirrors.tuna.tsinghua.edu.cn/), and
[`hf-mirror.com`](https://hf-mirror.com).

Full credit list and links: [CREDITS.md](CREDITS.md).

Licensed **MIT** — see [LICENSE](LICENSE).
