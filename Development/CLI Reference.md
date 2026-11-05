---
title: CLI Reference (hsmutil)
tags:
  - cli
  - development
  - hsmutil
  - bubbletea
  - pkcs11
created: 2026-10-29
updated: 2026-11-05
references:
  - "[[Architecture/Decisions/ADR-0015 - HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS11 Equivalence|ADR-0015: HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS#11 Equivalence]]"
  - "[[Architecture/Decisions/ADR-0014 - Documentation Terminology Generalization to Generic HSM USB Key|ADR-0014: Documentation Terminology Generalization to Generic HSM USB Key]]"
  - "[[Plan/HSM CLI Restructuring, Static Compilation, Bubble Tea UX & PKCS11 Equivalence Implementation Plan|Implementation Plan: HSM CLI Restructuring, Static Compilation, Bubble Tea UX & PKCS#11 Equivalence]]"
---

# `hsmutil` CLI Reference

`hsmutil` is the cross-platform command-line interface for administering and managing the **Hardware Security Module (HSM) USB Key**. Built with Go using clean architecture, it compiles to a pure, hermetically static binary (`CGO_ENABLED=0`) with zero external shared library dependencies.

---

## Command Syntax & Binary Invocation

```bash
hsmutil [command] [subcommand] [flags]
```

### Ergonomic Typo Symlink
A symlink `bin/hsmutils -> bin/hsmutil` is automatically generated during compilation, allowing `hsmutils` to be invoked interchangeably.

### Global Flags

- `-p, --port <string>`: Explicitly specify serial port path (e.g. `/dev/tty.usbmodem...` on macOS, `/dev/ttyACM...` on Linux, `COM...` on Windows).
- `-s, --serial <string>`: Target a specific HSM device by 12-digit hex BLE address (e.g. `FD54E793A75540C9` or `FD54E793A755`) or 16-digit device UID.
- `--help, -h`: Help for `hsmutil`.

### Environment Variables

| Variable | Description |
| :--- | :--- |
| `HSM_PORT` (or `NRF52840_HSM_PORT`) | Explicit serial port path to use when `-p` is not passed. |
| `HSM_SERIAL` (or `NRF52840_HSM_SERIAL`) | BLE address or serial number to target when `-s` is not passed. |
| `HSM_PIN` | Default PIN for non-interactive / headless scripts. |

---

## Command Groups Overview

| Group | Command Pattern | Description |
| :--- | :--- | :--- |
| **Device Lifecycle** | `hsmutil device <subcommand>` | Physical device lifecycle: enumeration, hardware info, PIN initialization, session lock, zeroization, DFU firmware upgrade, and full device migration. |
| **Key Operations** | `hsmutil key <subcommand>` | Asymmetric key operations: keypair generation, enumeration, import, export, deletion, and single-slot migration. |
| **Session Control** | `hsmutil unlock` (alias `auth`) | Interactive masked PIN authentication (`bubbles/textinput`). |
| **Diagnostics & Utilities**| `hsmutil echo`, `hsmutil random`, `hsmutil p11` | Round-trip latency tests, TRNG byte sampling, and Cryptoki PKCS#11 diagnostics. |
| **Hidden Aliases** | `hsmutil <info\|reset\|lock\|init\|firmware>` | Legacy top-level commands retained for non-breaking backward compatibility. |

> [!NOTE]
> **Pruned Commands**: The deprecated monolithic full-screen dashboard (`hsmutil tui`) and embedded test runner (`hsmutil test`) have been removed in accordance with [[Architecture/Decisions/ADR-0015 - HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS11 Equivalence|ADR-0015]]. Testing is now managed through unit tests (`go test ./...`) and the workspace test harness (`./scripts/run_integration_tests.sh`).

---

## 1. Device Management (`hsmutil device`)

### `hsmutil device list` (Aliases: `hsmutil devices`, `hsmutil list`)
Enumerates all connected HSM USB Keys, displaying their detected hardware BLE addresses, dedicated AT Data port paths, Console logging port paths, and USB VID/PID descriptors.

