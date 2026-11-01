---
title: Workspace Git Status & Repository Topologies
tags:
  - git
  - status
  - repository
  - branches
created: 2026-10-31
updated: 2026-11-02
lifecycle_phase: Sub-Repository Scaffolding & Yocto Image Build Verification (5,844 Tasks)
feature: "[[Architecture/07 - Air-Gapped RPi HSM OS Architecture|Air-Gapped RPi HSM OS Architecture]]"
changelog: "[[Changelog|Workspace Release Changelog]]"
---

# Workspace Git Status & Repository Topologies

This document tracks real-time Git branch topologies, commit records, working directory statuses, and merged feature branches across the multi-repository workspace.

---

## 1. Repository Status Overview

| Repository | Path | Active Branch | Tracking / Upstream | Latest Commit | Working Tree Status | Release / Lifecycle Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Workspace Root** | [nrf52840-hsm](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm) | `master` | Clean | `61a400e` (*docs(root): update README with reset safeguards and multi-device disambiguation*) | **Clean** | **Merged (Fast-Forward to `master`) / Root Makefile & AGENTS.md updated** |
| **Firmware** | [nrf52840-hsm-firmware](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware) | `main` | Clean | `ae087c1` (*docs(firmware): update README with CC310 factory reset and AT+RESET*) | **Clean** | **Merged (Fast-Forward to `main`) / v0.3.0 Released** |
| **CLI** | [nrf52840-hsm-cli](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli) | `main` | Clean | `c110845` (*docs(cli): update README with reset safeguards and multi-dongle disambiguation*) | **Clean** | **Merged (Fast-Forward to `main`) / v0.3.0 Released** |
| **PKCS#11 Provider** | [nrf52840-hsm-libpkcs11](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-libpkcs11) | `main` | Clean | `4673116` (*docs(p11): update README with AT port prioritization and serial targeting*) | **Clean** | Up to date (`main`) |
| **Air-Gapped OS** | [rpi-z2w-yocto-hsm](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm) | `main` | Clean | `1ce95a1` (*feat(yocto): complete air-gapped hardened OS recipes and configuration*) | **Clean** | **Yocto Build Verified (`hsm-image-dev` 5,844 Tasks Succeeded) / v0.4.0-alpha** |
| **Obsidian Vault** | [nrf52840-hsm-obsidian](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-obsidian) | `master` | Clean | `5f9834a` (*docs(vault): document complete Yocto implementation for rpi-z2w-yocto-hsm*) | **Clean** | Up to date (`master`) |

---

## 1.1 New Sub-Repository: `rpi-z2w-yocto-hsm`

