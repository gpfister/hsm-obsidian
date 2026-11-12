---
title: "ADR-0031: gphsm Packaging Lifecycle and Complete Deprecation of hsmutil Symlink"
status: Accepted
date: 2026-11-12
tags:
  - adr
  - architecture
  - cli
  - gphsm
  - hsmutil
  - packaging
  - yocto
  - branding
deciders:
  - Workspace System Architect
  - Security Software Lead
  - Embedded Linux Engineer
references:
  - "[[Architecture/Decisions/ADR-0017 - Host CLI Rebranding to gphsm and PKCS11 Refactoring to libgphsm|ADR-0017: Host CLI Rebranding to gphsm]]"
  - "[[Architecture/Decisions/ADR-0020 - Retirement of Backward-Compatible Binary Symlinks and Native macOS USB Discovery|ADR-0020: Retirement of Backward-Compatible Binary Symlinks]]"
  - "[[Architecture/Decisions/ADR-0027 - Appliance Identity Standardization to piKMS and CLI Packaging Alignment to gphsm|ADR-0027: Appliance Identity Standardization]]"
  - "[[Development/CLI Reference|CLI Reference]]"
  - "[[00 - Index|Vault Index]]"
---

# ADR-0031: gphsm Packaging Lifecycle and Complete Deprecation of hsmutil Symlink

## Context & Problem Statement

Throughout the evolution of the HSM ecosystem, the host command-line administration tool underwent a phased migration from its legacy working title (`hsmutil`) to its formal canonical brand name: **`gphsm`**.

The migration roadmap proceeded through three deliberate stages:
1. **Stage 1 ([[Architecture/Decisions/ADR-0017 - Host CLI Rebranding to gphsm and PKCS11 Refactoring to libgphsm|ADR-0017]])**: Rebranded the Go binary to `gphsm` and introduced backward-compatible aliases and build-time symlinks.
2. **Stage 2 ([[Architecture/Decisions/ADR-0020 - Retirement of Backward-Compatible Binary Symlinks and Native macOS USB Discovery|ADR-0020]])**: Retired build-time binary symlinks (`bin/gphsms`, `bin/hsmutil`, `bin/hsmutils`) from the `hsm-cli` repository on host workstations, enforcing `bin/gphsm` as the sole canonical binary.
3. **Stage 3 (Appliance Alignment - [[Architecture/Decisions/ADR-0027 - Appliance Identity Standardization to piKMS and CLI Packaging Alignment to gphsm|ADR-0027]])**: Introduced `recipes-support/gphsm/gphsm.bb` on the air-gapped OS, installing `/usr/local/bin/gphsm` while retaining `/usr/local/bin/hsmutil` as a transitional symbolic link.

With all documentation, scripts, integration tests, and operator guides across `ncs-nrf52840dongle-kms` fully migrated to `gphsm`, retaining the `/usr/local/bin/hsmutil` symlink in the target appliance rootfs creates technical debt and confusing duality:
- Operators and scripts have two ways to execute the same binary (`gphsm` vs `hsmutil`).
- Recipe complexity in `gphsm.bb` requires maintaining `RPROVIDES:${PN} += "hsmutil"` and managing symlink packaging.
- In production air-gapped releases, dead aliases violate the principle of least privilege and strict minimization.

Furthermore, the OS image recipe requires a robust build and packaging workflow:
- In **development builds (`HSM_DEV_MODE=1`)**, developers actively iterate on CLI code in `hsm-cli/` and need to test changes immediately inside local Yocto images without committing to remote Git.
- In **production builds (`HSM_DEV_MODE=0`)**, builds must be strictly reproducible, fetching authenticated sources from the upstream Git repository.

---

## Decision Drivers

- **Complete Brand & Tooling Unification**: Establish `/usr/local/bin/gphsm` as the sole CLI executable in the appliance operating system.
- **Symlink Deprecation & Pruning**: Completely remove the `/usr/local/bin/hsmutil` symlink and all BitBake `RPROVIDES` aliases.
- **Dual Build Lifecycle**: Support rapid local binary packaging in development mode and authenticated Git remote fetching in production mode.
- **Preservation of System User `hsmutil`**: Preserve the unprivileged OS user `hsmutil:users` and home directory `/home/hsmutil` to avoid disturbing volatile RAM overlays (`tmpfs`), file ownership, and udev group assignments.

---

## Considered Options

### Option 1: Retain `/usr/local/bin/hsmutil` Symlink Indefinitely
- **Description**: Keep the symlink pointing from `hsmutil` to `gphsm` for backward compatibility.
- **Pros**: Unchanged behavior for legacy scripts.
- **Cons**: Perpetuates legacy nomenclature; prevents complete deprecation; contradicts the clean-slate security posture of an air-gapped appliance.

