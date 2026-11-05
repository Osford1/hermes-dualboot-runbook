# 06 — Ubuntu-side robotics stack (SO-101 / LeRobot)

Optional and self-contained: skip it if you only wanted the dual boot and the Hermes port. It is here
because a native Ubuntu boot is the *better* host for a USB-serial arm than WSL — the arm appears as
`/dev/ttyACM*` directly, with no `usbipd` passthrough and no VM latency.

Two things are covered: **explicit control** (ROS 2 Jazzy + `ros2_control`, command the arm from
software and mirror it in RViz and MuJoCo) and **robot learning** (LeRobot + HIL-SERL). The arm in the
reference setup is an **SO-101** with Feetech **STS3215**-family bus servos.

## 6.0 Before you touch the arm

- **Torque is live whenever the controllers are active.** Support the arm and keep a hand on the
  **12 V power switch** — that is the only true stop. A soft stop exists (`/emergency_stop`), but the
  switch is the real one.
- **The calibration file is the one thing re-calibrating cannot restore** and it is tied to every
  dataset recorded under it. If you have a lab copy, fetch it before you calibrate anything here.
- **Copy tooling off NTFS and strip CRLF** before running it. Windows `git` with `core.autocrlf=true`
  contaminates shell scripts with CRLF, and bash dies on `$'\r': command not found`:

  ```bash
  cp -a /mnt/data/robot_101 ~/robot_101          # or wherever your scripts live on the shared volume
  find ~/robot_101 -name '*.sh' -exec sed -i 's/\r$//' {} +
  ```

  Better still, export code for Linux rather than copying a worktree — `git -c core.autocrlf=false
  -c core.eol=lf archive --format=tar.gz -o <native-path> HEAD` — then verify: `grep -rlU $'\r'
  <extracted> | wc -l` should be 0.

## 6.1 ROS 2 Jazzy + the workspace

Fetch anything that needs GitHub **while Windows is still bootable** and park it on the shared volume —
the ROS 2 apt signing key is the download that bites, because it succeeds through a Windows proxy and
fails without one, surfacing much later as an apt error.

```bash
# apt from a mirror, and the ROS 2 repo from TUNA
sudo apt update
sudo apt install -y software-properties-common curl gnupg lsb-release
# ROS 2 Jazzy apt key + repo (use the pre-fetched key file if you staged one)
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] https://mirrors.tuna.tsinghua.edu.cn/ros2/ubuntu noble main" \
  | sudo tee /etc/apt/sources.list.d/ros2.list
sudo apt update
sudo apt install -y ros-jazzy-desktop ros-dev-tools python3-colcon-common-extensions
source /opt/ros/jazzy/setup.bash
```

Build the workspace from the upstream packages (`so_arm_ros2` + `sts_hardware_interface`, plus your
own overlay). If you staged git-archive tarballs, extract them into `~/so101_ws/src` and build:

```bash
mkdir -p ~/so101_ws/src && cd ~/so101_ws/src
# extract your staged tarballs here (so_arm_ros2, sts_hardware_interface, ...)
cd ~/so101_ws
colcon build --symlink-install
source install/setup.bash
ros2 pkg list | grep -Ei 'so_arm|sts_hardware'
# expect so_arm_bringup, so_arm_control, so_arm_description, so_arm_mujoco, sts_hardware_interface
```

**Stop if** the build errors or a package is missing. Never take a broken build to hardware.

## 6.2 Serial access — the udev rule

The adapter is a **CH343-class** USB-serial chip. Give it a stable name instead of `/dev/ttyACM0`
(which renumbers on replug), and put yourself in `dialout`:

```bash
sudo usermod -aG dialout "$USER"        # log out and back in for this to take effect
sudo tee /etc/udev/rules.d/99-so101.rules >/dev/null <<'EOF'
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{idProduct}=="55d3", MODE="0666", SYMLINK+="ttySERVO"
EOF
sudo udevadm control --reload-rules && sudo udevadm trigger
```

