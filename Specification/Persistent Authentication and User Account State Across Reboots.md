---
title: "Technical Specification: Persistent Authentication & User Account State Across Reboots"
status: Approved
date: 2026-11-12
tags:
  - specification
  - architecture
  - yocto
  - security
  - authentication
  - pam
  - storage
references:
  - "[[Architecture/Decisions/ADR-0032 - Targeted Authentication State Persistence to data-etc with Ephemeral etc OverlayFS|ADR-0032: Targeted Authentication State Persistence to /data/etc with Ephemeral /etc OverlayFS]]"
  - "[[Architecture/Decisions/ADR-0033 - Real-Time Atomic Credential Synchronization and Reboot-Shutdown Flushes|ADR-0033: Real-Time Atomic Credential Synchronization and Reboot/Shutdown Flushes]]"
  - "[[Plan/Persistent Authentication and User Account State Across Reboots Implementation Plan|Implementation Plan: Persistent Authentication & User Account State Across Reboots]]"
  - "[[features/yocto-persist-auth-state|Feature Specification: Persistent Authentication & User Account State Across Reboots]]"
  - "[[Architecture/07 - Air-Gapped RPi HSM OS Architecture|07 - Air-Gapped RPi HSM OS Architecture]]"
---

# Technical Specification: Persistent Authentication & User Account State Across Reboots

**Document ID**: `SPEC-2026-YOCTO-PERSIST-AUTH`  
**Target Systems**: `hsm-yocto-offgrid-rpi` (meta-hsm-os)  
**Status**: APPROVED  
**Author**: `coding-specify`  

---

## 1. Executive Summary & Problem Motivation

The **nRF52840 Hardware Security Module (HSM)** air-gapped operating system (`hsm-yocto-offgrid-rpi`) uses an immutable root filesystem (`read-only-rootfs`) with an ephemeral RAM-only tmpfs overlay mounted over `/etc` (`lowerdir=/etc,upperdir=/run/overlay/etc-upper,workdir=/run/overlay/etc-work`). This architecture guarantees that runtime configuration changes do not contaminate the underlying read-only rootfs image, preserving A/B partition integrity during RAUC updates.

However, because `/run` is an in-memory `tmpfs`, all writes to `/etc`—specifically user authentication and credential databases (`/etc/shadow`, `/etc/passwd`, `/etc/group`, `/etc/gshadow`)—disappear upon system reboot. As a consequence:
1. When an operator updates their password via Linux PAM on first login, the change is written only to `/run/overlay/etc-upper/shadow`.
2. Upon power-cycling or rebooting the appliance, the RAM tmpfs overlay is discarded.
3. `/etc/shadow` reverts to the default template on the read-only rootfs, where the `hsmutil` account password has expired (`sp_lstchg=0`).
4. The operator is repeatedly forced to change their password on every boot, preventing persistent unattended operation and disrupting automated workflows.

A persistent solution is required that maintains the read-only rootfs and RAUC update isolation while ensuring that operator password updates, account states, and group memberships persist reliably across reboots.

---

## 2. Architecture & Design Principles

### 2.1 Targeted Auth Persistence Strategy

Instead of making the entire `/etc` directory persistent (which risks configuration drift and conflicts across RAUC A/B operating system upgrades), the appliance implements a **Targeted Authentication State Persistence** architecture.

- `/etc` remains an ephemeral RAM-only tmpfs overlay, guaranteeing immutability for system configuration files.
- The persistent ext4 partition (`/data`, mounted by filesystem label `LABEL=data`) hosts a secure directory `/data/etc` (mode `0700`, owner `root:root`).
- Only authentication and user databases are synchronized:
  - `/data/etc/shadow`
  - `/data/etc/passwd`
  - `/data/etc/group`
  - `/data/etc/gshadow`

