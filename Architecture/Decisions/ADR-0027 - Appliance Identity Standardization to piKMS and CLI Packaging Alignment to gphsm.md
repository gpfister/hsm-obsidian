---
title: "ADR-0027: Appliance Identity Standardization to piKMS and CLI Packaging Alignment to gphsm"
status: Accepted
date: 2026-11-11
tags:
  - adr
  - architecture
  - cli
  - gphsm
  - branding
  - hostname
  - pikms
  - yocto
  - appliance
deciders:
  - Workspace System Architect
  - Security Software Lead
  - Host Tooling Engineer
references:
  - "[[Architecture/Decisions/ADR-0017 - Host CLI Rebranding to gphsm and PKCS11 Refactoring to libgphsm|ADR-0017: Host CLI Rebranding to gphsm and PKCS#11 Refactoring to libgphsm]]"
  - "[[Architecture/Decisions/ADR-0020 - Retirement of Backward-Compatible Binary Symlinks and Native macOS USB Discovery|ADR-0020: Retirement of Backward-Compatible Binary Symlinks and Native macOS USB Discovery]]"
  - "[[Architecture/07 - Air-Gapped RPi HSM OS Architecture|07 - Air-Gapped RPi HSM OS Architecture]]"
  - "[[Specification/HSM Yocto OS Hardening, Clean Console, U-Boot Network Deactivation, and gphsm CLI Alignment|Specification: HSM Yocto OS Hardening, Clean Console & gphsm Alignment]]"
  - "[[Development/CLI Reference|CLI Reference: gphsm]]"
  - "[[00 - Index|Vault Index]]"
---

# ADR-0027: Appliance Identity Standardization to piKMS and CLI Packaging Alignment to gphsm

## Context & Problem Statement

Across the **nRF52840 Hardware Security Module (HSM)** ecosystem, two naming and packaging discrepancies existed between the host workstation tooling and the air-gapped embedded Linux operating system ([`hsm-yocto-offgrid-rpi`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi)):

1. **System Identity & Hostname Discrepancy**:
   - The appliance image defaulted to the generic hostname `airgap-hsm` via `hostname:pn-base-files` in `hsm-image-base.bb`.
   - The canonical project identity for the dedicated Raspberry Pi Key Management System appliance is **`piKMS`**. The inconsistent hostname caused ambiguity in audit logs, terminal prompts (`hsmutil@airgap-hsm` vs `hsmutil@piKMS`), and cryptographic key ceremony documentation.

2. **CLI Executable Naming Discrepancy (`hsmutil` vs `gphsm`)**:
   - In [[Architecture/Decisions/ADR-0017 - Host CLI Rebranding to gphsm and PKCS11 Refactoring to libgphsm|ADR-0017]] and [[Architecture/Decisions/ADR-0020 - Retirement of Backward-Compatible Binary Symlinks and Native macOS USB Discovery|ADR-0020]], the host administration CLI was standardized as **`gphsm`** across macOS, Linux, and Windows.
   - However, the Yocto layer [`meta-hsm-os`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi/meta-hsm-os) still maintained `recipes-support/hsmutil/hsmutil.bb`, installing the executable exclusively as `/usr/local/bin/hsmutil`.
   - As a result, operators trained on workstation documentation (`gphsm device list`, `gphsm unlock`, etc.) found `gphsm: command not found` when working inside the air-gapped console, while appliance scripts could not be ported directly to host environments without editing command names.

---

## Decision Drivers

- **Canonical Identity Harmonization**: Establish **`piKMS`** as the single authoritative system identity across prompts, audit logs, and host resolution.
- **Cross-Platform CLI Parity**: Ensure `gphsm` is the primary executable available in the appliance user path (`/usr/local/bin/gphsm`), matching macOS, Linux, and Windows installations.
- **100% Backward Compatibility**: Guarantee that existing operator muscle memory or legacy automation calling `hsmutil` executes without error via an automated symbolic link.
- **Preservation of User Space Isolation**: Maintain the dedicated unprivileged user account `hsmutil` (`/home/hsmutil`) to avoid breaking volatile `tmpfs` mounts, `sudoers` policies, or shadow password structures.

---

## Considered Options

### Option 1: Retain `hsmutil` in Yocto and `gphsm` on Workstations
- **Description**: Leave the appliance naming as-is and document the difference in CLI manuals.
- **Pros**: Zero BitBake recipe modifications.
- **Cons**: Ongoing operator confusion, divergent documentation, friction during air-gapped ceremonies.
- **Outcome**: **Rejected**.

