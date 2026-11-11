---
title: "Specification: HSM Yocto OS Hardening, Clean Console, U-Boot Network Deactivation, and gphsm CLI Alignment"
tags:
  - specification
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
  - "[[Plan/HSM Yocto OS Hardening, Clean Console, U-Boot Network Deactivation, and gphsm CLI Alignment Implementation Plan|Implementation Plan]]"
  - "[[features/yocto-hardening-clean-console-pikms|Feature Specification]]"
  - "[[Architecture/Decisions/ADR-0024 - Native Shadow PAM Password Expiration vs First-Login Wizard Deduplication|ADR-0024]]"
  - "[[Architecture/Decisions/ADR-0025 - Complete U-Boot Network Subsystem Deactivation in Air-Gapped Appliance|ADR-0025]]"
  - "[[Architecture/Decisions/ADR-0026 - HDMI Console Noise Suppression and Log Level Hardening|ADR-0026]]"
  - "[[Architecture/Decisions/ADR-0027 - Appliance Identity Standardization to piKMS and CLI Packaging Alignment to gphsm|ADR-0027]]"
---

# Technical Specification: HSM Yocto OS Hardening, Clean Console, U-Boot Network Deactivation, and gphsm CLI Alignment

**Document ID**: `SPEC-2026-YOCTO-HARDENING-CLEAN-CONSOLE`  
**Target Systems**: [`hsm-yocto-offgrid-rpi`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi) (`meta-hsm-os`), Root Workspace Orchestration  
**Status**: APPROVED  
**Author**: `coding-specify`  

---

## 1. Executive Summary & Problem Motivation

The **nRF52840 Hardware Security Module (HSM)** ecosystem employs an immutable, air-gapped embedded Linux OS ([`hsm-yocto-offgrid-rpi`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi)) running on 64-bit Raspberry Pi hardware. While previous enhancements established core air-gap kernel lockdowns, RAUC dual-slot updates, and HW-084 (DS3231) hardware RTC time synchronization, operational review identified key usability, security hardening, and branding inconsistencies across the appliance firmware:

1. **Password Change Duplication**: Operators logging into a fresh installation encounter two consecutive, conflicting password change requests: first via Linux PAM (`pam_pwquality` triggered by shadow password expiration `chage -d 0`), and immediately afterward via a custom bash password loop in `first-login-wizard.sh`.
2. **Raspberry Pi Boot Logo Leakage**: During early boot, the Raspberry Pi VideoCore GPU firmware and Linux kernel render four multi-color splash screen logos on the HDMI console (`tty1`), violating the strict visual minimalism and stealth branding required of a dedicated security appliance.
3. **Residual Network Attack Surface in U-Boot**: Although the Linux kernel completely disables all networking stacks (Wi-Fi, Bluetooth, Ethernet), the U-Boot secondary bootloader still builds and includes network commands and drivers (`CONFIG_NET`, `CONFIG_CMD_DHCP`, `CONFIG_CMD_PING`, `CONFIG_CMD_TFTPBOOT`).
4. **System Identity Alignment**: The system hostname defaults to `airgap-hsm` instead of the project standard appliance hostname **`piKMS`**.
5. **TTY Console Chatter and Watchdog Spam**: On HDMI `tty1`, systemd service unit startup announcements, periodic software watchdog notifications, and kernel log buffer flushes intermittently disrupt the operator TUI experience and clock configuration prompt.
6. **Binary Name Discrepancy (`hsmutil` vs `gphsm`)**: The host CLI and management tool is standardized across macOS, Linux, and Windows as `gphsm`, but the Yocto layer installs it as `hsmutil`.
7. **U-Boot Serial Output Mode Split**: Development troubleshooting requires diagnostic serial output and bootloader interaction over UART, whereas production deployments require total silence and zero-delay boot.

---

## 2. Architecture & Design Principles

