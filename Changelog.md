---
title: Workspace Release Changelog
tags:
  - release
  - changelog
  - history
  - milestones
created: 2026-10-31
updated: 2026-10-31
---

# Workspace Release Changelog

This document tracks all formal releases, milestone deliveries, and post-merge changelog notes across the orchestrated multi-repository workspace (`ncs-nrf52840dongle-kms`).

---

## [v0.2.0] — 2026-10-31: USB Descriptors & Device Identification

> [!NOTE]
> **Release Highlights**: Complete implementation of deterministic hardware BLE address serial numbers, runtime USB protocol descriptor patching, macOS dual CDC-ACM device node predictability (`suffix 1` for Console, `suffix 3` for AT data), CLI and PKCS#11 multi-device autodiscovery and `--serial` targeting.

### 1. Summary of Changes

| Subsystem | Merged Branch | Merge Commit | Key Additions |
| :--- | :--- | :--- | :--- |
| **Firmware** (`nrf52840-hsm-firmware`) | `feature/usb-descriptors-ble-address` $\rightarrow$ `main` | [`adb9c1e`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware) | Dynamic FICR BLE serial number hook, runtime protocol patching (`0x0001`/`0x0002`), custom string descriptors. |
| **CLI** (`nrf52840-hsm-cli`) | `feature/usb-descriptors-ble-address` $\rightarrow$ `main` | [`c9f4590`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli) | Port autodiscovery heuristics (score suffix `3`), `hsmutil devices` (`list`) command, `--serial` (`-s`) flag, `NRF52840_HSM_SERIAL`. |
| **PKCS#11** (`nrf52840-hsm-libpkcs11`) | `feature/usb-descriptors-ble-address` $\rightarrow$ `main` | [`4673116`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-libpkcs11) | Port candidate scoring (`serial_score_candidate`), quicksort candidate prioritization, `NRF52840_HSM_SERIAL` support. |
| **Docs Vault** (`nrf52840-hsm-obsidian`) | `master` | [`c29b40a`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-obsidian) | Feature specification, implementation plan, ADR-0001, ADR-0002, ADR-0003, Phase 3-5 session logs. |
| **Workspace Root** (`nrf52840-hsm`) | `master` / `main` | [`4b6d55b`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm) | Root README update with deterministic macOS port mappings, interface tables, and device targeting instructions. |

---

### 2. Detailed Subsystem Changes

#### Firmware (`nrf52840-hsm-firmware`)
- **Dynamic BLE Serial Number**:
  - Implemented `compute_ble_sn()` reading `NRF_FICR->DEVICEADDR[1]` (with BLE Random Static bitmask `0xC000`) and `NRF_FICR->DEVICEADDR[0]`.
  - Formatted 12-character uppercase hex serial (`%04X%08X`, MSB to LSB).
  - Strong symbol override of `usb_update_sn_string_descriptor()` returning dynamic BLE serial string.
  - Set `CONFIG_USB_DEVICE_SN="000000000000"` and enabled `CONFIG_HWINFO=y` in `prj.conf`.
- **Protocol Identifiers & String Descriptors**:
  - Registered user string descriptors via `USBD_STRING_DESCR_USER_DEFINE(primary)`:
    - `"Zephyr CDC ACM Console"`
    - `"nRF52840 HSM AT Interface"`
  - Implemented `usb_serial_patch_descriptors()` traversing `__usb_descriptor_start` prior to `usb_enable(NULL)`:
    - Interface 0 (Console CDC Comm): `bInterfaceProtocol = 0x01`, `iInterface = str_idx_console`.
    - Interface 2 (HSM AT CDC Comm): `bInterfaceProtocol = 0x02`, `iInterface = str_idx_hsm`.
- **Signed Binary Artifacts**:
  - MCUboot signed application: `firmware/app/build-prod/app/zephyr/zephyr.signed.bin`
  - Merged flash image: `firmware/app/build-prod/merged.hex`
  - DFU package: `firmware/app/build-prod/dfu_application.zip`

#### CLI (`nrf52840-hsm-cli`)
- **Port Autodiscovery Heuristics**:
  - Implemented candidate port scoring assigning suffix `3` score `+100` and demoting suffix `1` by `-50`.
  - Added `hsmutil devices` (alias `hsmutil list`) with optional `--json` flag to inspect connected dongles with BLE serial, Console port, and AT data port.
