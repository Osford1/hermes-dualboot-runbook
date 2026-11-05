# 00 — Prerequisites

Everything here is gathered **before** the first irreversible step. Do not shrink a partition or
boot an installer until every box in this file is ticked.

Substitute your own paths for `C:\Users\<you>\…` and `/home/<you>/…`; they are placeholders for the
Windows and Ubuntu usernames on the target machine.

## What you need on the Windows side

- A working Windows 11 install you can afford to keep. This runbook keeps it — it does not replace it.
- **BitLocker key access.** If C: is encrypted (most pre-installed laptops are), you must be able to
  read the 48-digit recovery key from your Microsoft account at `account.microsoft.com/devices/recoverykey`
  or from a printed/exported copy. Without it, a wrong click is unrecoverable.
- An **8 GB or larger USB stick** you can erase. It becomes the Ubuntu installer.
- **Enough free space on D:** (or whichever non-C: data volume you will shrink). You need the size of
  the Ubuntu partition (this runbook uses **100 GB**) plus headroom. Do **not** plan to take space
  from C: — see the warning below.
- A spare disk or off-machine location for the backup in [01-risk-and-backup.md](01-risk-and-backup.md).

### Do not touch C:

On the machine this was written for, **C: is ~87 % full (450 GB total, ~63 GB free) and must not be
shrunk.** Windows needs free space to update and to hibernate, and a nearly-full C: is the worst
place to carve a partition. All the space for Ubuntu comes from the tail of **D:** — a data volume
that held nothing but a WSL disk image. Check your own numbers before you start:

```powershell
Get-Volume | Where-Object DriveLetter | Select-Object DriveLetter, FileSystemLabel, SizeRemaining, Size
```

Windows converts `Size` and `SizeRemaining` from bytes to GB when it prints; read the free-space
percentage, not the raw number. If your C: is above ~80 % full, do not shrink it.

### Gather the machine facts

Run these and keep the output — you will compare against them after the install.

```powershell
# firmware mode, Secure Boot, and whether the disk is GPT or MBR
Confirm-SecureBootUEFI
Get-Disk | Select-Object Number, FriendlyName, PartitionStyle, @{n='SizeGB';e={[math]::Round($_.Size/1GB)}}
Get-Partition | Select-Object DiskNumber, PartitionNumber, DriveLetter, @{n='SizeGB';e={[math]::Round($_.Size/1GB)}}

# BitLocker state on the Windows volume
manage-bde -status C:

# fast startup and hibernation (both must be off before you shrink/resize)
powercfg /a
```

What "good" looks like (measured on the reference machine):

| Fact | Reference value |
| --- | --- |
| Firmware | UEFI, Secure Boot **On** |
| Disk | 932 GB NVMe, **GPT** (not MBR) |
| C: | 450 GB NTFS, ~87 % full — **untouched** |
| D: | 455 GB NTFS → **355 GB** after the 100 GB shrink |
| BitLocker on C: | `Protection On`, `XTS-AES 128`, protectors TPM + Numerical Password |

If the disk is **MBR**, stop and read the `windows-linux-dual-boot` skill's conversion section first —
Windows can convert MBR→GPT in place with `mbr2gpt.exe` (`mbr2gpt /validate` then `/convert
/allowFullOS`), but only after your firmware is switched to UEFI boot. Dual-booting on MBR is possible but is a different,
rougher route and is out of scope here.

## What you need on the Ubuntu side (to be installed)

- **Ubuntu 24.04 LTS "noble"** desktop ISO. This runbook's timings and package names assume 24.04.
  Download from <https://releases.ubuntu.com/24.04/> or a fast mirror (see the mirror note below).
- A **wired connection** if you have one, or your Wi-Fi credentials to hand — the installer can join
  Wi-Fi, and post-install package fetches are far faster wired.
- The **`hermes-port-payload.tar.gz`** and any robotics staging you built on Windows, copied to the
  machine before you start. See [payload-and-checksums.md](payload-and-checksums.md).

## What you need on the Hermes side

- A **working Hermes Agent install on the Windows side**, with a profile directory you can copy.
  Default location: `%LOCALAPPDATA%\hermes` (that is `C:\Users\<you>\AppData\Local\hermes`).
- The **provider key(s)** your install uses (e.g. a `DEEPSEEK_API_KEY`), so you can re-enter them.
  Never paste a key into a script or a note that could be published — see the credential rule in
  [01-risk-and-backup.md](01-risk-and-backup.md).

## Mirrors (there is a reason this section exists)

If raw `archive.ubuntu.com` / `github.com` / `huggingface.co` is slow or intermittent on your
network, put a mirror in front of it. On a China-facing network a local proxy on `127.0.0.1` is often
already running:

```bash
# Ubuntu apt: switch to a nearby mirror (Tsinghua TUNA example)
sudo sed -i 's|http://archive.ubuntu.com/ubuntu|https://mirrors.tuna.tsinghua.edu.cn/ubuntu|g' \
  /etc/apt/sources.list.d/ubuntu.sources
sudo apt update

# Hugging Face model/dataset downloads
export HF_ENDPOINT=https://hf-mirror.com

# git / pip through a local HTTP proxy, if you have one
export https_proxy=http://127.0.0.1:7897 http_proxy=http://127.0.0.1:7897
```

The `127.0.0.1:7897` proxy is a placeholder for whatever your local proxy listens on — check your
proxy tool's settings. Do not hard-code it into anything you publish.

## Checklist — all true before you continue

- [ ] I can read my BitLocker recovery key right now.
- [ ] I know my C: is not the volume I am shrinking.
- [ ] My target data volume has ≥ 110 GB free (100 GB for Ubuntu + headroom).
- [ ] `powercfg /a` confirms hibernation/fast-startup can be turned off.
- [ ] I have a ≥ 8 GB USB stick and a spare/off-machine target for the backup.
- [ ] I have the Ubuntu 24.04 ISO and can write it to the USB stick.
- [ ] I have my Hermes provider key(s) somewhere I can retype them.

Next: [01-risk-and-backup.md](01-risk-and-backup.md) — the gate. Read it before you touch a partition.
