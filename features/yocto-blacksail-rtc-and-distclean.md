---
title: "Feature: Yocto 6.1 (Blacksail), HW-084 (DS3231) RTC & Workspace Distclean"
tags:
  - feature
  - yocto
  - blacksail
  - rtc
  - ds3231
  - distclean
  - air-gap
  - rpi
created: 2026-11-08
status: Approved
repository: hsm-yocto-offgrid-rpi
references:
  - "[[Specification/Yocto 6.1 Blacksail Upgrade, HW-084 DS3231 RTC Integration and Workspace Distclean|Technical Specification]]"
  - "[[Plan/Yocto 6.1 Blacksail Upgrade, HW-084 DS3231 RTC Integration and Workspace Distclean Implementation Plan|Implementation Plan]]"
  - "[[Architecture/07 - Air-Gapped RPi HSM OS Architecture|07 - Air-Gapped RPi HSM OS Architecture]]"
  - "[[Architecture/08 - Air-Gapped Time Synchronization and Hardware RTC Architecture|08 - Air-Gapped Time Synchronization and Hardware RTC Architecture]]"
  - "[[Architecture/Build Lifecycle and Workspace Clean Routines|Build Lifecycle and Workspace Clean Routines]]"
  - "[[Architecture/Decisions/ADR-0021 - Hardware I2C-1 Bus Mapping for HW-084 DS3231 RTC Module|ADR-0021]]"
  - "[[Architecture/Decisions/ADR-0022 - RTC-Aware Informational Login Prompt with Transactional Hardware Clock Sync|ADR-0022]]"
  - "[[Architecture/Decisions/ADR-0023 - Container Volume and Workspace Cache Purging during Distclean|ADR-0023]]"
---

# Feature: Yocto 6.1 (Blacksail), HW-084 (DS3231) RTC & Workspace Distclean

This feature document synthesizes the capabilities, user journeys, configuration contracts, and verification criteria for modernizing the **air-gapped Raspberry Pi HSM OS** to **Yocto 6.1 (Blacksail)**, integrating the **HW-084 (DS3231)** hardware Real-Time Clock, and implementing atomic **`make distclean`** automation across the workspace.

---

## 1. Feature Overview & Value Proposition

1. **Yocto Project 6.1 (Blacksail) Modernization**:
   - Upgrades BitBake to branch 2.20 and layer manifests to Blacksail.
   - Eliminates deprecated `*_OPTED_OUT` variables in favor of strict `:remove` overrides.
   - Enforces `${UNPACKDIR}` compliance across all custom BitBake recipes.
   - Verifies Linux kernel 6.12+ compatibility for `nrf52840-hsm-kmod`.
2. **HW-084 (DS3231) Hardware Real-Time Clock Integration**:
   - Provides a battery-backed physical hardware timekeeper for air-gapped cryptographic ceremonies without network connectivity.
   - Early boot kernel synchronization via `CONFIG_RTC_DRV_DS1307=y` and `CONFIG_RTC_HCTOSYS=y`.
   - Dedicated primary hardware I2C-1 bus mapping (Pins 1, 3, 5, 9) through the Waveshare USB HAT (B) GPIO passthrough.
   - RTC-aware login prompt with zero-friction `[Enter]` to retain time and transactional SUID `hsm-set-date` helper for calibration.
3. **Workspace Distclean Automation**:
   - Total clean-slate reset target in `hsm-yocto-offgrid-rpi/Makefile` that purges `build/`, `downloads/`, `sstate-cache/`, Kas cloned layers, and the Docker/Podman named volume `rpi_yocto_hsm_tmp`.
   - Root `Makefile` orchestration forwarding via `make yocto-distclean` and root `make distclean`.

---

## 2. Operator User Journeys

### User Journey 1: Standard Routine Operator Login
- **Given**: An air-gapped appliance equipped with the HW-084 DS3231 RTC module with a healthy CR2032 battery.
- **When**: The operator powers on the appliance and logs into console `tty1` as user `hsmutil`.
- **Then**:
  1. Linux kernel reads `/dev/rtc0` at boot and initializes `CLOCK_REALTIME`.
  2. `clock-setup.sh` verifies `/dev/rtc0` via `hwclock -u -r` and displays:
     ```text
     ============================================================
              HSM Appliance Hardware Clock Status
     ============================================================
     [OK] Hardware RTC detected (/dev/rtc0): active and synchronized
     Current System Time (UTC): 2026-11-08 14:30:15
     ============================================================
     Enter current UTC date/time [YYYY-MM-DD HH:MM:SS] (or Enter to keep): 
     ```
  3. The operator presses `[Enter]`.
  4. The system retains the RTC time and proceeds immediately to the `gphsm` command prompt.