```mermaid
flowchart TD
    subgraph BootTime["Boot & Initialization Phase"]
        direction TB
        FSTAB["fstab: mount LABEL=data on /data"] --> MOUNT_CHK{"Is /data mounted?"}
        MOUNT_CHK -->|Yes| DIR_INIT["Ensure /data/etc exists (mode 0700 root:root)"]
        DIR_INIT --> SEED_CHK{"Does /data/etc/shadow exist & non-empty?"}
        SEED_CHK -->|No (First Boot)| SEED["Seed /data/etc/ from /etc base template"]
        SEED_CHK -->|Yes (Subsequent Boot)| SANITY["Validate file integrity"]
        SANITY --> RESTORE["Restore /data/etc/* into active /etc overlay<br/>(Apply strict ownership & permissions)"]
        SEED --> RESTORE
    end

    subgraph Runtime["Runtime State & Change Synchronization"]
        direction TB
        PAM["PAM / passwd / useradd modifies /etc/shadow or /etc/passwd"] --> OVL["Written to /run/overlay/etc-upper/"]
        OVL --> PATH["systemd path unit: hsm-persist-auth.path"]
        PATH -->|Trigger| SVC["systemd service: hsm-persist-auth.service"]
        SVC --> SCRIPT["/usr/local/bin/hsm-sync-auth.sh"]
        SCRIPT --> ATOMIC["Write to /data/etc/.<file>.tmp.$$<br/>Rotate backup to .bak<br/>Atomic rename to /data/etc/<file><br/>Execute sync flush"]
    end

    subgraph Shutdown["Shutdown / Reboot Phase"]
        direction TB
        SHUTDOWN["systemd-halt / reboot.target"] --> FLUSH["ExecStop flush in hsm-persist-auth.service"]
        FLUSH --> DISK["MMC Storage Flush (/data synced)"]
    end

    BootTime --> Runtime
    Runtime --> Shutdown
```

---

## 3. Functional Requirements

### 3.1 Filesystem Mount Standardization (FR-1)

- **FR-1.1**: The system `/etc/fstab` deployed via [recipes-core/base-files/files/fstab](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi/meta-hsm-os/recipes-core/base-files/files/fstab) shall reference the persistent data partition and home partition by volume label rather than static device node indices:
  ```text
  LABEL=data    /data    ext4    defaults                   0    2
  LABEL=homefs  /home    ext4    defaults,x-systemd.growfs  0    2
  ```
- **FR-1.2**: If `/data` or `/home` is already mounted via systemd mount generators or init scripts, mount operations shall be idempotent.

### 3.2 Boot-Time Initialization & Restoration (FR-2)

- **FR-2.1**: An early boot service (integrated into [hsm-volatile-storage.service](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi/meta-hsm-os/recipes-core/volatile-storage/files/hsm-volatile-storage.service) or a dedicated unit ordered `After=data.mount`, `Before=systemd-sysusers.service systemd-logind.service`) shall execute the initialization script `hsm-setup-volatile.sh`.
- **FR-2.2**: The initialization script shall verify that `/data` is mounted and writable.
- **FR-2.3**: The script shall ensure `/data/etc` exists with ownership `root:root` and permission mode `0700` (`drwx------`).
- **FR-2.4 (First-Boot Seeding)**:
  - If `/data/etc/shadow` is missing or has a file size of 0 bytes, the script shall seed the persistent store by copying `/etc/shadow`, `/etc/passwd`, `/etc/group`, and `/etc/gshadow` from the lower rootfs template to `/data/etc/`.
  - Proper permissions shall be established immediately on the seeded files:
    - `/data/etc/shadow`: mode `0400` (or `0640`), `root:shadow`
    - `/data/etc/gshadow`: mode `0400` (or `0640`), `root:shadow`
    - `/data/etc/passwd`: mode `0644`, `root:root`
    - `/data/etc/group`: mode `0644`, `root:root`
- **FR-2.5 (Subsequent Boot Restoration)**:
  - If `/data/etc/shadow` exists and contains valid data, the script shall copy `/data/etc/{shadow,passwd,group,gshadow}` into the active `/etc` tmpfs overlay.
  - The script shall strictly enforce ownership and permissions on the restored `/etc/` files:
    - `chmod 0400 /etc/shadow && chown root:shadow /etc/shadow`
    - `chmod 0400 /etc/gshadow && chown root:shadow /etc/gshadow`
    - `chmod 0644 /etc/passwd && chown root:root /etc/passwd`
    - `chmod 0644 /etc/group && chown root:root /etc/group`
