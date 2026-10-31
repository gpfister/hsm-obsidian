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
- [[Architecture/Dual-Stage Secure Boot|Dual-Stage Secure Boot]] — B0 immutable bootloader, MCUboot, and ECDSA P-256 signature chain.
- [[Architecture/USB Composite Stack|USB Composite Stack]] — Dual CDC-ACM Virtual COM Ports (Console vs HSM API).
- [[Architecture/HSM State Machine|HSM State Machine]] — State transitions and RGB LED indicators.

#### 📋 Architecture Decision Records (ADRs)
- [[Architecture/Decisions/ADR-0001 - Hardware BLE Address for Deterministic USB Serial Number|ADR-0001: Hardware BLE Address for USB Serial Number]] — Deterministic macOS TTY synthesis.
- [[Architecture/Decisions/ADR-0002 - In-Tree Protocol Descriptor Patching vs macOS DriverKit Extension|ADR-0002: In-Tree Protocol Descriptor Patching vs DriverKit]] — Zero-driver CDC-ACM interface tagging.
- [[Architecture/Decisions/ADR-0003 - Host Port Discovery and Serial Targeting Strategy|ADR-0003: Host Port Discovery and Serial Targeting Strategy]] — Multi-tiered resolution hierarchy for `hsmutil` and PKCS#11.
- [[Architecture/Decisions/ADR-0004 - Factory Reset Semantics and Safeguards|ADR-0004: Factory Reset Semantics and Safe Erasure Safeguards]] — Irreversible cryptographic teardown, NVS clearing, and interactive CLI safeguards.

### 🧩 Feature Specifications
- [[features/usb-descriptors-ble-address|USB Descriptors & Device Identification]] — Deterministic hardware BLE serial numbers, macOS CDC-ACM port allocation, and multi-device discovery.
- [[features/hsmutil-reset-and-multi-device-disambiguation|Reset Confirmation & Multi-Device Disambiguation]] — Interactive confirmation safeguards, firmware CC310 zeroization, and deterministic multi-dongle CLI targeting.

### 📐 Specifications
- [[Specification/USB Descriptors and Device Identification|USB Descriptors & Device Identification]] — FICR BLE address serial number, protocol IDs, and macOS device node mapping.
- [[Specification/YubiHSM Protocol|YubiHSM Protocol]] — Binary packet framing, command opcodes, and response payloads.
- [[Specification/Firmware DFU Protocol|Firmware DFU Protocol]] — Streaming format for `hsmutil firmware update`.

### 🔌 Hardware
- [[Hardware/nRF52840 Dongle PCA10059|nRF52840 Dongle PCA10059]] — Pinout, RGB LEDs, buttons, and crystal oscillators.
- [[Hardware/JLink Debug Header|JLink Debug Header]] — SWD connector pinout and Segger J-Link integration.

### 💻 Development & Workflows
- [[Development/Environment Setup|Environment Setup]] — Toolchain prerequisites (NCS v3.4.1, UV, Go 1.24+).
- [[Development/Build and Flash|Build and Flash]] — Firmware compilation, sysbuild, and J-Link programming.
- [[Development/CLI Reference|CLI Reference]] — Complete documentation for `hsmutil` CLI and BubbleTea TUI.

### 📜 Sessions & Transcripts
- [[Sessions/9f9ae0e3-f181-44f3-8dde-7b25c55cf4e3|2026-10-31 - USB Descriptors & Device Identification (Phases 3-5 Lifecycle)]] — Implementation, hardware validation, and pre-merge documentation tidy-up.
- [[Sessions/bd3d2f5d-cc3c-4513-97fd-6b35e8fb0ddc|2026-10-31 - USB Descriptors & Device Identification (Phase 2 Specification)]] — Technical specification, architectural mapping, and verification gates.
- [[Sessions/2026-10-29 - Multi-Repo Initialization|2026-10-29 - Multi-Repo Initialization]] — Genesis session establishing the multirepo workspace.

### 📊 Monitoring & Telemetry
- [[Monitoring/2026-10-31 - Production Firmware Flashing and First Boot Telemetry|Production Firmware Flashing & First Boot Telemetry]] — J-Link SWD flashing (S/N: 802007333), first-boot verification, CC310 crypto initialization, and dual CDC-ACM enumeration.
- [[Monitoring/2026-10-30 - Hardware Serial Console Monitoring|Hardware Serial Console Monitoring (PCA10059)]] — Stream surveillance for `/dev/cu.usbmodem83101` (115200 baud), boot sequence, PSA crypto, and health verification.

### 🔍 Investigations & Issue Assessments
- [[investigations/2026-10-31-hsmutil-reset-and-multi-device-disambiguation|2026-10-31 - hsmutil Reset Confirmation & Multi-Device Disambiguation]] — Root-cause analysis and remediation architecture for factory reset erasure semantics and multi-dongle auto-discovery disambiguation.

### 🌿 Repository, Releases & Status
- [[Changelog|Workspace Release Changelog]] — Formal releases, milestone deliveries, and post-merge changelog notes.
- [[git-status|Workspace Git Status]] — Multi-repository branch topologies, commits, active feature tracking, and worktrees.

### 🎯 Roadmap & Planning
- [[Plan/Project Roadmap|Project Roadmap]] — Milestones, future cryptographic extensions, and security certification targets.
- [[Plan/USB Descriptors and BLE Identification Implementation Plan|Implementation Plan: USB Descriptors & BLE Device Identification]] — Multi-repository implementation plan across firmware, CLI, PKCS#11, and documentation.