```bash
$ hsmutil device list
Connected HSM USB Keys (2 detected):

[1] Serial / BLE:  FD54E793A75540C9
    AT Data Port:  /dev/tty.usbmodem31424103
    Console Port:  /dev/tty.usbmodem31424101
    USB ID:        1209:0001 (HSM USB Key)

[2] Serial / BLE:  3302311888499AC7
    AT Data Port:  /dev/tty.usbmodem31424403
    Console Port:  /dev/tty.usbmodem31424401
    USB ID:        1209:0001 (HSM USB Key)
```

**Flags**:
- `--json`: Output discovered device records in structured JSON format for automation and scripts.

---

### `hsmutil device info` (Legacy Alias: `hsmutil info`)
Queries standard device information (active firmware version, hardware revision, device UID, bootloader status, and supported algorithms):

```bash
$ hsmutil device info -s FD54E793A75540C9
==================================================
                 HSM Device Info                  
==================================================
 Port:           /dev/tty.usbmodem31424103
 Firmware:       1.0.0
 Hardware:       PCA10059
 Device UID:     EB53D74B8A9CAB28
 Bootloader:     B0+MCUboot
==================================================
```

---

### `hsmutil device init` (Legacy Alias: `hsmutil init`)
Initializes the master secret and sets the administrator PIN on an unconfigured device (`HSM_STATE_UNCONFIGURED`).

**Flags**:
- `--pin <string>`: Master administrator PIN (8–16 characters). If omitted in an interactive TTY, securely prompts with masked input.

---

### `hsmutil device lock` (Legacy Alias: `hsmutil lock`)
Closes the active authenticated session, revokes access tokens in RAM, transitions the device state machine to locked (`HSM_STATE_LOCKED`), and illuminates the solid **RED** status LED.

---

### `hsmutil device reset` (Legacy Alias: `hsmutil reset`)
Performs a complete cryptographic factory reset of the HSM key:
1. Zeroizes all 32 hardware key slots.
2. Wipes the Shamir Secret Sharing master secret from volatile RAM and non-volatile flash (NVS).
3. Restores factory defaults (PIN: `"12345678"`, retries: 3).
4. Transitions hardware to `HSM_STATE_UNCONFIGURED` (pulsating/breathing **BLUE** LED).

**Flags**:
- `-y, --yes`: Bypasses interactive confirmation safeguard.

**Interactive Safeguard**:
When executed without `-y`, prompts for explicit operator confirmation:
```text
WARNING: This will permanently delete all stored cryptographic keys,
zeroize the Shamir master secret, and return the HSM to an unconfigured state.
This action cannot be undone!

Are you sure you want to proceed? (yes/no): 
```

---

### `hsmutil device status` (Legacy Alias: `hsmutil status`)
Queries device authentication state, remaining PIN retry attempts, and stored key count (`AT+STATUS?`):

```bash
$ hsmutil device status
==================================================
                    HSM Status                    
==================================================
 Authentication: Unlocked (Authenticated)
 PIN Retries:    3 / 3
 Keys Stored:    2 / 16
==================================================
```

---

### `hsmutil device firmware <subcommand>` (Legacy Alias: `hsmutil firmware`)

#### `hsmutil device firmware info`
Queries the device via `AT+DFU_INFO?` for:
- Active firmware semantic version (`VER`).
- Active MCUboot partition slot (`SLOT=0` Primary vs `SLOT=1` Secondary candidate).
- Application binary image size (`SIZE`).
- 32-byte PSA SHA-256 binary digest (`SHA256`).

#### `hsmutil device firmware update --file <path>`
Streams a signed MCUboot application binary (`zephyr.signed.bin`) over the ASCII AT command channel in 256-byte chunks (`AT+DFU_CHUNK=<offset>,<512-hex-chars>`), triggers a deferred test swap via `AT+DFU_APPLY`, and reboots into the updated image.