### Option 2: Full Renaming of All User Accounts, Filesystems, and Binaries to `pikms`
- **Description**: Rename user `hsmutil` to `pikms`, change home to `/home/pikms`, rename CLI to `pikms`.
- **Pros**: Uniform name across all dimensions.
- **Cons**: 
  - Massive breaking changes across volatile mount scripts, PAM configs, shadow recipes, and udev permission rules.
  - Conflicts with the established `gphsm` CLI brand.
- **Outcome**: **Rejected**.

### Option 3: Standardize Hostname to `piKMS`, Package `gphsm` with `hsmutil` Symlink, Retain `hsmutil` User
- **Description**:
  1. Update `hostname:pn-base-files = "piKMS"` in `hsm-image-base.bb`.
  2. Create a modern recipe `recipes-support/gphsm/gphsm.bb` that installs `/usr/local/bin/gphsm` (mode `0755`) and establishes a symbolic link `/usr/local/bin/hsmutil -> gphsm`.
  3. Declare `RPROVIDES:${PN} += "hsmutil"` and replace `hsmutil` with `gphsm` in `hsm-image-base.bb` `IMAGE_INSTALL`.
  4. Retain user `hsmutil` and `/home/hsmutil` for non-privileged operator execution.
- **Pros**:
  - Perfect alignment with project identity (`piKMS`) and host CLI tooling (`gphsm`).
  - Seamless backward compatibility: both `gphsm` and `hsmutil` commands execute identically.
  - Zero disruption to volatile storage overlays, udev permissions, or PAM password configs.
- **Outcome**: **Accepted**.

---

## Decision Outcome

### 1. Hostname Standardization (`hsm-image-base.bb`)
In [`meta-hsm-os/recipes-core/images/hsm-image-base.bb`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi/meta-hsm-os/recipes-core/images/hsm-image-base.bb):
```bitbake
hostname:pn-base-files = "piKMS"
```
The console login banner, shell prompt (`hsmutil@piKMS:~$`), `/etc/hostname`, and `/etc/hosts` now consistently reflect `piKMS`.

### 2. New CLI Recipe (`recipes-support/gphsm/gphsm.bb`)
Created in [`meta-hsm-os/recipes-support/gphsm/gphsm.bb`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi/meta-hsm-os/recipes-support/gphsm/gphsm.bb):
```bitbake
SUMMARY = "nRF52840 HSM CLI and Management Tool (gphsm)"
DESCRIPTION = "Cross-platform Go command line utility and TUI dashboard for interacting with nRF52840 HSM dongles."
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = " \
    file://gphsm-arm64 \
"

S = "${UNPACKDIR}"

INSANE_SKIP:${PN} += "already-stripped ldflags file-rdeps"
INHIBIT_PACKAGE_STRIP = "1"
INHIBIT_PACKAGE_DEBUG_SPLIT = "1"

do_install() {
    install -d ${D}/usr/local/bin
    install -m 0755 ${UNPACKDIR}/gphsm-arm64 ${D}/usr/local/bin/gphsm
    ln -s gphsm ${D}/usr/local/bin/hsmutil
}

FILES:${PN} += " \
    /usr/local/bin/gphsm \
    /usr/local/bin/hsmutil \
"
RPROVIDES:${PN} += "hsmutil"
```

### 3. Image Package Manifest Alignment
In `hsm-image-base.bb`:
```bitbake
IMAGE_INSTALL:append = " \
    ...
    gphsm \
    ...
"
```
The standalone recipe `recipes-support/hsmutil/` is retired, with `RPROVIDES:${PN} += "hsmutil"` ensuring all internal package references resolve cleanly.

---

## Consequences & Verification

### Positive Consequences
- **Harmonized Tooling Experience**: Operators use `gphsm` everywhere—on their host workstations and on the air-gapped appliance.
- **Zero Breakage**: Invocations of `hsmutil` continue to execute flawlessly via the relative symlink.
- **Professional Identity**: Hostname `piKMS` provides an unambiguous, clear identity in audit records, cryptographic certificates, and console headers.
- **Preserved Isolation Architecture**: The underlying user model (`hsmutil:users`, `/home/hsmutil`) remains stable without cascading refactorings.

### Negative / Neutral Consequences
- User account remains `hsmutil` while primary command is `gphsm`, but this follows standard UNIX conventions (e.g. user `git` running `git-shell`, or user `postgres` running `psql`).