- **Serial Device Targeting**:
  - Added persistent `--serial` (`-s`) flag to target specific dongles by 12-digit hex BLE address or device UID.
  - Supported `NRF52840_HSM_SERIAL` environment variable.
- **Cross-Platform Compilation**:
  - Built binaries across 5 host targets: `darwin-arm64`, `linux-amd64`, `linux-arm64`, `windows-amd64.exe`, `windows-arm64.exe`.
  - Verified 100% Go unit test coverage (`go test ./...`).

#### PKCS#11 Library Provider (`nrf52840-hsm-libpkcs11`)
- **Candidate Port Scoring**:
  - Implemented `serial_score_candidate()`: assigns score `1000` to `tty.usbmodem*3`, `900` to `cu.usbmodem*3`, demoting suffix `1` candidates.
  - Ordered candidates via `qsort` and `compare_port_candidates` before probing.
- **Direct Serial Targeting**:
  - Supports `NRF52840_HSM_SERIAL` environment variable for direct node synthesis `/dev/tty.usbmodem<SERIAL>3`.
- **Test Harness Verification**:
  - 100% CTest pass on `test_p11_mock` and hardware validation on physical dongles.

---

### 3. Verification & Validation Evidence

- **Dual-Device Isolation**: Tested simultaneously with two connected PCA10059 dongles:
  - Dongle 1: BLE `FD54E793A75540C9` $\rightarrow$ AT `/dev/tty.usbmodem31424103`, Console `/dev/tty.usbmodem31424101`
  - Dongle 2: BLE `3302311888499AC7` $\rightarrow$ AT `/dev/tty.usbmodem31424403`, Console `/dev/tty.usbmodem31424401`
- **Zero Crosstalk**: Targeted queries via `hsmutil info -s FD54E793A75540C9` and `hsmutil info -s 3302311888499AC7` confirmed complete channel separation.

---

### 4. Architecture Decision Records

- [[Architecture/Decisions/ADR-0001 - Hardware BLE Address for Deterministic USB Serial Number|ADR-0001: Hardware BLE Address for Deterministic USB Serial Number]]
- [[Architecture/Decisions/ADR-0002 - In-Tree Protocol Descriptor Patching vs macOS DriverKit Extension|ADR-0002: In-Tree Protocol Descriptor Patching vs macOS DriverKit Extension]]
- [[Architecture/Decisions/ADR-0003 - Host Port Discovery and Serial Targeting Strategy|ADR-0003: Host Port Discovery and Serial Targeting Strategy]]

---

## [v0.1.0] — 2026-10-29: Multi-Repository Genesis Architecture & Dual-Stage Secure Boot

> [!NOTE]
> **Genesis Release**: Initial multi-repository workspace initialization, dual-stage secure boot chain, dual CDC-ACM USB endpoints, cross-platform CLI, PKCS#11 shared library provider, and documentation vault.

### 1. Key Additions
- **Dual-Stage Secure Bootloader**: Stage 1 B0 immutable bootloader with CC310 hardware crypto validation $\rightarrow$ Stage 2 MCUboot with dual-slot partition manager $\rightarrow$ Signed Application using root ECDSA P-256 keys (`keys/root-ec-p256.pem`).
- **Dual CDC-ACM Endpoints**: Zephyr kernel logging and console on `cdc_acm_uart0`, HSM binary/AT commands on `cdc_acm_uart1`.
- **RGB LED State Engine**: Visual status indication for unconfigured (pulsing blue), locked (solid red), and unlocked (solid green) states.
- **`hsmutil` CLI / TUI**: Built with Go 1.24, Cobra, and BubbleTea for macOS, Linux, and Windows.
- **PKCS#11 Shared Library Provider**: Cryptoki v2.40 compliant shared library (`libnrf52840_pkcs11.dylib`/`.so`/`.dll`).
- **Root Automation**: Root `Makefile` coordinating `sync`, `status`, `firmware-build`, `firmware-flash`, `cli-build`, and `libpkcs11-build`.
- **Obsidian Documentation Vault**: Centralized vault in `nrf52840-hsm-obsidian/`.