**Bubble Tea & Terminal UX**:
- **Interactive TTY**: Renders an animated gradient progress bar using `bubbles/progress`:
  ```text
  Updating Firmware [zephyr.signed.bin] (Slot 1 Candidate)
  [████████████████████░░░░░░░░░░░░░░░░░░░░] 52% (67/128 chunks) [ETA: 00:14]
  ```
- **Headless Fallback (`!term.IsTerminal`)**: Automatically switches to discrete stdout line logging for CI/CD runners and non-interactive scripts:
  ```text
  [INFO] DFU Transfer: Chunk 67/128 (52%) acknowledged.
  ```

---

### `hsmutil device migrate` (Whole-Device Replacement)
Performs an atomic, whole-device cryptographic migration from a source HSM to a target replacement HSM:
1. Resolves and authenticates both source (`--source`) and target (`--target`) devices.
2. Enumerates all occupied key slots (0..31) on the source device.
3. Wraps and transfers each key slot under the target device's ephemeral ECDH key.
4. Verifies target slot public key digests.
5. **Fail-Fast Lockdown**: If any slot fails verification, partial target slots are wiped, the source device is immediately locked, and execution aborts.
6. **Decommissioning Zeroization**: Upon 100% successful verification of all migrated keys, the source device is automatically zeroized (`AT+FACTORYRESET`) to prevent dangerous key duplication in production environments.

---

## 2. Key Management (`hsmutil key`)

### `hsmutil key list`
Lists all occupied key slots, displaying algorithm type, bit length, and user labels (`AT+SLOTS?`).

### `hsmutil key generate --slot <1-16> [--algo EC_P256]`
Generates a hardware-backed SECP256R1 (`prime256v1`) ECDSA keypair in the specified storage slot using the CryptoCell CC310 TRNG (`AT+KEYGEN`).

**Flags**:
- `--slot <int>`: Target key slot identifier (1..16, required).
- `--algo <string>`: Key algorithm (default: `EC_P256`).

### `hsmutil key pubkey --slot <1-16>`
Displays the uncompressed 65-byte public key point in hex format from the specified slot (`AT+PUBKEY`).

**Flags**:
- `--slot <int>`: Source key slot identifier (1..16, required).

### `hsmutil key sign --slot <1-16> --hash <hex>`
Computes a hardware ECDSA signature over a 32-byte SHA-256 hash digest using the on-chip accelerator (`AT+SIGN`).

**Flags**:
- `--slot <int>`: Key slot identifier for signing (1..16, required).
- `--hash <string>`: SHA-256 digest in hex (64 hex characters / 32 bytes, required).

### `hsmutil key migrate --slot <1-16> --target <serial> [--source <serial>]`
Migrates a **single** key slot to another HSM device:
- The single slot is wrapped under the target device's ephemeral public key and imported into the target slot.
- Source slot is preserved on the source device.
- Does not zeroize the source device.

---

## 3. Session Authentication (`hsmutil unlock`)

### `hsmutil unlock` (Alias: `hsmutil auth`)
Authenticates the operator session, unlocks the HSM hardware state machine (`HSM_STATE_AUTHENTICATED`), and illuminates the solid **GREEN** status LED.

**Bubble Tea & Terminal UX**:
- **Interactive TTY**: If `--pin` is not passed, prompts with a masked `bubbles/textinput` widget where characters are masked as `••••••••`:
  ```text
  Enter HSM PIN: ••••••••
  ```
- **Headless Fallback**: In automated environments, accepts `--pin <string>`, reads a single line from piped `os.Stdin`, or checks `HSM_PIN`.

---

## 4. Diagnostics & Utilities

