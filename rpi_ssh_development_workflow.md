# Raspberry Pi Development Workflow (SSH-Based) — Reference Guide

A reusable reference for developing ROS2-based robotics projects on a Raspberry Pi, including integration with a microcontroller (ESP32/STM32). Use this as a checklist/manual for any future project with a similar architecture.

---

## 1. The Core Idea

Unlike a microcontroller (where you *flash* code and it runs standalone), a Raspberry Pi runs a **full Linux OS**. You never "upload" code to it in the Arduino sense — you write code as regular files, transfer them to the Pi, and **run them as programs** over an SSH connection. The Pi has no display attached ("headless"), so SSH is your only way in.

```
[Your Laptop]  <-- SSH / WiFi -->  [Raspberry Pi, headless]
   - Write code                        - Runs ROS2 nodes
   - Git push                          - Talks to sensors (LiDAR, camera)
   - RViz2 (visualize remotely)        - Talks to MCU over serial
```

---

## 2. One-Time Setup

### 2.1 Flash the OS
- Use **Raspberry Pi Imager** (from your laptop).
- Choose **Ubuntu Server 24.04 LTS (64-bit)** — match your ROS2 distro's target OS (Jazzy → Noble/24.04). Avoid Raspberry Pi OS unless your ROS2 workflow specifically needs it; Ubuntu Server keeps package versions consistent with your laptop.
- In the Imager's advanced settings (gear icon) **before writing**, pre-configure:
  - Hostname (e.g., `amr-pi`)
  - Enable SSH, set a username/password (or add your SSH public key — preferred)
  - WiFi SSID/password, so it connects on first boot without a monitor

### 2.2 First Boot & Connect
```bash
# From your laptop, find the Pi on the network (or check your router's device list)
ping amr-pi.local

# SSH in
ssh <username>@amr-pi.local
```
If `.local` mDNS resolution doesn't work, find the Pi's IP from your router's admin page and use `ssh user@<ip>` instead.

