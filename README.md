# AIC8800D80 Linux Driver

Out-of-tree USB Wi-Fi + Bluetooth driver for the AIC8800D80 chipset. Supports devices such as the Tenda U11, Tenda AX913B, Brostrend adapters, and various OEM variants.

**Tested kernels:** 6.1 (Debian 12), 6.16 (Ubuntu 25.04), 7.0.9 (Fedora)

### Disclaimer

I did not develop this software. The code originates from the Tenda U11 vendor driver. I made modifications to adapt it to newer kernel versions and to add Bluetooth support. I can only address compilation issues, not hardware-specific problems.

### Attention

Before installing, delete any existing AIC8800 firmware:

```bash
sudo rm -rf /lib/firmware/aic8800*
```

Stale firmware from a previous install causes system freezes on driver load.

---

## Kernel Modules

Three modules are built and installed:

| Module | Purpose |
|--------|---------|
| `aic8800_fdrv` | Main Wi-Fi driver (cfg80211/fullmac) |
| `aic_load_fw` | Firmware + Bluetooth firmware loader |
| `aic_btusb` | Custom AIC Bluetooth USB driver |

The `aic_btusb` module must handle the Bluetooth interface instead of the kernel's generic `btusb` driver. The `modprobe/aic8800-bt.conf` file (installed to `/etc/modprobe.d/`) enforces this via a `softdep` rule.

---

## Supported Devices

Vendor IDs: `0xA69C` (AIC), `0x368B` (AIC v2), `0x2604` (Tenda), `0x3625` (Tenda v2), `0x2357` (TP-Link/Mercury)

Notable supported adapters:
- Tenda U11, U11 PRO, U2
- AIC8800D80/D81/D83/D84/D85 variants
- AIC8800M80 CUS1–CUS8 series
- AIC8800FC CUS1–CUS6 series
- UGREEN AX900 (AIC8800D80)
- TP-Link Archer TX1U Nano
- Mercury OEM variants
- Brostrend adapters
- "Pandora" clone (USB ID `1111:1111` — auto mode-switched via udev + `usb_modeswitch`)

---

## Installation

### Method 1: Quick Installation (Recommended)

See [INSTALL_SCRIPT.md](INSTALL_SCRIPT.md) for the automated installer which handles distro detection, DKMS setup, firmware install, and Secure Boot.

```bash
sudo ./install.sh
```

### Method 2: Manual Installation

#### 1. Copy udev rules

```bash
sudo cp aic.rules /lib/udev/rules.d/
sudo udevadm control --reload-rules
```

#### 2. Copy firmware

All firmware variants must be installed:

```bash
for d in fw/aic8800*; do sudo cp -r "$d" /lib/firmware/; done
```

Or individually:

```bash
sudo cp -r fw/aic8800D80  /lib/firmware/
sudo cp -r fw/aic8800D80X2 /lib/firmware/
sudo cp -r fw/aic8800DC   /lib/firmware/
sudo cp -r fw/aic8800DLN  /lib/firmware/
```

#### 3. Install usb_modeswitch config (Pandora clone only)

```bash
sudo mkdir -p /etc/usb_modeswitch.d
sudo cp usb_modeswitch/1111_1111 /etc/usb_modeswitch.d/1111:1111
```

#### 4. Install modprobe config for Bluetooth

```bash
sudo cp modprobe/aic8800-bt.conf /etc/modprobe.d/
```

#### 5. Build and install the driver

```bash
cd drivers/aic8800
make
sudo make install
```

After a kernel update, rebuild:

```bash
make clean && make && sudo make install
```

---

## Loading the Driver

```bash
sudo modprobe aic8800_fdrv
```

The `aic_load_fw` and `aic_btusb` modules load automatically as dependencies.

### Verify modules are loaded

```bash
lsmod | grep aic
```

Expected output:

```
aic_btusb       196608  0
aic8800_fdrv    536576  0
aic_load_fw      69632  1  aic8800_fdrv
```

---

## Verify Wi-Fi

```bash
iwconfig
```

Check kernel logs if the interface does not appear:

```bash
sudo dmesg | grep -i aic
```

---

## Bluetooth Support

Bluetooth is handled by the `aic_btusb` module (included in this repo), **not** the kernel's generic `btusb` driver. The `aic_load_fw` module uploads the firmware; `aic_btusb` then drives the HCI interface.

The `modprobe/aic8800-bt.conf` installs a `softdep btusb pre: aic_btusb` rule so the correct driver claims the device automatically.

### Verify Bluetooth

```bash
hciconfig -a
```

You should see an `hci0` device. If it is down:

```bash
sudo hciconfig hci0 up
```

Scan for devices:

```bash
bluetoothctl
power on
scan on
```

### Bluetooth Troubleshooting

Run the diagnostic script for a full report:

```bash
chmod +x diagnose_bt.sh
sudo ./diagnose_bt.sh
```

Common issues:

1. **HCI_Reset timeout / btusb claiming the device**
   Ensure the modprobe config is installed:
   ```bash
   sudo cp modprobe/aic8800-bt.conf /etc/modprobe.d/
   sudo modprobe -r btusb aic_btusb && sudo modprobe aic_btusb
   ```

2. **Firmware not found**
   Key Bluetooth firmware files (must be in `/lib/firmware/aic8800D80/`):
   - `fw_patch_8800d80_u02.bin`
   - `fw_patch_table_8800d80_u02.bin`
   - `fw_adid_8800d80_u02.bin`
   ```bash
   sudo dmesg | grep -iE "aicbt|bluetooth|hci|fw_patch"
   ```

3. **RF-Kill blocking Bluetooth**
   ```bash
   rfkill list bluetooth
   sudo rfkill unblock bluetooth
   ```

---

## Pandora Clone (`1111:1111`)

Some adapters enumerate as USB ID `1111:1111` (mass storage mode) and require a mode switch before the Wi-Fi/BT interfaces appear. The udev rules in `aic.rules` and the `usb_modeswitch/1111_1111` config handle this automatically once installed.

After mode switch the device re-enumerates as `a69c:8d81` or `a69c:8d83` and the driver binds normally.

---

## Bazzite / Immutable OS

See [bazzite/README.md](bazzite/README.md) for building an RPM via `rpmbuild` and installing with `rpm-ostree`.

---

## Debugging

```bash
sudo dmesg | grep -i aic       # driver messages
sudo dmesg | grep -iE "bt|hci" # bluetooth messages
iwconfig                        # Wi-Fi interface
hciconfig -a                    # Bluetooth interface
sudo ./diagnose_bt.sh           # full BT diagnostic
```