### User Journey 2: Clock Calibration / Initial Deployment
- **Given**: A newly assembled appliance where the DS3231 RTC module is uncalibrated or drifted.
- **When**: The operator logs in and is prompted for date/time.
- **Then**:
  1. The operator types: `2026-11-08 14:35:00`.
  2. SUID helper `hsm-set-date` validates format and calls `settimeofday()`.
  3. SUID helper opens `/dev/rtc0` and updates the DS3231 registers via `ioctl(RTC_SET_TIME)`.
  4. Terminal outputs:
     ```text
     System time set to: 2026-11-08 14:35:00 UTC
     Hardware RTC (/dev/rtc0) synchronized successfully.
     ```
  5. Subsequent boots retain the updated time across power cycles.

### User Journey 3: Clean-Slate Rebuild via Root `make distclean`
- **Given**: A developer or CI runner preparing for a major version release or after switching Yocto series.
- **When**: The operator executes `make distclean` from the workspace root.
- **Then**:
  1. Root Makefile cleans firmware, CLI, and PKCS#11 provider build trees.
  2. Root Makefile invokes `yocto-distclean`.
  3. `hsm-yocto-offgrid-rpi/Makefile` wipes `build/`, `downloads/`, `sstate-cache/`, and Kas layers.
  4. Container volume `rpi_yocto_hsm_tmp` is pruned via `docker volume rm` or `podman volume rm`.
  5. The entire multi-repo workspace is restored to a 100% pristine git clone state.

---

## 3. Configuration & Interface Contracts

### 3.1 Kernel Configuration (`meta-hsm-os/recipes-kernel/linux/files/hardened.cfg`)
```kconfig
CONFIG_RTC_CLASS=y
CONFIG_RTC_DRV_DS1307=y
CONFIG_RTC_HCTOSYS=y
CONFIG_RTC_HCTOSYS_DEVICE="rtc0"
CONFIG_I2C=y
CONFIG_I2C_CHARDEV=y
CONFIG_I2C_BCM2835=y
```

### 3.2 Raspberry Pi Boot Configuration (`rpi-config_%.bbappend`)
```bitbake
ENABLE_I2C = "1"
RPI_EXTRA_CONFIG:append = "\n# HW-084 (DS3231) Hardware RTC on I2C-1\ndtoverlay=i2c-rtc,ds3231\n"
```

### 3.3 Kas Manifest (`hsm-yocto-offgrid-rpi/kas-project.yml`)
```yaml
header:
  version: 14
distro: hsm-os
target:
  - hsm-image-base
repos:
  openembedded-core:
    url: "https://git.openembedded.org/openembedded-core"
    branch: "blacksail"
    path: "openembedded-core"
    layers:
      meta:
  bitbake:
    url: "https://git.openembedded.org/bitbake"
    branch: "2.20"
    path: "bitbake"
  meta-yocto:
    url: "https://git.yoctoproject.org/meta-yocto"
    branch: "blacksail"
    path: "meta-yocto"
    layers:
      meta-poky:
      meta-yocto-bsp:
  meta-openembedded:
    url: "https://git.openembedded.org/meta-openembedded"
    branch: "blacksail"
    path: "meta-openembedded"
    layers:
      meta-oe:
      meta-python:
      meta-networking:
      meta-filesystems:
  meta-raspberrypi:
    url: "https://github.com/agherzan/meta-raspberrypi.git"
    branch: "master"
    path: "meta-raspberrypi"
  meta-rauc:
    url: "https://github.com/rauc/meta-rauc.git"
    branch: "master"
    path: "meta-rauc"
  meta-hsm-os:
    path: "meta-hsm-os"
```

---

## 4. Acceptance Criteria & Test Matrix

- [x] **AC-1 (Kas Manifest Syntax)**: `kas-project.yml` parses with Kas with zero BitBake deprecation warnings.
- [x] **AC-2 (Kernel RTC & I2C)**: Target kernel contains `CONFIG_RTC_DRV_DS1307=y`, `CONFIG_RTC_HCTOSYS=y`, `CONFIG_RTC_HCTOSYS_DEVICE="rtc0"`, `CONFIG_I2C=y`, `CONFIG_I2C_BCM2835=y`.
- [x] **AC-3 (Firmware Overlays)**: In `boot.img`, `/boot/config.txt` contains `dtparam=i2c_arm=on` and `dtoverlay=i2c-rtc,ds3231`.
- [x] **AC-4 (Hardware Tooling)**: Image contains `/usr/sbin/i2cdetect` (`i2c-tools`) and `hwclock`.
- [x] **AC-5 (Clock Setup Prompt)**:
  - When `/dev/rtc0` is available, `clock-setup.sh` displays active status banner.
  - Pressing `[Enter]` keeps time without mutation.
  - Supplying timestamp updates kernel clock and synchronizes `/dev/rtc0`.
- [x] **AC-6 (Subrepo Distclean)**: Running `make -C hsm-yocto-offgrid-rpi distclean` purges build, downloads, sstate-cache, layers, and removes volume `rpi_yocto_hsm_tmp`.
- [x] **AC-7 (Root Workspace Distclean)**: Running `make distclean` triggers `yocto-distclean` alongside all subrepo cleans.
- [x] **AC-8 (Kernel 6.12+ Compatibility)**: `nrf52840-hsm-kmod` compiles cleanly against Linux 6.12+ headers.
