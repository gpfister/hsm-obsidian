---
title: nRF52840 HSM Dongle Knowledge Vault
tags:
  - hsm
  - nrf52840
  - zephyr
  - security
  - index
created: 2026-10-29
---

# nRF52840 HSM Dongle Documentation Vault

Welcome to the central engineering documentation vault for the **nRF52840 Hardware Security Module (HSM) USB Dongle** project (`ncs-nrf52840dongle-kms`).

---

## 🧭 Vault Navigation

### 🏗️ Architecture
- [[Architecture/System Architecture|System Architecture]] — Multi-repository layout, system blocks, and runtime interactions.
- [[Architecture/07 - Air-Gapped RPi HSM OS Architecture|Air-Gapped RPi HSM OS Architecture]] — Ultra-hardened embedded Linux appliance, physical network isolation, out-of-tree kernel driver, volatile RAM overlays, and dual-slot RAUC A/B updates.
- [[Architecture/Dual-Stage Secure Boot|Dual-Stage Secure Boot]] — B0 immutable bootloader, MCUboot, and ECDSA P-256 signature chain.
- [[Architecture/USB Composite Stack|USB Composite Stack]] — Dual CDC-ACM Virtual COM Ports (Console vs HSM API).
- [[Architecture/HSM State Machine|HSM State Machine]] — State transitions and RGB LED indicators.

#### 📋 Architecture Decision Records (ADRs)
- [[Architecture/Decisions/ADR-0001 - Hardware BLE Address for Deterministic USB Serial Number|ADR-0001: Hardware BLE Address for USB Serial Number]] — Deterministic macOS TTY synthesis.
- [[Architecture/Decisions/ADR-0002 - In-Tree Protocol Descriptor Patching vs macOS DriverKit Extension|ADR-0002: In-Tree Protocol Descriptor Patching vs DriverKit]] — Zero-driver CDC-ACM interface tagging.
- [[Architecture/Decisions/ADR-0003 - Host Port Discovery and Serial Targeting Strategy|ADR-0003: Host Port Discovery and Serial Targeting Strategy]] — Multi-tiered resolution hierarchy for `hsmutil` and PKCS#11.
- [[Architecture/Decisions/ADR-0004 - Factory Reset Semantics and Safeguards|ADR-0004: Factory Reset Semantics and Safe Erasure Safeguards]] — Irreversible cryptographic teardown, NVS clearing, and interactive CLI safeguards.
- [[Architecture/Decisions/ADR-0005 - Custom BitBake img Image Type via WIC Rename|ADR-0005: Custom BitBake img Image Type via WIC Rename]] — Race-free in-place `.wic` to `.img` rename via `IMAGE_TYPEDEP:img = "wic"`.
- [[Architecture/Decisions/ADR-0006 - AT-Protocol DFU Chunk Streaming and Deferred MCUboot Swap|ADR-0006: AT-Protocol DFU Chunk Streaming & Deferred MCUboot Swap Architecture]] — Callback-decoupled 256-byte hex chunk streaming (`AT+DFU_*`), `k_work_delayable` deferred reboot, and startup image confirmation.
- [[Architecture/Decisions/ADR-0007 - Decompose Poky into openembedded-core meta-yocto and bitbake|ADR-0007: Decompose Poky into openembedded-core, meta-yocto and bitbake]] — Yocto 6.1 layer topology.
- [[Architecture/Decisions/ADR-0008 - Internalize RPi RAUC Integration into meta-hsm-os|ADR-0008: Internalize RPi RAUC Integration into meta-hsm-os]] — Drop meta-rauc-community.
- [[Architecture/Decisions/ADR-0009 - Kernel Driver llseek Version Guard and UNPACKDIR Migration|ADR-0009: Kernel llseek Guard & UNPACKDIR Migration]] — Linux 6.12+ and Yocto 6.1 recipe conventions.
- [[Architecture/Decisions/ADR-0010 - Kernel Boot Console Multiplexing and Command Line Specialization|ADR-0010: Kernel Boot Console Multiplexing & Command Line Specialization]] — Dev dual-console broadcast vs prod silence via `rpi-cmdline.bbappend`.
- [[Architecture/Decisions/ADR-0011 - Silent Non-Interactive U-Boot Bootloader Hardening|ADR-0011: Silent Non-Interactive U-Boot Bootloader Hardening]] — Production non-interactive boot (`CONFIG_BOOTDELAY=-2`), silent console, and modular Kconfig fragments.
- [[Architecture/Decisions/ADR-0012 - PAM pwquality Password Complexity and Expiration Policy|ADR-0012: PAM pwquality Password Complexity & Expiration Policy]] — Dual-layer defense via `pam_pwquality.so`, 90-day password aging, and wizard validation.
- [[Architecture/Decisions/ADR-0013 - Standard CDC-ACM AT Protocol Descriptors for macOS Dual Enumeration|ADR-0013: Standard CDC-ACM AT Protocol Descriptors for macOS Dual Enumeration]] — Standardization of Interface 2 and IAD protocol to `0x01` (ITU-T V.250) restoring dual CDC-ACM device enumeration on macOS.

