# 🖥️ Unraid Fan Controller Fix
### Corsair Commander Pro — Persistent USB Device Path via udev

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Unraid](https://img.shields.io/badge/Platform-Unraid-orange.svg)](https://unraid.net)
[![Device](https://img.shields.io/badge/Device-Corsair%20Commander%20Pro-red.svg)](https://www.corsair.com)
[![Plugin](https://img.shields.io/badge/Plugin-Dynamic%20Fan%20Controller-green.svg)](https://forums.unraid.net)

> Fix for the Corsair Commander Pro USB fan controller losing its device address on every Unraid reboot, causing the Dynamic Fan Controller plugin to stop working.

---

## 🔍 The Problem

Every time Unraid restarts, the Linux kernel reassigns USB bus addresses dynamically based on device discovery order. This means your Corsair Commander Pro may be at `/dev/bus/usb/001/006` one boot and `/dev/bus/usb/001/009` the next. The Dynamic Fan Controller plugin loses track of the device and stops controlling your fans.

## ✅ The Fix

Create a persistent **udev symlink** at `/dev/corsair-commander` that is pinned to the device's unique serial number. This stable path survives every reboot regardless of what bus address the kernel assigns, and the plugin can always find the device.

---

## 📋 Prerequisites

- SSH access to your Unraid server
- Corsair Commander Pro connected via USB
- Dynamic Fan Controller plugin installed

---

## 🚀 Quick Start

### Step 1 — Locate your device

```bash
lsusb
```

Look for:
```
Bus 001 Device 006: ID 1b1c:0c10 Corsair Commander PRO
```

### Step 2 — Get the serial number

Replace `001/006` with your actual bus/device numbers:

```bash
udevadm info --name=/dev/bus/usb/001/006 --attribute-walk | grep -E "serial|idVendor|idProduct"
```

Note the value of `ATTR{serial}` from the first block of results.

### Step 3 — Create the udev rule

Replace `YOUR_SERIAL_HERE` with your serial number from Step 2:

```bash
echo 'SUBSYSTEM=="usb", ATTRS{idVendor}=="1b1c", ATTRS{idProduct}=="0c10", ATTRS{serial}=="YOUR_SERIAL_HERE", SYMLINK+="corsair-commander"' > /etc/udev/rules.d/99-corsair-commander.rules
```

Reload and verify:

```bash
udevadm control --reload-rules && udevadm trigger
ls -la /dev/corsair-commander
```

### Step 4 — Persist across reboots

Add to `/boot/config/go` (before any `exit 0`):

```bash
nano /boot/config/go
```

```bash
# Persistent udev rule for Corsair Commander Pro
echo 'SUBSYSTEM=="usb", ATTRS{idVendor}=="1b1c", ATTRS{idProduct}=="0c10", ATTRS{serial}=="YOUR_SERIAL_HERE", SYMLINK+="corsair-commander"' > /etc/udev/rules.d/99-corsair-commander.rules
udevadm control --reload-rules
udevadm trigger
```

Save: **Ctrl+O** → **Enter** → **Ctrl+X**

### Step 5 — Update the plugin

In the Unraid web UI, set the Dynamic Fan Controller device path to:

```
/dev/corsair-commander
```

### Step 6 — Reboot and verify

```bash
ls -la /dev/corsair-commander
```

The symlink should be present after every reboot. ✅

---

## 📖 Full Guide

See [corsair-commander-unraid-fix.md](corsair-commander-unraid-fix.md) for the complete step-by-step walkthrough including detailed explanations, expected output, and troubleshooting.

---

## 🛠️ Troubleshooting

| Problem | Solution |
|---|---|
| Symlink not created | Unplug and replug the USB cable after running `udevadm trigger` |
| Rule lost after reboot | Verify `/boot/config/go` was saved — run `cat /boot/config/go` to check |
| Plugin still loses device | Open plugin settings, make a minor change, hit Apply to reinitialize |
| Multiple Commander Pro units | Use separate rule files with unique symlink names per serial number |

---

## 📦 Repository Contents

| File | Description |
|---|---|
| `README.md` | This file — quick start and overview |
| `corsair-commander-unraid-fix.md` | Full step-by-step guide |

---

## 📝 Changelog

### v1.0.0 — March 2026
- Initial release
- udev symlink fix for Corsair Commander Pro on Unraid
- Persistent `/boot/config/go` method documented
- Troubleshooting section added
- Full step-by-step guide included

---

## 📄 License

This project is licensed under the **MIT License** — see the [LICENSE](LICENSE) file for details.

You are free to use, copy, modify, and distribute this guide. If it helped you, a ⭐ on the repo is always appreciated!

---

## 👤 Credits

**Author:** [@mejacobarussell](https://github.com/mejacobarussell)

**Built with help from:**
- The Unraid community forums for background on the Dynamic Fan Controller plugin behavior
- Linux `udev` documentation for persistent device naming best practices

---

## 🔗 Related Resources

- [Unraid Forums](https://forums.unraid.net)
- [Dynamic Fan Controller Plugin Thread](https://forums.unraid.net/topic/59270-dynamic-fan-controller-plugin/)
- [Unraid Community Apps](https://unraid.net/community/apps)

---

*If this fix helped you, consider sharing the link on the [Unraid forums](https://forums.unraid.net) or [r/unraid](https://reddit.com/r/unraid) so others can find it.*
