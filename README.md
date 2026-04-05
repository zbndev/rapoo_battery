# Rapoo VT3 Pro Max Battery Reader

Read battery percentage and charging status from Rapoo gaming mice on Linux via their 2.4GHz USB dongle. No kernel module or proprietary driver needed.

## Supported Devices

| Device | USB ID (VID:PID) |
|--------|-----------------|
| VT3 Pro Max | 24ae:1244 |
| VT3 Max Gen-2 | 24ae:3102 |
| VT9 Pro | 24ae:1417 |
| VT9 Pro Mini | 24ae:3103 |

## How It Works

The USB dongle exposes two HID interfaces:
- **Interface 0** — Standard mouse input (X/Y/buttons/wheel)
- **Interface 1** — Vendor-defined HID collections including battery status

The mouse sends a battery report (Report ID `0xBB`, 7 bytes) every ~3 seconds:

```
Byte 0: 0xBB       — Report marker
Byte 1: 0xB0       — Status flags
Byte 2: 0x81       — Mode flags
Byte 3: 0x20       — Polling/connection info
Byte 4: 0x03       — Connection type (2.4GHz)
Byte 5: 0x01/0x02  — Charging state (discharging / charging)
Byte 6: 0x00-0x64  — Battery percentage (0-100)
```

A second report type (`0xBC`, 10 bytes) may appear during cable plug/unplug transitions and is ignored.

### Battery update frequency

Reports arrive every ~3 seconds, but the **battery percentage value itself only changes every 5-10 minutes** — this is a firmware design choice by Rapoo (saves power by not constantly reading the ADC). The charging state (`0x01`/`0x02`) updates instantly when you plug or unplug the cable.

## Installation

```bash
# Copy script
sudo cp rapoo-battery.py /usr/local/bin/rapoo-battery
sudo chmod +x /usr/local/bin/rapoo-battery

# (Optional) Allow non-root access via udev rule
echo 'KERNEL=="hidraw*", ATTRS{idVendor}=="24ae", MODE="0666"' | sudo tee /etc/udev/rules.d/99-rapoo-mouse.rules
sudo udevadm control --reload-rules && sudo udevadm trigger
# Log out and back in for the rule to take full effect
```

## Usage

```bash
# Read battery once
rapoo-battery

# Monitor continuously (single-line, updates every ~3 seconds)
rapoo-battery --watch

# JSON output (for waybar, polybar, i3bar, etc.)
rapoo-battery --json

# Show raw report bytes
rapoo-battery --raw

# Specify device explicitly (auto-detection if omitted)
rapoo-battery --device /dev/hidraw7

# Custom max wait time for watch mode
rapoo-battery --watch --interval 10
```

### Example output

```
$ rapoo-battery
Rapoo Mouse:   ++[=========-----------]  45%  Charging

$ rapoo-battery --json
{"battery_percent": 45, "charging": true, "device": "/dev/hidraw7"}

$ rapoo-battery --watch
Monitoring Rapoo mouse on /dev/hidraw7
Press Ctrl+C to stop.

[20:30:15]   ++[=========-----------]  45%  ⚡ CHARGING  #42
```

## Waybar Integration

```json
"custom/rapoo": {
    "exec": "rapoo-battery --json",
    "return-type": "json",
    "format": "{charging_icon} {battery_percent}%",
    "format-charging": "⚡ {battery_percent}%",
    "interval": 30
}
```

## Requirements

- Python 3.6+
- Linux with hidraw support (any modern kernel)
- `lsusb` and `udevadm` (from `usbutils` and `systemd` packages)
- Root access or udev rule for `/dev/hidraw*` permissions
