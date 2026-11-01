---
title: Yocto Build and RAUC Update Guide
tags:
  - operational-guide
  - yocto
  - kas
  - rauc
  - flashing
  - updates
  - rpi
created: 2026-11-01
updated: 2026-11-01
status: Approved
repository: rpi-z2w-yocto-hsm
---

# Yocto Build and RAUC Update Guide

This operational guide provides end-to-end procedures for compiling, flashing, configuring, and updating the **`rpi-z2w-yocto-hsm`** air-gapped embedded Linux operating system for the Raspberry Pi appliance and nRF52840 HSM Dongle (`PCA10059`).

---

## 1. Build System Architecture & Tooling

The build pipeline leverages **Kas** (`ghcr.io/siemens/kas/kas:latest`) running inside Docker or Podman. Kas provides declarative layer aggregation, reproducible BitBake environments, and complete isolation from host dependencies.

```mermaid
flowchart LR
    HOST["Host Machine\n(Docker / Podman)"] --> KAS["Kas Build Container\n(ghcr.io/siemens/kas)"]
    KAS --> LAYERS["Yocto 5.0 (Scarthgap)\npoky, meta-raspberrypi, meta-rauc, meta-hsm-os"]
    LAYERS --> WIC["Rootfs WIC Image\n(.wic.bz2)"]
    LAYERS --> RAUCB["Signed Update Bundle\n(.raucb)"]
    WIC --> SD["MicroSD Card\n(flash-sd.sh)"]
    RAUCB --> USB["USB Flash Drive\n(Offline Ingestion)"]
```

### Host Prerequisites
- **Container Engine**: Docker or Podman installed and running.
- **Disk Space**: At least 50 GB free disk space (downloads and sstate cache are shared across builds).
- **Architecture**: Supports macOS (Apple Silicon / Intel) and Linux (`x86_64` / `aarch64`).
- **Host Tools for Flashing**: `bzip2` or `pbzip2`, `dd`, `sudo`.

---

## 2. Building OS Images with Kas

Build targets can be triggered either from the **workspace root** or directly within the **`rpi-z2w-yocto-hsm`** sub-repository.

### 2.1 Workspace Root Makefile Targets

```bash
# Build development image (hsm-image-dev) for default machine (raspberrypi0-2w-64)
make yocto-build-dev

# Build production air-gapped image (hsm-image-prod)
make yocto-build-prod

# Target specific Raspberry Pi model (Pi 3, Pi 4, Pi 5)
make -C rpi-z2w-yocto-hsm build-prod MACHINE=raspberrypi4-64

# Build signed RAUC A/B update bundle (.raucb)
make yocto-bundle

# Launch interactive Kas shell inside build container
make yocto-shell

# Clean build artifacts while preserving downloads/sstate cache
make yocto-clean
```

### 2.2 Sub-Repository Makefile Targets

Navigate to [`rpi-z2w-yocto-hsm/`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm):

```bash
cd rpi-z2w-yocto-hsm

# 1. Build development image (features sudo, debug console port, RAM logging)
make build-dev

# 2. Build production hardened image (quiet kernel, locked root, AT port only, silent journald)
make build-prod

# 3. Build signed RAUC update bundle (.raucb)
make bundle

# 4. Open interactive Bitbake shell
make shell
```

### 2.3 Kas Configuration Hierarchy

Kas configurations are organized hierarchically:
- [`kas-project.yml`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/kas-project.yml): Declares upstream Scarthgap repositories (`poky`, `meta-raspberrypi`, `meta-openembedded`, `meta-rauc`), machine defaults (`raspberrypi0-2w-64`), and base distro configurations.
- [`kas-dev.yml`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/kas-dev.yml): Inherits `kas-project.yml`, builds target `hsm-image-dev`, sets `HSM_DEV_MODE=1`, enables `debug-tweaks`, root password `root`, sudo escalation for user `hsmutil`, and RAM-only logging (`Storage=volatile`).
- [`kas-prod.yml`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/kas-prod.yml): Inherits `kas-project.yml`, builds target `hsm-image-prod`, sets `HSM_DEV_MODE=0`, enforces quiet kernel, locks root account (`usermod -L root`), and enables silent journald (`Storage=none`).

