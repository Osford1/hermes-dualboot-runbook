# Clock and RTC between Windows and Linux

**This is the single most common dual-boot annoyance, and it is optional to fix — the machine boots
fine with a wrong clock. But if your timestamps, logs, or `tf2` transforms drift by exactly your UTC
offset, this is the cause.**

## The problem, plainly

There is one Real-Time Clock chip. Two OSes write it on shutdown with different conventions:

- **Windows default:** the RTC holds **local** time.
- **Ubuntu (Linux) default:** the RTC holds **UTC**.

With no decision made, each OS "corrects" the clock the way it thinks is right, so they ping-pong:
fix the time in Windows and Linux boots 8 hours off next time (8 h = the UTC offset for China); fix it
in Linux and Windows is off. The error is **exactly the offset**, not minutes — that is how you know it
is the RTC convention and not crystal drift.

## Pick one branch — never two

Set **exactly one** of these:

**Option A — RTC holds UTC (what this machine uses).** Tell Windows the RTC is UTC, and leave Linux on
its UTC default:

```powershell
# Windows, admin. Add (or set) the DWORD:
reg add "HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /t REG_DWORD /d 1 /f
```

```bash
# Ubuntu: confirm it is on the UTC branch
sudo timedatectl set-local-rtc 0     # RTC in UTC (no --adjust-system-clock)
sudo systemctl restart systemd-timesyncd
```

**Option B — RTC holds local time.** The alternative, if you prefer to leave Windows alone and are in a
zone **without DST** (true for China):

```bash
sudo timedatectl set-local-rtc 1
```

Use `set-local-rtc 1` **without** `--adjust-system-clock` if you only want to change the convention.
Doing it with `--adjust-system-clock` immediately shifts the running clock, which is confusing mid-debug.

> **The two guides disagree and this is worth stating.** The earlier guide (2026-09-21) recommended
> Option B (`set-local-rtc 1`). The later runbook doc (2026-10-18) settled on **Option A — UTC, with
> `RealTimeIsUniversal=1` on Windows** — and that is the state this machine is in. Option A is the
> better choice in a zone with DST (China has none, but portability is the point of this repo): local
> time in the RTC makes Linux read the wrong hour after a DST change. If you copied a plan that says
> `set-local-rtc 1`, Option A is the newer, preferred answer.

**Never set both.** `set-local-rtc 1` *and* `RealTimeIsUniversal=1` puts the two OSes 8 h out in
opposite directions.

## Pick the branch from the RTC's actual contents

Don't guess. Read what the RTC currently holds and set the OSes to match:

```bash
sudo hwclock --show --utc       # interpret the chip as UTC
sudo hwclock --show --localtime # interpret the chip as local
```

Whichever interpretation equals the true current time tells you the convention already in force; align
both OSes to it rather than moving the chip.

## Verify

```bash
timedatectl | grep -E 'Local time|Universal time|RTC in local TZ'
# RTC in local TZ: no   <- UTC branch (Option A)
```

Then boot Windows and confirm its clock is correct. Boot back to Linux and confirm it still is.

## If Windows will not sync (part of the same story)

- `w32time`'s "The computer did not resync because no time data was available" says nothing about the
  cause. Check in order: `w32tm /query /peers`, then
  `w32tm /stripchart /computer:<server> /samples:3 /dataonly`, then the NTP **source port**.
- `/manualpeerlist:` is **space**-delimited. `a.com,b.com` is read as one host name (a comma only
  introduces a flag like `,0x8`); use `"a.com b.com c.com"`.
- Some networks silently drop NTP sent from **source port 123** — the port `w32time` always uses — so
  no server choice fixes it. Probe source-123 vs an ephemeral port; if it is filtered, just set the
  clock directly (admin): `Set-Date -Adjust (New-TimeSpan -Seconds $offset)`.
- All the config commands (`w32tm /config`, `/resync`, `Set-Date`, `net stop w32time`) need an **admin**
  shell. An unelevated session cannot do them.

Back to [README](../README.md).
