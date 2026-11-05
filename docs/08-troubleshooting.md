# 08 — Troubleshooting

Errors that actually happened during the reference build, with the fix that worked. If your symptom
matches, try the fix before inventing a new theory.

| Symptom (exact text or observation) | Cause | Fix |
| --- | --- | --- |
| Clock is exactly the UTC offset out (8 h in China) between the two OSes | RTC convention split: one OS writes local time, the other UTC | Pick one branch and stay on it. See [clock-and-rtc.md](clock-and-rtc.md) |
| `tf2` / RViz: "extrapolation into the future" | System clock behind the data timestamp | Fix the clock (same root cause as the row above) |
| `$'\r': command not found` when running a `.sh` | Script copied from NTFS has CRLF line endings | `find <dir> -name '*.sh' -exec sed -i 's/\r$//' {} +`, or re-export with `git -c core.autocrlf=false archive` |
| `-o /d/...`: "Cannot connect to C: resolve failed" (or a file written somewhere odd) | Native `tar`/`git` gets an MSYS `/d/...` path it does not translate | Pass a native path: `-o D:/payload/x.tar.gz` |
| `Permission denied` opening `/dev/ttyACM0` / `/dev/ttySERVO` | User not in `dialout`, or not re-logged in | `sudo usermod -aG dialout "$USER"`, then **log out and back in** |
| `/dev/ttySERVO` does not exist | udev rule not loaded, or device plugged before boot | `sudo udevadm control --reload-rules && sudo udevadm trigger`, replug; or fall back to `/dev/ttyACM*` |
| All six servos answer the same ID | Motor IDs never assigned | Assign one at a time with the driver's ID test |
| Bus health `BusHealth` is poor / servos missing | Cable, controller board, or a servo — hardware | Check cable, board, power, adapter. **No software fixes it** |
| `mount: unknown filesystem type 'BitLocker'` on `/dev/...C...` | C: is BitLocker-encrypted; no Linux driver | Expected. Never mount C: — use D: (shared) or a payload. Do **not** run `dislocker` casually |
| MuJoCo launch fails to load the model | A mesh is not ASCII STL, or the path is wrong | Read the log; convert meshes to ASCII; check the model path |
| RViz black or segfaults on AMD/Radeon | GPU driver / OpenGL | Update drivers; try the software GL path (`LIBGL_ALWAYS_SOFTWARE=1`) once to confirm it is GL |
| `colcon build` fails or keeps old behaviour after editing | Stale `build/` and `install/` | `rm -rf build install log && colcon build --symlink-install` |
| `E: Unable to locate package ros-jazzy-…` | ROS 2 apt key/repo missing, or fetch failed behind the proxy | Pre-fetch the ROS key while Windows is bootable; use the TUNA `ros2/ubuntu` repo (see [06](06-robotics-stack.md)) |
| `pip install torch` reverted to CPU | Plain `pip install lerobot` / `pip install torch` reinstalled the CPU wheel | Reinstall the cu126 wheel with `--no-deps`; never re-run plain `pip install torch` |
| `File exists: .../eval_xxxx` during eval | Previous eval output dir not cleaned | Delete the `eval_*` folder and re-run |
| Eval reports `mean is infinity` | Camera keyword does not match the recording config | Fix the camera keyword in the config to match the recording |
| Policy learns garbage | Robot not reset to a consistent pose between episodes | Fix the reset pose before adding more episodes |
| Windows boots 8 h behind / ahead after switching OS | RTC convention (see first rows) | [clock-and-rtc.md](clock-and-rtc.md) |
| `The computer did not resync because no time data was available` | Says nothing about the cause | Check `w32tm /query /peers`, then `w32tm /stripchart /computer:<server> /samples:3 /dataonly`, then NTP source port (below) |
| `w32tm /query /peers` shows `#Peers: 1` and Event ID 47 names a comma string | `/manualpeerlist:` is **space**-delimited; commas are flags | Use `"a.com b.com c.com"` |
| No NTP server ever syncs, any config | Some networks drop NTP from **source port 123** (the port w32time always uses) | Probe source 123 vs ephemeral; if filtered, set the clock directly (`Set-Date -Adjust …`) |
| Installer shows a different partition size than you planned | Truncated/resized view or wrong free-space selection | **Stop.** Re-read the partition table; never accept a surprise size |
| After install, Windows is missing from GRUB | Windows entry dropped | `sudo os-prober && sudo update-grub` (ensure `os-prober` is enabled in `/etc/default/grub`) |
| BitLocker asks for the recovery key after install | Protectors not re-enabled / Secure Boot state changed | Enter the 48-digit key; then `manage-bde -protectors -enable C:`. Keep Secure Boot as-is |
| Windows shows the Linux partition as an unknown/RAW disk and offers to format | Normal — it is ext4 | **Click Cancel.** Never let Windows format or `chkdsk` it |

## When it is the network, not you

A **Windows-side proxy** (e.g. Clash on `127.0.0.1:7897`) does not exist in an Ubuntu boot. Inside
Ubuntu use the Tsinghua mirrors (`mirrors.tuna.tsinghua.edu.cn/ubuntu`, `…/ros2/ubuntu`), the TUNA PyPI
index, and `HF_ENDPOINT=https://hf-mirror.com`. Anything that needs GitHub, fetch it while Windows is
still bootable and park it on the shared NTFS volume.

Back to [README](../README.md).