- **Repository Path**: [`rpi-z2w-yocto-hsm/`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm)
- **Active Branch**: `main`
- **Latest Commit**: [`1ce95a1`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm) — *feat(yocto): complete air-gapped hardened OS recipes and configuration*
- **Base Scaffolding Commit**: [`580b452`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm) — *feat: initial scaffolding of rpi-z2w-yocto-hsm air-gapped embedded OS*
- **Architecture Note**: [[Architecture/07 - Air-Gapped RPi HSM OS Architecture|07 - Air-Gapped RPi HSM OS Architecture]]
- **Hardware Integration Note**: [[Hardware/Waveshare USB HAT (B) Integration|Waveshare USB HAT (B) Integration]]
- **Operational Guide**: [[Development/Yocto Build and RAUC Update Guide|Yocto Build and RAUC Update Guide]]
- **Implemented Components & Recipes**:
  - **Declarative Kas Build Configurations**: `kas-project.yml` (Scarthgap 5.0 layers), `kas-dev.yml` (development tweaks, sudo), `kas-prod.yml` (production hardening, quiet kernel, locked root).
  - **Out-of-Tree Driver (`nrf52840-hsm-kmod`)**:
    - Character driver matching USB `0x1209:0x0001` extracting device BLE address.
    - Blacklisted generic ACM driver (`blacklist-cdc-acm.conf`).
    - Exposes deterministic `/dev/tty.hsm<ble_address><port>` nodes with `/dev/tty.hsm` symlink and `0660` permissions owned by group `dialout`.
    - In production mode: strictly binds AT command interface (`cdc_acm_uart1`) and suppresses debug logger interface (`cdc_acm_uart0`).
  - **Kernel Defconfig Hardening & Device Tree Overlays**:
    - `hardened.cfg`: completely removes wireless (`CFG80211=n`, `BT=n`), Ethernet, USB network drivers, and userspace GPIO sysfs.
    - Memory hardening: `CONFIG_SECURITY=y`, `CONFIG_HARDENED_USERCOPY=y`, `CONFIG_FORTIFY_SOURCE=y`, `CONFIG_STRICT_KERNEL_RWX=y`, `CONFIG_RANDOMIZE_BASE=y`.
    - `rpi-config_%.bbappend`: disables onboard WiFi/Bluetooth (`disable-wifi`, `disable-bt`), enables DWC2 USB host mode (`dtoverlay=dwc2,dr_mode=host`) for Waveshare USB HAT (B), and enables HDMI force hotplug (`hdmi_force_hotplug=1`, `hdmi_drive=2`).
  - **Packet Filter Firewall (`nftables`)**:
    - Unconditional default drop policy across input, forward, and output chains (`nftables.conf`).
  - **Volatile Storage & RAM Overlays**:
    - `volatile-storage` service mounts `tmpfs` on `/media`, `overlayfs` on `/etc` (`/run/overlay/etc-upper`), and `tmpfs` on `/home/hsmutil`.
    - Allows volatile runtime operations (like `passwd`) without modifying the read-only flash rootfs.
  - **Silent Journald Configuration**:
    - Production: `Storage=none` with all console/syslog/kmsg forwards disabled.
    - Development: `Storage=volatile` with 64 MB RAM limit.
  - **User Management & Login Wizard**:
    - `hsmutil-user.bb`: dedicated unprivileged `hsmutil` user with groups `users`, `dialout`, `tty`, `storage`.
    - Production image locks superuser (`usermod -L root;`); dev image configures password `root` and sudo.
    - First-login setup wizard (`first-login-wizard.sh`): forced password reset, interactive keyboard selection (`loadkeys`), and UTC timezone standard.
    - Interactive login clock prompt (`clock-setup.sh`) backed by setuid root C helper (`hsm-set-date`), allowing unprivileged time calibration without sudo.
  - **USB Mass Storage Automount & Offline RAUC Updates**:
    - `usb-mount.sh` automatically mounts FAT32/ext4 partitions under `/media/usb-*` with ownership `hsmutil:storage`.
    - Detects `*.raucb` update bundles and automatically triggers `rauc-usb-autoinstall.service`.
    - Dual-slot A/B partition management with U-Boot boot counter and watchdog fallback.
  - **Automation Scripts**: `scripts/run-kas.sh`, `scripts/flash-sd.sh`, `scripts/generate-rauc-keys.sh`, `scripts/create-rauc-bundle.sh`.
- **Root Makefile Targets**:
  - `make yocto-build-dev` — builds development image (`hsm-image-dev`).
  - `make yocto-build-prod` — builds production air-gapped image (`hsm-image-prod`).
  - `make yocto-bundle` — creates signed RAUC update bundle (`.raucb`).
  - `make yocto-shell` — interactive Kas shell inside build container.
  - `make yocto-clean` — cleans build artifacts.

### 1.1.1 Verified Yocto Development Image Build & Deployed Artifacts