- **FR-2.6 (Corruption Fallback)**:
  - If any persistent file in `/data/etc/` is 0 bytes or corrupted, the service shall log a warning to the kernel journal and restore the base template from the read-only rootfs layer, preventing lockout.

### 3.3 Real-Time Synchronization Mechanism (FR-3)

- **FR-3.1**: A systemd path unit `hsm-persist-auth.path` shall monitor file modifications on the active authentication databases:
  ```ini
  [Unit]
  Description=Monitor Authentication and Account Database Changes
  DefaultDependencies=no
  After=data.mount

  [Path]
  PathModified=/etc/shadow
  PathModified=/etc/passwd
  PathModified=/etc/group
  PathModified=/etc/gshadow
  Unit=hsm-persist-auth.service

  [Install]
  WantedBy=multi-user.target
  ```
- **FR-3.2**: A systemd service unit `hsm-persist-auth.service` shall be triggered by `hsm-persist-auth.path`:
  ```ini
  [Unit]
  Description=Synchronize Authentication Database to Persistent Storage
  DefaultDependencies=no
  After=data.mount
  Conflicts=shutdown.target

  [Service]
  Type=oneshot
  ExecStart=/usr/local/bin/hsm-sync-auth.sh
  ExecStop=/usr/local/bin/hsm-sync-auth.sh
  RemainAfterExit=yes

  [Install]
  WantedBy=multi-user.target
  ```
- **FR-3.3**: The synchronization script `/usr/local/bin/hsm-sync-auth.sh` shall:
  1. Verify `/data` is mounted and writable.
  2. Ensure `/data/etc` exists with mode `0700`.
  3. For each file in `shadow`, `passwd`, `group`, `gshadow`:
     - Validate that the source file in `/etc` exists and is non-empty.
     - Write the source file to a temporary file in `/data/etc/` (e.g. `/data/etc/.<file>.tmp.$$`).
     - Set ownership and file mode matching requirements (`0400 root:shadow` for shadow/gshadow, `0644 root:root` for passwd/group).
     - If an existing target `/data/etc/<file>` exists, create a backup `/data/etc/<file>.bak`.
     - Atomically rename the temporary file over `/data/etc/<file>` via `mv -f`.
  4. Invoke `sync` to immediately commit dirty blocks to non-volatile MMC storage.
- **FR-3.4 (Shutdown Hook)**:
  - `ExecStop=/usr/local/bin/hsm-sync-auth.sh` ensures that during system halt, poweroff, or reboot, all pending credential changes are committed before unmounting `/data`.

### 3.4 Persistent User Home Directory Alignment (FR-4)

- **FR-4.1**: In [hsm-setup-volatile.sh](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi/meta-hsm-os/recipes-core/volatile-storage/files/hsm-setup-volatile.sh), the script must check if `/home` is already mounted (from partition `LABEL=homefs`). If `/home` is an active filesystem mount, the script shall NOT mount a `tmpfs` over `/home/hsmutil`, ensuring that operator home directory state (`.bashrc`, `.keymap`, `.wizard_completed`) also persists reliably.

---

## 4. Security & Hardening Requirements

- **NFR-1 (Least Privilege & Confidentiality)**:
  - `/data/etc` directory permissions must remain strictly `0700` (`drwx------ root:root`), preventing non-root users from listing or traversing stored password hashes.
  - `/data/etc/shadow` and `/data/etc/gshadow` must have mode `0400` (or `0640`), readable only by `root` or `shadow` group.
- **NFR-2 (Power-Fail Safety & Atomicity)**:
  - Synchronization must never write directly to the target file. Using temporary files and atomic `rename` (`mv`) guarantees that an abrupt power loss cannot result in an empty or partially written `/data/etc/shadow`.
  - Maintaining `.bak` copies ensures that even if filesystem recovery truncates a file, the previous valid credential state can be restored.
- **NFR-3 (Read-Only Rootfs Integrity & Update Isolation)**:
  - The underlying rootfs partition (`/dev/mmcblk0p2` or `/dev/mmcblk0p3`) remains 100% read-only.
  - RAUC updates to Slot A or Slot B do not overwrite user-created passwords or accounts, as `/data/etc` is stored on the shared data partition.

