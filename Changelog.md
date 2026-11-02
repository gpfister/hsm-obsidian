---
title: Workspace Release Changelog
tags:
  - release
  - changelog
  - history
  - milestones
created: 2026-10-31
updated: 2026-11-02
---

# Workspace Release Changelog

This document tracks all formal releases, milestone deliveries, and post-merge changelog notes across the orchestrated multi-repository workspace (`ncs-nrf52840dongle-kms`).

---

## [v0.4.0-alpha] — 2026-11-01: Air-Gapped Raspberry Pi Embedded Linux OS (`rpi-z2w-yocto-hsm`)

> [!NOTE]
> **Release Highlights**: Addition and initialization of the `rpi-z2w-yocto-hsm` sub-repository delivering an ultra-hardened, air-gapped 64-bit embedded Linux operating system built with Yocto Project 5.0 LTS (`scarthgap`) and Linux Kernel 6.6 LTS. Features containerized Kas build workflows, complete physical network elimination (WiFi, BLE, Ethernet disabled at hardware/DTB and kernel level), `nftables` 100% total-drop firewall, out-of-tree `nrf52840-hsm-kmod` kernel driver, mandatory first-login wizard with clock prompt, read-only root filesystem, USB mass storage automount, and dual-slot RAUC A/B update verification using x509 PKI certificates.
>
> **Release Status**: Scaffolding complete; development OS image (`hsm-image-dev`) verified with 5,844 BitBake tasks successfully completed; bootable WIC image and fast block map deployed; flashing instructions documented; root Makefile targets integrated; architecture documentation indexed in vault.

### 0. Milestone: Verified Yocto Development Build (5,844 Tasks Succeeded)

- **Target Image**: `hsm-image-dev` for `raspberrypi0-2w-64` (Raspberry Pi Zero 2 W 64-bit).
- **Build Execution**: Successfully executed `kas build kas-dev.yml` (via `make yocto-build-dev`): **5,844 tasks attempted and succeeded** with zero errors or task failures.
- **Deployed Artifacts** in [`rpi-z2w-yocto-hsm/build/deploy/images/raspberrypi0-2w-64/`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/build/deploy/images/raspberrypi0-2w-64):
  - `hsm-image-dev-raspberrypi0-2w-64.rootfs.wic.bz2` (**95 MB**): Raw bootable SD disk image with dual-slot A/B partitions and U-Boot.
  - `hsm-image-dev-raspberrypi0-2w-64.rootfs.wic.bmap` (**4.0 KB**): Block map file for accelerated flashing.
  - `hsm-image-dev-raspberrypi0-2w-64.rootfs.tar.bz2` (**73 MB**): Rootfs compressed tarball containing packages and `hsmutil`.
  - Kernel Device Tree Overlays & Boot Files: `disable-bt.dtbo` (1.0 KB), `disable-wifi.dtbo` (387 B), `dwc2.dtbo` (801 B), `bcm2710-rpi-zero-2.dtb` (33.7 KB), `boot.scr` (1.3 KB).
- **Flashing Procedures**:
  - Direct streaming flasher: `./scripts/flash-sd.sh build/deploy/images/raspberrypi0-2w-64/hsm-image-dev-raspberrypi0-2w-64.rootfs.wic.bz2 <target_device>`.
  - Fast block map copy: `sudo bmaptool copy --bmap build/deploy/images/raspberrypi0-2w-64/hsm-image-dev-raspberrypi0-2w-64.rootfs.wic.bmap build/deploy/images/raspberrypi0-2w-64/hsm-image-dev-raspberrypi0-2w-64.rootfs.wic <target_device>`.

### 1. Summary of Changes

| Subsystem | Branch | Commit | Key Additions |
| :--- | :--- | :--- | :--- |
| **Air-Gapped OS** (`rpi-z2w-yocto-hsm`) | `main` | [`91428c8`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm) | Fix U-Boot device tree handoff (`${fdt_addr}`), VideoCore `bootargs` extraction, and dual-slot WIC layout (`sdimage-dual-raspberrypi.wks.in`). Packaging conflicts resolved (`053bf75`, `cb4e8ca`). Complete Yocto 5.0 layer (`1ce95a1`). |
| **Workspace Root** (`nrf52840-hsm`) | `master` | Working Tree | Added `rpi-z2w-yocto-hsm` to `SUBREPOS` in root `Makefile`; added `yocto-build-dev`, `yocto-build-prod`, `yocto-bundle`, `yocto-shell`, and `yocto-clean` targets; updated root `AGENTS.md`. |
| **Obsidian Vault** (`nrf52840-hsm-obsidian`) | `master` | [`5f9834a`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-obsidian) | Authored `Architecture/07 - Air-Gapped RPi HSM OS Architecture.md`, `Hardware/Waveshare USB HAT (B) Integration.md`, `Development/Yocto Build and RAUC Update Guide.md`, updated `00 - Index.md`, `git-status.md`, and `Changelog.md`. |

---

### 2. Detailed Subsystem Changes

#### Air-Gapped OS Subsystem (`rpi-z2w-yocto-hsm`)
- **Yocto Layer & Architecture**:
  - Initialized custom layer `meta-hsm-os` compatible with Yocto 5.0 (`scarthgap`).
  - Integrated `kas` orchestration with reproducible container runtime (`ghcr.io/siemens/kas/kas:latest`).
  - Configured `hsm-image-base.bb` with `read-only-rootfs` and common utilities.
  - Implemented `hsm-image-prod.bb` with locked root superuser (`usermod -L root`) and `hsm-image-dev.bb` with root password `root`, sudo escalation, and debugging packages.
