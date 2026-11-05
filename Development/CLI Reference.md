---
title: "CLI Reference (gphsm / hsmutil)"
tags:
  - cli
  - development
  - gphsm
  - hsmutil
  - bubbletea
  - pkcs11
  - dual-mode-auth
  - pwquality
created: 2026-10-29
updated: 2026-11-05
references:
  - "[[Architecture/Decisions/ADR-0017 - Host CLI Rebranding to gphsm and PKCS11 Refactoring to libgphsm|ADR-0017: Host CLI Rebranding to gphsm and PKCS#11 Refactoring to libgphsm]]"
  - "[[Architecture/Decisions/ADR-0016 - Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)|ADR-0016: Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)]]"
  - "[[Architecture/Decisions/ADR-0015 - HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS11 Equivalence|ADR-0015: HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS#11 Equivalence]]"
  - "[[Architecture/Decisions/ADR-0014 - Documentation Terminology Generalization to Generic HSM USB Key|ADR-0014: Documentation Terminology Generalization to Generic HSM USB Key]]"
  - "[[Plan/HSM Dual-Mode Authentication, CLI Renaming (gphsm), and PKCS11 Refactoring Implementation Plan|Implementation Plan: Dual-Mode Auth, CLI Renaming & PKCS#11 Refactoring]]"
---

# `gphsm` (hsmutil) CLI Reference

`gphsm` is the canonical cross-platform command-line administration utility for the **Hardware Security Module (HSM) USB Key** ecosystem. Built in Go using Clean Architecture, it compiles to a 100% hermetic static binary (`CGO_ENABLED=0`) with zero external shared library dependencies.

---

## Command Syntax & Binary Invocation

```bash
gphsm [command] [subcommand] [flags]
```

### Backward-Compatible Symlinks
Building `gphsm` automatically establishes relative symlinks to guarantee 100% backward compatibility:
```bash
bin/gphsms  -> gphsm
bin/hsmutil -> gphsm
bin/hsmutils -> gphsm
```
Operators, existing scripts, and CI/CD pipelines can invoke `hsmutil` or `hsmutils` interchangeably.

### Global Flags

- `-p, --port <string>`: Explicitly specify serial port path (e.g. `/dev/tty.usbmodem...3` on macOS, `/dev/ttyACM1` on Linux, `COM...` on Windows).
- `-s, --serial <string>`: Target a specific HSM device by 12-digit hex BLE address (e.g. `FD54E793A75540C9` or `FD54E793A755`) or 16-digit device UID.
- `--help, -h`: Help for `gphsm`.

### Environment Variables

| Variable | Description |
| :--- | :--- |
| `GPHSM_PORT` (fallbacks: `HSM_PORT`, `NRF52840_HSM_PORT`) | Explicit serial port path to use when `-p` is not passed. |
| `GPHSM_SERIAL` (fallbacks: `HSM_SERIAL`, `NRF52840_HSM_SERIAL`) | BLE address or serial number to target when `-s` is not passed. |
| `GPHSM_PIN` (fallback: `HSM_PIN`) | Default PIN for non-interactive / headless scripts in PIN mode. |

---

## Command Groups Overview

| Group | Command Pattern | Description |
| :--- | :--- | :--- |
| **Device Lifecycle** | `gphsm device <subcommand>` | Physical device lifecycle: enumeration, hardware info, dual-mode initialization (PIN / Shamir), session lock, zeroization, DFU firmware upgrade, and full device migration. |
| **Key Operations** | `gphsm key <subcommand>` | Asymmetric key operations: keypair generation, enumeration, public key export, digital signing, and single-slot migration. |
| **Session Control** | `gphsm unlock` (alias `auth`) | Auto-detecting session authentication: masked PIN prompt (`bubbles/textinput`) or 3-of-5 Shamir custodian ceremony. |
| **Diagnostics & Utilities**| `gphsm echo`, `gphsm random` | Round-trip serial latency tests and TRNG byte sampling from CC310 hardware. |
| **Hidden Aliases** | `gphsm <info\|reset\|lock\|init\|firmware\|auth>` | Legacy root-level commands retained for non-breaking backward compatibility. |

> [!NOTE]
> **Pruned Commands**: In accordance with [[Architecture/Decisions/ADR-0017 - Host CLI Rebranding to gphsm and PKCS11 Refactoring to libgphsm|ADR-0017]], the internal `p11` command group has been purged. PKCS#11 operations are cleanly delegated to the dedicated [`libgphsm`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/libgphsm) provider library and standard OpenSC tooling (`pkcs11-tool`).

---

## 1. Device Management (`gphsm device`)

### `gphsm device list` (Aliases: `gphsm devices`, `gphsm ls`)
Enumerates all connected HSM USB Keys, displaying their detected hardware BLE addresses, dedicated AT Data port paths, Console logging port paths, and USB VID/PID descriptors.