```mermaid
flowchart TD
    subgraph Bootloader["Stage 2 Bootloader (U-Boot)"]
        direction TB
        UB_CFG["U-Boot Config (no-net.cfg)"] -->|Strip CONFIG_NET| UB_CORE["Minimal U-Boot Core"]
        UB_MODE{"HSM_DEV_MODE"}
        UB_MODE -->|0 (PROD)| UB_PROD["silent=1, bootdelay=-2<br/>No UART/VideoCore chatter"]
        UB_MODE -->|1 (DEV)| UB_DEV["console=serial0,115200<br/>Interactive prompt, RAUC slot echo"]
    end

    subgraph Kernel["Linux Kernel 6.12+ (hardened.cfg + rpi-cmdline)"]
        direction TB
        K_LOGO["# CONFIG_LOGO is not set<br/>logo.nologo"] --> K_SILENT["Zero Splash / Zero Logo"]
        K_PRINTK["kernel.printk = 1 1 1 1<br/>quiet loglevel=0"] --> K_TTY["Clean Console (tty1)"]
    end

    subgraph UserSpace["Systemd & Console User Space"]
        direction TB
        SYS_CONF["system.conf.d/10-clean-console.conf<br/>LogLevel=notice, ShowStatus=no"] --> C_JOURNAL["Zero Watchdog Spam"]
        PAM["PAM + Shadow (chage -d 0)"] -->|Step 1: Enforce PAM Reset| PAM_OK["Password Changed in PAM"]
        PAM_OK --> WIZARD["first-login-wizard.sh<br/>Step 2: Keyboard Layout<br/>Step 3: Locale / UTC"]
        WIZARD --> APP["/usr/local/bin/gphsm<br/>(symlink: /usr/local/bin/hsmutil)"]
    end

    Bootloader --> Kernel
    Kernel --> UserSpace
```

---

## 3. Functional Requirements

### 3.1 Password Change Duplication Resolution (FR-1)
- **FR-1.1**: The script `first-login-wizard.sh` shall be stripped of all bash-based password prompts, confirmation inputs, complexity evaluation functions (`validate_password`), and direct password-setting commands (`chpasswd`, `passwd`).
- **FR-1.2**: Password expiration shall be strictly and natively managed by shadow password expiration (`chage -d 0` or setting `sp_lstchg` to `0` in `/etc/shadow` during `pkg_postinst` of `hsmutil-user.bb`).
- **FR-1.3**: When an operator logs in for the first time, Linux PAM shall intercept the login before launching the shell and enforce a password update according to `/etc/security/pwquality.conf`.
- **FR-1.4**: Upon successful completion of the PAM password change, the user shell starts, executing `first-login-wizard.sh`, which begins directly at **Step 1: Keyboard Layout Selection** and proceeds to **Step 2: Locale & Timezone**, writing `${HOME}/.wizard_completed`.

### 3.2 Raspberry Pi Boot Logo Suppression (FR-2)
- **FR-2.1**: The kernel configuration fragment `hardened.cfg` shall explicitly disable kernel boot logos:
  ```config
  # CONFIG_LOGO is not set
  # CONFIG_LOGO_LINUX_MONO is not set
  # CONFIG_LOGO_LINUX_VGA16 is not set
  # CONFIG_LOGO_LINUX_CLUT224 is not set
  ```
- **FR-2.2**: The kernel command line in `rpi-cmdline.bbappend` shall include the boot argument `logo.nologo` across both `CMDLINE_PROD` and `CMDLINE_DEV`.
- **FR-2.3**: The Raspberry Pi firmware configuration `rpi-config_%.bbappend` shall maintain `disable_splash=1`, and `kas-project.yml` shall declare `DISABLE_RPI_BOOT_LOGO = "1"`.

### 3.3 Deactivate Networking in U-Boot (FR-3)
- **FR-3.1**: A new configuration fragment `recipes-bsp/u-boot/files/no-net.cfg` shall be created, disabling all network-related subsystems in U-Boot:
  ```config
  # CONFIG_NET is not set
  # CONFIG_CMD_NET is not set
  # CONFIG_CMD_DHCP is not set
  # CONFIG_CMD_PING is not set
  # CONFIG_CMD_NFS is not set
  # CONFIG_CMD_TFTPBOOT is not set
  # CONFIG_NETDEVICES is not set
  ```
- **FR-3.2**: `recipes-bsp/u-boot/u-boot_%.bbappend` shall append `file://no-net.cfg` to `SRC_URI:append:rpi`.

### 3.4 System Name Standard (uname / hostname) (FR-4)
- **FR-4.1**: In `recipes-core/images/hsm-image-base.bb`, the hostname assignment for `base-files` shall be changed to:
  ```bitbake
  hostname:pn-base-files = "piKMS"
  ```
- **FR-4.2**: `/etc/hostname` and `/etc/hosts` in the generated rootfs shall resolve to `piKMS`.

### 3.5 Clean Console & Prevent TTY Disruption (FR-5)
- **FR-5.1**: Systemd manager configuration shall provide `/etc/systemd/system.conf.d/10-clean-console.conf` specifying:
  ```ini
  [Manager]
  LogLevel=notice
  LogTarget=journal
  LogColor=no
  ShowStatus=no
  ```
