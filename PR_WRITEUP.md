# PR: Bluetooth Support, New Chip Variants, Kernel 6.13–6.17+ Compatibility, SDK v5.0

## The Problem This Solves

The AIC8800D80 is a combo Wi-Fi + Bluetooth chip. Every device in the wild — Tenda U11,
TP-Link adapters, and AIC-branded dongles — ships with both radios on the same USB device.
Yet until now, this driver only exposed Wi-Fi. Bluetooth was silently ignored.

Users who plugged in their adapter and ran `bluetoothctl list` saw nothing. No errors, no
module, no hint that BT hardware exists. The chip was capable; the driver wasn't.

This PR changes that.

---

## What Changed

### 1. Full Bluetooth Module — `aic_btusb`

A new kernel module (`drivers/aic8800/aic_btusb/`) provides a complete HCI driver for the
AIC8800D80's Bluetooth radio. The module:

- Registers a full HCI controller with the Linux Bluetooth subsystem
- Handles BT firmware upload over USB (separate from Wi-Fi firmware)
- Supports BLE (advertising, central + peripheral roles)
- Supports A2DP, HFP, AVRCP, OBEX — full Classic BT profile stack
- Auto-loads via `softdep` when the USB adapter is plugged in

This was verified live: after loading `aic_btusb`, a scan discovered nearby devices including
a Pixel 8a and Sony WH-1000XM4 headphones. The controller negotiated pairing correctly.

### 2. New Chip Variants

Three new chip variants now have full support:

| Variant | Compat files added | Firmware added |
|---------|--------------------|----------------|
| AIC8800D80N | `aicwf_compat_8800d80n.c/h` | `fw/aic8800D80N/` (11 blobs) |
| AIC8800D80X2 | `aicwf_compat_8800d80x2.c/h` updated | `fw/aic8800D80X2/` (11 blobs) |
| AIC8800DLN | `aicwf_compat_8800dln.c/h` | `fw/aic8800DLN/` (4 blobs) |

Without these, users with newer hardware revisions get silent probe failures at `aicwf_usb_probe()`.

### 3. Firmware Updates

All existing chip firmware blobs updated to SDK v5.0 levels:

- `fw/aic8800D80/` — updated `fmacfw`, `lmacfw_rf`, `fw_ble_scan`, BT patch files
- `fw/aic8800DC/` — updated calibration, patch, and RF firmware
- Added `fw_patch_8800d80_u04.bin` for u04 silicon revision support

**Why this matters:** Stale firmware causes system freezes on newer silicon revisions (u04).
The u02 firmware was silently running on u04 chips and producing hangs that looked like
kernel bugs.

### 4. Kernel 6.13–6.17+ Compatibility

Two breaking kernel API changes handled:

**Linux 6.13 — `MODULE_IMPORT_NS`**
The `MODULE_IMPORT_NS` macro signature changed. Without this fix, the module fails to load
with:
```
modprobe: ERROR: could not insert 'aic8800_fdrv': Invalid argument
```

**Linux 6.17 — `get_tx_power` callback**
The `cfg80211_ops.get_tx_power` callback signature changed (added `link_id` parameter).
Without this fix, the driver fails to compile on 6.17+:
```
error: incompatible function pointer types
```

Both fixes are gated behind version checks in `rwnx_compat.h` so older kernels are unaffected.

### 5. TP-Link / Mercury USB IDs

Users with TP-Link and Mercury branded AIC8800D80 adapters (USB VID `0x2357`) reported
their devices not being recognized. Their hardware is identical — just a different vendor
string burned into the USB descriptor. Their IDs are now in `aicwf_usb.h`.

### 6. Install Script and DKMS Improvements

- `install.sh` now copies all firmware variants (D80, D80N, D80X2, DC, DLN) — previously
  only D80 was copied, leaving other variant users with missing firmware on fresh installs
- BT module auto-load wired via `/etc/modprobe.d/aic8800-bt.conf` (softdep)
- udev rule added (`aic.rules`) to trigger firmware load on USB hotplug
- DKMS `make` now correctly receives `KDIR` — fixes rebuild failures on custom kernel paths
- `diagnose_bt.sh` added for first-time BT debugging

### 7. Bazzite / Immutable OS Support

Users on Bazzite, Silverblue, and other rpm-ostree systems cannot install DKMS modules
conventionally. An RPM spec file (`bazzite/aic8800d80.spec`) and `bazzite/README.md` now
document how to build and install via `rpmbuild` + `rpm-ostree override replace`.

---

## Why This Should Be Merged

**1. The hardware already ships with Bluetooth.** Every user who buys one of these adapters
expects Wi-Fi + BT. They're getting half a device today.

**2. Kernel breakage is real and recent.** The 6.17 compile error and 6.13 load error affect
any user on a current Fedora, Arch, or Ubuntu 25.04 system. The driver is unusable for them
without these fixes.

**3. New chip variants are in the wild.** The D80N and D80X2 are shipping now. Users report
probe failures that look identical to "driver not working" — they disappear silently.

**4. Firmware staleness causes freezes.** This is the kind of bug that gets filed against
the kernel, not the driver. Users lose hours debugging the wrong thing.

**5. This was tested on real hardware.** The following was verified on kernel `7.0.9-202.fc44.x86_64`:

```
Wi-Fi:  wlan0 connected, -55 dBm signal, 480 Mbps RX, Wi-Fi 6 (HE-MCS 9)
BT:     aic_btusb loaded, HCI controller registered, scan discovers devices
```

---

## Files Changed (Summary)

```
drivers/aic8800/Makefile                 — add aic_btusb build/install/clean targets
drivers/aic8800/aic_btusb/              — new: full BT HCI driver module (5500+ lines)
drivers/aic8800/aic8800_fdrv/           — new chip variant compat files, kernel compat fixes
drivers/aic8800/aic_load_fw/            — BT firmware upload updates, new variant support
fw/aic8800D80N/, fw/aic8800DLN/         — new firmware directories
fw/aic8800D80X2/                        — extended firmware for D80X2
fw/aic8800D80/, fw/aic8800DC/           — updated firmware blobs (SDK v5.0)
install.sh                              — full firmware install, BT auto-load, udev
modprobe/aic8800-bt.conf               — new: softdep for BT auto-load
aic.rules                               — new: udev hotplug rule
diagnose_bt.sh                          — new: BT diagnostic script
bazzite/                               — new: RPM spec + instructions for immutable OSes
```

Total: **158 files changed, 11,758 insertions, 1,554 deletions**