Plug the adapter in **after** Ubuntu has booted, once, so the rule lands. Confirm:

```bash
ls -l /dev/ttySERVO /dev/ttyACM* 2>/dev/null
groups | grep dialout
```

Motor IDs are fixed by the description package:
`shoulder_pan=1 shoulder_lift=2 elbow_flex=3 wrist_flex=4 wrist_roll=5 gripper=6`, baud `1000000`.

## 6.3 Gates — run them in order, stop at the first failure

The reference session packaged the work as gates. The point is to keep a failure localised: never
debug goal 3 on a stack that did not pass goal 1.

| Gate | Goal | Command | Pass condition |
| --- | --- | --- | --- |
| 0 | install | build the workspace (6.1) | 5 packages built |
| 1 | controller stack, mock | `ros2 launch so_arm_bringup so_arm.launch.py use_mock:=true` | `joint_trajectory_controller` and `joint_state_broadcaster` **active**, `/joint_states` streams 6 joints, tracking error < 0.08 rad |
| 2 | RViz twin | same launch, RViz opens | the arm renders and mirrors the commanded pose live |
| 3a | MuJoCo physics | `ros2 launch so_arm_bringup so_arm.launch.py ros2_control_hardware_type:=mujoco` | the viewer follows both commanded poses |
| 3b | real arm | bring-up sequence below | the real arm matches the sim |
| 4 | vision model | wrist camera + a policy | feed confirmed before any model work (**stretch**) |

Check the controller stack (gate 1) directly:

```bash
ros2 control list_controllers          # both should read 'active'
ros2 topic echo /joint_states --once
```

Send an explicit command (works in sim and on real hardware):

```bash
# e.g. a 6-value target; the exact script you staged may differ — the topic is the contract
ros2 topic pub /joint_commands sensor_msgs/msg/JointState "{name: [...], position: [...]}" --once
```

The interface contract, so nothing has to be guessed:

```
command path:  /joint_commands (sensor_msgs/JointState)
               -> joint_trajectory_bridge -> joint_trajectory_controller -> hardware
state path:    /joint_states, /dynamic_joint_states  (voltage, temperature, current, is_moving)
services:      /emergency_stop (std_srvs/SetBool)
               /one_key_calibration (sts_hardware_interface/srv/OneKeyCalibration)
launch args:   model=so100|so101  serial_port  use_mock  use_sim_time
               ros2_control_hardware_type=real|mujoco  mujoco_headless  mujoco_arena
               frame_prefix  replay_loops  wrist_camera  wrist_camera_urdf
```

Keep the RViz twin and the real arm on the **same command topic** so "what you see is what the arm
does" stays literally true.

### Gate 3b — the real arm, in order

Steps 1–4 never command motion.

```bash
# 1. raw bus quality — NO motion
# 2. per-servo voltage + temperature — NO motion
# 3. which IDs actually answer
# 4. assign IDs ONLY if wrong/missing — ONE servo at a time
# 5. launch the controllers            (terminal A)
# 6. calibrate ONE motor first          (terminal B)
# 7. home — the first commanded motion  (terminal B)
```

The reference implementation exposes these as `real-arm-bringup.sh {health,diag,scan,ids,launch,calib,home,estop}`;
the underlying driver tests are:

```bash
ros2 launch sts_hardware_interface single_motor_position.launch.py serial_port:=/dev/ttySERVO motor_id:=1
ros2 launch sts_hardware_interface motor_diagnostics.launch.py
```

**Stop rules**

- Bus `health` bad ⇒ cable / controller board / servo / adapter fault. **No software fixes it.**
- All six servos answering the same ID ⇒ IDs were never assigned; that is the `ids` step, one servo
  at a time.
- **Calibration is the delicate one.** The driver has its own `OneKeyCalibration` service
  (`uint16[] motor_ids`) that runs each servo's `CalibrationOfs`, recentering each motor onto raw step
  `2048` — i.e. it *defines* that joint's zero. Torque drops during it; support the arm. Do **one**
  motor, confirm the verify line, then continue. `sts_hardware_interface`'s one-key calibration means
  **LeRobot is not required just to calibrate**.