---

## 3. Flashing to MicroSD Card

Once the build finishes, the raw partitioned image is available under:
`rpi-z2w-yocto-hsm/build/tmp/deploy/images/<machine>/hsm-image-<variant>.rootfs.wic.bz2`

### 3.1 Device Identification Safety

> [!CAUTION]
> Flashing raw disk images completely overwrites the target device. Always identify your physical MicroSD card device path with absolute certainty before proceeding.

- **macOS Identification**:
  ```bash
  diskutil list
  # Identify external disk (e.g., /dev/disk4)
  # Use the raw character device /dev/rdisk4 for 10x faster flashing speed
  ```
- **Linux Identification**:
  ```bash
  lsblk -p
  # Identify target SD card (e.g., /dev/sdb or /dev/mmcblk0)
  ```

### 3.2 Executing Flashing Script

Use the built-in flasher [`scripts/flash-sd.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/scripts/flash-sd.sh):

```bash
# Syntax: ./scripts/flash-sd.sh <image_path> <target_device>

./scripts/flash-sd.sh \
  build/tmp/deploy/images/raspberrypi0-2w-64/hsm-image-prod.rootfs.wic.bz2 \
  /dev/rdisk4
```

The script performs safety checks:
1. Confirms the source compressed image exists.
2. Displays source and target device path.
3. Requires explicit interactive typing of `"yes"` before invoking `dd`.
4. Streams decompression via `bzip2 -dc` directly to the raw block device with `bs=4M status=progress conv=fsync`.
5. Executes `sync` to ensure buffered blocks are flushed to physical flash.

---

## 4. Appliance Assembly & Cold Boot Procedure

```mermaid
graph TD
    PI["1. Mount Waveshare USB HAT (B)\nonto Raspberry Pi Zero 2 W"] --> HDMI["2. Connect HDMI Monitor\n(Direct tty1 Console)"]
    HDMI --> USB_DEVS["3. Connect Peripherals to HAT:\n- USB Keyboard\n- Primary nRF52840 HSM Dongle\n- (Optional) Secondary HSM Dongle\n- (Optional) USB Storage Drive"]
    USB_DEVS --> POWER["4. Connect 5V 2.5A Power\nBoot into Linux 6.6 LTS"]
```

1. **Physical Assembly**:
   - Mount the [[Hardware/Waveshare USB HAT (B) Integration|Waveshare USB HAT (B)]] onto the Raspberry Pi Zero 2 W (or Pi 3/4/5) using brass standoffs.
   - Verify firm contact of pogo pins with test points (or attach the Micro-USB bridge).
2. **Connect Peripherals**:
   - Local HDMI display to HDMI port (console will appear on `tty1` via `hdmi_force_hotplug=1`).
   - USB Keyboard into Waveshare USB HAT Port 1.
   - nRF52840 HSM Dongle into Waveshare USB HAT Port 2.
3. **Power On**:
   - Apply power via 5V 2.5A Micro-USB adapter.
   - U-Boot initializes hardware, verifies partition slot (Slot A), and boots the hardened kernel.
   - The appliance issue banner displays on the local HDMI monitor.

---

## 5. Operator Initialization & Workflows

### 5.1 First-Login Setup Wizard
1. At the login prompt, enter default operator credentials:
   - **Login**: `hsmutil`
   - **Password**: `hsmutil`
2. [`first-login-wizard.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-hsm-os/recipes-core/init-wizard/files/first-login-wizard.sh) executes automatically:
   - **Step 1: Forced Password Change**:
     ```text
     You must now change the default password for account 'hsmutil'.
     Current password:
     New password:
     Retype new password:
     ```
     Password update applies to `/etc/shadow` residing in volatile RAM overlay (`/run/overlay/etc-upper`), preserving the underlying immutable root filesystem.
   - **Step 2: Keyboard Layout Selection**:
     ```text
     Select your physical keyboard layout:
       1) US English (us)
       2) French (fr)
       3) German (de)
       4) UK English (uk)
       5) Swiss French (fr_CH)
       6) Swiss German (sg)
     Enter choice [1-6] (default: 1):
     ```
     Applies `loadkeys` and persists choice to `~/.keymap`.
   - **Step 3: Timezone / Locale**:
     Applies `export TZ=UTC` ensuring standard timestamp logging.
   - Creates marker `~/.wizard_completed` so the wizard does not re-fire on subsequent logins.

