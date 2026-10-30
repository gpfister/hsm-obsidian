---
title: CLI Reference (hsmutil)
tags:
  - cli
  - development
  - hsmutil
created: 2026-10-29
---

# `hsmutil` CLI Reference

`hsmutil` is the cross-platform command-line tool and interactive terminal dashboard for communicating with the nRF52840 HSM Dongle.

---

## Command Syntax

```bash
hsmutil [command] [flags]
```

### Global Flags
- `-p, --port <string>`: Specify serial port path (e.g. `/dev/cu.usbmodem...` on macOS, `/dev/ttyACM...` on Linux, `COM...` on Windows). If omitted, `hsmutil` auto-detects the dongle.

---

## Commands

### 1. Firmware Management
- **`hsmutil firmware info`**: Queries the dongle for its active firmware version, MCUboot partition slot, image size, and SHA-256 digest.
- **`hsmutil firmware update --file <path>`**: Streams a signed MCUboot application binary (`zephyr.signed.bin`) in chunked packets to the secondary candidate slot, validates the payload, marks it for upgrade, and requests a reboot into MCUboot.

### 2. Device Status & Diagnostics
- **`hsmutil info`**: Queries standard YubiHSM 2 device information (serial number, version, supported crypto algorithms).
- **`hsmutil echo [message]`**: Sends an echo packet and calculates round-trip latency.

### 3. Session & Security State
- **`hsmutil unlock`** (alias **`auth`**): Authenticates session, transitions the dongle to unlocked state, and turns the RGB LED solid **GREEN**.
- **`hsmutil lock`**: Closes session, transitions the dongle to locked state, and turns the RGB LED solid **RED**.
- **`hsmutil reset`**: Resets device state back to unconfigured, transitioning the RGB LED to pulsating **BLUE**.

### 4. Interactive TUI & Tests
- **`hsmutil tui`**: Launches full-screen BubbleTea interactive terminal dashboard.
- **`hsmutil test`**: Executes the full automated integration test suite against the connected dongle.