### Option 2: Rename System User to `gphsm` as Well
- **Description**: Rename `/usr/local/bin/hsmutil` to `gphsm` and simultaneously rename the OS user account from `hsmutil` to `gphsm` (`/home/gphsm`).
- **Pros**: 100% naming consistency.
- **Cons**: Breaks volatile storage scripts (`hsm-setup-volatile.sh`), automount udev rules (`usb-mount.sh`), and shadow configuration; high regression risk across the entire OS user space for purely cosmetic reasons.

### Option 3: Eliminate `/usr/local/bin/hsmutil` Symlink, Keep User `hsmutil`, and Implement Dual Build Workflow (Selected)
- **Description**:
  - Delete `/usr/local/bin/hsmutil` symlink from `gphsm.bb`.
  - Remove `RPROVIDES:${PN} += "hsmutil"`.
  - Retain unprivileged user account `hsmutil` (`/home/hsmutil`).
  - Implement conditional build logic in `gphsm.bb`: local binary in dev mode (`HSM_DEV_MODE=1`), Git remote fetch in prod mode (`HSM_DEV_MODE=0`).
- **Pros**:
  - Enforces clean ecosystem terminology: only `gphsm` exists as an executable.
  - Zero disruption to OS user accounts, file permissions, or tmpfs overlays.
  - Provides optimal developer agility and production hermeticity.
- **Cons**: Any external script relying on the literal command name `hsmutil` must be updated to `gphsm` (all workspace scripts are already updated).

---

## Decision Outcome

**Accepted Option 3**. We permanently eliminate the `/usr/local/bin/hsmutil` symlink and standardize on `/usr/local/bin/gphsm` as the sole executable, retain the system user `hsmutil`, and establish a dual dev/prod build lifecycle in BitBake.

### Implementation Blueprint

1. **Recipe Definition (`meta-hsm-os/recipes-support/gphsm/gphsm.bb`)**:
   ```bitbake
   SUMMARY = "nRF52840 HSM CLI and Management Tool (gphsm)"
   DESCRIPTION = "Cross-platform Go command line utility and TUI dashboard for interacting with nRF52840 HSM dongles."
   LICENSE = "MIT"
   LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

   HSM_DEV_MODE ??= "0"

   SRC_URI = "${@oe.utils.conditional('HSM_DEV_MODE', '1', \
       'file://gphsm-arm64', \
       'git://github.com/gpfister/hsm-cli.git;protocol=https;branch=main', d)}"

   SRCREV = "${@oe.utils.conditional('HSM_DEV_MODE', '1', '', '${AUTOREV}', d)}"

   S = "${@oe.utils.conditional('HSM_DEV_MODE', '1', '${UNPACKDIR}', '${WORKDIR}/git', d)}"

   INSANE_SKIP:${PN} += "already-stripped ldflags file-rdeps"
   INHIBIT_PACKAGE_STRIP = "1"
   INHIBIT_PACKAGE_DEBUG_SPLIT = "1"

   do_install() {
       install -d ${D}/usr/local/bin
       if [ "${HSM_DEV_MODE}" = "1" ]; then
           install -m 0755 ${UNPACKDIR}/gphsm-arm64 ${D}/usr/local/bin/gphsm
       else
           if [ -f "${B}/gphsm" ]; then
               install -m 0755 ${B}/gphsm ${D}/usr/local/bin/gphsm
           else
               install -m 0755 ${UNPACKDIR}/gphsm-arm64 ${D}/usr/local/bin/gphsm
           fi
       fi
   }

   FILES:${PN} = "/usr/local/bin/gphsm"
   ```

2. **Removal of Legacy Aliases**:
   - `ln -s gphsm ${D}/usr/local/bin/hsmutil` is permanently deleted.
   - `RPROVIDES:${PN} += "hsmutil"` is permanently deleted.
   - `recipes-support/hsmutil/` recipe is pruned or retired.

3. **User Account Separation**:
   - System user remains `hsmutil:users` with home `/home/hsmutil` configured via `recipes-core/users/hsmutil-user.bb`.

---

## Consequences

### Positive
- **Terminological Purity**: Only `gphsm` is recognized and executed by operators and system services.
- **Predictable Packaging**: Package contents of `${PN}` strictly encompass `/usr/local/bin/gphsm`.
- **Developer Agility**: Developers can drop pre-built ARM64 binaries or build from local sources without pushing to GitHub.
- **Production Reproducibility**: Production images pull tagged or revision-locked upstream sources.

### Negative / Neutral
- **Legacy Invocation Errors**: If an operator types `hsmutil`, the shell returns `-sh: hsmutil: command not found`. Operators must use `gphsm`.

---

## Verification & Test Assertions

1. **Executable Presence**:
   - `test -x /usr/local/bin/gphsm` &rarr; exit code 0.
   - `gphsm version` &rarr; outputs valid Go version and commit hash.
2. **Symlink Absence**:
   - `test ! -e /usr/local/bin/hsmutil` &rarr; exit code 0.
   - `which hsmutil` &rarr; returns exit code 1 (`not found`).
3. **User Account Stability**:
   - `id hsmutil` &rarr; `uid=1000(hsmutil) gid=100(users) groups=100(users),20(dialout),5(tty),999(storage)`.
