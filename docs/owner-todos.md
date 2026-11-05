# Owner TODOs

Open items for whoever maintains this runbook. None block the text; they make it better. Each names the
file and section so it can be picked up without re-reading everything.

## Screenshots / captures to add

Everything below is a place where a picture beats prose. Put images in `docs/screenshots/` and link them
from the named file. See `docs/screenshots/README.md` for the exact filenames.

- **`docs/02-windows-preparation.md` — the shrink step.** A **Disk Management** capture showing the
  ~100 GB **Unallocated** block immediately after shrinking D:, with C: unchanged. This is the single
  most useful image in the repo: it is what tells a reader "the gap is at the end, C: is untouched".
- **`docs/02-windows-preparation.md` — BitLocker state.** `manage-bde -status C:` showing
  **Protection Off** (suspended), captured *after* `-protectors -disable` and *before* the install.
- **`docs/03-install-ubuntu.md` — the manual partitioner.** The "Something else" screen with the free
  space selected, one ext4 `/` and the existing ESP assigned — no formatting of any Windows partition.
- **`docs/03-install-ubuntu.md` — boot menu.** The GRUB screen listing both Ubuntu and Windows Boot
  Manager, right after install.
- **`docs/04-first-boot.md` — the dead C: mount.** The terminal output of the `mount … /mnt/win` failure
  that proves C: is unreadable from Linux.
- **`docs/clock-and-rtc.md` — the twin clocks.** Side by side: Ubuntu `timedatectl` and the Windows tray
  clock, both showing the same correct time after the fix.
- **`docs/06-robotics-stack.md` — the twin arm.** RViz next to the MuJoCo viewer following the same
  commanded pose. Great hero image for the robotics section.
- **`docs/screenshots/` — a repo hero.** Something for the README first screen: the GRUB menu, or the
  dual-boot desktop.

## Text fixes / follow-ups

- **Stale partition sizes in the source plan.** `_plans/house`-era notes carry "**150 GB**" (plan
  header) and "**~200 GB from D:**" (plan §2) for the Linux partition. The decided and executed size is
  **100 GB**. This runbook uses 100 GB throughout; if the source plan is reused, fix those two lines.
- **The two source guides disagree on the RTC.** The 2026-09-21 guide says `set-local-rtc 1` (Option B);
  the 2026-10-18 doc says `RealTimeIsUniversal=1` (Option A, UTC) — and Option A is the machine's actual
  state. `docs/clock-and-rtc.md` documents the conflict and the resolution; keep it that way if either
  source changes.
- **Robotics verification.** Gates 1–2 (controller stack + RViz, mock) were pre-verified in simulation;
  gate 3b (real arm) and gate 4 (vision) are recorded per session and are **not** claimed verified. Run
  a full pass and update the status note at the end of `docs/06-robotics-stack.md`.
- **Fill in real measured numbers** where the text uses the reference machine's values (free space on
  C: at plan time, the verify counts in `docs/07-verification.md`) if you run this again on a different
  machine — or mark them clearly as the reference machine's.
- **Add the exact ISO filename** you used (point release) in `docs/00-prerequisites.md`, once frozen.
- **Confirm the ROS 2 apt key file name** (`ros.key` vs `ros-archive-keyring.gpg` output path) against
  the current ROS 2 Jazzy install docs before publishing the robotics section as tested.

## Not published on purpose

- The raw drafting docs (`Ubuntu-Native-Port-Plan-2026-09-19.md`, `Ubuntu-SO101-Full-Setup-Guide-2026-09-19.md`)
  and the owner's payload (`.env`, `auth.json`, `state.db`, `sessions/`) are **not** in this repo — see
  Exclusions in the plan. Do not add them.

Back to [README](../README.md).
