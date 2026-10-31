---
title: Workspace Git Status & Repository Topologies
tags:
  - git
  - status
  - repository
  - branches
created: 2026-10-31
updated: 2026-10-31
lifecycle_phase: Phase 6 (Merge / Post-Merge Complete)
feature: "[[features/usb-descriptors-ble-address|USB Descriptors & Device Identification]]"
changelog: "[[Changelog|Workspace Release Changelog]]"
---

# Workspace Git Status & Repository Topologies

This document tracks real-time Git branch topologies, commit records, working directory statuses, and merged feature branches across the multi-repository workspace.

---

## 1. Repository Status Overview

| Repository | Path | Active Branch | Tracking / Upstream | Latest Commit | Working Tree Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Workspace Root** | [nrf52840-hsm](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm) | `master` / `main` | Clean | `4b6d55b` (*docs(root): update workspace README with USB descriptors and BLE device targeting*) | **Merged & Clean** |
| **Firmware** | [nrf52840-hsm-firmware](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware) | `main` | Clean | `adb9c1e` (*docs(firmware): update README with USB descriptors and BLE address details*) | **Merged & Clean** |
| **CLI** | [nrf52840-hsm-cli](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli) | `main` | Clean | `c9f4590` (*docs(cli): update README with serial port autodiscovery heuristics and serial targeting*) | **Merged & Clean** |
| **PKCS#11 Provider** | [nrf52840-hsm-libpkcs11](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-libpkcs11) | `main` | Clean | `4673116` (*docs(p11): update README with AT port prioritization and serial targeting*) | **Merged & Clean** |
| **Obsidian Vault** | [nrf52840-hsm-obsidian](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-obsidian) | `master` | Clean | `1f347e8` (*docs(vault): update git status, branch tracking, and release changelog for v0.2.0 post-merge*) | **Merged & Clean** |

---

## 2. Feature Merge History: USB Descriptors & Device Identification

> [!NOTE]
> **Status**: Feature branch `feature/usb-descriptors-ble-address` successfully merged into `main` / `master` across all component repositories.  
> Detailed feature note: [[features/usb-descriptors-ble-address|USB Descriptors & Device Identification]].  
> System release note: [[Changelog|Release v0.2.0 - USB Descriptors & Device Identification]].

### 2.1 Multi-Repository Git Topology

```mermaid
gitGraph
    commit id: "Genesis"
    branch "feature/usb-descriptors-ble-address" order: 1
    checkout "feature/usb-descriptors-ble-address"
    commit id: "53f764b (Firmware: BLE Serial & Proto Descriptors)"
    commit id: "201764b (CLI: Autodiscovery & Devices Command)"
    commit id: "65b1dd0 (PKCS#11: Suffix 3 Scoring & Serial Targeting)"
    checkout main
    merge "feature/usb-descriptors-ble-address" id: "adb9c1e (Firmware main)"
    merge "feature/usb-descriptors-ble-address" id: "c9f4590 (CLI main)"
    merge "feature/usb-descriptors-ble-address" id: "4673116 (PKCS#11 main)"
    commit id: "c29b40a (Vault master)"
    commit id: "4b6d55b (Workspace root)"
```

### 2.2 Subsystem Merged Commit Details

#### 1. Firmware (`nrf52840-hsm-firmware`)
- **Active Branch**: `main` (Merged `feature/usb-descriptors-ble-address`)
- **Merge Commit**: [`adb9c1e`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware) — *docs(firmware): update README with USB descriptors and BLE address details*
- **Feature Commit**: [`53f764b`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware) — *feat(usb): implement dynamic serial from BLE address and configure interface descriptors*
- **Delivered Capabilities**:
  - Dynamic extraction of 48-bit factory BLE address from `NRF_FICR->DEVICEADDR[1]` (with BLE Random Static bitmask `0xC000`) and `NRF_FICR->DEVICEADDR[0]`.
  - Formatted 12-character uppercase hex serial (`%04X%08X`) registered via strong override of `usb_update_sn_string_descriptor()`.
  - User string descriptors (`USBD_STRING_DESCR_USER_DEFINE(primary)`): `"Zephyr CDC ACM Console"` and `"nRF52840 HSM AT Interface"`.
  - Dynamic protocol patching via `usb_serial_patch_descriptors()`: assigns Protocol `0x0001` (Console) on Interface 0 and `0x0002` (HSM AT API) on Interface 2 prior to calling `usb_enable(NULL)`.
  - Configured `CONFIG_USB_DEVICE_SN="000000000000"` and `CONFIG_HWINFO=y` in `prj.conf`.
- **Signed Binary Artifacts**:
  - `firmware/app/build-prod/app/zephyr/zephyr.signed.bin`
  - `firmware/app/build-prod/merged.hex`
  - `firmware/app/build-prod/dfu_application.zip`

#### 2. CLI (`nrf52840-hsm-cli`)
- **Active Branch**: `main` (Merged `feature/usb-descriptors-ble-address`)
- **Merge Commit**: [`c9f4590`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli) — *docs(cli): update README with serial port autodiscovery heuristics and serial targeting*
- **Feature Commit**: [`201764b`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli) — *feat(cli): implement serial port autodiscovery heuristics and serial targeting*
- **Delivered Capabilities**:
  - Port candidate scoring: assigns macOS suffix `3` (AT data channel) score `+100` and demotes suffix `1` (Console logging) by `-50`.
  - Added `hsmutil devices` (alias `hsmutil list`) with optional `--json` output to enumerate connected dongles, displaying BLE serial, Console port, and AT data port.
  - Added persistent `--serial` (`-s`) flag across commands to target specific dongles by 12-digit hex BLE address or device UID.
  - Supported `NRF52840_HSM_SERIAL` environment variable.
  - Cross-compiled binaries for 5 host platforms: `darwin-arm64`, `linux-amd64`, `linux-arm64`, `windows-amd64.exe`, `windows-arm64.exe`.