```bash
$ gphsm device list
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

### `gphsm device info` (Legacy Alias: `gphsm info`)
Queries standard device information (active firmware version, hardware revision, device UID, bootloader status, and supported algorithms):

```bash
$ gphsm device info -s FD54E793A75540C9
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

### `gphsm device init` (Legacy Alias: `gphsm init`)
Initializes authentication mode on an unconfigured device (`CONFIGURED=0`):

1. **Interactive Mode (TTY)**:
   Launches a Bubble Tea wizard prompting the operator to select authentication paradigm:
   ```text
   Select Initialization Mode:
     [1] PIN Mode (Single-User Password Protection)
     [2] Shamir Secret Sharing Mode (3-of-5 Custodian Threshold)
   ```
2. **PIN Mode (`--mode pin`)**:
   Enforces Yocto-compliant `pam_pwquality` password complexity:
   - Minimum 10 characters, maximum 64 characters.
   - At least 3 of 4 character classes (upper, lower, digit, special).
   - Maximum 1 repeating consecutive identical character.
   - Maximum 2 sequential monotonic characters.
   - Flags: `--pin <string>` (or env `GPHSM_PIN`).
3. **Shamir Mode (`--mode shamir`)**:
   Executes a 3-of-5 threshold ceremony, emitting 5 custodian shares and requiring confirmation before zeroizing RAM buffers.

---

### `gphsm device status` (Legacy Alias: `gphsm status`)
Queries device authentication state, initialization status, configured method, remaining PIN retries, and stored key count (`AT+STATUS?`):

```bash
$ gphsm device status
==================================================
                    HSM Status                    
==================================================
 Authentication: Locked (Authentication Required)
 Configured:     Yes
 Method:         PIN
 PIN Retries:    3 remaining
 Stored Keys:    2
==================================================
```

---

### `gphsm device lock` (Legacy Alias: `gphsm lock`)
Closes the active authenticated session, revokes access tokens in RAM, transitions the device state machine to locked, and illuminates the solid **RED** status LED.

---

### `gphsm device reset` (Legacy Alias: `gphsm reset`)
Performs a complete cryptographic factory reset of the HSM key:
1. Zeroizes all 32 hardware key slots.
2. Zeroizes PIN salt/hash and Shamir master secret in RAM and NVS.
3. Restores default retries (3) and unconfigured state (`METHOD=NONE`).
4. Transitions hardware to unconfigured state (pulsating/breathing **BLUE** LED).

**Flags**:
- `-y, --yes`: Bypasses interactive confirmation safeguard.

**Interactive Safeguard**:
When executed without `-y`, prompts for explicit operator confirmation:
```text
WARNING: This operation will perform a destructive factory reset on the HSM.
All persistent private keys, cryptographic material, and secrets will be permanently destroyed.
The device will transition to an unconfigured state (pulsating blue LED).

Are you sure you want to proceed? (yes/no): 
```

---

### `gphsm device firmware <subcommand>` (Legacy Alias: `gphsm firmware`)

#### `gphsm device firmware info`
Queries active firmware semantic version, active MCUboot partition slot (`Slot 0` Primary vs `Slot 1` Secondary candidate), application binary image size, and SHA-256 hash.

#### `gphsm device firmware update --file <path>`
Streams a signed MCUboot application binary (`zephyr.signed.bin`) over the AT command channel in 256-byte chunks (`AT+DFU_CHUNK`), verifies slot image SHA-256, and schedules a deferred swap reboot via `AT+DFU_APPLY`.

**Bubble Tea & Terminal UX**:
- **Interactive TTY**: Renders an animated gradient progress bar using `bubbles/progress`.
- **Headless Fallback**: Switches to discrete stdout percentage logs.

---

### `gphsm device migrate` (Whole-Device Replacement)
Performs an atomic, whole-device cryptographic migration from a source HSM to a target replacement HSM:
1. **Homogeneous Method Check**: Validates that both source and target devices are initialized with identical authentication methods (`source.METHOD == target.METHOD`). Fails fast if methods mismatch.
2. Enumerates all occupied key slots on the source device.
3. Wraps and transfers each key slot under the target device's ephemeral ECDH public key.
4. Verifies target slot public key digests.
5. **Fail-Fast Lockdown**: If any slot fails verification, partial target slots are wiped, both source and target devices are immediately locked, and execution halts.
6. **Optional Source Zeroization**: With `--zeroize-source`, automatically resets the source device upon 100% transfer confirmation.

---

## 2. Key Management (`gphsm key`)

### `gphsm key list`
Lists all occupied key slots, displaying slot ID, algorithm type, bit size, and user label.

### `gphsm key generate --slot <1-16> [--algo EC_P256]`
Generates a hardware-backed SECP256R1 (`prime256v1`) ECDSA keypair in the specified storage slot using the CryptoCell CC310 TRNG (`AT+KEYGEN`). Device automatically self-locks upon generation.

### `gphsm key pubkey --slot <1-16>`
Displays the uncompressed 65-byte public key point in hex format from the specified slot (`AT+PUBKEY`).