### 5.2 Every-Login UTC Timestamp Calibration
Because the appliance is air-gapped without NTP network synchronization, and Raspberry Pi boards lack an onboard battery RTC:
1. On every login shell, [`clock-setup.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-hsm-os/recipes-core/init-wizard/files/clock-setup.sh) prompts:
   ```text
   ------------------------------------------------------------------
   Air-Gapped Clock Notice:
   Current System Time: 2026-11-01 12:00:00 UTC
   ------------------------------------------------------------------
   Because this machine is strictly air-gapped without NTP network sync
   or battery-backed RTC, verify and update the system clock for
   accurate cryptographic certificate timestamps and audit logs.

   Enter current UTC date/time [YYYY-MM-DD HH:MM:SS] (or Enter to keep):
   ```
2. Operator inputs current UTC time (e.g. `2026-11-01 19:30:00`).
3. The script calls the setuid root C helper [`hsm-set-date`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-hsm-os/recipes-core/init-wizard/files/hsm-set-date.c) (`/usr/local/bin/hsm-set-date`), which validates the format using `strptime`/`timegm` and calls `settimeofday()`.
4. The system clock is updated without requiring superuser credentials, ensuring valid x509 certificate timestamps.

### 5.3 USB Mass Storage Automounting
When a USB flash drive (FAT32 or ext4) is inserted into an available Waveshare HAT port:
- `udev` invokes [`usb-mount.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-hsm-os/recipes-support/usb-mount/files/usb-mount.sh) via `usb-mount@<dev>.service`.
- Mounts the partition under `/media/usb-<devname>` with ownership `hsmutil:storage` and permissions `0770`.
- Operator can read and write keys, certificates, and backups:
  ```bash
  # Generate CSR or export public key to USB drive:
  hsmutil key list
  hsmutil key export --slot 1 --out /media/usb-sda1/root-ca.pub.pem
  ```

---

## 6. Offline RAUC A/B Firmware Update Protocol

System updates are cryptographically signed using an x509 PKI infrastructure and delivered offline via USB mass storage.

### 6.1 RAUC PKI Architecture

```mermaid
graph TD
    CA_KEY["rauc-ca.key.pem\n(Root CA Private Key - Kept Offline)"]
    CA_CERT["rauc-ca.cert.pem\n(Root CA Public Certificate - Baked into Rootfs)"]
    DEV_KEY["development-signing.key.pem\n(Release Signing Key)"]
    DEV_CERT["development-signing.cert.pem\n(Release Signer Certificate)"]
    BUNDLE["update.raucb\n(CMS Cryptographic Container)"]
    APPLIANCE["Air-Gapped Appliance\n(/etc/rauc/ca.cert.pem)"]

    CA_KEY -->|Signs| CA_CERT
    CA_KEY -->|Signs| DEV_CERT
    DEV_KEY -->|Signs Manifest| BUNDLE
    DEV_CERT -->|Included in| BUNDLE
    CA_CERT -->|Embedded in| APPLIANCE
    BUNDLE -. Verified by .-> APPLIANCE
```