#### 3. PKCS#11 Library (`nrf52840-hsm-libpkcs11`)
- **Active Branch**: `main` (Merged `feature/usb-descriptors-ble-address`)
- **Merge Commit**: [`4673116`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-libpkcs11) — *docs(p11): update README with AT port prioritization and serial targeting*
- **Feature Commit**: [`65b1dd0`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-libpkcs11) — *feat(p11): prioritize suffix '3' AT port and support NRF52840_HSM_SERIAL targeting*
- **Delivered Capabilities**:
  - Candidate port scoring (`serial_score_candidate`): rates macOS `tty.usbmodem*` suffix `3` at `1000`, `cu.usbmodem*` suffix `3` at `900`, demoting suffix `1` to `200`/`100`.
  - Quicksort candidate ordering (`qsort` with `compare_port_candidates`) to probe AT channels first.
  - Direct hardware targeting via `NRF52840_HSM_SERIAL` environment variable.
  - 100% CTest passage on `test_p11_mock` and physical hardware compatibility.

#### 4. Obsidian Vault (`nrf52840-hsm-obsidian`)
- **Active Branch**: `master`
- **Latest Commit**: [`1f347e8`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-obsidian) — *docs(vault): update git status, branch tracking, and release changelog for v0.2.0 post-merge*
- **Delivered Records**:
  - Feature note [[features/usb-descriptors-ble-address|USB Descriptors & Device Identification]].
  - System Specification [[Specification/USB Descriptors and Device Identification|USB Descriptors and Device Identification]].
  - Implementation Plan [[Plan/USB Descriptors and BLE Identification Implementation Plan|Implementation Plan: USB Descriptors & BLE Device Identification]].
  - ADRs: [[Architecture/Decisions/ADR-0001 - Hardware BLE Address for Deterministic USB Serial Number|ADR-0001]], [[Architecture/Decisions/ADR-0002 - In-Tree Protocol Descriptor Patching vs macOS DriverKit Extension|ADR-0002]], [[Architecture/Decisions/ADR-0003 - Host Port Discovery and Serial Targeting Strategy|ADR-0003]].
  - Session archival [[Sessions/9f9ae0e3-f181-44f3-8dde-7b25c55cf4e3|Phase 3-5 Lifecycle Session]].
  - Release Changelog [[Changelog|Workspace Release Changelog]].

#### 5. Workspace Root Orchestrator (`nrf52840-hsm`)
- **Active Branch**: `master` / `main`
- **Latest Commit**: [`4b6d55b`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm) — *docs(root): update workspace README with USB descriptors and BLE device targeting*
- **Delivered Updates**:
  - Updated root [README.md](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/README.md) with deterministic macOS TTY device node mapping, USB interface table, and multi-device targeting commands.

---

## 3. Physical Hardware Topologies & Validation

Connected hardware dongles detected and verified via `hsmutil devices`:

```text
[1] Serial / BLE:  FD54E793A75540C9
    AT Data Port:  /dev/tty.usbmodem31424103
    Console Port:  /dev/tty.usbmodem31424101
    USB ID:        1209:0001 (nRF52840 HSM Dongle)

[2] Serial / BLE:  3302311888499AC7
    AT Data Port:  /dev/tty.usbmodem31424403
    Console Port:  /dev/tty.usbmodem31424401
    USB ID:        1209:0001 (nRF52840 HSM Dongle)
```

### 3.1 Validation Matrix (100% Passed)
- **USB Descriptors**: Validated on physical PCA10059 dongles with dynamic 12-digit hex BLE serial numbers (`%04X%08X`).
- **Protocol Identifiers**: Dynamic runtime descriptor patching sets Protocol `0x0001` (Console) on Interface 0 and `0x0002` (HSM AT API) on Interface 2.
- **macOS TTY Mapping**: Suffix `1` attached to Zephyr console logging; Suffix `3` attached to HSM AT data stream.
- **Dual-Device Concurrency & Serial Targeting**: Concurrently tested two connected hardware dongles with `hsmutil devices` and `hsmutil info -s <SERIAL>`, confirming complete isolation without port crosstalk.

---

## 4. Architecture Decision Records (ADRs)

- [[Architecture/Decisions/ADR-0001 - Hardware BLE Address for Deterministic USB Serial Number|ADR-0001: Hardware BLE Address for Deterministic USB Serial Number]]
- [[Architecture/Decisions/ADR-0002 - In-Tree Protocol Descriptor Patching vs macOS DriverKit Extension|ADR-0002: In-Tree Protocol Descriptor Patching vs macOS DriverKit Extension]]
- [[Architecture/Decisions/ADR-0003 - Host Port Discovery and Serial Targeting Strategy|ADR-0003: Host Port Discovery and Serial Targeting Strategy]]

---

## 5. Related Documentation

- [[00 - Index|Vault Index]]
- [[Changelog|Workspace Release Changelog]]
- [[features/usb-descriptors-ble-address|USB Descriptors & Device Identification Feature Specification & Implementation]]
- [[Architecture/USB Composite Stack|USB Composite Stack Architecture]]
- [[Development/CLI Reference|hsmutil CLI Reference]]
