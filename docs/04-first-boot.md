# 04 — First boot fixes

Ubuntu boots; now make it usable and, more importantly, understand what you **cannot** reach.

## 4.1 Update and mirrors

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential git curl wget python3-pip python3-venv \
                    pciutils usbutils ntfs-3g exfatprogs
```

If apt is slow, switch the mirror first (see [00-prerequisites.md](00-prerequisites.md)). On a
China-facing network with a proxy on `127.0.0.1`, export `http_proxy`/`https_proxy` before apt/git/pip.

## 4.2 Graphics — the hybrid-GPU reality

The reference laptop has an AMD iGPU plus an NVIDIA RTX 3050 (hybrid). Ubuntu 24.04 installs the
open AMD driver; the NVIDIA proprietary driver is a separate step.

```bash
# what the kernel sees
lspci | grep -Ei 'vga|3d|display'
ubuntu-drivers devices

# install the recommended NVIDIA driver (auto-picks the version)
sudo ubuntu-drivers autoinstall
sudo reboot

# after reboot — confirm the driver is loaded
nvidia-smi
glxinfo | grep -i "opengl renderer"    # glxinfo is in mesa-utils
```

Watch for the classic hybrid-GPU symptom: after a suspend/resume the NVIDIA card may not wake, or the
desktop falls back to software rendering. If `nvidia-smi` fails after resume, a reboot fixes it; the
permanent fix is a `systemd` resume hook or keeping the dGPU off when idle (the driver's
`nvidia-powerd` handles this on newer drivers).

## 4.3 Mounting the Windows volumes — and the one you cannot

NTFS volumes mount read-write in Linux via `ntfs-3g`. **But an encrypted C: is not mountable at all.**
This is the single most important correction to make to any older plan that says "symlink
`/mnt/win/Users/<you>/…`".

```bash
# D: is plain NTFS and mounts fine. Find it by label, not by a hard-coded device node:
lsblk -o NAME,FSTYPE,SIZE,LABEL,MOUNTPOINT
sudo mkdir -p /mnt/data
sudo mount -t ntfs3 /dev/nvme0n1p3 /mnt/data    # device node will differ on your disk

# C: — expect this to FAIL if BitLocker is On:
sudo mount -t ntfs3 /dev/nvme0n1p2 /mnt/win
#   mount: unknown filesystem type 'BitLocker' (or: wrong fs type / bad option)
```

That failure is expected and correct. **Linux has no in-kernel BitLocker driver.** To get files off an
encrypted C:, you decrypt it from Windows (only possible after `manage-bde -off` and a long rewrite) or
you simply **stage what you need on D: from Windows while Windows is running**. The reference port does
the latter: a `hermes-port-payload.tar.gz` on D: carries everything the Linux side needs.

Make the NTFS mounts survive reboot with an fstab entry (use the volume **UUID**, and `windows_names`
so Windows-illegal filenames are rejected rather than silently mangled):

```bash
sudo blkid                       # read the UUID of your NTFS data volume
# /etc/fstab — one line (D: mounts at /mnt/data):
UUID=<uuid-of-D>  /mnt/data  ntfs3  defaults,uid=1000,gid=1000,umask=022,windows_names,nofail  0 0
```

`nofail` means a missing/unmountable volume does not drop you to a rescue shell at boot.

## 4.4 Serial / USB devices

Robotics gear is USB-serial and shows up under `/dev/ttyUSB*` or `/dev/ttyACM*`. Confirm it appears
and note the adapter chipset:

```bash
lsusb
dmesg | grep -Ei 'tty(USB|ACM)|ch34|cp210|ftdi'
ls -l /dev/serial/by-id/
```

The `by-id` path is stable across replugs; use it in configs instead of `/dev/ttyUSB0`. On the
reference machine the arm's adapter is a **CH343-class** USB-serial chip (COM`<n>` under Windows).
A udev rule to give it a stable name is in [06-robotics-stack.md](06-robotics-stack.md).

## 4.5 Power, fans, and the laptop basics

```bash
sudo apt install -y power-profiles-daemon tlp   # tlp for battery life; tune to taste
sudo systemctl enable --now fstrim.timer        # weekly SSD trim
```

## 4.6 Where the payload is

Copy the staged payload from D: to your home directory and verify it before you trust it:

```bash
mkdir -p ~/hermes-port
cp /mnt/data/hermes-port/hermes-port-payload.tar.gz ~/hermes-port/
cd ~/hermes-port
sha256sum -c SHA256SUMS.txt          # if you staged a checksum file (you should have)
chmod 600 hermes-port-payload.tar.gz # it contains live keys
```

See [payload-and-checksums.md](payload-and-checksums.md) for how the payload is built and why the
checksum matters.

## Checkpoint 4

- [ ] `sudo apt update` succeeds; mirror/proxy works.
- [ ] `nvidia-smi` prints the GPU and driver version.
- [ ] The NTFS data volume mounts at `/mnt/data` and survives a reboot (fstab entry present).
- [ ] `sudo mount /dev/...C... /mnt/win` **fails** as expected — C: is encrypted.
- [ ] The payload is copied to `~/hermes-port/` and its checksum verifies.
- [ ] `date` on Linux and the clock in Windows agree (see [clock-and-rtc.md](clock-and-rtc.md)).

Next: [05-hermes-port.md](05-hermes-port.md) — move the Hermes install across.