### 🧩 Feature Specifications
- [[features/fix-macos-cdc-acm-enumeration|Fix macOS CDC-ACM Enumeration for Interface 2]] — Restoration of dual `/dev/cu.usbmodem*` enumeration on Darwin by aligning protocol codes with `AppleUSBACMControl`.
- [[features/at-protocol-dfu-support|AT-Protocol DFU Support]] — End-to-end Device Firmware Upgrade (`AT+DFU_INFO?`, `AT+DFU_CHUNK`, `AT+DFU_APPLY`) across Zephyr firmware, Go CLI (`hsmutil`), and mock HSM PTY.
- [[features/uncompressed-img-disk-image-output|Uncompressed Disk Image .img Output]] — Custom BitBake `img` image type (`IMAGE_TYPEDEP:img = "wic"`), `.img` output retention, and SD flash tooling alignment.
- [[features/uncompressed-wic-image-and-sd-flash-tooling|Uncompressed WIC Image Generation & SD Flash Tooling Alignment]] — Uncompressed `.wic` artifact retention in Kas/BitBake, deploy directory path alignment, and cross-platform macOS/Linux `dd` support in `flash-sd.sh`.
- [[features/usb-descriptors-ble-address|USB Descriptors & Device Identification]] — Deterministic hardware BLE serial numbers, macOS CDC-ACM port allocation, and multi-device discovery.
- [[features/hsmutil-reset-and-multi-device-disambiguation|Reset Confirmation & Multi-Device Disambiguation]] — Interactive confirmation safeguards, firmware CC310 zeroization, and deterministic multi-dongle CLI targeting.
- [[features/yocto-boot-visibility-and-password-policy|Yocto Boot Visibility & Password Policy]] — Dev/prod boot log multiplexing, silent U-Boot hardening, default password `password`, PAM pwquality complexity, and 90-day expiration.

### 📐 Specifications
- [[Specification/Fix macOS CDC-ACM Enumeration for Interface 2|Fix macOS CDC-ACM Enumeration for Interface 2]] — Technical specification for descriptor patch routines in `usb_serial.c`.
- [[Specification/AT-Protocol DFU Support|AT-Protocol DFU Support]] — Technical specification for `AT+DFU_INFO?`, `AT+DFU_CHUNK`, and `AT+DFU_APPLY`, MCUboot slot confirmation, and `ATClient` integration.
- [[Specification/Uncompressed Disk Image img Output in rpi-z2w-yocto-hsm|Uncompressed Disk Image .img Output in rpi-z2w-yocto-hsm]] — Technical specification for replacing uncompressed `.wic` output with `.img` via `IMAGE_CMD:img`.
- [[Specification/Uncompressed WIC Image Generation and SD Flash Tooling Alignment|Uncompressed WIC Image Generation & SD Flash Tooling Alignment]] — Technical specification for `IMAGE_FSTYPES` `.wic` emission and cross-platform SD card flashing.
- [[Specification/Reset Confirmation and Multi-Device Disambiguation|Reset Confirmation & Multi-Device Disambiguation]] — Interactive confirmation safeguards, CC310 zeroization, and deterministic multi-device auto-discovery.
- [[Specification/Yocto 6.1 Blacksail Upgrade|Yocto 6.1 (Blacksail) Upgrade]] — Upgrade of `rpi-z2w-yocto-hsm` from Scarthgap: layer decomposition, RAUC internalization, UNPACKDIR, kernel 6.12+.
- [[Specification/Yocto Dev-Prod Boot Visibility, U-Boot Hardening, and Password Policy|Yocto Dev-Prod Boot Visibility, U-Boot Hardening, and Password Policy]] — Technical specification for dev/prod boot visibility, U-Boot silence, password complexity, and expiration policy.
- [[Specification/USB Descriptors and Device Identification|USB Descriptors & Device Identification]] — FICR BLE address serial number, protocol IDs, and macOS device node mapping.
- [[Specification/YubiHSM Protocol|YubiHSM Protocol]] — Binary packet framing, command opcodes, and response payloads.
- [[Specification/Firmware DFU Protocol|Firmware DFU Protocol]] — Streaming format for `hsmutil firmware update`.