---

## 5. Affected Repositories & Files

| Component | File Path | Scope of Modification |
| :--- | :--- | :--- |
| **Base Files** | `recipes-core/base-files/files/fstab` | Update `/data` and `/home` mount entries to use `LABEL=data` and `LABEL=homefs`. |
| **Volatile Storage** | `recipes-core/volatile-storage/files/hsm-setup-volatile.sh` | Add boot-time seeding and restoration of `/data/etc/{shadow,passwd,group,gshadow}`. Avoid mounting `tmpfs` on `/home/hsmutil` if persistent `/home` is mounted. |
| **Auth Persist Script** | `recipes-core/volatile-storage/files/hsm-sync-auth.sh` | **New script**: Implements atomic sync of auth files from `/etc` to `/data/etc` with permissions enforcement and `sync` flush. |
| **Auth Persist Service** | `recipes-core/volatile-storage/files/hsm-persist-auth.service` | **New service**: Executes `hsm-sync-auth.sh` on changes and shutdown. |
| **Auth Persist Path** | `recipes-core/volatile-storage/files/hsm-persist-auth.path` | **New path unit**: Monitors `/etc/{shadow,passwd,group,gshadow}` for modifications. |
| **Volatile Storage Recipe** | `recipes-core/volatile-storage/volatile-storage.bb` | Install `hsm-sync-auth.sh`, `hsm-persist-auth.service`, and `hsm-persist-auth.path`. Enable path unit via systemd class. |

---

## 6. Edge Cases & Mitigations

| Edge Case | Impact | Mitigation Strategy |
| :--- | :--- | :--- |
| **Power Loss During Password Change** | File corruption or zero-byte file in `/data/etc/`. | Atomic `mv` replaces file metadata in single transaction; `sync` commits to disk; `.bak` files preserved for fallback. |
| **Corrupted Persistent Password File** | System boot cannot authenticate users. | Restoration logic verifies file size > 0. If file is corrupted or empty, falls back to base template from rootfs, permitting emergency console recovery. |
| **Data Partition Missing or Unformatted** | `/data` mount fails. | Service logs warning; system proceeds with default rootfs credentials in volatile RAM mode; console remains functional. |
| **Concurrent Writes to Password DB** | Race condition between `passwd` and sync. | Systemd path unit serializes service executions; temporary PID-based file naming prevents write collision. |
| **User Adds Multiple Accounts** | `/etc/passwd` and `/etc/shadow` change simultaneously. | Systemd path unit aggregates multiple file events into a single sync execution. |

---

## 7. Verifiable Acceptance Criteria

- [ ] **AC-1 (Fstab Label-Based Mounting)**:
  - In booted appliance, `mount | grep /data` shows `/data` mounted from partition labeled `data`.
  - `mount | grep /home` shows `/home` mounted from partition labeled `homefs`.
- [ ] **AC-2 (First Boot Seeding & Permissions)**:
  - On a newly partitioned device, `/data/etc` is automatically created with mode `0700` (`root:root`).
  - `/data/etc/{shadow,passwd,group,gshadow}` are seeded from rootfs template.
  - `/data/etc/shadow` has mode `0400` (or `0640`) and ownership `root:shadow`.
- [ ] **AC-3 (Password Change Real-Time Sync)**:
  - Changing operator password via `passwd` immediately triggers `hsm-persist-auth.path` and `hsm-persist-auth.service`.
  - `/data/etc/shadow` reflects the new password hash within 2 seconds of change.
  - `/data/etc/shadow.bak` contains the previous hash.
- [ ] **AC-4 (Reboot Persistence)**:
  - Operator changes password and reboots the appliance (`reboot`).
  - Upon reboot, operator logs in using the **new** password. The default initial password is rejected.
  - PAM does NOT prompt for password reset on subsequent boots (`sp_lstchg` remains valid).
  - Setup wizard does not re-prompt for setup once completed (`.wizard_completed` persists in `/home/hsmutil`).
- [ ] **AC-5 (Corruption Fallback)**:
  - Truncating `/data/etc/shadow` to 0 bytes and rebooting results in clean recovery from base template without hanging boot.
