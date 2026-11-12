---
title: "Feature: Persistent Authentication & User Account State Across Reboots"
status: Approved
date: 2026-11-12
tags:
  - feature
  - architecture
  - yocto
  - security
  - authentication
  - pam
  - storage
references:
  - "[[Architecture/Decisions/ADR-0032 - Targeted Authentication State Persistence to data-etc with Ephemeral etc OverlayFS|ADR-0032: Targeted Authentication State Persistence to /data/etc with Ephemeral /etc OverlayFS]]"
  - "[[Architecture/Decisions/ADR-0033 - Real-Time Atomic Credential Synchronization and Reboot-Shutdown Flushes|ADR-0033: Real-Time Atomic Credential Synchronization and Reboot/Shutdown Flushes]]"
  - "[[Specification/Persistent Authentication and User Account State Across Reboots|Technical Specification: Persistent Authentication & User Account State Across Reboots]]"
  - "[[Plan/Persistent Authentication and User Account State Across Reboots Implementation Plan|Implementation Plan: Persistent Authentication & User Account State Across Reboots]]"
  - "[[Architecture/07 - Air-Gapped RPi HSM OS Architecture|07 - Air-Gapped RPi HSM OS Architecture]]"
---

# Feature: Persistent Authentication & User Account State Across Reboots

## 1. Feature Summary

This feature resolves credential amnesia on the **air-gapped Raspberry Pi HSM OS** ([`hsm-yocto-offgrid-rpi`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi)), ensuring that operator password updates, account states, and group memberships persist seamlessly across reboots while strictly preserving root filesystem immutability (`read-only-rootfs`) and RAUC update isolation:

1. **Filesystem Mount Standardization**: Replaces static device node paths in `/etc/fstab` with volume labels (`LABEL=data` on `/data`, `LABEL=homefs` on `/home`), ensuring deterministic mount behavior regardless of MMC driver enumeration order.
2. **Targeted Authentication State Persistence (`/data/etc`)**: Retains `/etc` as an ephemeral RAM-only tmpfs overlay (`/run/overlay/etc-upper`), while storing only user authentication and account databases (`shadow`, `passwd`, `group`, `gshadow`) on the persistent data partition under `/data/etc` (mode `0700`, owner `root:root`).
3. **Early Boot Seeding & Restoration**: On early boot, `hsm-setup-volatile.sh` automatically seeds `/data/etc/` from the rootfs template on first boot, and restores persistent credentials into the active `/etc` overlay on subsequent boots with strict ownership (`root:shadow` for shadow/gshadow, `root:root` for passwd/group) and permissions (`0400`/`0644`). Corrupt files fall back safely to the rootfs template.
4. **Real-Time Atomic Synchronization**: Deploys systemd path unit `hsm-persist-auth.path` and service unit `hsm-persist-auth.service` monitoring `/etc/{shadow,passwd,group,gshadow}`. Changes trigger `/usr/local/bin/hsm-sync-auth.sh`, which performs PID-staged writes (`.<file>.tmp.$$`), rotates `.bak` backups, executes atomic POSIX renames, and commits dirty blocks to physical flash via `sync`.
5. **Clean Shutdown Flush**: Declares `Conflicts=shutdown.target` and `ExecStop=/usr/local/bin/hsm-sync-auth.sh`, ensuring all pending in-memory credential changes are flushed to persistent media before `/data` is unmounted during reboot or poweroff.
6. **Persistent Home Directory Retention**: Checks if `/home` is mounted from `LABEL=homefs`, bypassing ephemeral tmpfs mounting over `/home/hsmutil` to preserve operator shell preferences (`.bashrc`), keyboard layout (`.keymap`), and wizard completion markers (`.wizard_completed`).

---

## 2. User Stories & Acceptance Criteria

### User Story 1: Reboot Credential Persistence
> *As a cryptographic custodian or appliance operator, I want my password update from the first-login wizard to persist across reboots, so that I can log in using my chosen password without being repeatedly forced to reset an expired password on every power cycle.*

- **AC-1.1**: The initial password change executed during first boot persists across reboots.
- **AC-1.2**: On subsequent boots, the operator successfully logs in with the newly established password; the factory default password is rejected.
- **AC-1.3**: Linux PAM does not prompt for password expiration on subsequent boots (`sp_lstchg` remains valid).
- **AC-1.4**: First-login setup wizard (`first-login-wizard.sh`) does not re-trigger once completed (`~/.wizard_completed` persists in `/home/hsmutil`).

### User Story 2: Power-Fail Safe Atomic Synchronization
> *As a security administrator, I want credential modifications to be committed atomically to persistent storage with backup rotation, so that unexpected power loss never results in empty, truncated, or corrupt password databases.*

- **AC-2.1**: Runtime password changes (`passwd`) trigger `hsm-persist-auth.path` and `hsm-persist-auth.service` in real time.
- **AC-2.2**: Updates are staged in temporary files (`.<file>.tmp.$$`), given strict permissions (`0400 root:shadow` / `0644 root:root`), backed up to `.bak`, and atomically renamed (`mv -f`).
- **AC-2.3**: Kernel filesystem buffers are committed via `sync`.
- **AC-2.4**: Halting or rebooting triggers `ExecStop` to flush pending state before unmounting.

### User Story 3: System Upgrade & Immutability Protection
> *As an embedded release engineer, I want system configuration files in /etc to remain ephemeral in RAM tmpfs, so that RAUC A/B operating system upgrades never experience configuration drift or conflicts.*

- **AC-3.1**: The root filesystem (`/dev/mmcblk0p2` / `/dev/mmcblk0p3`) remains 100% read-only.
- **AC-3.2**: `/etc` remains backed by `/run/overlay/etc-upper` in RAM tmpfs.
- **AC-3.3**: Only user accounts and credentials persist in `/data/etc`; no other system configuration files in `/etc` are persisted, preventing drift across RAUC updates.
- **AC-3.4**: If `/data/etc/shadow` is missing or corrupted, the system falls back safely to the clean rootfs template.

---

## 3. Architecture & Traceability Matrix

| Capability Track | Architecture Decision Record (ADR) | Target Components |
| :--- | :--- | :--- |
| **Targeted Persistence & OverlayFS** | [[Architecture/Decisions/ADR-0032 - Targeted Authentication State Persistence to data-etc with Ephemeral etc OverlayFS\|ADR-0032]] | `fstab`, `hsm-setup-volatile.sh`, `/data/etc` |
| **Real-Time Atomic Sync & Shutdown Flush** | [[Architecture/Decisions/ADR-0033 - Real-Time Atomic Credential Synchronization and Reboot-Shutdown Flushes\|ADR-0033]] | `hsm-persist-auth.path`, `hsm-persist-auth.service`, `hsm-sync-auth.sh` |
| **Volatile Packaging & Systemd Units** | [[Architecture/Decisions/ADR-0032 - Targeted Authentication State Persistence to data-etc with Ephemeral etc OverlayFS\|ADR-0032]] | `volatile-storage.bb` |
| **Appliance Architecture Integration** | [[Architecture/07 - Air-Gapped RPi HSM OS Architecture\|07 - Air-Gapped RPi HSM OS Architecture]] | OS architecture document, partition table, runtime workflows |
