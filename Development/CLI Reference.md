---
title: CLI Reference (hsmutil)
tags:
  - cli
  - development
  - hsmutil
created: 2026-10-29
updated: 2026-10-31
---

# `hsmutil` CLI Reference

`hsmutil` is the cross-platform command-line tool and interactive terminal dashboard for communicating with the nRF52840 HSM Dongle.

---

## Command Syntax

```bash
hsmutil [command] [flags]
```

### Global Flags

- `-p, --port <string>`: Explicitly specify serial port path (e.g. `/dev/tty.usbmodem...` on macOS, `/dev/ttyACM...` on Linux, `COM...` on Windows).
- `-s, --serial <string>`: Target a specific HSM dongle by 12-digit hex BLE address (e.g. `FD54E793A75540C9` or `FD54E793A755`) or 16-digit device UID.
- `--help, -h`: Help for `hsmutil`.

### Environment Variables

| Variable | Description |
| :--- | :--- |
| `NRF52840_HSM_PORT` | Explicit serial port path to use when `-p` is not passed. |
| `NRF52840_HSM_SERIAL` | BLE address or serial number to target when `-s` is not passed. |

---

## Commands

### 1. Device Discovery & Enumeration

#### `hsmutil devices` (alias: `hsmutil list`)
Enumerates all connected nRF52840 HSM Dongles, displaying their detected BLE addresses, dedicated AT Data port paths, Console logging port paths, and USB VID/PID descriptors.

```bash
$ hsmutil devices
Connected nRF52840 HSM Dongles (2 detected):

[1] Serial / BLE:  FD54E793A75540C9
    AT Data Port:  /dev/tty.usbmodem31424103
    Console Port:  /dev/tty.usbmodem31424101
    USB ID:        1209:0001 (nRF52840 HSM Dongle)

[2] Serial / BLE:  3302311888499AC7
    AT Data Port:  /dev/tty.usbmodem31424403
    Console Port:  /dev/tty.usbmodem31424401
    USB ID:        1209:0001 (nRF52840 HSM Dongle)
```

**Flags**:
- `--json`: Output discovered device records in structured JSON format for scripting and orchestration.

```bash
$ hsmutil devices --json
[
  {
    "serial": "FD54E793A75540C9",
    "data_port": "/dev/tty.usbmodem31424103",
    "console_port": "/dev/tty.usbmodem31424101",
    "vid": "1209",
    "pid": "0001",
    "product": "nRF52840 HSM Dongle"
  }
]
```

---

### 2. Device Status & Diagnostics

#### `hsmutil info`
Queries standard device information (active firmware version, hardware revision, device UID, bootloader status, and supported algorithms). Supports serial targeting with `-s`:

```bash
$ hsmutil info -s FD54E793A75540C9
==================================================
              nRF52840 HSM Device Info            
==================================================
 Port:           /dev/tty.usbmodem31424103
 Firmware:       1.0.0
 Hardware:       PCA10059
 Device UID:     EB53D74B8A9CAB28
 Bootloader:     B0+MCUboot
==================================================
```

#### `hsmutil echo [message]`
Sends an echo packet to verify round-trip latency and communication integrity.

---

### 3. Firmware Management (AT-Protocol DFU)

#### `hsmutil firmware info`
Queries the dongle via `AT+DFU_INFO?` for active firmware metadata (`VER`), active MCUboot partition slot (`SLOT=0` Primary vs `SLOT=1` Secondary candidate), image size (`SIZE`), and 32-byte PSA SHA-256 digest (`SHA256`). Automatically falls back to `AT+INFO?` if connected to legacy firmware that returns `+CME ERROR: 8`.

#### `hsmutil firmware update --file <path>`
Streams a signed MCUboot application binary (`zephyr.signed.bin`) over the ASCII AT command channel (`cdc_acm_uart1`) in 256-byte chunks (`AT+DFU_CHUNK=<offset>,<512-hex-chars>`, `10s` timeout per chunk to accommodate `FLASH_AREA_ID(image_1)` erase at `offset=0`), triggers a deferred MCUboot test swap via `AT+DFU_APPLY` (`boot_request_upgrade(BOOT_UPGRADE_TEST)` + `200 ms` `k_work_delayable` reboot), and confirms the swapped slot permanently via `boot_write_img_confirmed()` on next startup. Requires an authenticated session (`hsmutil auth`).

---

### 4. Session & Security State

- **`hsmutil unlock`** (alias **`auth`**): Authenticates session, transitions dongle to unlocked state, and sets RGB LED solid **GREEN**.
- **`hsmutil lock`**: Closes session, transitions dongle to locked state, and sets RGB LED solid **RED**.
- **`hsmutil reset`**: Performs complete cryptographic factory reset of the HSM dongle, zeroizing all 32 CC310 hardware key slots, wiping the Shamir Secret Sharing master secret from RAM and NVS, restoring factory defaults (PIN: `"12345678"`, retries: 3), and transitioning hardware to `HSM_STATE_UNCONFIGURED` (pulsating/breathing **BLUE** LED).
  - **Flags**:
    - `-y, --yes`: Bypasses interactive confirmation prompt.
  - **Interactive Safeguard**:
    - If `--yes` is not passed, prompts:
      ```text
      WARNING: This will permanently delete all stored cryptographic keys,
      zeroize the Shamir master secret, and return the HSM to an unconfigured state.
      This action cannot be undone!

      Are you sure you want to proceed? (yes/no): 
      ```
    - Typing anything other than `"y"` or `"yes"` aborts with `Reset cancelled.` (exit code 0) without connecting to the serial device or sending `AT+RESET`.
  - **Multi-Device Behavior**:
    - If multiple dongles are plugged in without `-p` or `-s`, fails with `ErrMultipleDevicesDetected`, printing all connected dongles and prompting to specify `-s <BLE>` or `-p <PORT>`.

---

### 5. Interactive TUI & Tests

- **`hsmutil tui`**: Launches full-screen BubbleTea interactive terminal dashboard.
- **`hsmutil test`**: Executes the automated integration test suite against the target dongle.

---

## Cross-References

- [[features/hsmutil-reset-and-multi-device-disambiguation|Feature: Reset Confirmation & Multi-Device Disambiguation]]
- [[Specification/Reset Confirmation and Multi-Device Disambiguation|Specification: Reset Confirmation and Multi-Device Disambiguation]]
- [[features/usb-descriptors-ble-address|USB Descriptors & Device Identification Feature Spec]]
- [[Architecture/USB Composite Stack|USB Composite Stack Architecture]]
- [[Architecture/Decisions/ADR-0003 - Host Port Discovery and Serial Targeting Strategy|ADR-0003: Host Port Discovery and Serial Targeting Strategy]]
- [[Architecture/Decisions/ADR-0004 - Factory Reset Semantics and Safeguards|ADR-0004: Factory Reset Semantics and Safe Erasure Safeguards]]
