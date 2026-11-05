# Credits

## Author

Written by a graduate researcher working on precision motion control, who needed one laptop to run
Windows engineering tools and a native Linux robotics + agent stack. Ideas and corrections welcome via
an issue.

## Hermes skills used

This runbook was assembled from two of the author's Hermes skills, which did the actual work of the
job and are credited by name:

- **`windows-linux-dual-boot`** — disk/partition/encryption reconnaissance; install-media discipline
  (ISO internals, ISO vs DD write mode, Ventoy's NTFS requirement); installer choices (never format the
  existing EFI System Partition; ESP placement for external installs); reversibility of the shrink;
  the clock-skew branch (Windows `RealTimeIsUniversal`, Linux `timedatectl set-local-rtc`) and which
  branch the RTC is actually on.
- **`ubuntu-dualboot-hermes-port`** — Windows pre-flight checks (BitLocker needs admin, back up the
  recovery key, suspend before a bootloader change, keep Secure Boot on); the finding that a
  BitLocker-protected `C:` **cannot be mounted from Linux**, which makes staged payloads the only route
  in; the Hermes-profile port inventory (what moves, what is rebuilt, re-path cron, keep the payload off
  cloud); staging the payload twice; shipping with checksums and a runbook.

Referenced by the source notes and followed here:

- **`lerobot-environment-setup`** — the pin gotcha: plain `pip install torch` / `pip install lerobot`
  silently reverts a CUDA install to CPU torch.
- **`hermes-profile-migration`** — payload mechanics, scoped ports, and re-syncing a moved profile.

## External guides and upstream projects

Followed or relied on by the author's notes; all credit to their authors:

| Project | What it provided |
| --- | --- |
| [TheRobotStudio/SO-ARM100](https://github.com/TheRobotStudio/SO-ARM100) | the SO-101 arm design and servo selection |
| [adityakamath/so_arm_ros2](https://github.com/adityakamath/so_arm_ros2) | the ROS 2 control / description / sim packages |
| `sts_hardware_interface` | the Feetech STS3215 `ros2_control` hardware interface and calibration service |
| [huggingface/lerobot](https://github.com/huggingface/lerobot) | LeRobot, `lerobot-calibrate` / `-record` / `-train`, HIL-SERL |
| [google-deepmind/mujoco](https://github.com/google-deepmind/mujoco) | physics simulation and viewer |
| [Hermes Agent docs](https://hermes-agent.nousresearch.com/docs) | Hermes install, profile layout, and porting |
| [Rufus](https://rufus.ie/) | Windows USB installer writer |
| [Tsinghua TUNA mirrors](https://mirrors.tuna.tsinghua.edu.cn/) | Ubuntu / ROS 2 apt and PyPI mirrors |
| [hf-mirror.com](https://hf-mirror.com) | Hugging Face mirror (`HF_ENDPOINT`) |

## Machine-specific documentation

Microsoft `manage-bde`, `powercfg`, and `w32tm` documentation; Ubuntu/Debian `timedatectl`, `hwclock`,
`lsblk`, and `mount(8)` (`ntfs3`) man pages; Canonical's Ubuntu 24.04 installation guide.
