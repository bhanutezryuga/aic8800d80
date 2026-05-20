# AIC8800D80 Driver — Field Verification Case Study

## Overview

This document captures a live verification session of the AIC8800D80 out-of-tree Linux driver
on Fedora 44 (kernel `7.0.9-202.fc44.x86_64`). It covers every command used, what it revealed,
and why it matters — intended as a learning reference for Linux driver debugging and Wi-Fi/BT
verification on modern kernels.

---

## 1. Verifying Driver Load

### Command
```bash
lsmod | grep aic
```

### Output
```
aic8800_fdrv         1110016  0
aic_btusb             118784  0
cfg80211             1601536  4 ath,mac80211,ath10k_core,aic8800_fdrv
aic_load_fw           155648  1 aic8800_fdrv
```

### What This Tells You

| Column | Meaning |
|--------|---------|
| Module name | The `.ko` file loaded into the kernel |
| Size (bytes) | Memory footprint of the module |
| `0` / `1` after size | Reference count — how many other modules depend on it |
| Modules listed after | Reverse dependency chain |

- `aic_load_fw` has ref count `1` (used by `aic8800_fdrv`) — correct, firmware loader must stay alive while Wi-Fi runs
- `aic_btusb` ref count `0` — BT module loaded but not actively claimed by another module (normal for a standalone HCI driver)
- `cfg80211` lists `aic8800_fdrv` as a user — confirms the driver registered with the kernel's wireless subsystem

---

## 2. Discovering Wireless Interfaces

### Why `iwconfig` Is Deprecated
`iwconfig` comes from the `wireless-tools` package, which targets the older WEXT (Wireless Extensions)
kernel API. Modern kernels use **cfg80211/nl80211**. The `iw` tool speaks nl80211 directly.

### Command
```bash
iw dev
```

### Output Breakdown
```
phy#1                          ← physical radio #1 (AIC8800D80 USB adapter)
    Interface wlan0            ← kernel network interface name
        ifindex 4              ← kernel interface index (unique ID)
        wdev 0x100000001       ← wireless device handle (used internally)
        addr 22:e5:92:b7:1b:ca ← MAC address (randomized on this system)
        ssid Isha-3-1-5G       ← currently associated SSID
        type managed           ← station/client mode (vs AP, monitor, mesh)
        channel 153 (5765 MHz) ← current channel and frequency
        width: 80 MHz          ← channel width (20/40/80/160 MHz)
        center1: 5775 MHz      ← center frequency of the 80 MHz block
        txpower 18.00 dBm      ← transmit power (~63 mW)

phy#0                          ← physical radio #0 (built-in laptop/desktop card)
    Interface wlp3s0           ← predictable interface name (PCI slot 3, function 0)
```

**Key insight:** Two `phy#` entries = two independent Wi-Fi radios. The USB adapter appears as
`phy#1`/`wlan0`; the built-in PCIe card is `phy#0`/`wlp3s0`.

---

## 3. Signal Strength and Connection Speed

### Command
```bash
iw dev wlan0 link
iw dev wlan0 station dump
```

### `iw dev wlan0 link` — Quick Status
```
Connected to b4:b0:24:2f:f1:63   ← AP MAC address (BSSID)
SSID: Isha-3-1-5G
freq: 5765.0                      ← frequency in MHz
signal: -55 dBm                   ← received signal strength
rx bitrate: 480.3 MBit/s 80MHz HE-MCS 9 HE-NSS 1 HE-GI 0 HE-DCM 0
tx bitrate: 360.3 MBit/s 80MHz HE-MCS 7 HE-NSS 1 HE-GI 0 HE-DCM 0
```

### Signal Strength Reference

| dBm range | Quality |
|-----------|---------|
| -30 to -50 | Excellent |
| -50 to -65 | Good |
| -65 to -75 | Fair |
| -75 to -85 | Poor |
| below -85 | Unusable |

**-55 dBm = good signal.**

### Decoding the Bitrate String

`480.3 MBit/s 80MHz HE-MCS 9 HE-NSS 1 HE-GI 0 HE-DCM 0`

| Token | Meaning |
|-------|---------|
| `HE` | High Efficiency = 802.11ax (Wi-Fi 6) |
| `MCS 9` | Modulation and Coding Scheme index — 9 = 1024-QAM, 5/6 coding rate (highest efficiency) |
| `NSS 1` | Number of Spatial Streams — 1 stream (single antenna) |
| `GI 0` | Guard Interval — 0 = 0.8µs (normal) |
| `DCM 0` | Dual Carrier Modulation off (only used at very low SNR) |
| `80MHz` | Channel width used for this transmission |

**Theoretical max for HE 80MHz NSS=1 MCS=9: ~600 Mbps.** Getting 480 Mbps RX is ~80% efficiency — healthy.

Contrast with built-in card:
```
rx bitrate: 292.6 MBit/s VHT-MCS 6 80MHz short GI VHT-NSS 1
```
`VHT` = Very High Throughput = 802.11ac (Wi-Fi 5). Older standard, lower throughput ceiling.

### `iw dev wlan0 station dump` — Detailed Counters
This command queries the AP's station table entry for this client. Key fields:

```
signal:      -55 [-55, -73] dBm   ← [primary chain, secondary chain]
signal avg:  -54 dBm              ← rolling average
tx retries:  0                    ← zero retries = clean channel
tx failed:   0                    ← no dropped packets
beacon loss: 0                    ← AP reachable consistently
inactive time: 16909 ms           ← ms since last frame exchange
```

