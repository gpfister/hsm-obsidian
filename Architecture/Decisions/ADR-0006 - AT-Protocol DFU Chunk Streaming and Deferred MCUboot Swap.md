---
title: "ADR-0006: AT-Protocol DFU Chunk Streaming & Deferred MCUboot Swap Architecture"
status: Accepted
date: 2026-11-05
tags:
  - adr
  - architecture
  - dfu
  - mcuboot
  - at-protocol
  - firmware
  - cli
deciders:
  - Embedded Systems Architect
  - Security Firmware Lead
references:
  - "[[features/at-protocol-dfu-support|Feature: AT-Protocol DFU Support]]"
  - "[[Specification/AT-Protocol DFU Support|Specification: AT-Protocol DFU Support]]"
  - "[[Plan/AT-Protocol DFU Support Implementation Plan|Implementation Plan: AT-Protocol DFU Support]]"
  - "[[Architecture/Dual-Stage Secure Boot|Dual-Stage Secure Boot]]"
  - "[[Architecture/USB Composite Stack|USB Composite Stack]]"
---

# ADR-0006: AT-Protocol DFU Chunk Streaming & Deferred MCUboot Swap Architecture

## Context & Problem Statement

During the migration of the nRF52840 Dongle HSM (`PCA10059`) from legacy binary YubiHSM-style framing (`s.transport`) to the ASCII AT command protocol (`s.atClient`) over the dedicated USB CDC-ACM cryptographic channel (`cdc_acm_uart1`), the Device Firmware Upgrade (DFU) subsystem remained partially unmigrated:

