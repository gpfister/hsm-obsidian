---
title: "ADR-0010: Kernel Boot Console Multiplexing & Command Line Specialization"
status: Accepted
date: 2026-11-05
tags:
  - adr
  - architecture
  - yocto
  - boot
  - cmdline
  - raspberrypi
deciders:
  - Embedded Systems Architect
  - Security Firmware Lead
references:
  - "[[features/yocto-boot-visibility-and-password-policy|Feature: Yocto Boot Visibility & Password Policy]]"
  - "[[Specification/Yocto Dev-Prod Boot Visibility, U-Boot Hardening, and Password Policy|Specification: Yocto Dev-Prod Boot Visibility, U-Boot Hardening, and Password Policy]]"
  - "[[Plan/Yocto Dev-Prod Boot Visibility, U-Boot Hardening, and Password Policy Implementation Plan|Implementation Plan: Yocto Dev-Prod Boot Visibility, U-Boot Hardening, and Password Policy]]"
  - "[[Architecture/07 - Air-Gapped RPi HSM OS Architecture|Architecture: Air-Gapped RPi HSM OS]]"
---

# ADR-0010: Kernel Boot Console Multiplexing & Command Line Specialization

## Context & Problem Statement

In the air-gapped embedded Linux operating system ([`meta-hsm-os`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-hsm-os)), boot visibility requirements differ fundamentally between development and production environments:

- **Development Builds (`HSM_DEV_MODE = "1"`)**: Engineers require simultaneous visibility of kernel `printk` diagnostic output on both the local HDMI monitor (`tty1`) and the serial UART console (`serial0,115200`), as well as early debug logs (`loglevel=7 debug earlyprintk`) to diagnose peripheral setup and driver initialization.
- **Production Builds (`HSM_DEV_MODE = "0"`)**: Security invariants require complete silence (`quiet loglevel=0 console=tty1 fsck.mode=skip`), zero serial console leakage, and no terminal interfaces exposed on serial pins.

Previously, `kas-dev.yml` and `kas-prod.yml` attempted to configure this behavior via:
```yaml
# kas-dev.yml
APPEND += "debug earlyprintk"

# kas-prod.yml
APPEND += "quiet loglevel=0 console=tty1 fsck.mode=skip"
```

However, `meta-raspberrypi`'s [`rpi-cmdline.bb`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-raspberrypi/recipes-bsp/bootfiles/rpi-cmdline.bb) constructs `/boot/cmdline.txt` using its own internal `CMDLINE` and `CMDLINE_*` variables, completely ignoring the generic OpenEmbedded `APPEND` variable. As a result, the configured parameters were never written to `/boot/cmdline.txt`, preventing early boot logs from appearing on HDMI in development and failing to silence the serial output in production.

---

## Decision Drivers

- **Simultaneous Dev Logging**: Kernel boot logs must be visible on both the serial port and the physical monitor without requiring rebuilds or choosing one sink over the other.
- **Production Air-Gap & Zero Leakage**: Production builds must eliminate serial kernel logs to prevent side-channel information exposure on physical headers.
- **Proper Yocto Layering**: Modifications must override `rpi-cmdline` cleanly within `meta-hsm-os` without patching external layers (`meta-raspberrypi`).
- **Compatibility with VideoCore & U-Boot**: The generated `cmdline.txt` is read by VideoCore firmware, passed into FDT `/chosen/bootargs`, and consumed by U-Boot (`boot.cmd.in`) using `fdt get value bootargs /chosen bootargs`.

---

## Considered Options

### Option 1: Standard Yocto `APPEND` Variable
- *Mechanism*: Rely on `APPEND += "..."` in image or machine configuration.
- *Evaluation*: Rejected because `meta-raspberrypi/recipes-bsp/bootfiles/rpi-cmdline.bb` does not consult `APPEND` when assembling `cmdline.txt`.

### Option 2: Machine-Level `CMDLINE_SERIAL` & `CMDLINE_DEBUG` Overrides in Kas Configs
- *Mechanism*: Define `CMDLINE_SERIAL` and `CMDLINE_DEBUG` in `kas-dev.yml` and `kas-prod.yml`.
- *Evaluation*: Fragile because `rpi-cmdline.bb` appends `console=tty1` conditionally only when certain features are present and does not easily allow multiplexing `console=serial0,115200 console=tty1` while preserving `root=` parameters.

### Option 3: Dedicated `meta-hsm-os/recipes-bsp/bootfiles/rpi-cmdline.bbappend` (Chosen)
- *Mechanism*: Implement `rpi-cmdline.bbappend` in `meta-hsm-os` that directly controls `CMDLINE` based on the value of `HSM_DEV_MODE`:
  - When `HSM_DEV_MODE == "1"`:
    `CMDLINE = "console=serial0,115200 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 rootwait loglevel=7 debug earlyprintk"`
  - When `HSM_DEV_MODE == "0"`:
    `CMDLINE = "console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 rootwait quiet loglevel=0 fsck.mode=skip"`
- *Evaluation*: Clean, deterministic, and leverages Linux kernel console multiplexing semantics.

---

## Decision

We adopt **Option 3**: Implement `meta-hsm-os/recipes-bsp/bootfiles/rpi-cmdline.bbappend` to conditionally construct `CMDLINE`.

### Kernel Console Multiplexing Semantics
In the Linux kernel, when multiple `console=` parameters are supplied:
1. `console=serial0,115200` registers the UART serial port as a kernel logging sink.
2. `console=tty1` registers the virtual terminal framebuffer (HDMI monitor) as a second kernel logging sink.
3. Kernel `printk` messages are emitted to **both** consoles simultaneously.
4. Userspace `/dev/console` is attached to the **last** registered console (`tty1`).
5. `systemd` inspects the kernel command line and spawns both `serial-getty@ttyS0.service` (for serial debugging) and `getty@tty1.service` (for local operator console).

In production mode (`HSM_DEV_MODE = "0"`), `console=serial0` is omitted entirely and `quiet loglevel=0` is asserted, ensuring zero kernel output is transmitted over physical serial pins.

---

## Consequences

- **Positive**:
  - Developers have full visibility of early boot issues on both serial terminal and HDMI monitor.
  - Production builds guarantee zero serial console leakage on hardware GPIO headers.
  - Eliminated dead/ineffective `APPEND` directives from Kas configuration files.
- **Negative / Neutral**:
  - `meta-hsm-os` explicitly manages `CMDLINE` construction, requiring maintenance if `meta-raspberrypi` changes its root device addressing scheme.
