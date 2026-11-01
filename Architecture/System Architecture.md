---
title: System Architecture
tags:
  - architecture
  - multi-repo
  - system-design
created: 2026-10-29
---

# System Architecture

The **ncs-nrf52840dongle-kms** workspace implements a Hardware Security Module (HSM) on an **nRF52840 Dongle** (`PCA10059`), communicating via a dual CDC-ACM USB interface and controlled by the cross-platform CLI `hsmutil`.

```mermaid
graph TD
    subgraph Host ["Host Workstation (Mac / Linux / Windows)"]
        CLI["hsmutil (Go CLI & BubbleTea TUI)"]
        CONSOLE["Serial Terminal / Monitor"]
    end

    subgraph USB ["USB Composite Device"]
        CDC0["Port 0: CDC-ACM UART (Console / Logs)"]
        CDC1["Port 1: CDC-ACM UART (YubiHSM 2 API)"]
    end

    subgraph Dongle ["nRF52840 Dongle (PCA10059)"]
        subgraph Boot ["Dual-Stage Secure Boot"]
            B0["B0 Immutable Bootloader\n(CC310 Hardware SHA256 / ECDSA)"]
            MCUBOOT["MCUboot Upgradable Stage"]
        end

        subgraph Firmware ["Firmware Application (firmware/app)"]
            CORE["HSM Core Logic Engine (firmware/lib)"]
            DFU["DFU Service (Flash Map Slot 1)"]
            LED["RGB LED Driver (PWM / GPIO)"]
        end
    end

    CONSOLE <--> CDC0
    CLI <--> CDC1
    CDC0 <--> Firmware
    CDC1 <--> CORE
    CORE --> LED
    CORE --> DFU
```

---

## Workspace Multi-Repo Structure

The workspace coordinates five orchestrated sub-repositories:

1. **`nrf52840-hsm-firmware`**: NCS/Zephyr workspace repository for the dongle firmware, including dual-stage secure boot configurations and pure HSM logic.
2. **`nrf52840-hsm-cli`**: Cross-platform Go repository delivering the `hsmutil` CLI and BubbleTea TUI for macOS, Linux, and Windows.
3. **`nrf52840-hsm-libpkcs11`**: Cryptoki PKCS#11 v2.40 shared library provider (`libnrf52840_pkcs11.dylib` / `.so` / `.dll`).
4. **`nrf52840-hsm-obsidian`**: Centralized Obsidian documentation vault tracking specifications, architecture, and sessions.
5. **`rpi-z2w-yocto-hsm`**: Ultra-hardened, air-gapped 64-bit embedded Linux operating system built with Yocto Project 5.0 LTS (`scarthgap`) for Raspberry Pi hardware appliances and dual-slot RAUC A/B updates.