### `gphsm key sign --slot <1-16> --hash <hex>`
Computes a hardware ECDSA signature over a 32-byte SHA-256 hash digest using the on-chip accelerator (`AT+SIGN`).

### `gphsm key migrate --slot <1-16> --target <serial> [--source <serial>]`
Migrates a **single** key slot to another HSM device:
- Strictly verifies that source and target devices share identical authentication methods.
- Single slot is wrapped under target's ephemeral ECDH public key.
- Source slot is preserved on the source device without zeroization.

---

## 3. Session Authentication (`gphsm unlock`)

### `gphsm unlock` (Alias: `gphsm auth`)
Authenticates the operator session, unlocks the HSM hardware state machine, and illuminates the solid **GREEN** status LED.

**Auto-Detection Logic**:
1. Queries device via `AT+STATUS?`.
2. If `CONFIGURED=0`: prompts operator to run `gphsm device init`.
3. If `AUTH=1`: reports device is already unlocked.
4. If `METHOD=PIN`:
   - Interactive TTY: prompts with a masked `bubbles/textinput` widget (`••••••••`).
   - Headless: reads `--pin`, `stdin`, or `GPHSM_PIN`.
5. If `METHOD=SHAMIR`:
   - Initiates turn-by-turn custodian threshold unlock ceremony, prompting for quorum shares.

---

## 4. Diagnostics & Utilities

- **`gphsm echo [message]`**: Sends a round-trip echo packet to verify serial communication integrity and latency (`AT`).
- **`gphsm random [--len <bytes>]`**: Retrieves cryptographically secure random bytes from the on-chip True Random Number Generator (TRNG) (`AT+RANDOM=<len>`).

---

## 5. PKCS#11 Equivalence Matrix (`libgphsm` / `pkcs11-tool`)

Standard OpenSC `pkcs11-tool` invocations map to `gphsm` and `libgphsm` commands as follows:

| Capability | `gphsm` Command | `pkcs11-tool` Command (`-m libgphsm.dylib`) | Parity |
| :--- | :--- | :--- | :--- |
| Enumerate Devices | `gphsm device list` | `pkcs11-tool -L` | Equivalent |
| Display Metadata | `gphsm device info` | `pkcs11-tool -M` / `-T` | Equivalent |
| Initialize Token | `gphsm device init` | `pkcs11-tool --init-token --init-pin` | Equivalent (PIN mode) |
| Authenticate / Login | `gphsm unlock` | `pkcs11-tool --login --pin <PIN>` | Equivalent (PIN mode) / Out-of-band (Shamir) |
| Logout / Lock | `gphsm device lock` | `pkcs11-tool --logout` | Equivalent |
| Factory Reset Zeroization | `gphsm device reset` | *None (Vendor Out-of-Band)* | `gphsm` Only |
| Firmware Upgrade (DFU) | `gphsm device firmware update` | *None (Host Management)* | `gphsm` Only |
| Generate Keypair | `gphsm key generate --slot <N>` | `pkcs11-tool --keypairgen --key-type EC:secp256r1 --id <N>` | Equivalent |
| List Key Objects | `gphsm key list` | `pkcs11-tool -O` | Equivalent |
| Digital Sign (ECDSA-SHA256) | `gphsm key sign` | `pkcs11-tool --sign --id <N> -m ECDSA-SHA256` | Equivalent |
| Single Key Migration | `gphsm key migrate --slot <N>` | *None (Vendor Out-of-Band)* | `gphsm` Only |
| Full Device Migration | `gphsm device migrate` | *None (Platform Specialty)* | `gphsm` Only |
| TRNG Random Sampling | `gphsm random [N]` | `pkcs11-tool --generate-random [N]` | Equivalent |

---

## Cross-References

- [[Architecture/Decisions/ADR-0017 - Host CLI Rebranding to gphsm and PKCS11 Refactoring to libgphsm|ADR-0017: Host CLI Rebranding to gphsm and PKCS#11 Refactoring to libgphsm]]
- [[Architecture/Decisions/ADR-0016 - Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)|ADR-0016: Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)]]
- [[Architecture/Decisions/ADR-0015 - HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS11 Equivalence|ADR-0015: HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS#11 Equivalence]]
- [[Architecture/Decisions/ADR-0014 - Documentation Terminology Generalization to Generic HSM USB Key|ADR-0014: Documentation Terminology Generalization to Generic HSM USB Key]]
- [[Architecture/Decisions/ADR-0003 - Host Port Discovery and Serial Targeting Strategy|ADR-0003: Host Port Discovery and Serial Targeting Strategy]]
- [[Architecture/Decisions/ADR-0004 - Factory Reset Semantics and Safeguards|ADR-0004: Factory Reset Semantics and Safeguards]]
- [[Architecture/Decisions/ADR-0006 - AT-Protocol DFU Chunk Streaming and Deferred MCUboot Swap|ADR-0006: AT-Protocol DFU Chunk Streaming and Deferred MCUboot Swap]]
- [[00 - Index|Vault Index]]