- **FR-5.2**: Sysctl configuration shall provide `/etc/sysctl.d/20-quiet-console.conf` specifying:
  ```ini
  kernel.printk = 1 1 1 1
  ```
- **FR-5.3**: Production kernel command line `CMDLINE_PROD` in `rpi-cmdline.bbappend` shall enforce quiet execution:
  ```text
  quiet loglevel=0 console=tty1 fsck.mode=skip vt.global_cursor_default=1 logo.nologo
  ```
- **FR-5.4**: Systemd journald logging configuration in `recipes-core/systemd-journald-airgap/files/` (`journald-prod.conf` and `journald-dev.conf`) shall enforce `ForwardToConsole=no` and `ForwardToKMsg=no`.

### 3.6 Executable Renaming (`hsmutil` -> `gphsm`) (FR-6)
- **FR-6.1**: A new recipe `recipes-support/gphsm/gphsm.bb` shall install `/usr/local/bin/gphsm` (mode `0755`) and create a backward-compatibility symbolic link: `/usr/local/bin/hsmutil -> /usr/local/bin/gphsm`.
- **FR-6.2**: `recipes-core/images/hsm-image-base.bb` shall replace dependency `hsmutil` with `gphsm`.
- **FR-6.3**: The dedicated unprivileged user account defined in `recipes-core/users/hsmutil-user.bb` shall remain named `hsmutil` with home directory `/home/hsmutil`.

### 3.7 Conditional U-Boot Serial Output (Dev vs Prod) (FR-7)
- **FR-7.1**: In **Production Mode** (`HSM_DEV_MODE=0`), U-Boot compiles with `silent-boot.cfg` (`CONFIG_BOOTDELAY=-2`, `CONFIG_SILENT_CONSOLE=y`, `CONFIG_SYS_DEVICE_NULLDEV=y`, `CONFIG_SILENT_U_BOOT_ONLY=y`). `boot.cmd.in` sets `setenv silent 1` and strips all `echo` statements.
- **FR-7.2**: In **Development Mode** (`HSM_DEV_MODE=1`), U-Boot serial console is enabled on `serial0,115200`, boot delay allows intervention, and `boot.cmd.in` emits clear diagnostic messages regarding RAUC slot selection.

---

## 4. Non-Functional Requirements

- **NFR-1 (Air-Gap Defense-in-Depth)**: Deactivating network stacks in U-Boot guarantees that even before the Linux kernel boots, no network stack exists in volatile memory or peripheral registers.
- **NFR-2 (Deterministic Operator UX)**: TTY1 output must remain strictly dedicated to the operator workflow without unsolicited background logging.
- **NFR-3 (Backward Compatibility)**: Existing operator scripts or habits invoking `hsmutil` must seamlessly execute `gphsm` via symlink.
- **NFR-4 (Build Idempotency)**: Adding `no-net.cfg` and `gphsm.bb` must cleanly build across all supported target machines (`raspberrypi0-2w-64`, `raspberrypi3-64`, `raspberrypi4-64`, `raspberrypi5`).

---

## 5. Subsystem Cross-References

- **Implementation Plan**: [[Plan/HSM Yocto OS Hardening, Clean Console, U-Boot Network Deactivation, and gphsm CLI Alignment Implementation Plan|Implementation Plan]]
- **Feature Specification**: [[features/yocto-hardening-clean-console-pikms|Feature Specification]]
- **Native PAM Expiration ADR**: [[Architecture/Decisions/ADR-0024 - Native Shadow PAM Password Expiration vs First-Login Wizard Deduplication|ADR-0024]]
- **U-Boot Network Deactivation ADR**: [[Architecture/Decisions/ADR-0025 - Complete U-Boot Network Subsystem Deactivation in Air-Gapped Appliance|ADR-0025]]
- **Clean Console Hardening ADR**: [[Architecture/Decisions/ADR-0026 - HDMI Console Noise Suppression and Log Level Hardening|ADR-0026]]
- **Appliance Identity & CLI Alignment ADR**: [[Architecture/Decisions/ADR-0027 - Appliance Identity Standardization to piKMS and CLI Packaging Alignment to gphsm|ADR-0027]]
- **Appliance Architecture**: [[Architecture/07 - Air-Gapped RPi HSM OS Architecture|07 - Air-Gapped RPi HSM OS Architecture]]