### 🔌 Hardware
- [[Hardware/nRF52840 Dongle PCA10059|nRF52840 Dongle PCA10059]] — Pinout, RGB LEDs, buttons, and crystal oscillators.
- [[Hardware/Waveshare USB HAT (B) Integration|Waveshare USB HAT (B) Integration]] — 4-port USB 2.0 expansion hub, DWC2 host overlay, power budgeting, and peripheral port allocation.
- [[Hardware/JLink Debug Header|JLink Debug Header]] — SWD connector pinout and Segger J-Link integration.

### 💻 Development & Workflows
- [[Development/Environment Setup|Environment Setup]] — Toolchain prerequisites (NCS v3.4.1, UV, Go 1.24+).
- [[Development/Build and Flash|Build and Flash]] — Firmware compilation, sysbuild, and J-Link programming.
- [[Development/CLI Reference|CLI Reference]] — Complete documentation for `hsmutil` CLI and BubbleTea TUI.
- [[Development/Yocto Build and RAUC Update Guide|Yocto Build & RAUC Update Guide]] — Kas containerized builds, SD card flashing, RAUC x509 PKI bundles, and offline update procedures.

### 📜 Sessions & Transcripts
- [[Sessions/4b1bb45a-82b0-4966-bacf-254b577ad6f2|2026-10-31 - Reset Confirmation & Multi-Device Disambiguation (Phase 3 Planning)]] — Approved multi-repository implementation plan, architectural blueprint, and task breakdown.
- [[Sessions/36b74c4b-dd01-44e1-9405-48fdc8d21d87|2026-10-31 - Reset Confirmation & Multi-Device Disambiguation (Phase 2 Specification)]] — Approved technical specification, zeroization flow, and multi-device disambiguation protocol.
- [[Sessions/9f9ae0e3-f181-44f3-8dde-7b25c55cf4e3|2026-10-31 - USB Descriptors & Device Identification (Phases 3-5 Lifecycle)]] — Implementation, hardware validation, and pre-merge documentation tidy-up.
- [[Sessions/bd3d2f5d-cc3c-4513-97fd-6b35e8fb0ddc|2026-10-31 - USB Descriptors & Device Identification (Phase 2 Specification)]] — Technical specification, architectural mapping, and verification gates.
- [[Sessions/2026-10-29 - Multi-Repo Initialization|2026-10-29 - Multi-Repo Initialization]] — Genesis session establishing the multirepo workspace.

### 📊 Monitoring & Telemetry
- [[Monitoring/2026-11-05 - Production Flashing, DFU Verification, and Factory Reset - Second Dongle|Production Flashing, DFU & Reset (Second Dongle)]] — J-Link SWD flashing (S/N: 802007333, Unit: D74B8A9CAB28), macOS dual CDC-ACM enumeration (ADR-0013 validation), device info, auth, DFU chunk streaming upgrade, and factory reset verification.
- [[Monitoring/2026-11-05 - Production Flashing and USB CDC-ACM Interface Anomaly|Production Flashing & USB CDC-ACM Anomaly]] — J-Link SWD flashing (S/N: 802007333), first-boot verification, stream anomaly alert (Interface 2 rejection by macOS AppleUSBACMControl via proto 0x02).
- [[Monitoring/2026-10-31 - Production Firmware Flashing and First Boot Telemetry|Production Firmware Flashing & First Boot Telemetry]] — J-Link SWD flashing (S/N: 802007333), first-boot verification, CC310 crypto initialization, and dual CDC-ACM enumeration.
- [[Monitoring/2026-10-30 - Hardware Serial Console Monitoring|Hardware Serial Console Monitoring (PCA10059)]] — Stream surveillance for `/dev/cu.usbmodem83101` (115200 baud), boot sequence, PSA crypto, and health verification.

