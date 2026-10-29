---
title: CLI Reference (hmsutil)
tags:
  - cli
  - development
  - hmsutil
created: 2026-10-29
---

# `hmsutil` CLI Reference

`hmsutil` is the cross-platform command-line tool and interactive terminal dashboard for communicating with the nRF52840 HSM Dongle.

---

## Command Syntax

```bash
hmsutil [command] [flags]
```

### Global Flags
- `-p, --port <string>`: Specify serial port path (e.g. `/dev/cu.usbmodem...` on macOS, `/dev/ttyACM...` on Linux, `COM...` on Windows). If omitted, `hmsutil` auto-detects the dongle.

---

## Commands

### 1. Firmware Management
- **`hmsutil firmware info`**: Queries the dongle for its active firmware version, MCUboot partition slot, image size, and SHA-256 digest.
- **`hmsutil firmware update --file <path>`**: Streams a signed MCUboot application binary (`zephyr.signed.bin`) in chunked packets to the secondary candidate slot, validates the payload, marks it for upgrade, and requests a reboot into MCUboot.

### 2. Device Status & Diagnostics
- **`hmsutil info`**: Queries standard YubiHSM 2 device information (serial number, version, supported crypto algorithms).
- **`hmsutil echo [message]`**: Sends an echo packet and calculates round-trip latency.

### 3. Session & Security State
- **`hmsutil unlock`** (alias **`auth`**): Authenticates session, transitions the dongle to unlocked state, and turns the RGB LED solid **GREEN**.
- **`hmsutil lock`**: Closes session, transitions the dongle to locked state, and turns the RGB LED solid **RED**.
- **`hmsutil reset`**: Resets device state back to unconfigured, transitioning the RGB LED to pulsating **BLUE**.

### 4. Interactive TUI & Tests
- **`hmsutil tui`**: Launches full-screen BubbleTea interactive terminal dashboard.
- **`hmsutil test`**: Executes the full automated integration test suite against the connected dongle.
