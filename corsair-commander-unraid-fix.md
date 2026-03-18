# Fix: Corsair Commander Pro USB Address Changes on Unraid Reboot

> **Affected plugin:** Dynamic Fan Controller  
> **Device:** Corsair Commander Pro (`1b1c:0c10`)  
> **Time to fix:** ~5 minutes

## Background

The Linux kernel dynamically assigns USB bus addresses (e.g. `/dev/bus/usb/001/006`) based on the order devices are discovered at boot. This address can change on every restart, causing the Dynamic Fan Controller plugin to lose track of the Corsair Commander Pro.

The fix is a persistent **udev symlink** — a stable device path (`/dev/corsair-commander`) that always resolves to the correct device regardless of what bus address the kernel assigns.

---

## Prerequisites

- SSH access to your Unraid server
- Corsair Commander Pro connected via USB
- Dynamic Fan Controller plugin installed

---

## Step 1 — Locate Your Device on the USB Bus

SSH into your Unraid server and confirm the Commander Pro is visible:

```bash
lsusb
```

You should see a line like:

```
Bus 001 Device 006: ID 1b1c:0c10 Corsair Commander PRO
```

Note the **Bus** and **Device** numbers — you'll need them in Step 2.

---

## Step 2 — Find the Device Serial Number

Run `udevadm` to inspect the device attributes. Replace `001/006` with your actual bus/device numbers if different:

```bash
udevadm info --name=/dev/bus/usb/001/006 --attribute-walk | grep -E "serial|idVendor|idProduct"
```

Look at the **first block** of results (your device's own attributes, not the parent hub):

```
    ATTR{idProduct}=="0c10"
    ATTR{idVendor}=="1b1c"
    ATTR{serial}=="160503C48427131A"
```

> ⚠️ **Note:** The serial number above is an example. Use **your own serial number** from the command output in all steps below.

---

## Step 3 — Create the udev Rule

Create a udev rule that generates a persistent symlink at `/dev/corsair-commander`, pinned to your device's exact serial number. Replace `YOUR_SERIAL_HERE` with the serial from Step 2:

```bash
echo 'SUBSYSTEM=="usb", ATTRS{idVendor}=="1b1c", ATTRS{idProduct}=="0c10", ATTRS{serial}=="YOUR_SERIAL_HERE", SYMLINK+="corsair-commander"' > /etc/udev/rules.d/99-corsair-commander.rules
```

Reload udev and trigger the rule:

```bash
udevadm control --reload-rules && udevadm trigger
```

Verify the symlink was created:

```bash
ls -la /dev/corsair-commander
```

Expected output:

```
lrwxrwxrwx 1 root root 15 Mar 18 08:18 /dev/corsair-commander -> bus/usb/001/006
```

> ✅ If you see the symlink pointing to a `bus/usb/...` path, the rule is working. The exact bus/device numbers in the arrow don't matter — they will change on reboot, but the symlink will always follow the correct device.

---

## Step 4 — Persist the Rule Across Reboots

Unraid stores `/etc/udev/rules.d/` in RAM — it resets on every reboot. Add the rule to `/boot/config/go` so it runs on every startup:

```bash
nano /boot/config/go
```

Add the following block **before** any `exit 0` line. Replace `YOUR_SERIAL_HERE` with your actual serial number:

```bash
# Persistent udev rule for Corsair Commander Pro
echo 'SUBSYSTEM=="usb", ATTRS{idVendor}=="1b1c", ATTRS{idProduct}=="0c10", ATTRS{serial}=="YOUR_SERIAL_HERE", SYMLINK+="corsair-commander"' > /etc/udev/rules.d/99-corsair-commander.rules
udevadm control --reload-rules
udevadm trigger
```

Save: **Ctrl+O** → **Enter**, then exit: **Ctrl+X**

---

## Step 5 — Update the Dynamic Fan Controller Plugin

In the Unraid web UI, open the **Dynamic Fan Controller** plugin settings and change the device path to:

```
/dev/corsair-commander
```

Save and apply the settings.

---

## Step 6 — Test With a Reboot

Reboot Unraid, then verify the symlink is still present:

```bash
ls -la /dev/corsair-commander
```

The symlink should be pointing to your Commander Pro at whatever bus address the kernel assigned this boot. The fan controller plugin should initialize automatically without any intervention.

---

## Troubleshooting

**Symlink not created after running the commands**  
Try unplugging and replugging the Commander Pro USB cable after running `udevadm trigger`, then check again with `ls -la /dev/corsair-commander`.

**Rule doesn't persist after reboot**  
Check that your `/boot/config/go` additions were saved to the USB boot drive (not just RAM). Run `cat /boot/config/go` to confirm the lines are present.

**Plugin still loses the device after reboot**  
Some versions of the Dynamic Fan Controller plugin have a known initialization issue on boot. As a workaround, open the plugin settings, make a minor change, and hit Apply after the system finishes booting.

**Multiple Commander Pro devices**  
Create a separate rule file for each unit using distinct symlink names (e.g. `corsair-commander-1`, `corsair-commander-2`) differentiated by their unique serial numbers.

---

## Reference

| Field | Value |
|---|---|
| Vendor ID | `1b1c` |
| Product ID | `0c10` |
| Stable symlink | `/dev/corsair-commander` |
| udev rule file | `/etc/udev/rules.d/99-corsair-commander.rules` |
| Startup script | `/boot/config/go` |
