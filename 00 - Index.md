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
- [[Sessions/2026-10-29 - Multi-Repo Initialization|2026-10-29 - Multi-Repo Initialization]] — Genesis session establishing the multirepo workspace.

### 📊 Monitoring & Telemetry
- [[Monitoring/2026-10-31 - Production Firmware Flashing and First Boot Telemetry|Production Firmware Flashing & First Boot Telemetry]] — J-Link SWD flashing (S/N: 802007333), first-boot verification, CC310 crypto initialization, and dual CDC-ACM enumeration.
- [[Monitoring/2026-10-30 - Hardware Serial Console Monitoring|Hardware Serial Console Monitoring (PCA10059)]] — Stream surveillance for `/dev/cu.usbmodem83101` (115200 baud), boot sequence, PSA crypto, and health verification.

### 🎯 Roadmap & Planning
- [[Plan/Project Roadmap|Project Roadmap]] — Milestones, future cryptographic extensions, and security certification targets.
