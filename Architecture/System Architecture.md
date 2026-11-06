---
title: System Architecture
tags:
  - architecture
  - multi-repo
  - system-design
created: 2026-10-29
---

# System Architecture

The **ncs-nrf52840dongle-kms** workspace implements a Hardware Security Module (HSM) ecosystem with firmware currently targeting an **nRF52840 Dongle** (`PCA10059`), communicating via a dual CDC-ACM USB interface and controlled by the cross-platform CLI `gphsm` (with `hsmutil` backward-compatible symlinks) or standard Cryptoki PKCS#11 tooling via `libgphsm` (see [[Architecture/Decisions/ADR-0014 - Documentation Terminology Generalization to Generic HSM USB Key|ADR-0014]], [[Architecture/Decisions/ADR-0016 - Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)|ADR-0016]], [[Architecture/Decisions/ADR-0017 - Host CLI Rebranding to gphsm and PKCS11 Refactoring to libgphsm|ADR-0017]], and [[Architecture/Decisions/ADR-0018 - Sub-Repository Renaming to hsm-pkcs11 with libgphsm ABI Invariance|ADR-0018]]).

```mermaid
graph TD
    subgraph Host ["Host Workstation (Mac / Linux / Windows) / Offgrid OS"]
        CLI["gphsm (Go CLI & Bubble Tea UX)"]
        PKCS11["libgphsm (OpenSC / OpenSSL)"]
        CONSOLE["Serial Terminal / Monitor"]
    end

    subgraph USB ["USB Composite Device"]
        CDC0["Port 0: CDC-ACM UART (Console / Logs)"]
        CDC1["Port 1: CDC-ACM UART (AT Protocol Channel)"]
    end

    subgraph Dongle ["HSM USB Key / Dongle (nRF52840 PCA10059)"]
        subgraph Boot ["Dual-Stage Secure Boot"]
            B0["B0 Immutable Bootloader\n(CC310 Hardware SHA256 / ECDSA)"]
            MCUBOOT["MCUboot Upgradable Stage"]
        end

        subgraph Firmware ["Firmware Application (hsm-ncs-firmware)"]
            CORE["HSM Core Logic Engine (firmware/lib)\nDual-Mode Auth (PIN / Shamir)"]
            DFU["DFU Service (Flash Map Slot 1)"]
            LED["RGB LED Driver (PWM / GPIO)"]
        end
    end

    CONSOLE <--> CDC0
    CLI <--> CDC1
    PKCS11 <--> CDC1
    CDC0 <--> Firmware
    CDC1 <--> CORE
    CORE --> LED
    CORE --> DFU
```

---

## Workspace Multi-Repo Structure

The workspace coordinates five orchestrated sub-repositories:

1. **[`hsm-ncs-firmware`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware)**: NCS v3.4.1 / Zephyr RTOS workspace repository for the dongle firmware (currently exclusively supporting the nRF52840 USB Dongle `PCA10059`), including dual-stage secure boot configurations, dual-mode authentication (`auth_method_t`), and pure HSM logic.
2. **[`hsm-cli`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli)**: Cross-platform Go repository delivering the `gphsm` CLI (with `hsmutil` symlinks) and Bubble Tea UX for administering the HSM USB key on macOS, Linux, and Windows with `pam_pwquality` PIN validation.
3. **[`hsm-pkcs11`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-pkcs11)** (formerly `libgphsm` / `hsm-libpkcs11`): Cryptoki PKCS#11 v2.40 shared library provider (`libgphsm.dylib` / `.so` / `.dll`) allowing utilities like `pkcs11-tool` to interact with the HSM key with dual-mode auth support while preserving strict ABI compatibility.
4. **[`hsm-obsidian`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian)**: Centralized Obsidian documentation vault tracking specifications, architecture, and sessions.
5. **[`hsm-yocto-offgrid-rpi`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi)**: Ultra-hardened, air-gapped 64-bit embedded Linux operating system built with Yocto Project 6.1 (`blacksail`) for performing sensitive cryptographic operations (e.g. offline Root CA management) with the HSM key.