### 🔍 Investigations & Issue Assessments
- [[investigations/2026-11-05 - BitBake scdoc-native do_fetch Failure|2026-11-05 - BitBake scdoc-native do_fetch Failure]] — Root cause analysis and resolution for BitBake Git fetch failure caused by interrupted clone leaving `refs/heads/.invalid` in bare clone `HEAD`, plus automated sanity guard.
- [[investigations/01 - RPi Zero 2 W First Boot Device Tree Failure|01 - RPi Zero 2 W First Boot Device Tree Failure]] — Root cause analysis and remediation for U-Boot device tree handoff (`${fdt_addr}`), VideoCore `bootargs` extraction, and dual-slot WIC partition layout.
- [[investigations/2026-10-31-hsmutil-reset-and-multi-device-disambiguation|2026-10-31 - hsmutil Reset Confirmation & Multi-Device Disambiguation]] — Root-cause analysis and remediation architecture for factory reset erasure semantics and multi-dongle auto-discovery disambiguation.

### 🌿 Repository, Releases & Status
- [[Changelog|Workspace Release Changelog]] — Formal releases, milestone deliveries, and post-merge changelog notes.
- [[git-status|Workspace Git Status]] — Multi-repository branch topologies, commits, active feature tracking, and worktrees.

### 🎯 Roadmap & Planning
- [[Plan/Project Roadmap|Project Roadmap]] — Milestones, future cryptographic extensions, and security certification targets.
- [[Plan/Fix macOS CDC-ACM Enumeration for Interface 2 Implementation Plan|Implementation Plan: Fix macOS CDC-ACM Enumeration for Interface 2]] — Multi-repository implementation plan, descriptor patching, J-Link flashing, and `hsmutil` verification.
- [[Plan/AT-Protocol DFU Support Implementation Plan|Implementation Plan: AT-Protocol DFU Support]] — Multi-repository implementation plan across `nrf52840-hsm-firmware`, `nrf52840-hsm-cli`, and `ncs-nrf52840dongle-kms` (`scripts/mock_hsm_pty.py`).
- [[Plan/Uncompressed Disk Image img Output Implementation Plan|Implementation Plan: Uncompressed Disk Image .img Output]] — Repository work breakdown and BitBake `IMAGE_CMD:img` pipeline plan for `rpi-z2w-yocto-hsm`.
- [[Plan/Uncompressed WIC Image Generation and SD Flash Tooling Implementation Plan|Implementation Plan: Uncompressed WIC Image Generation & SD Flash Tooling Alignment]] — Repository work breakdown and build verification plan for `rpi-z2w-yocto-hsm`.
- [[Plan/Reset Confirmation and Multi-Device Disambiguation Implementation Plan|Implementation Plan: Reset Confirmation & Multi-Device Disambiguation]] — Multi-repository implementation plan across firmware keystore zeroization, CLI confirmation safeguards, and serial disambiguation.
- [[Plan/USB Descriptors and BLE Identification Implementation Plan|Implementation Plan: USB Descriptors & BLE Device Identification]] — Multi-repository implementation plan across firmware, CLI, PKCS#11, and documentation.
- [[Plan/Yocto 6.1 Blacksail Upgrade Implementation Plan|Implementation Plan: Yocto 6.1 (Blacksail) Upgrade]] — Work breakdown, deployment verification and risks for `rpi-z2w-yocto-hsm`.
- [[Plan/Yocto Dev-Prod Boot Visibility, U-Boot Hardening, and Password Policy Implementation Plan|Implementation Plan: Yocto Dev-Prod Boot Visibility, U-Boot Hardening, and Password Policy]] — Repository work breakdown, Kconfig fragments, PAM configuration, and test matrix.