- **`hsmutil echo [message]`**: Sends a round-trip echo packet to verify serial communication integrity and latency (`AT`).
- **`hsmutil random [--len <bytes>]`**: Retrieves cryptographically secure random bytes from the on-chip True Random Number Generator (TRNG) (`AT+RANDOM=<len>`).
- **`hsmutil pkcs11`** (Alias: `p11`): Diagnostic and testing suite for `hsm-libpkcs11`:
  - `hsmutil pkcs11 info`: Display Cryptoki provider library version and manufacturer info.
  - `hsmutil pkcs11 slots`: Enumerate PKCS#11 slots, token status, and supported mechanisms.
  - `hsmutil pkcs11 init-token --slot <id> [--label <label>] [--pin <pin>]`: Initialize token via PKCS#11.
  - `hsmutil pkcs11 keygen --slot <id> --key-id <id>`: Generate EC keypair via PKCS#11 interface.
  - `hsmutil pkcs11 sign --slot <id> --key-id <id> --data <string>`: Test ECDSA signature via PKCS#11 interface.
  - `hsmutil pkcs11 test`: Run automated end-to-end Cryptoki test suite.

---

## 5. PKCS#11 Equivalence Matrix (`pkcs11-tool`)

Standard OpenSC `pkcs11-tool` invocations map to `hsmutil` commands as follows:

| Capability | `hsmutil` Command | `pkcs11-tool` Command (`-m libnrf52840_pkcs11.dylib`) | Parity |
| :--- | :--- | :--- | :--- |
| Enumerate Devices | `hsmutil device list` | `pkcs11-tool -L` | Equivalent |
| Display Metadata | `hsmutil device info` | `pkcs11-tool -M` / `-T` | Equivalent |
| Initialize Token | `hsmutil device init` | `pkcs11-tool --init-token --init-pin` | Equivalent |
| Authenticate / Login | `hsmutil unlock` | `pkcs11-tool --login --pin <PIN>` | Equivalent |
| Logout / Lock | `hsmutil device lock` | `pkcs11-tool --logout` | Equivalent |
| Factory Reset Zeroization | `hsmutil device reset` | *None (Vendor Out-of-Band)* | `hsmutil` Only |
| Firmware Upgrade (DFU) | `hsmutil device firmware update` | *None (Host Management)* | `hsmutil` Only |
| Generate Keypair | `hsmutil key generate --slot <N>` | `pkcs11-tool --keypairgen --key-type EC:secp256r1 --id <N>` | Equivalent |
| List Key Objects | `hsmutil key list` | `pkcs11-tool -O` | Equivalent |
| Digital Sign (ECDSA-SHA256) | `hsmutil key sign` | `pkcs11-tool --sign --id <N> -m ECDSA-SHA256` | Equivalent |
| Delete Key Object | `hsmutil key delete --slot <N>` | `pkcs11-tool --delete-object --type privkey --id <N>` | Equivalent |
| Single Key Migration | `hsmutil key migrate --slot <N>` | `pkcs11-tool --wrap` / `--unwrap` | Partial Gap |
| Full Device Migration | `hsmutil device migrate` | *None (Platform Specialty)* | `hsmutil` Only |
| TRNG Random Sampling | `hsmutil random [N]` | `pkcs11-tool --generate-random [N]` | Equivalent |

---

## Cross-References

- [[Architecture/Decisions/ADR-0015 - HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS11 Equivalence|ADR-0015: HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS#11 Equivalence]]
- [[Plan/HSM CLI Restructuring, Static Compilation, Bubble Tea UX & PKCS11 Equivalence Implementation Plan|Implementation Plan: HSM CLI Restructuring, Static Compilation, Bubble Tea UX & PKCS#11 Equivalence]]
- [[Architecture/Decisions/ADR-0014 - Documentation Terminology Generalization to Generic HSM USB Key|ADR-0014: Documentation Terminology Generalization to Generic HSM USB Key]]
- [[Architecture/Decisions/ADR-0003 - Host Port Discovery and Serial Targeting Strategy|ADR-0003: Host Port Discovery and Serial Targeting Strategy]]
- [[Architecture/Decisions/ADR-0004 - Factory Reset Semantics and Safeguards|ADR-0004: Factory Reset Semantics and Safeguards]]
- [[Architecture/Decisions/ADR-0006 - AT-Protocol DFU Chunk Streaming and Deferred MCUboot Swap|ADR-0006: AT-Protocol DFU Chunk Streaming and Deferred MCUboot Swap]]
- [[00 - Index|Vault Index]]
