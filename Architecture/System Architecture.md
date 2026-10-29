---
title: System Architecture
tags:
  - architecture
  - multi-repo
  - system-design
created: 2026-10-29
---

# System Architecture

The **ncs-nrf52840dongle-kms** workspace implements a Hardware Security Module (HSM) on an **nRF52840 Dongle** (`PCA10059`), communicating via a dual CDC-ACM USB interface and controlled by the cross-platform CLI `hmsutil`.

```mermaid
graph TD
    subgraph Host ["Host Workstation (Mac / Linux / Windows)"]
        CLI["hmsutil (Go CLI & BubbleTea TUI)"]
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

The workspace coordinates three independent sub-repositories:

1. **`nrf52840-hms-firmware`**: NCS/Zephyr workspace repository for the dongle firmware, including dual-stage secure boot configurations and pure HSM logic.
2. **`nrf52840-hms-cli`**: Go repository delivering the `hmsutil` binary for macOS, Linux, and Windows.
3. **`nrf52840-hms-obsidian`**: Obsidian documentation vault tracking specifications, architecture, and sessions.