Zero retries and zero failures = driver and RF path are healthy.

---

## 4. Fair Comparison: Same AP, Two Adapters

To compare the USB adapter vs built-in fairly, `wlp3s0` was connected to the same AP:

```bash
# Check saved connection profiles
nmcli connection show | grep -i isha

# Connect built-in card to same AP
nmcli device wifi connect Isha-3-1-5G ifname wlp3s0
```

`nmcli` (NetworkManager CLI) manages connections at a higher level than `iw` — it handles
authentication, DHCP, and connection profiles. `iw` is read-only diagnostic; `nmcli` is
for managing connections.

### Results: Same AP, Same Time

| Metric | wlan0 (AIC8800D80 USB) | wlp3s0 (built-in) |
|--------|------------------------|-------------------|
| Signal | **-55 dBm** | -78 dBm |
| RX speed | **480.3 Mbps** | 292.6 Mbps |
| Standard | **Wi-Fi 6 (HE/802.11ax)** | Wi-Fi 5 (VHT/802.11ac) |
| MCS index | 9 | 6 |

**23 dBm signal difference** = ~200× stronger received power from USB adapter.
Better antenna placement (USB dongle positioned openly) vs internal card behind laptop chassis.

### Reconnect built-in to original network
```bash
nmcli device wifi connect Airtel_Isha3-2 ifname wlp3s0
```

---

## 5. Bluetooth Driver Verification

### Check modules loaded
```bash
lsmod | grep aic
```
Confirmed `aic_btusb` loaded.

### Identify BT controllers
```bash
bluetoothctl list
```
```
Controller E8:2A:44:D2:BB:16 fedora #1 [default]   ← built-in
Controller 8C:77:3B:61:E6:17 fedora                ← AIC8800D80 USB
```

### Determine which is which via sysfs
```bash
cat /sys/class/bluetooth/hci0/device/uevent
cat /sys/class/bluetooth/hci1/device/uevent
```

The `uevent` file in sysfs exposes kernel device metadata:

```
# hci0 (built-in)
PRODUCT=4ca/3015/1     ← USB VID=0x04CA (Lite-On Technology, common laptop BT)

# hci1 (AIC)
PRODUCT=368b/8d81/100  ← USB VID=0x368B (AIC v2 vendor ID — matches aicwf_usb.h)
```

Cross-referencing against `drivers/aic8800/aic8800_fdrv/aicwf_usb.h` confirmed `368B` is
the AIC vendor ID. `hci1` = AIC adapter.

### Select AIC controller and scan
```bash
bluetoothctl select 8C:77:3B:61:E6:17    # switch default controller
bluetoothctl scan on                      # active scan for 10s
bluetoothctl devices                      # list discovered devices
```

Pixel 8a appeared at `08:8B:C8:56:C4:5B` after active scan (was absent from cached list
because it wasn't previously paired — cached list only shows known/paired devices).

### BT controller capabilities confirmed
`bluetoothctl show` revealed the AIC controller supports:
- **A2DP** (audio streaming — Source and Sink)
- **HFP/HSP** (handsfree/headset profile)
- **AVRCP** (media remote control)
- **OBEX** (file transfer)
- **BLE** (Bluetooth Low Energy — advertising, central + peripheral roles)
- **MIDI over BLE** (UUID `03b80e5a`)

---

## 6. Key Commands Reference

| Command | Purpose |
|---------|---------|
| `lsmod \| grep aic` | Verify driver modules are loaded |
| `iw dev` | List all wireless interfaces and their state |
| `iw dev <iface> link` | Current connection: AP, signal, bitrate |
| `iw dev <iface> station dump` | Detailed packet counters and signal stats |
| `nmcli connection show` | List saved NetworkManager profiles |
| `nmcli device wifi connect <SSID> ifname <iface>` | Connect specific interface to SSID |
| `nmcli device status` | Overview of all network devices |
| `bluetoothctl list` | List all BT controllers |
| `bluetoothctl select <MAC>` | Switch active controller |
| `bluetoothctl show <MAC>` | Controller details and capabilities |
| `bluetoothctl scan on` | Active scan for nearby BT devices |
| `bluetoothctl devices` | List discovered/paired devices |
| `bluetoothctl pair <MAC>` | Initiate pairing with device |
| `cat /sys/class/bluetooth/hciX/device/uevent` | Identify controller hardware via sysfs |
| `sudo dmesg \| grep -i aic` | Kernel log messages from AIC driver |

---

## 7. What Was Verified

- [x] Kernel modules load without errors on kernel 7.0.9-202.fc44.x86_64
- [x] Wi-Fi interface appears and associates (`wlan0` on 5 GHz, Wi-Fi 6)
- [x] Signal strength healthy (-55 dBm)
- [x] Throughput healthy (480 Mbps RX, 360 Mbps TX)
- [x] Driver negotiates Wi-Fi 6 (HE/802.11ax) — not just Wi-Fi 5
- [x] USB adapter outperforms built-in card on same AP
- [x] BT module (`aic_btusb`) loads and registers HCI controller
- [x] BT controller powered, pairable, supports A2DP + BLE
- [x] BT scan discovers nearby devices (Pixel 8a, Sony WH-1000XM4, etc.)