To initialize or rotate RAUC keys:
```bash
./scripts/generate-rauc-keys.sh
```
- Generates an ECDSA P-256 (`prime256v1`) Root CA certificate valid for 10 years.
- Embeds `keys/rauc-ca.cert.pem` directly into `/etc/rauc/ca.cert.pem` during Yocto image compilation.
- Generates a developer signing certificate (`development-signing.cert.pem`) signed by the Root CA.

### 6.2 Creating Signed Update Bundles
Run the bundle generator:
```bash
# From workspace root:
make yocto-bundle

# Or from sub-repo:
cd rpi-z2w-yocto-hsm
make bundle
```
The resulting `.raucb` bundle encapsulates the new kernel, device tree, and rootfs image, cryptographically signed with CMS/PKCS#7.

### 6.3 Offline Automated USB Installation
1. **Copy Bundle to USB Storage**:
   ```bash
   cp build/tmp/deploy/images/raspberrypi0-2w-64/update-bundle-raspberrypi0-2w-64.raucb /Volumes/SECURE_USB/update.raucb
   ```
2. **Insert USB Drive into Appliance**:
   - Plug the drive into an available Waveshare USB HAT port.
3. **Automated Detection & Installation**:
   - `udev` mounts the drive under `/media/usb-sda1`.
   - `usb-mount.sh` detects `update.raucb` and automatically starts `rauc-usb-autoinstall.service`.
   - RAUC verifies the CMS signature against `/etc/rauc/ca.cert.pem`.
   - If valid, RAUC streams the image into the inactive partition slot (e.g. Slot B).
   - U-Boot environment variables are updated: `upgrade_available=1`, `BOOT_ORDER="B A"`.
4. **Reboot & Mark Good**:
   - Power-cycle or reboot the appliance.
   - U-Boot boots into the updated slot.
   - Confirm active slot and health:
     ```bash
     rauc status
     rauc status mark-good
     ```
   - If the update fails to boot, U-Boot's boot counter automatically falls back to Slot A.

---

## 7. Verification & Troubleshooting Commands

Inside the appliance terminal, verify system state with:

```bash
# 1. Inspect RAUC dual-slot status and active slot:
rauc status

# 2. Inspect attached nRF52840 HSM character device nodes:
ls -la /dev/tty.hsm*
# Output: crw-rw---- 1 root dialout ... /dev/tty.hsmF4CE3682910A0
# Symlink: lrwxrwxrwx 1 root root ... /dev/tty.hsm -> tty.hsmF4CE3682910A0

# 3. Test communication with HSM via hsmutil:
hsmutil info
hsmutil echo --message "Air-gap verification"

# 4. Verify total-drop firewall rules:
nft list ruleset
# Output confirms: table inet filter { chain input { type filter hook input priority filter; policy drop; } ... }

# 5. Check volatile RAM mounts:
df -h | grep -E "media|overlay|hsmutil"
# Confirms tmpfs mounts on /media, /home/hsmutil, and overlay on /etc

# 6. Check silent journald configuration:
cat /etc/systemd/journald.conf.d/airgap-logging.conf
# Output (prod): Storage=none, ForwardToSyslog=no, ForwardToKMsg=no, ForwardToConsole=no

# 7. Test unprivileged clock calibration:
hsm-set-date "2026-11-01 20:00:00"
date -u
```

---

## 8. Cross-References

- **Appliance Architecture**: [[Architecture/07 - Air-Gapped RPi HSM OS Architecture|07 - Air-Gapped RPi HSM OS Architecture]]
- **Hardware Integration**: [[Hardware/Waveshare USB HAT (B) Integration|Waveshare USB HAT (B) Integration]]
- **Dongle Specifications**: [[Hardware/nRF52840 Dongle PCA10059|nRF52840 Dongle PCA10059]]
- **CLI Reference**: [[Development/CLI Reference|CLI Reference]]
- **Root Orchestration**: [`Makefile`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/Makefile)
