---
title: "ADR-0011: Silent Non-Interactive U-Boot Bootloader Hardening"
status: Accepted
date: 2026-11-05
tags:
  - adr
  - architecture
  - yocto
  - u-boot
  - hardening
  - bootloader
  - security
deciders:
  - Embedded Systems Architect
  - Security Firmware Lead
references:
  - "[[features/yocto-boot-visibility-and-password-policy|Feature: Yocto Boot Visibility & Password Policy]]"
  - "[[Specification/Yocto Dev-Prod Boot Visibility, U-Boot Hardening, and Password Policy|Specification: Yocto Dev-Prod Boot Visibility, U-Boot Hardening, and Password Policy]]"
  - "[[Plan/Yocto Dev-Prod Boot Visibility, U-Boot Hardening, and Password Policy Implementation Plan|Implementation Plan: Yocto Dev-Prod Boot Visibility, U-Boot Hardening, and Password Policy]]"
  - "[[Architecture/07 - Air-Gapped RPi HSM OS Architecture|Architecture: Air-Gapped RPi HSM OS]]"
  - "[[Architecture/Decisions/ADR-0010 - Kernel Boot Console Multiplexing and Command Line Specialization|ADR-0010: Kernel Boot Console Multiplexing]]"
---

# ADR-0011: Silent Non-Interactive U-Boot Bootloader Hardening

## Context & Problem Statement

The **`rpi-z2w-yocto-hsm`** operating system uses **Das U-Boot** to manage dual-slot A/B partition booting and integration with the RAUC update controller.

Two major architectural deficiencies were observed in the bootloader subsystem:

1. **Production Boot Interactivity & Information Exposure**:
   - In production builds (`HSM_DEV_MODE = "0"`), U-Boot printed startup banners, CPU clock speeds, DRAM size, and partition tables to the UART console.
   - It printed the autoboot countdown (`Hit any key to stop autoboot: 2`) and paused for keyboard/serial break sequences. A physical attacker or compromised peripheral with serial access could interrupt boot and execute arbitrary commands in U-Boot shell (`printenv`, `md`, `mw`, `bootm`), bypassing OS security controls.
   - U-Boot bootscripts ([`boot.cmd.in`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-hsm-os/recipes-bsp/rpi-u-boot-scr/files/boot.cmd.in)) printed verbose RAUC diagnostic messages (`RAUC: Booting slot A...`).
2. **Defconfig Patch Fragility across Raspberry Pi Targets**:
   - `meta-rauc-raspberrypi` previously patched `configs/rpi_arm64_defconfig` via a static diff (`rpi_arm64_defconfig.patch`).
   - When building for boards using specific defconfigs (e.g. `rpi_0_w_defconfig`, `rpi_3_defconfig`) or when U-Boot source versions differed, the patch failed to apply or missed enabling SquashFS support (`CONFIG_CMD_SQUASHFS=y`, `CONFIG_FS_SQUASHFS=y`).

---

## Decision Drivers

- **Zero Physical Interactivity in Production**: The bootloader must boot immediately into the verified kernel without an interactive countdown window or escape sequences.
- **Console Silence**: In production, U-Boot must emit zero characters on UART and HDMI to prevent timing attacks and side-channel leakage.
- **Developer Observability**: Development builds (`HSM_DEV_MODE = "1"`) must preserve the interactive console and diagnostic RAUC output.
- **Portability across RPi Boards**: The configuration mechanism must work seamlessly on `raspberrypi0-2w-64`, `raspberrypi3-64`, `raspberrypi4-64`, and `raspberrypi5` without maintaining separate defconfig patch files.

---

## Considered Options

### Option 1: Static Patching of Board Defconfigs
- *Mechanism*: Maintain separate `.patch` files for each board's defconfig in `meta-hsm-os`.
- *Evaluation*: Highly brittle, prone to merge conflicts during Yocto/U-Boot version bumps.

### Option 2: Environment Variable Overrides via U-Boot Environment File (`uboot.env`)
- *Mechanism*: Burn pre-compiled U-Boot environment with `bootdelay=-2` and `silent=1`.
- *Evaluation*: Fails to guarantee security because U-Boot prints initial banner lines before reading environment variables, and `CONFIG_BOOTDELAY` is compiled into the binary.

### Option 3: Modular Kconfig Fragments & Dynamic Bootscript Generation (Chosen)
- *Mechanism*:
  1. Replace defconfig patches with a generic Kconfig fragment `squashfs.cfg`:
     ```text
     CONFIG_CMD_SQUASHFS=y
     CONFIG_FS_SQUASHFS=y
     ```
  2. Implement production silent boot fragment `silent-boot.cfg`:
     ```text
     CONFIG_BOOTDELAY=-2
     CONFIG_SILENT_CONSOLE=y
     CONFIG_SYS_DEVICE_NULLDEV=y
     CONFIG_SILENT_U_BOOT_ONLY=y
     ```
  3. Conditionally append `file://silent-boot.cfg` to `SRC_URI` in `u-boot_%.bbappend` when `HSM_DEV_MODE == "0"`.
  4. In `boot.cmd.in`, conditionally assert `setenv silent 1` and guard `echo "RAUC: ..."` statements when `HSM_DEV_MODE == "0"`.

---

## Decision

We adopt **Option 3**:
- In production builds (`HSM_DEV_MODE = "0"`):
  - `CONFIG_BOOTDELAY=-2`: Setting `CONFIG_BOOTDELAY` to a negative value disables autoboot delay. Crucially, U-Boot defines:
    - `-1`: Autoboot without delay, but key press can still abort if pressed during init.
    - `-2`: Autoboot without delay, and aborting via key press is **completely disabled**.
  - `CONFIG_SILENT_CONSOLE=y` and `CONFIG_SYS_DEVICE_NULLDEV=y`: Allows routing all console output to a null device.
  - `CONFIG_SILENT_U_BOOT_ONLY=y`: Ensures that silencing U-Boot does not unintentionally silence the downstream Linux kernel (the kernel receives its own explicit console arguments).
  - `boot.cmd.in` executes `setenv silent 1` and suppresses all informational `echo` lines.
- In development builds (`HSM_DEV_MODE = "1"`):
  - U-Boot compiles with default interactive delay (`CONFIG_BOOTDELAY=2`).
  - Bootscript emits full RAUC slot selection diagnostics.
  - `squashfs.cfg` ensures SquashFS filesystem support is enabled across all 4 machine architectures.

---

## Consequences

- **Positive**:
  - Production appliances cannot be hijacked or interrupted at the bootloader stage via serial break keys or keyboard input.
  - Boot time is accelerated because U-Boot skips countdown timers and expensive UART console character transmissions.
  - Multi-board compatibility is achieved without maintaining board-specific patches.
- **Negative / Neutral**:
  - In production, manual recovery via U-Boot prompt is impossible; appliance recovery relies on automated RAUC A/B watchdog fallback.