1. **CLI Layer (`nrf52840-hsm-cli`)**:
   - [`HSMService.GetFirmwareInfo`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli/internal/usecases/hsm_service.go#L93-L131) returned a hardcoded stub when `s.transport == nil`.
   - [`HSMService.UpdateFirmware`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli/internal/usecases/hsm_service.go#L178-L230) immediately failed with `DFU update requires legacy binary serial connection` when connected over `s.atClient`.
2. **Firmware Layer (`nrf52840-hsm-firmware`)**:
   - [`at_commands_execute`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware/firmware/lib/src/at_commands.c#L94-L687) had no handlers for `AT+DFU_INFO?`, `AT+DFU_CHUNK=<offset>,<hex_data>`, or `AT+DFU_APPLY`, and the pure C HSM library (`firmware/lib`) was decoupled from the Zephyr MCUboot flash service ([`dfu_service.c`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware/firmware/app/src/dfu_service.c)).
   - [`AT_PARSER_MAX_LINE_LEN`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware/firmware/lib/include/hsm/at_parser.h#L12) was `512` bytes, which could not hold a 256-byte binary chunk (`512` hex characters) plus the `AT+DFU_CHUNK=<offset>,` prefix and `\r\n` terminator without triggering `CME_ERR_BUFFER_OVERFLOW` (`+CME ERROR: 7`).
   - Synchronous execution of `sys_reboot(SYS_REBOOT_COLD)` inside `AT+DFU_APPLY` before the USB CDC-ACM TX ring buffer flushed `OK\r\n` caused the host CLI to experience an abrupt USB disconnect / EOF error instead of a clean response.
   - [`dfu_service_init`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware/firmware/app/src/dfu_service.c#L10-L13) did not call `boot_write_img_confirmed()`, causing MCUboot to revert a `BOOT_UPGRADE_TEST` image swap on the subsequent reset.

We need an architecture that enables authenticated, deterministic DFU streaming over the ASCII AT command channel while preserving the clean separation between `firmware/lib` (platform-agnostic HSM core) and `firmware/app` (Zephyr RTOS, Flash map, and MCUboot APIs).

---

## Decision Drivers

- **Clean Library Decoupling**: `firmware/lib` (`at_commands.c`) must remain buildable and testable in host unit tests without depending directly on Zephyr headers (`<zephyr/dfu/mcuboot.h>`, `<zephyr/storage/flash_map.h>`).
- **Reliable USB CDC-ACM Response Delivery**: The host CLI (`hsmutil firmware update`) must receive `OK\r\n` for `AT+DFU_APPLY` before the nRF52840 executes a cold reboot into MCUboot.
- **Permanent Slot Confirmation after Self-Test**: Newly booted candidate images (`BOOT_UPGRADE_TEST`) must confirm themselves once `dfu_service_init()` runs, preventing unintended rollback on subsequent power cycles while retaining MCUboot fallback protection if a bad image crashes prior to initialization.
- **Flash Erase Latency Resilience**: Erasing the secondary MCUboot slot (`FLASH_AREA_ID(image_1)`, ~256 KB on nRF52840 internal flash) at `offset == 0` takes 1–4 seconds; the AT protocol client must accommodate this without premature serial timeout.
- **Cryptographic Access Control**: Staging chunks and triggering slot swaps must be restricted to authenticated HSM sessions (`keystore_is_authenticated()`), complementing MCUboot's hardware-backed ECDSA P-256 signature verification (`keys/root-ec-p256.pem`).

---

## Considered Options

### Option 1: Re-enable Legacy Binary Framing Alongside AT Protocol on `cdc_acm_uart1`
- *Mechanism*: Multiplex binary YubiHSM-style packets and ASCII `AT+` lines on the same UART channel, or switch the UART into binary mode during DFU.
- *Pros*: Reuses legacy binary DFU framing code.
- *Cons*: Breaks the pure ASCII line-oriented state machine in [`at_parser.c`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware/firmware/lib/src/at_parser.c), complicates [`mock_hsm_pty.py`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts/mock_hsm_pty.py) and kernel driver filtering (`nrf52840-hsm-kmod`), and introduces framing ambiguity when binary payload bytes collide with `\r` (`0x0D`) or `\n` (`0x0A`).

### Option 2: Direct Zephyr MCUboot Calls Inside `at_commands.c` with Immediate Reboot
- *Mechanism*: Include Zephyr `mcuboot.h` and `flash_map.h` directly in `firmware/lib/src/at_commands.c` and call `sys_reboot(SYS_REBOOT_COLD)` immediately on `AT+DFU_APPLY`.
- *Pros*: Minimal indirection.
- *Cons*: Violates the architectural boundary between `firmware/lib` and `firmware/app`, breaks host-side C unit tests for `at_commands.c`, and drops the USB CDC-ACM connection before `OK\r\n` is transmitted to the host.

### Option 3: Callback-Decoupled 256-Byte Hex Chunk Streaming (`AT+DFU_*`), `k_work_delayable` Deferred Reboot, and Startup Image Confirmation (Chosen)
- *Mechanism*:
  1. **1024-Byte Line Buffer**: Increase [`AT_PARSER_MAX_LINE_LEN`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware/firmware/lib/include/hsm/at_parser.h#L12) (and [`mock_hsm_pty.py`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts/mock_hsm_pty.py#L625-L629)) from `512` to `1024` bytes so 256-byte binary chunks (`512` hex chars + `AT+DFU_CHUNK=<offset>,` + `\r\n`) fit comfortably.
  2. **Dependency-Inverted DFU Callback Table (`at_dfu_callbacks_t`)**: Define `at_dfu_callbacks_t` (`write_chunk`, `apply`, `get_info`) and `at_commands_register_dfu_callbacks()` in [`at_commands.h`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware/firmware/lib/include/hsm/at_commands.h), wired at runtime in [`main.c`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware/firmware/app/src/main.c).
  3. **Three AT DFU Commands**:
     - `AT+DFU_INFO?` $\rightarrow$ `+DFU_INFO: VER="<major>.<minor>.<patch>",SLOT=<slot>,SIZE=<size>,SHA256="<64-hex>"\r\nOK\r\n` (unauthenticated read allowed; CLI falls back to `AT+INFO?` if legacy firmware returns `+CME ERROR: 8`).
     - `AT+DFU_CHUNK=<offset>,<hex_data>` $\rightarrow$ Requires `keystore_is_authenticated()` (`+CME ERROR: 1` if locked), validates `1..256` decoded bytes (`+CME ERROR: 5` on invalid params), erases `image_1` when `offset == 0`, writes flash chunk, and responds `OK\r\n` (`10s` timeout in CLI).
     - `AT+DFU_APPLY` $\rightarrow$ Requires `keystore_is_authenticated()`, calls `boot_request_upgrade(BOOT_UPGRADE_TEST)`, schedules a `k_work_delayable` cold reboot (`200 ms` delay), and immediately returns `OK\r\n` so `usb_serial_poll()` flushes the response over USB before reset.
  4. **Boot Confirmation**: In [`dfu_service_init()`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware/firmware/app/src/dfu_service.c#L10-L13), check `!boot_is_img_confirmed()` and invoke `boot_write_img_confirmed()` so MCUboot marks the swapped slot permanent once the application initializes cleanly.

---

## Decision Outcome

**Chosen Option: Option 3 — Callback-Decoupled 256-Byte Hex Chunk Streaming (`AT+DFU_INFO?`, `AT+DFU_CHUNK`, `AT+DFU_APPLY`), `k_work_delayable` Deferred Reboot, and Startup Image Confirmation.**

### End-to-End DFU Sequence & Deferred Reboot Flow

```mermaid
sequenceDiagram
    participant CLI as hsmutil (ATClient)
    participant USB as USB CDC-ACM (cdc_acm_uart1)
    participant AT as at_commands.c (firmware/lib)
    participant DFU as dfu_service.c (firmware/app)
    participantBOOT as MCUboot (Stage 2)

    CLI->>USB: AT+DFU_INFO?\r\n
    USB->>AT: at_commands_execute("AT+DFU_INFO?")
    AT->>DFU: s_dfu_cbs.get_info(...)
    DFU-->>AT: VER=1.0.0, SLOT=0, SIZE, SHA256
    AT-->>USB: +DFU_INFO: VER="1.0.0",SLOT=0,SIZE=262144,SHA256="..."\r\nOK\r\n
    USB-->>CLI: FirmwareInfo struct

    loop Stream 256-byte chunks (512 hex chars)
        CLI->>USB: AT+DFU_CHUNK=<offset>,<hex_data>\r\n (10s timeout)
        USB->>AT: at_commands_execute("AT+DFU_CHUNK=...")
        Note over AT: Verify keystore_is_authenticated()
        AT->>DFU: s_dfu_cbs.write_chunk(offset, bin_buf, len)
        Note over DFU: If offset == 0, flash_area_erase(image_1)
        DFU-->>AT: 0 (OK)
        AT-->>USB: OK\r\n
        USB-->>CLI: OK
    end

    CLI->>USB: AT+DFU_APPLY\r\n
    USB->>AT: at_commands_execute("AT+DFU_APPLY")
    AT->>DFU: s_dfu_cbs.apply()
    DFU->>BOOT: boot_request_upgrade(BOOT_UPGRADE_TEST)
    Note over DFU: k_work_schedule(&s_reboot_work, K_MSEC(200))
    DFU-->>AT: 0 (OK)
    AT-->>USB: OK\r\n
    USB-->>CLI: OK (Clean completion before USB bus reset)
    Note over DFU,BOOT: 200ms timer fires -> sys_reboot(SYS_REBOOT_COLD)
    BOOT->>BOOT: Verify ECDSA P-256 signature & swap image_1 -> image_0
    BOOT->>DFU: Boot new application -> dfu_service_init()
    DFU->>BOOT: !boot_is_img_confirmed() -> boot_write_img_confirmed()
```

---

## Consequences

### Positive
- **Unified ASCII AT Transport**: All HSM operations — key management, ECDSA signing, factory reset, and firmware updates — now operate over a single, auditable ASCII AT command grammar on `cdc_acm_uart1`.
- **Clean Architectural Boundaries**: `firmware/lib` remains 100% platform-independent via `at_dfu_callbacks_t`, enabling fast host-side testing and mock PTY simulation (`scripts/mock_hsm_pty.py`).
- **Zero-Rollback Surprises & Safe Failover**: Automatic `boot_write_img_confirmed()` in `dfu_service_init()` makes valid upgrades permanent on first boot while preserving MCUboot's automatic rollback if a faulty image fails to reach application initialization.
- **Deterministic CLI UX**: Deferred reboot (`200 ms` via `k_work_delayable`) guarantees `hsmutil firmware update` receives the final `OK\r\n` and renders 100% progress completion cleanly.

### Negative
- **Hex Encoding Bandwidth Overhead**: Hex-encoding 256 binary bytes into 512 ASCII characters doubles serial wire bytes (~500 KB transfer for a 250 KB image), taking ~15–25 seconds over USB CDC-ACM — completely acceptable for administrative HSM firmware updates.
- **One-Time Hardware J-Link Bootstrap**: Physical dongles (`3302311888499AC7` and `FD54E793A75540C9`) running older pre-DFU firmware (`53f764b`) require a one-time SWD flash (`make firmware-flash`) before subsequent upgrades can be performed over USB via `hsmutil firmware update`.