- **Kernel & Hardware Lockdown**:
  - Implemented kernel config fragment (`hardened.cfg`) stripping wireless (`CONFIG_CFG80211=n`, `CONFIG_BT=n`), Ethernet, USB net drivers, and user GPIO probing (`CONFIG_GPIO_SYSFS=n`).
  - Enforced kernel memory hardening: `CONFIG_SECURITY=y`, `CONFIG_HARDENED_USERCOPY=y`, `CONFIG_FORTIFY_SOURCE=y`, `CONFIG_STRICT_KERNEL_RWX=y`, `CONFIG_RANDOMIZE_BASE=y`.
  - Configured `/boot/config.txt` via `rpi-config_%.bbappend`: disabled WiFi/Bluetooth (`disable-wifi`, `disable-bt`), forced DWC2 host mode (`dtoverlay=dwc2,dr_mode=host`) for Waveshare USB HAT (B), and enabled HDMI console force hotplug (`hdmi_force_hotplug=1`, `hdmi_drive=2`).
- **Out-of-Tree Driver (`nrf52840-hsm-kmod`)**:
  - Dedicated character driver matching USB `0x1209:0x0001` and extracting device BLE address from USB serial number.
  - Blacklisted generic Linux ACM driver (`blacklist-cdc-acm.conf`) for exclusive dongle claiming.
  - In production mode: binds and exposes AT command channel (`cdc_acm_uart1`) under `/dev/tty.hsm<ble><num>` with symlink `/dev/tty.hsm` while suppressing logger port (`cdc_acm_uart0`).
  - In development mode: exposes both console and AT command ports.
  - Installed udev rules (`99-nrf52840-hsm.rules`) setting permissions to `0660` with group `dialout`.
- **Volatile Storage & Logging**:
  - Implemented `volatile-storage` service and `hsm-setup-volatile.sh`:
    - Mounts `tmpfs` on `/media` for unprivileged USB storage mount points.
    - Mounts `overlayfs` on `/etc` (`/run/overlay/etc-upper`), allowing runtime password updates in volatile RAM without altering immutable flash.
    - Mounts `tmpfs` on `/home/hsmutil` populated from `/etc/skel`.
  - Implemented `systemd-journald-airgap`:
    - Production: `Storage=none`, zero console/syslog/kmsg forwarding (no disk or console data leakage).
    - Development: `Storage=volatile` with 64 MB RAM buffer limit.
- **Network Security & User Space**:
  - Enforced `nftables` default drop across input, output, and forward chains.
  - Configured dedicated unprivileged user `hsmutil` in groups `users`, `dialout`, `tty`, `storage` with SHA-512 crypt password.
  - Implemented mandatory first-login wizard (`first-login-wizard.sh`) enforcing user password change, physical keyboard layout selection (`loadkeys`), and UTC timezone standard.
  - Implemented every-login UTC clock synchronization prompt (`clock-setup.sh`) backed by setuid root C helper (`hsm-set-date`), allowing unprivileged time synchronization without sudo.
  - Implemented USB storage automount service under `/media/usb-*` for `hsmutil` user space key exports and backups.
- **Dual-Slot RAUC A/B Updates**:
  - Integrated `meta-rauc` and `meta-rauc-raspberrypi` with U-Boot boot counter and watchdog fallback.
  - Implemented ECDSA P-256 PKI key generation script (`generate-rauc-keys.sh`) and offline bundle generator (`create-rauc-bundle.sh`).
  - Automated USB update installer service (`rauc-usb-autoinstall`): detects `*.raucb` bundles on USB insertion via `usb-mount.sh` and verifies x509 CMS signatures before streaming into the inactive slot.

---

### 3. Investigation & Remediation: First-Boot Device Tree Failure (`91428c8`)

- **Root Cause Assessment**: [[investigations/01 - RPi Zero 2 W First Boot Device Tree Failure|01 - RPi Zero 2 W First Boot Device Tree Failure]] (Session `7727fe1d-b544-42ae-848b-ab6b2767f731`).
- **Defects Remediated**:
  1. **Device Tree Handoff (`boot.cmd.in`)**: Eliminated erroneous `load mmc 0:1 ${fdt_addr_r} booti.dtb` which halted with error `-14`. Passed firmware-prepared device tree pointer at `${fdt_addr}` directly to `booti ${kernel_addr_r} - ${fdt_addr}`, preserving VideoCore hardware overlays (`disable-bt`, `disable-wifi`, `dwc2`) and dynamic board configuration.
  2. **VideoCore Bootargs Extraction**: Added `fdt addr ${fdt_addr} && fdt get value bootargs /chosen bootargs` to initialize serial console and kernel parameters from `cmdline.txt`.
  3. **Dual-Slot Kernel Isolation**: Updated RAUC boot logic to load `/boot/Image` directly from the active slot (`mmc 0:2` for Slot A, `mmc 0:3` for Slot B) with fallback to `mmc 0:1`, ensuring atomic kernel updates and rollbacks.
  4. **Dual-Slot WIC Partitioning**: Set `WKS_FILE = "sdimage-dual-raspberrypi.wks.in"` in `kas-project.yml` ensuring Partition 1 (`boot`), Partition 2 (`rootfs_A`), and Partition 3 (`rootfs_B`) are provisioned in `.wic` images.

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
