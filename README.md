# LinkerHand L10 Control Dashboard

Graphical dashboard for controlling a LinkerHand L10 dexterous hand from a Linux laptop.

The dashboard uses the official LinkerHand Python SDK and the project GUI entry point:

```bash
python3 example/gui_control/gui_control.py
```

Reference SDK:
[linker-bot/linkerhand-python-sdk](https://github.com/linker-bot/linkerhand-python-sdk).

## Safety First

- Mount or hold the hand securely before moving any sliders or presets.
- Keep fingers, tools, and cables away from the hand during tests.
- Run only one control method at a time. Close terminal tools, ROS nodes, glove controllers, and other GUI dashboards before launching this dashboard.
- Start with CAN detection and a safe preset such as `Open Hand` before using sliders.
- `Stop All Actions` stops the dashboard preset loop. It is not a hardware emergency stop. Use the hardware power switch or emergency stop if motion is unsafe.

## What You Need

- Ubuntu 20.04 or newer is recommended.
- Python 3.8 or newer.
- A desktop session or working X11 display for PyQt5.
- Git.
- A USB-to-CAN adapter supported by SocketCAN.
- A powered LinkerHand L10.
- Linux CAN tools: `iproute2`, `can-utils`, and `ethtool`.
- Optional but recommended: Conda or another Python virtual environment.

## First Setup On A Linux Laptop

### 1. Clone the project

```bash
cd ~
git clone https://github.com/notalexander30/linkerhand-L10-control-dashboard.git
cd linkerhand-L10-control-dashboard
```

If you cloned somewhere else, always `cd` into the project directory before running the commands below.

### 2. Create and activate a Python environment

Using Conda:

```bash
conda create -n linkerhand-l10-dashboard python=3.10 -y
conda activate linkerhand-l10-dashboard
python3 -m pip install --upgrade pip
```

Without Conda:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install --upgrade pip
```

### 3. Install Linux CAN tools

```bash
sudo apt update
sudo apt install -y git iproute2 can-utils ethtool
```

For a local desktop GUI, PyQt usually works after the Python install below. If you are on a minimal Ubuntu image, also install common Qt/X11 libraries:

```bash
sudo apt install -y libxcb-xinerama0 libxkbcommon-x11-0 libxcb-cursor0
```

### 4. Install Python packages

Recommended minimal install for this dashboard:

```bash
python3 -m pip install PyYAML==6.0.1 pyqt5==5.15.0 pyqtgraph==0.13.3 python-can python-can-candle numpy
```

If you want all original SDK examples and simulation packages:

```bash
python3 -m pip install -r requirements.txt
```

The full `requirements.txt` can take longer and may install packages that are not needed for this dashboard.

### 5. Configure the hand

Open:

```bash
nano LinkerHand/config/setting.yaml
```

Set only the hand you are using to `EXISTS: True`. The dashboard supports one hand at a time and prefers the left hand if both are set to `True`.

Example for a left L10 hand:

```yaml
LINKER_HAND:
  LEFT_HAND:
    EXISTS: True
    TOUCH: True
    MODBUS: "None"
    JOINT: L10
  RIGHT_HAND:
    EXISTS: False
    TOUCH: True
    MODBUS: "None"
    JOINT: L10
PASSWORD: "your_sudo_password_if_you_use_sdk_auto_can_open"
```

Example for a right L10 hand:

```yaml
LINKER_HAND:
  LEFT_HAND:
    EXISTS: False
    TOUCH: True
    MODBUS: "None"
    JOINT: L10
  RIGHT_HAND:
    EXISTS: True
    TOUCH: True
    MODBUS: "None"
    JOINT: L10
PASSWORD: "your_sudo_password_if_you_use_sdk_auto_can_open"
```

Note: the current dashboard creates `LinkerHandApi(...)` without an explicit `can=` argument, so the SDK default is `can0`. If your adapter appears as `can1`, see [Using `can1`](#using-can1).

### 6. Connect the hardware

1. Connect the USB-to-CAN adapter to the laptop.
2. Connect CAN-H, CAN-L, and ground according to your adapter and hand wiring.
3. Power on the LinkerHand L10.
4. Make sure no other LinkerHand controller is running.

### 7. Find the CAN interface

```bash
ip link
ip -br link show type can
```

You should see `can0` or `can1`.

Optional helper:

```bash
chmod +x find_can.sh
./find_can.sh
```

If no CAN interface appears, stop here and fix the USB-to-CAN adapter, driver, cable, or power.

### 8. Reset CAN: down, configure, up

Most L10 setups use 1 Mbps.

For `can0`:

```bash
sudo ip link set can0 down
sudo ip link set can0 type can bitrate 1000000 restart-ms 100
sudo ip link set can0 txqueuelen 1000
sudo ip link set can0 up
ip -details link show can0
```

You can watch CAN traffic with:

```bash
candump can0
```

Press `Ctrl+C` to stop `candump`.

### 9. Launch the dashboard

```bash
python3 example/gui_control/gui_control.py
```

When the window opens:

1. Confirm the header shows the expected hand side and `L10`.
2. Start with a safe preset such as `Open Hand`.
3. Move sliders slowly.
4. Use `Stop All Actions` to stop preset looping.
5. Use the hardware power switch or emergency stop for unsafe motion.

## Daily Start

```bash
cd ~/linkerhand-L10-control-dashboard
conda activate linkerhand-l10-dashboard
sudo ip link set can0 down
sudo ip link set can0 type can bitrate 1000000 restart-ms 100
sudo ip link set can0 txqueuelen 1000
sudo ip link set can0 up
python3 example/gui_control/gui_control.py
```

## Dashboard Controls

| Control | What it does |
| --- | --- |
| Joint sliders | Sends live joint values through `api.finger_move(...)`. |
| System preset buttons | Sends saved poses from `LinkerHand/config/L10_positions.yaml`. |
| New preset name + Add | Saves the current slider values as a new preset. |
| Loop Preset Action | Loops through valid presets from the YAML file. |
| Stop All Actions | Stops the preset loop and clears highlighting. |
| Return to Initial Position | Sends the dashboard initial pose. |
| All Joints 0 | Sends all displayed joints to zero. Use carefully. |

## Presets

L10 dashboard presets live in:

```text
LinkerHand/config/L10_positions.yaml
```

Each preset must contain:

- `ACTION_NAME`
- `POSITION`
- exactly ten values
- each value between `0` and `255`

Example:

```yaml
RIGHT_HAND:
- ACTION_NAME: Open Hand
  POSITION: [255, 70, 255, 255, 255, 255, 255, 255, 255, 255]
```

The L10 joint order is:

1. Thumb Base
2. Thumb Side Swing
3. Index Base
4. Middle Base
5. Ring Base
6. Little Base
7. Index Side Swing
8. Ring Side Swing
9. Little Side Swing
10. Thumb Rotation

## Using `can1`

The current GUI uses the SDK default `can0`.

Option A: rename the interface to `can0` before launching:

```bash
sudo ip link set can1 down
sudo ip link set can1 name can0
sudo ip link set can0 type can bitrate 1000000 restart-ms 100
sudo ip link set can0 txqueuelen 1000
sudo ip link set can0 up
python3 example/gui_control/gui_control.py
```

Option B: edit `example/gui_control/gui_control.py` and pass `can="can1"`:

```python
self.api = LinkerHandApi(hand_joint=self.hand_joint, hand_type=self.hand_type, can="can1")
```

Then launch the dashboard normally.

## Troubleshooting

| Problem or return message | What to do first |
| --- | --- |
| `git clone` fails on Windows with `Filename too long` | Use a Linux laptop for this SocketCAN project, or clone into a very short path such as `C:\src`. On Windows you can also try `git config --global core.longpaths true`, then clone again. |
| The GUI does not open over SSH | Run on the laptop desktop, use `ssh -X`, or set up a working X11/VNC display. Check `echo $DISPLAY`. |
| `qt.qpa.plugin: Could not load the Qt platform plugin "xcb"` | Install Qt/X11 libraries: `sudo apt install -y libxcb-xinerama0 libxkbcommon-x11-0 libxcb-cursor0`. |
| `ModuleNotFoundError: No module named 'PyQt5'` | Activate the environment and run `python3 -m pip install pyqt5==5.15.0`. |
| `ModuleNotFoundError` for `yaml`, `can`, or `LinkerHand` | Activate the environment, install dependencies, and launch from the repository root. |
| `ip: command not found` | Run `sudo apt update && sudo apt install -y iproute2`. |
| `candump: command not found` | Run `sudo apt update && sudo apt install -y can-utils`. |
| `can0` does not appear in `ip link` | Check USB-to-CAN connection, driver, USB port, and cable. Replug the adapter. Run `ip -br link show type can` and `./find_can.sh`. |
| `can0 interface is not open` | Run the CAN down/configure/up commands, then launch the dashboard again. |
| Your adapter appears as `can1` | Rename it to `can0` before launch, or edit the dashboard to pass `can="can1"` to `LinkerHandApi`. |
| `RTNETLINK answers: Device or resource busy` | Close other controllers, then run `sudo ip link set can0 down` and bring it back up. |
| `Operation not permitted` | Use `sudo` for `ip link` commands. |
| `SIOCSIFBITRATE: Invalid argument` | The adapter/driver may not support SocketCAN bitrate changes, or the interface is not a CAN interface. Check the USB-to-CAN driver. |
| `candump can0` shows no traffic | Confirm hand power, CAN-H/CAN-L/GND wiring, termination, 1 Mbps bitrate, and interface name. |
| CAN error counters increase | Check `ip -statistics -details link show can0`, reset CAN, then check wiring, termination, and bitrate. |
| `Warning: Hardware version number not recognized` | Power-cycle the hand, replug USB-to-CAN, reset CAN, and confirm the correct hand side in `setting.yaml`. |
| Dashboard opens but the hand does not move | Check CAN setup, hand power, selected hand side, and whether another controller is active. Start with `Open Hand`. |
| Wrong hand side is shown | Edit `LinkerHand/config/setting.yaml` so only the correct side has `EXISTS: True`. |
| Both hands are set to `EXISTS: True` | Set one side to `False`. The dashboard supports one hand at a time and prefers left if both are enabled. |
| Preset buttons are missing | Check `LinkerHand/config/L10_positions.yaml`. The selected side (`LEFT_HAND` or `RIGHT_HAND`) must contain a list of actions. |
| A preset does not move or is ignored | Make sure the preset `POSITION` has exactly ten values from `0` to `255`. |
| Slider movement is too sudden | Stop moving sliders, use a safe preset, and lower speed in code/config if needed before more testing. |
| `PASSWORD` in `setting.yaml` is wrong | Prefer manual `sudo ip link ...` CAN setup before launch. If relying on SDK auto-open, update `PASSWORD`. |
| Another tool seems to control the hand | Close terminal control, ROS nodes, glove/mocap controllers, and other Python scripts. |
| Full `requirements.txt` install fails | Use the minimal install listed above. The full file includes extra SDK/simulation packages not required for the dashboard. |

## When You Are Done

Close the dashboard window.

Optionally bring CAN down:

```bash
sudo ip link set can0 down
```

Power off the hand when it is safe to do so.