### 2.3 Set a Static IP (strongly recommended)
DHCP-assigned IPs can change on reboot, breaking your SSH shortcuts and ROS2 discovery. Set a static IP either:
- On your router (DHCP reservation tied to the Pi's MAC address) — easiest, do this.
- Or via netplan on the Pi itself if router access isn't available.

### 2.4 SSH Key-Based Login (skip typing passwords every time)
```bash
# On your laptop, generate a key if you don't have one
ssh-keygen -t ed25519

# Copy it to the Pi
ssh-copy-id <username>@amr-pi.local
```
After this, `ssh <username>@amr-pi.local` logs in without a password prompt.

### 2.5 Install ROS2 on the Pi
Same steps as your laptop, but install the **base** variant (no GUI tools needed on the Pi itself — you'll visualize remotely):
```bash
sudo apt update && sudo apt install curl -y
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

sudo apt update
sudo apt install ros-jazzy-ros-base -y
sudo apt install ros-dev-tools -y
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc
```

### 2.6 Match ROS_DOMAIN_ID on Both Machines
So the laptop and Pi discover each other's topics/nodes on the same network:
```bash
# Add to ~/.bashrc on BOTH laptop and Pi
echo "export ROS_DOMAIN_ID=42" >> ~/.bashrc   # any number 0-101, just match it on both
source ~/.bashrc
```

### 2.7 VS Code Remote-SSH (recommended dev environment)
- Install the **Remote - SSH** extension in VS Code on your laptop.
- `Ctrl+Shift+P` → "Remote-SSH: Connect to Host" → enter `<username>@amr-pi.local`.
- VS Code now opens a window that's *actually editing files on the Pi*, with full IntelliSense, terminal, and debugging — but the Pi is doing the heavy lifting, not your laptop.
- This is the smoothest option once set up; use it for most of your day-to-day work.

---

## 3. Day-to-Day Development Workflow

There are two common patterns — pick whichever fits your habits, or mix them:

### Pattern A: Git-based (best for tracking history / working solo across sessions)
```bash
# On laptop: write code, commit, push
git add .
git commit -m "add lidar bringup node"
git push origin main

# On Pi (via SSH terminal): pull latest
ssh amr-pi.local
cd ~/amr_ws/src/your_repo
git pull
cd ~/amr_ws
colcon build
source install/setup.bash
```

### Pattern B: Direct edit via VS Code Remote-SSH (best for fast iteration)
- Just edit files directly in the Remote-SSH VS Code window — no push/pull needed, changes save straight to the Pi.
- Still recommended to `git commit` periodically for backup/versioning, just not required for every test run.

### Running Your Code
```bash
# Inside SSH session or VS Code integrated terminal (connected to Pi)
cd ~/amr_ws
colcon build --symlink-install     # --symlink-install lets you edit Python files without rebuilding
source install/setup.bash
ros2 launch your_package your_launch_file.launch.py
```

### Visualizing From Your Laptop
Because ROS2 nodes discover each other over the network automatically (same `ROS_DOMAIN_ID`, same subnet), you can run heavy GUI tools on your laptop while the Pi does the sensing/computing:
```bash
# On your LAPTOP (not the Pi) — Pi has no GPU/display, don't run RViz there
rviz2
```
The Pi publishes topics (`/scan`, `/odom`, `/map`, etc.); RViz2 on your laptop subscribes and renders them.

### Auto-Start on Boot (optional, once stable)
Once your launch file is reliable, set it up as a `systemd` service so the Pi starts your ROS2 stack automatically on power-on — useful once you're past active development and want the robot to "just work" when switched on. Not needed while you're still debugging.

---

## 4. Integrating a Microcontroller (ESP32/STM32) with the Pi

This is the part that differs most from the Pi's own workflow, so it's worth being explicit about which machine does what.

### 4.1 The Two-Brain Split (why this exists)
| | Raspberry Pi | ESP32 / STM32 |
|---|---|---|
| OS | Full Linux (Ubuntu) | None — runs your code directly (bare-metal/RTOS) |
| Role | High-level: ROS2, SLAM, Nav2, path planning | Low-level: real-time PID, encoder reading, motor PWM |
| Timing | Not hard real-time (Linux scheduler) | Hard real-time (deterministic loop timing) |
| How code runs | Executed as a process, started via SSH/launch file | Flashed once, runs forever until reflashed |
| How you develop | Git pull / VS Code Remote-SSH, no "upload" step | Arduino IDE / PlatformIO, physical USB **upload/flash** step |

This split exists because a PID control loop needs consistent, predictable timing — something a general-purpose Linux OS running dozens of other processes can't always guarantee. The MCU handles that tight loop; the Pi handles everything that can tolerate small timing variance.

### 4.2 Programming the MCU — Still the "Old" Way
This part **does not change** regardless of the Pi/ROS2 integration:
- Write firmware in **Arduino IDE** or **PlatformIO** (C/C++).
- Connect the MCU to your **laptop** via USB.
- Hit **Upload/Flash** — code is written directly onto the chip's flash memory and starts running immediately, standalone, no OS.
- The MCU does **not** run ROS2 itself (in the basic setup) — it just listens for simple commands over serial and replies with sensor data.

### 4.3 Connecting the MCU to the Pi (not your laptop) for actual operation
- During MCU development/testing, plug it into your **laptop** to flash and debug via Serial Monitor.
- Once firmware is stable, physically connect the MCU to the **Raspberry Pi** via USB (or UART GPIO pins) — this is its permanent home once the robot is assembled.
- The Pi will see it as a serial device: `/dev/ttyUSB0` or `/dev/ttyACM0`.
```bash
# On the Pi, check the MCU is detected
ls /dev/tty*
dmesg | grep tty      # confirms which port it landed on after plugging in
```
- **Tip**: USB device names can shift between reboots if you have multiple USB devices. Use **udev rules** to assign a fixed symlink (e.g., `/dev/mcu`) so your code always finds it reliably:
```bash
# Find the MCU's vendor/product ID
udevadm info -a -n /dev/ttyUSB0 | grep -E "idVendor|idProduct"

# Create a rule (adjust IDs accordingly)
echo 'SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="ea60", SYMLINK+="mcu"' | sudo tee /etc/udev/rules.d/99-mcu.rules
sudo udevadm control --reload-rules && sudo udevadm trigger
```

### 4.4 The Software Bridge (this is the piece you write)
On the Pi, you write a **ROS2 node** (Python, using `pyserial`, or C++) that acts as a translator between ROS2 topics and the MCU's serial protocol:
```bash
pip install pyserial --break-system-packages
```
Responsibilities of this node:
- Subscribe to `/cmd_vel` (velocity commands from Nav2 or teleop)
- Convert to whatever simple protocol your MCU firmware expects (e.g., `"V,0.3,0.1\n"`)
- Write that over serial to `/dev/mcu`
- Read encoder/sensor data the MCU sends back over serial
- Publish that as `/odom` or other ROS2 topics

This node is the **only** piece of code that needs to know both "ROS2 language" and "MCU serial language" — everything else on the Pi just talks normal ROS2 topics, and everything on the MCU just talks serial.

### 4.5 Optional Upgrade Path: micro-ROS
If you want the MCU to speak native ROS2 messages directly (skipping the custom serial-bridge node), look into **micro-ROS** — a ROS2 client library that runs on microcontrollers. It's more elegant long-term but adds real setup complexity (transport configuration, agent setup on the Pi side). Recommended only as a later optimization, not a starting point — get the simple serial-bridge version working first.

---

## 5. Quick Reference Checklist (for any new project)

**One-time setup:**
- [ ] Flash Ubuntu Server on Pi, enable SSH, set static IP
- [ ] SSH key-based login working
- [ ] ROS2 base installed on Pi, `ROS_DOMAIN_ID` matches laptop
- [ ] VS Code Remote-SSH connected
- [ ] Git repo created for the workspace

**Per coding session:**
- [ ] Edit code (VS Code Remote-SSH, or edit locally + git push/pull)
- [ ] `colcon build --symlink-install` on the Pi
- [ ] `source install/setup.bash`
- [ ] `ros2 launch ...` to run
- [ ] `rviz2` on laptop to visualize

**MCU integration specifically:**
- [ ] Flash MCU firmware via laptop (Arduino IDE/PlatformIO), test standalone first
- [ ] Move MCU to Pi via USB once firmware is stable
- [ ] Set a udev rule for a fixed device name
- [ ] Write/run the serial-bridge ROS2 node on the Pi
- [ ] Validate: send a test `/cmd_vel`, confirm MCU responds, confirm `/odom` publishes back correctly

---

## 6. Common Pitfalls
- **Forgetting to `source install/setup.bash`** after a build — commands silently use the old version or fail to find packages.
- **ROS_DOMAIN_ID mismatch** between laptop and Pi — nodes won't discover each other, but no error is thrown; it just looks like "nothing is happening."
- **USB device name changes** after reboot if multiple serial devices are plugged in — use udev rules (Section 4.3).
- **Running RViz2 on the Pi** — don't; it has no GPU and will be painfully slow or crash. Always visualize from the laptop.
- **Powering the Pi from the same battery rail as motors** without isolation — causes brownouts/crashes when motors draw current. Use a separate regulated supply for the Pi.