The containerized Kas build workflow for the development image has been executed and verified:
- **Build Target**: `hsm-image-dev` for machine `raspberrypi0-2w-64`
- **BitBake Execution**: **5,844 tasks attempted and succeeded** (zero build errors, zero task failures)
- **Deployed Artifacts** in [`rpi-z2w-yocto-hsm/build/deploy/images/raspberrypi0-2w-64/`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/build/deploy/images/raspberrypi0-2w-64):
  - `hsm-image-dev-raspberrypi0-2w-64.rootfs.wic.bz2` (**95 MB**): Compressed raw bootable SD image with dual-slot A/B partition topology and U-Boot.
  - `hsm-image-dev-raspberrypi0-2w-64.rootfs.wic.bmap` (**4.0 KB**): Fast block map file for accelerated flashing.
  - `hsm-image-dev-raspberrypi0-2w-64.rootfs.tar.bz2` (**73 MB**): Rootfs compressed tarball with hardened configs and `hsmutil`.
  - Kernel Device Tree Overlays & Boot Files:
    - `disable-bt.dtbo` (**1.0 KB**): Disables onboard Broadcom Bluetooth controller at boot.
    - `disable-wifi.dtbo` (**387 B**): Disables onboard Broadcom SDIO WiFi interface at boot.
    - `dwc2.dtbo` (**801 B**): Configures DWC2 USB host mode for Waveshare USB HAT (B).
    - `bcm2710-rpi-zero-2.dtb` (**33.7 KB**): Compiled device tree blob for Raspberry Pi Zero 2 W SoC.
    - `boot.scr` (**1.3 KB**): U-Boot boot script for A/B slot selection and fallback.
- **Flashing Procedures**:
  - Direct streaming flasher: `./scripts/flash-sd.sh build/deploy/images/raspberrypi0-2w-64/hsm-image-dev-raspberrypi0-2w-64.rootfs.wic.bz2 <target_device>`
  - High-speed block copy: `sudo bmaptool copy --bmap build/deploy/images/raspberrypi0-2w-64/hsm-image-dev-raspberrypi0-2w-64.rootfs.wic.bmap build/deploy/images/raspberrypi0-2w-64/hsm-image-dev-raspberrypi0-2w-64.rootfs.wic <target_device>`

---

## 2. Merged Feature Release: Reset Confirmation & Multi-Device Disambiguation (v0.3.0)

> [!NOTE]
> **Status**: Phase 6 (Merge & Release) Complete. Feature branches were fast-forward merged into base branches (`master` / `main`) and pruned. Comprehensive Quality Audit and post-merge regression test suites achieved **100% compliance** across all repositories.  
> Detailed feature note: [[features/hsmutil-reset-and-multi-device-disambiguation|Reset Confirmation & Multi-Device Disambiguation]].  
> Implementation Plan: [[Plan/Reset Confirmation and Multi-Device Disambiguation Implementation Plan|Reset Confirmation & Multi-Device Disambiguation Implementation Plan]].  
> Specification: [[Specification/Reset Confirmation and Multi-Device Disambiguation|Reset Confirmation & Multi-Device Disambiguation Specification]].  
> Architecture Decision Record: [[Architecture/Decisions/ADR-0004 - Factory Reset Semantics and Safeguards|ADR-0004: Factory Reset Semantics and Safeguards]].  
> Release Changelog: [[Changelog|Release v0.3.0 - Factory Reset Safeguards & Multi-Device Auto-Discovery Disambiguation]].

### 2.0 Post-Merge Quality Audit & Verification Summary

| Verification Gate | Target Subsystem | Command | Result | Audit Status |
| :--- | :--- | :--- | :--- | :--- |
| **Code Cleanliness & Style** | CLI, Firmware, Scripts | Static analysis & linter checks | Zero dead code, proper error wrapping, explicit comments preserved | **100% Passed** |
| **Firmware Unit Tests** | `nrf52840-hsm-firmware` | `ctest --test-dir firmware/lib/build --output-on-failure` | 100% pass across crypto, keystore, and AT protocol engine | **100% Passed** |
| **Firmware Sysbuild** | `nrf52840-hsm-firmware` | `west build -b nrf52840dongle/nrf52840 firmware/app -d firmware/app/build-prod` | Dual-stage bootloader + MCUboot application built cleanly | **100% Passed** |
| **CLI Unit Tests** | `nrf52840-hsm-cli` | `go test ./... -v` | 100% pass across discovery, domain errors, and reset command UI tests | **100% Passed** |
| **CLI Cross-Compilation** | `nrf52840-hsm-cli` | `make cli-build-all` | Clean builds for `darwin/arm64`, `linux/arm64`, `linux/amd64`, `windows/arm64`, `windows/amd64` | **100% Passed** |
| **Mock PTY Integration** | Workspace Root | `make integration-test` | End-to-end integration tests including factory reset zeroization and safeguards | **100% Passed** |

