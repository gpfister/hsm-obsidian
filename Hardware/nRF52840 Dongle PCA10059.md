---
title: nRF52840 Dongle PCA10059 Hardware Specification
tags:
  - hardware
  - nrf52840
  - pca10059
created: 2026-10-29
---

# nRF52840 Dongle (PCA10059) Hardware Reference

The **Nordic nRF52840 Dongle** (`PCA10059`) is a compact USB form-factor development platform featuring an ARM Cortex-M4 with FPU running at 64 MHz, 1 MB Flash, 256 KB RAM, and an onboard ARM CryptoCell 310 hardware crypto accelerator.

---

## 1. Onboard LEDs & GPIO Pinout

The dongle contains two LEDs:
1. `LED1`: Multi-color RGB LED.
2. `LED2`: Single green LED (`P0.06`).

| Color | GPIO Pin | Active Level | DTS Label | Function |
| :--- | :--- | :--- | :--- | :--- |
| **Red** | `P0.08` | Active Low | `led1_red` / `red_pwm_led` | Configured & Locked |
| **Green** | `P1.09` | Active Low | `led1_green` / `green_pwm_led` | Authenticated & Unlocked |
| **Blue** | `P0.12` | Active Low | `led1_blue` / `blue_pwm_led` | Pulsating when Unconfigured |

---

## 2. J-Link SWD Debug Connection

The dongle has SWD solder pads on the underside:
- `P1`: VDD
- `P2`: SWDIO
- `P3`: SWDCLK
- `P4`: GND

Connected to Segger J-Link (Serial: `802007333`) for flashing with `nrfutil device program` or `west flash`.
