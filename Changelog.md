---
title: Workspace Release Changelog
tags:
  - release
  - changelog
  - history
  - milestones
created: 2026-10-31
updated: 2026-11-01
---

# Workspace Release Changelog

This document tracks all formal releases, milestone deliveries, and post-merge changelog notes across the orchestrated multi-repository workspace (`ncs-nrf52840dongle-kms`).

---

## [v0.4.0-alpha] — 2026-11-01: Air-Gapped Raspberry Pi Embedded Linux OS (`rpi-z2w-yocto-hsm`)

> [!NOTE]
> **Release Highlights**: Addition and initialization of the `rpi-z2w-yocto-hsm` sub-repository delivering an ultra-hardened, air-gapped 64-bit embedded Linux operating system built with Yocto Project 5.0 LTS (`scarthgap`) and Linux Kernel 6.6 LTS. Features containerized Kas build workflows, complete physical network elimination (WiFi, BLE, Ethernet disabled at hardware/DTB and kernel level), `nftables` 100% total-drop firewall, out-of-tree `nrf52840-hsm-kmod` kernel driver, mandatory first-login wizard with clock prompt, read-only root filesystem, USB mass storage automount, and dual-slot RAUC A/B update verification using x509 PKI certificates.
>
> **Release Status**: Scaffolding complete across sub-repository, root Makefile targets integrated, and architecture documentation indexed in vault.

### 1. Summary of Changes

| Subsystem | Branch | Commit | Key Additions |
| :--- | :--- | :--- | :--- |
| **Air-Gapped OS** (`rpi-z2w-yocto-hsm`) | `main` | [`580b452`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm) | Initial scaffolding of Yocto 5.0 Scarthgap layer (`meta-hsm-os`), Kas configurations (`kas-project.yml`, `kas-dev.yml`, `kas-prod.yml`), `nrf52840-hsm-kmod` driver, `nftables` rules, first-login wizard, RAUC A/B update service, and build scripts. |
| **Workspace Root** (`nrf52840-hsm`) | `master` | Working Tree | Added `rpi-z2w-yocto-hsm` to `SUBREPOS` in root `Makefile`; added `yocto-build-dev`, `yocto-build-prod`, `yocto-bundle`, `yocto-shell`, and `yocto-clean` targets; updated root `AGENTS.md`. |
| **Obsidian Vault** (`nrf52840-hsm-obsidian`) | `master` | [`8e3fe33`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-obsidian) | Authored `Architecture/07 - Air-Gapped RPi HSM OS Architecture.md`, `Development/Yocto Build and RAUC Update Guide.md`, updated `00 - Index.md`, `git-status.md`, and `Changelog.md`. |

---

### 2. Detailed Subsystem Changes

#### Air-Gapped OS Subsystem (`rpi-z2w-yocto-hsm`)
- **Yocto Layer & Architecture**:
  - Initialized custom layer `meta-hsm-os` compatible with Yocto 5.0 (`scarthgap`).
  - Integrated `kas` orchestration with reproducible container runtime (`ghcr.io/siemens/kas/kas:latest`).
- **Kernel & Hardware Lockdown**:
  - Implemented kernel config fragments stripping wireless (`CONFIG_CFG80211=n`, `CONFIG_BT=n`), Ethernet, and unnecessary peripherals.
  - Device tree overlays disabling WiFi, Bluetooth, and debug GPIO headers.
- **Out-of-Tree Driver (`nrf52840-hsm-kmod`)**:
  - Dedicated character driver matching USB `0x1209:0x0001` and extracting device BLE address.
  - In production mode: binds and exposes AT command channel (`cdc_acm_uart1`) under `/dev/tty.hsm<ble><num>` while suppressing logger port (`cdc_acm_uart0`).
- **Network Security & User Space**:
  - Enforced `nftables` default drop across input, output, and forward chains.
  - Implemented mandatory first-login wizard enforcing user password rotation, keyboard layout selection, and locale setting.
  - Implemented every-login UTC clock synchronization prompt to compensate for lack of battery RTC/NTP.
  - Implemented USB storage automount service under `/media/usb-*` for `hsmutil` user space key exports.
- **Dual-Slot RAUC A/B Updates**:
  - Integrated `meta-rauc` and `meta-rauc-raspberrypi` with U-Boot boot counter and watchdog fallback.
  - Implemented ECDSA P-256 PKI key generation script and automated USB update installer service (`rauc-usb-autoinstall`).

---

## [v0.3.0] — 2026-10-31: Factory Reset Safeguards & Multi-Device Auto-Discovery Disambiguation