### 2.1 Multi-Repository Git Topology

```mermaid
gitGraph
    commit id: "v0.2.0"
    branch "feature/factory-reset-zeroization" order: 1
    checkout "feature/factory-reset-zeroization"
    commit id: "166c307 (Firmware: reset & AT)"
    commit id: "ae087c1 (Firmware: README)"
    checkout main
    branch "feature/reset-safeguard-and-disambiguation" order: 2
    checkout "feature/reset-safeguard-and-disambiguation"
    commit id: "29659c2 (CLI: -y flag & prompt)"
    commit id: "c110845 (CLI: README)"
    checkout main
    branch "feature/reset-and-disambiguation-tests" order: 3
    checkout "feature/reset-and-disambiguation-tests"
    commit id: "c4fb7d6 (Root: Mock PTY & test)"
    commit id: "61a400e (Root: README)"
    checkout main
    merge "feature/factory-reset-zeroization" id: "ae087c1 (Firmware main)"
    merge "feature/reset-safeguard-and-disambiguation" id: "c110845 (CLI main)"
    merge "feature/reset-and-disambiguation-tests" id: "61a400e (Root master)"
    commit id: "v0.3.0 (Release)"
```

### 2.2 Subsystem Development Commits & File Inventory

#### 1. Firmware Subsystem (`nrf52840-hsm-firmware`)
- **Active Branch**: `main` (Fast-forward merged from `feature/factory-reset-zeroization`; feature branch deleted)
- **Latest Commit**: [`ae087c1`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware) — *docs(firmware): update README with CC310 factory reset and AT+RESET*
- **Base Feature Commit**: [`166c307`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware) — *feat(keystore): implement keystore_factory_reset with CC310 zeroization and AT+FACTORYRESET*
- **Modified Files**:
  - [`README.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware/README.md): Documented hardware zeroization and AT+RESET/AT+FACTORYRESET commands.
  - [`firmware/lib/include/hsm/keystore.h`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware/firmware/lib/include/hsm/keystore.h) (+10 lines): Added `int keystore_factory_reset(void);` API prototype.
  - [`firmware/lib/src/keystore.c`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware/firmware/lib/src/keystore.c) (+21 lines): Implemented hardware zeroization looping slots 1..32, CC310 key destruction, `sss_zeroize()`, default PIN restoration, and NVS persistence.
  - [`firmware/lib/src/at_commands.c`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware/firmware/lib/src/at_commands.c) (+7 lines): Bound `AT+RESET` and alias `AT+FACTORYRESET` to `keystore_factory_reset()`.
- **Test Verification**:
  - `ctest` passed 100%.
  - Sysbuild production build passed cleanly.

#### 2. Host CLI Subsystem (`nrf52840-hsm-cli`)
- **Active Branch**: `main` (Fast-forward merged from `feature/reset-safeguard-and-disambiguation`; feature branch deleted)
- **Latest Commit**: [`c110845`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli) — *docs(cli): update README with reset safeguards and multi-dongle disambiguation*
- **Base Feature Commit**: [`29659c2`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli) — *feat(cli): add reset confirmation safeguard and multi-device auto-discovery disambiguation*
- **Modified Files**:
  - [`README.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli/README.md): Documented reset safeguards, `-y` flag, and multi-dongle auto-discovery disambiguation.
  - [`internal/domain/at_errors.go`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli/internal/domain/at_errors.go) (+18 lines): Added `ErrMultipleDevicesDetected` error type formatting multi-device instructions.
  - [`internal/infra/serial/discovery.go`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli/internal/infra/serial/discovery.go) (+79 lines): Refactored `AutoDiscoverPortTargeting()` to reject multi-dongle ambiguity; implemented `ResolvePortOrSerial()`.
  - [`internal/infra/serial/discovery_test.go`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli/internal/infra/serial/discovery_test.go) (+239 lines): Added unit test suite covering discovery edge cases.
  - [`internal/ui/cli/auth.go`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli/internal/ui/cli/auth.go) (+30 lines): Added `-y, --yes` flag handling, interactive stdin prompt, cancellation logic, and success feedback.
  - [`internal/ui/cli/cli_test.go`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli/internal/ui/cli/cli_test.go) (+97 lines): Added CLI test cases for `reset` command confirmation and bypass.
  - [`internal/ui/cli/migrate.go`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli/internal/ui/cli/migrate.go) (+20 lines): Updated migration targeting via `ResolvePortOrSerial()`.
  - [`internal/ui/cli/root.go`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli/internal/ui/cli/root.go) (+10 lines): Bound `--yes` flag and enhanced error formatting.