- Know the sign conventions before you trust a direction: tick-`2048`-as-zero, and `drive_mode` sign.

## 6.4 Robot learning (LeRobot + HIL-SERL)

```bash
# Miniforge + a python 3.12 env
curl -L -o ~/miniforge.sh https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh
bash ~/miniforge.sh -b -p ~/miniforge && ~/miniforge/bin/conda init bash && source ~/.bashrc
conda create -n lerobot python=3.12 -y && conda activate lerobot

# LeRobot from source (HIL-SERL needs the git source, not plain `pip install lerobot`)
git clone https://github.com/huggingface/lerobot.git ~/lerobot && cd ~/lerobot
pip install -e ".[hilserl]"

# CUDA torch — ORDER MATTERS (the #1 gotcha in this stack)
pip install --index-url https://download.pytorch.org/whl/cu126 --force-reinstall --no-deps \
  "torch==2.11.0+cu126" "torchvision==0.26.0+cu126"
pip install "torchcodec>=0.7,<0.12"                 # from PyPI, NOT the cu126 index
conda install -c conda-forge ffmpeg -y              # torchcodec needs it for video I/O
pip install feetech-servo-sdk deepdiff

python -c "import torch, lerobot; print(torch.__version__, torch.cuda.is_available(), lerobot.__version__)"
# expect: 2.11.0+cu126 True 0.6.x
```

Never re-run plain `pip install torch` or `pip install 'lerobot[feetech]'` afterwards — both silently
revert you to a **CPU** torch. That pin gotcha is documented in the `lerobot-environment-setup` skill.

Keep datasets and the HF cache on the shared volume so ext4 stays lean:

```bash
echo 'export HF_HOME=/mnt/data/hf' >> ~/.bashrc && source ~/.bashrc && mkdir -p /mnt/data/hf
```

Hardware sanity — **plug the arms first (confirm two ports), then the camera**; the USB ordering
pitfall otherwise scrambles the device mapping:

```bash
lerobot-find-port                 # expect the two arm ports (map to /dev/ttyACM0, /dev/ttyACM1)
lerobot-find-cameras opencv       # plug the camera only now
```

Calibrate once, then **back up the file immediately** to `/mnt/data` and the Windows side:

```bash
lerobot-calibrate --robot.type=so101_follower  --robot.port=/dev/ttyACM0 --robot.id=my_follower
lerobot-calibrate --teleop.type=so101_leader   --teleop.port=/dev/ttyACM1 --teleop.id=my_leader
```

`lerobot-record` / `lerobot-train` (ACT) / eval follow from there; the reference protocol lives in
`~/robot_101/experiment-episodes-vs-steps.md`. Two known eval errors: `File exists: .../eval_xxxx`
(delete the `eval_*` folder) and `mean is infinity` (your camera keyword does not match the recording
config). And the reset lesson: if the robot is not reset to a consistent pose between episodes, the
policy learns garbage regardless of episode count — fix the reset before adding data.

## Checkpoint 6

- [ ] `ros2 pkg list | grep -Ei 'so_arm|sts_hardware'` shows the five packages.
- [ ] `ros2 control list_controllers` shows both controllers `active`.
- [ ] `/dev/ttySERVO` exists after a replug; `groups` includes `dialout`.
- [ ] The mock gate (1) passes with tracking error < 0.08 rad before any hardware step.
- [ ] `python -c "import torch; print(torch.cuda.is_available())"` is `True`.
- [ ] The calibration file is backed up in two places.

> **Status honesty:** the gates above were packaged and pre-verified in simulation (MuJoCo model loads,
> tracks commands), but the full real-arm pass is recorded per session and is **not** claimed as
> verified here. Treat gates 3b and 4 as a checklist until your own run prints a pass.

Next: [07-verification.md](07-verification.md).