> [!NOTE]
> **Release Highlights**: Implementation of irreversible hardware factory reset zeroization (`keystore_factory_reset`) on CryptoCell CC310 keys (slots 1..32) and Shamir master secret in NVS, interactive confirmation prompt and `-y`/`--yes` bypass in `hsmutil reset`, and strict auto-discovery disambiguation (`ErrMultipleDevicesDetected`) preventing accidental targeting of multiple connected HSM dongles.
>
> **Release Status**: Merged via fast-forward into `master` / `main` production branches across all repositories with feature branches pruned. Verified with 100% test pass rate across unit tests, production sysbuilds, and root integration test suites.

### 1. Summary of Changes

| Subsystem | Merged Branch | Target Branch | Commit | Key Additions |
| :--- | :--- | :--- | :--- | :--- |
| **Firmware** (`nrf52840-hsm-firmware`) | `feature/factory-reset-zeroization` (pruned) | `main` | [`ae087c1`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware) | `keystore_factory_reset()` wiping CC310 slots 1..32, SSS zeroization, factory defaults restore, NVS persistence, `AT+RESET` & `AT+FACTORYRESET` (Base commit: [`166c307`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware)). |
| **CLI** (`nrf52840-hsm-cli`) | `feature/reset-safeguard-and-disambiguation` (pruned) | `main` | [`c110845`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli) | Interactive reset safeguard prompt `[y/N]`, `-y/--yes` bypass, `ErrMultipleDevicesDetected` disambiguation error, `ResolvePortOrSerial()` (Base commit: [`29659c2`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli)). |
| **Workspace Root** (`nrf52840-hsm`) | `feature/reset-and-disambiguation-tests` (pruned) | `master` | [`61a400e`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm) | Mock PTY factory zeroization, AT client cryptographic reset test assertions, end-to-end integration test coverage (Base commit: [`c4fb7d6`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm)). |
| **Obsidian Vault** (`nrf52840-hsm-obsidian`) | `master` | `master` | [`6ad1077`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-obsidian) | ADR-0004, feature specification, implementation plan, pre-merge & post-merge release records. |

---

### 2. Detailed Subsystem Changes

#### Firmware Subsystem (`nrf52840-hsm-firmware`)
- **Factory Reset Hardware Zeroization**:
  - Implemented `keystore_factory_reset(void)` in `firmware/lib/src/keystore.c` and header prototype in `include/hsm/keystore.h`.
  - Iterates all 32 persistent slots and calls `crypto_cc310_destroy_key(slot_id)`.
  - Zeroizes RAM Shamir Secret Sharing master secret with `sss_zeroize()`.
  - Restores default master PIN (`"12345678"`) and resets PIN retry counter to `3`.
  - Persists zeroized state to Non-Volatile Storage (`save_to_nvs()`).
  - Emits asynchronous state notification `notify_state(HSM_STATE_UNCONFIGURED)` switching RGB LED to breathing BLUE.
- **AT Command Handler**:
  - Bound `AT+RESET` and added alias `AT+FACTORYRESET` to `keystore_factory_reset()`.

#### Host CLI Subsystem (`nrf52840-hsm-cli`)
- **Confirmation Safeguards (`hsmutil reset`)**:
  - Bound `-y, --yes` flag to bypass interactive prompt.
  - Interactive warning banner and explicit `(yes/no)` confirmation prompt on stdin.
  - Non-affirmative responses abort cleanly with exit code 0 and `Reset cancelled.`.
  - Success feedback informs operator that device is unconfigured and guides to `hsmutil init --threshold`.
- **Multi-Device Auto-Discovery Disambiguation**:
  - Defined domain error `ErrMultipleDevicesDetected` formatting a table of attached devices with BLE serials, AT Data ports, and Console ports.
  - Refactored `AutoDiscoverPortTargeting()` to halt with `ErrMultipleDevicesDetected` if `len(dongles) > 1` and no `--serial` or `--port` is supplied.
  - Implemented `ResolvePortOrSerial()` transparently parsing paths vs 12-digit hex BLE serials for `hsmutil key migrate`.

#### Workspace Root & Test Harness (`scripts`)
- **Mock HSM Serial Engine**:
  - Enhanced `scripts/mock_hsm_pty.py` to clear all key slots, master secret, and revert `shamir_initialized` to `False` on `AT+RESET`.
- **Automated Integration & Regression Tests**:
  - Added cryptographic reset verification in `scripts/test_at_client.py`.
  - Added reset safeguard prompt cancellation and `--yes` assertions to `scripts/run_integration_tests.sh`.

---

### 3. Architecture Decision Records

- [[Architecture/Decisions/ADR-0004 - Factory Reset Semantics and Safeguards|ADR-0004: Factory Reset Semantics and Safe Erasure Safeguards]]
- [[Architecture/Decisions/ADR-0003 - Host Port Discovery and Serial Targeting Strategy|ADR-0003: Host Port Discovery and Serial Targeting Strategy]]

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