- **Test Verification**:
  - `go test ./... -v` passed 100% across all packages.
  - Cross-compilation verified cleanly for darwin/arm64, linux/arm64, linux/amd64, windows/arm64, windows/amd64.

#### 3. Workspace Root & Test Harness (`nrf52840-hsm`)
- **Active Branch**: `master` (Fast-forward merged from `feature/reset-and-disambiguation-tests`; feature branch deleted)
- **Latest Commit**: [`61a400e`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm) — *docs(root): update README with reset safeguards and multi-device disambiguation*
- **Base Feature Commit**: [`c4fb7d6`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm) — *feat(scripts): add factory reset zeroization and confirmation test harness*
- **Modified Files**:
  - [`README.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/README.md): Documented factory reset zeroization safeguards and multi-device targeting disambiguation.
  - [`scripts/mock_hsm_pty.py`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts/mock_hsm_pty.py) (+25 lines): Added `AT+RESET` slot zeroization and master secret erasure.
  - [`scripts/test_at_client.py`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts/test_at_client.py) (+84 lines): Added cryptographic reset assertion tests.
  - [`scripts/run_integration_tests.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts/run_integration_tests.sh) (+20 lines): Added integration assertions for reset prompt cancellation and `--yes` flag.
- **Test Verification**:
  - `make integration-test` passed 100%.

---

## 3. Previous Feature Release History: USB Descriptors & Device Identification (v0.2.0)

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
- [[Architecture/Decisions/ADR-0004 - Factory Reset Semantics and Safeguards|ADR-0004: Factory Reset Semantics and Safe Erasure Safeguards]]

---

## 5. Upcoming & In-Flight Features

- **[[features/hsmutil-reset-and-multi-device-disambiguation|Reset Confirmation & Multi-Device Disambiguation]]**:
  - **Status**: Completed & Released (v0.3.0) — Phase 6 Post-Merge Complete.
  - **Formal Spec**: [[Specification/Reset Confirmation and Multi-Device Disambiguation|Reset Confirmation and Multi-Device Disambiguation Specification]]
  - **Implementation Plan**: [[Plan/Reset Confirmation and Multi-Device Disambiguation Implementation Plan|Reset Confirmation & Multi-Device Disambiguation Implementation Plan]]
  - **Specification Session**: [[Sessions/36b74c4b-dd01-44e1-9405-48fdc8d21d87|Session 36b74c4b]]
  - **Planning Session**: [[Sessions/4b1bb45a-82b0-4966-bacf-254b577ad6f2|Session 4b1bb45a]]
  - **Merged Base Commits**:
    - Root: `master` at [`61a400e`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm)
    - Firmware: `main` at [`ae087c1`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware)
    - CLI: `main` at [`c110845`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli)
  - **Verification Status**: 100% passing across firmware `ctest`, sysbuild, CLI unit tests, cross-compilation, and root integration test harness. Working trees clean across all repositories.

---

## 6. Related Documentation

- [[00 - Index|Vault Index]]
- [[Changelog|Workspace Release Changelog]]
- [[features/usb-descriptors-ble-address|USB Descriptors & Device Identification Feature Specification & Implementation]]
- [[features/hsmutil-reset-and-multi-device-disambiguation|Reset Confirmation & Multi-Device Disambiguation Feature Specification]]
- [[Architecture/USB Composite Stack|USB Composite Stack Architecture]]
- [[Development/CLI Reference|hsmutil CLI Reference]]
