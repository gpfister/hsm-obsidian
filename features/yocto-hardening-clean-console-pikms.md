---
title: "Feature: Yocto OS Hardening, Clean Console, U-Boot Network Deactivation & piKMS Identity"
tags:
  - feature
  - yocto
  - security
  - hardening
  - u-boot
  - console
  - pikms
  - gphsm
created: 2026-11-11
status: Approved
repository: hsm-yocto-offgrid-rpi
references:
  - "[[Specification/HSM Yocto OS Hardening, Clean Console, U-Boot Network Deactivation, and gphsm CLI Alignment|Technical Specification]]"
  - "[[Plan/HSM Yocto OS Hardening, Clean Console, U-Boot Network Deactivation, and gphsm CLI Alignment Implementation Plan|Implementation Plan]]"
  - "[[Architecture/07 - Air-Gapped RPi HSM OS Architecture|07 - Air-Gapped RPi HSM OS Architecture]]"
  - "[[Architecture/Decisions/ADR-0024 - Native Shadow PAM Password Expiration vs First-Login Wizard Deduplication|ADR-0024]]"
  - "[[Architecture/Decisions/ADR-0025 - Complete U-Boot Network Subsystem Deactivation in Air-Gapped Appliance|ADR-0025]]"
  - "[[Architecture/Decisions/ADR-0026 - HDMI Console Noise Suppression and Log Level Hardening|ADR-0026]]"
  - "[[Architecture/Decisions/ADR-0027 - Appliance Identity Standardization to piKMS and CLI Packaging Alignment to gphsm|ADR-0027]]"
---

# Feature: Yocto OS Hardening, Clean Console, U-Boot Network Deactivation & piKMS Identity

This feature specification details the user journeys, capabilities, and verification matrix for hardening the **air-gapped Raspberry Pi HSM OS** ([`hsm-yocto-offgrid-rpi`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi)), suppressing visual and operational console noise, eliminating dormant bootloader network stacks, and standardizing system identity to **`piKMS`** and CLI tooling to **`gphsm`**.

---

## 1. Feature Overview & Capabilities

1. **Native PAM Password Rotation (Zero Duplication)**:
   - Eliminates redundant bash password prompts.
   - Enforces password complexity natively via Linux PAM and shadow expiration (`sp_lstchg=0`).
   - The first-login setup wizard begins cleanly at keyboard layout and timezone selection.
2. **Stealth Boot & Clean Console**:
   - Completely suppresses Raspberry Pi GPU/kernel framebuffer boot logos (`# CONFIG_LOGO is not set`, `logo.nologo`).
   - Hardens systemd and kernel printk (`10-clean-console.conf`, `20-quiet-console.conf`) to eliminate service unit chatter and watchdog interruptions on HDMI `tty1`.
3. **U-Boot Network Subsystem Deactivation**:
   - Strips `CONFIG_NET`, `CONFIG_CMD_DHCP`, `CONFIG_CMD_PING`, and related network drivers from U-Boot, enforcing physical air-gapping continuously across all bootloader stages.
4. **Appliance Identity Standard (`piKMS`)**:
   - Sets the system hostname to `piKMS` across `/etc/hostname`, `/etc/hosts`, and shell prompts.
5. **Cross-Platform CLI Parity (`gphsm`)**:
   - Packages the Go CLI as `/usr/local/bin/gphsm` with backward-compatible `/usr/local/bin/hsmutil` symlink.

---

## 2. Operator User Journeys

### User Journey 1: First-Boot Commissioning
- **Given**: A freshly flashed `hsm-image-prod` MicroSD card in a Raspberry Pi HSM appliance.
- **When**: The operator powers on the appliance with an HDMI monitor and USB keyboard attached.
- **Then**:
  1. Boot screen remains pitch black (zero berry logos, zero kernel messages, zero systemd service lines).
  2. The system presents the login prompt: `piKMS login: `.
  3. The operator logs in with `hsmutil` / `password`.
  4. Linux PAM immediately intercepts: `You are required to change your password immediately (administrator enforced)`.
  5. The operator enters a compliant password (validated by `pam_pwquality`).
  6. Upon success, the session starts and `first-login-wizard.sh` prompts for Keyboard Layout (Step 1) and Timezone (Step 2).
  7. `clock-setup.sh` displays hardware RTC status.
  8. The operator is ready at `hsmutil@piKMS:~$`.

### User Journey 2: CLI Invocation with Backward Compatibility
- **Given**: An operator executing cryptographic ceremonies on the appliance.
- **When**: The operator types `gphsm device list` or legacy command `hsmutil device list`.
- **Then**:
  - Both commands invoke the same Go executable, returning deterministic dongle serials and port mappings.

---

## 3. Acceptance Criteria & Test Matrix

- [x] **AC-1 (First Login)**: PAM intercepts expired password on initial login; wizard runs without duplicate password prompt.
- [x] **AC-2 (Logo Suppression)**: Zero berry logos during boot; `logo.nologo` present in `/proc/cmdline`.
- [x] **AC-3 (U-Boot No-Net)**: U-Boot binary compiled with `no-net.cfg`; zero network commands available.
- [x] **AC-4 (Hostname piKMS)**: `hostname` returns `piKMS`.
- [x] **AC-5 (Clean Console)**: `cat /proc/sys/kernel/printk` returns `1 1 1 1`; zero watchdog spam on `tty1`.
- [x] **AC-6 (gphsm Packaging)**: `/usr/local/bin/gphsm` and symlink `/usr/local/bin/hsmutil` both exist and function.
