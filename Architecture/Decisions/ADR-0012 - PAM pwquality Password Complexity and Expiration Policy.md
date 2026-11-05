---
title: "ADR-0012: PAM pwquality Password Complexity & Expiration Policy"
status: Accepted
date: 2026-11-05
tags:
  - adr
  - architecture
  - yocto
  - pam
  - security
  - hsmutil
  - shadow
deciders:
  - Embedded Systems Architect
  - Security Firmware Lead
references:
  - "[[features/yocto-boot-visibility-and-password-policy|Feature: Yocto Boot Visibility & Password Policy]]"
  - "[[Specification/Yocto Dev-Prod Boot Visibility, U-Boot Hardening, and Password Policy|Specification: Yocto Dev-Prod Boot Visibility, U-Boot Hardening, and Password Policy]]"
  - "[[Plan/Yocto Dev-Prod Boot Visibility, U-Boot Hardening, and Password Policy Implementation Plan|Implementation Plan: Yocto Dev-Prod Boot Visibility, U-Boot Hardening, and Password Policy]]"
  - "[[Architecture/07 - Air-Gapped RPi HSM OS Architecture|Architecture: Air-Gapped RPi HSM OS]]"
---

# ADR-0012: PAM pwquality Password Complexity & Expiration Policy

## Context & Problem Statement

The **`rpi-z2w-yocto-hsm`** operating system is designed as an air-gapped cryptographic signing appliance. The unprivileged operator account `hsmutil` interacts with cryptographic keys on the nRF52840 HSM Dongle, controls key ceremonies, and executes offline firmware updates.

Prior to this decision:
1. **Insecure Initial Credentials**:
   - The initial password in [`hsmutil-user.bb`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-hsm-os/recipes-core/users/hsmutil-user.bb#L10-L11) was set to `hsmutil`, conflicting with workspace standards requiring `password`.
2. **Absence of Password Complexity Enforcement**:
   - Neither the standard Linux `passwd` utility nor the custom [`first-login-wizard.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-hsm-os/recipes-core/init-wizard/files/first-login-wizard.sh) enforced password complexity. An operator could change the password to a trivial 1-character string, empty string, or simple sequence (e.g., `123456`), undermining appliance security.
3. **Absence of Password Expiration Lifecycle**:
   - Passwords had no maximum lifetime (`PASS_MAX_DAYS -1`), violating compliance standards for cryptographic appliances which require password rotation every 90 days.
4. **Distro Features & Shadow Configuration**:
   - Yocto's `pam` distro feature was not explicitly enabled in [`kas-project.yml`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/kas-project.yml), preventing integration of standard Linux Pluggable Authentication Modules.

---

## Decision Drivers

- **Defense in Depth**: Password complexity must be enforced both at the UI layer (`first-login-wizard.sh`) and at the underlying Linux PAM stack (`/etc/pam.d/common-password`) so that any invocation of `passwd`, `chpasswd`, or useradd is constrained.
- **Strict Compliance Rules**:
  - Minimum 10 characters (`minlen = 10`).
  - At least 1 numeric digit, 1 uppercase letter, and 1 symbol (`dcredit = -1`, `ucredit = -1`, `ocredit = -1`).
  - Prohibition of adjacent repeated characters (`maxrepeat = 1`, e.g., no `aa`, `11`).
  - Prohibition of monotonic sequences (`maxsequence = 2`, e.g., no `abc`, `123`).
  - Root account enforcement (`enforce_for_root`).
- **90-Day Rotation Policy**: Enforce `PASS_MAX_DAYS 90` in both system defaults (`/etc/login.defs`) and user shadow entries (`/etc/shadow`).
- **Volatile Rootfs Compatibility**: All shadow and PAM operations must function seamlessly within the RAM overlay on `/etc` (`tmpfs`/`overlayfs`).

---

## Considered Options

### Option 1: Shell-Only Validation in `first-login-wizard.sh`
- *Mechanism*: Implement validation logic exclusively in the first-login wizard bash script.
- *Evaluation*: Rejected because it leaves standard `passwd` or CLI password changes completely unconstrained.

### Option 2: Basic PAM `pam_unix.so` with Minlen
- *Mechanism*: Rely only on `pam_unix.so minlen=10`.
- *Evaluation*: Rejected because `pam_unix.so` cannot enforce character classes, repeat limits, or sequential character bans.

### Option 3: Dual-Layer Defense via `libpwquality` + PAM Stack + Wizard Pre-Validation (Chosen)
- *Mechanism*:
  1. Add `pam` to `DISTRO_FEATURES` in `kas-project.yml`.
  2. Include `libpwquality` in `hsm-image-base.bb`.
  3. Deploy `/etc/security/pwquality.conf` and `/etc/pam.d/common-password` via `meta-hsm-os/recipes-extended/libpwquality/libpwquality_%.bbappend`.
  4. Configure `/etc/login.defs` with `PASS_MAX_DAYS 90` via `meta-hsm-os/recipes-extended/shadow/shadow_%.bbappend`.
  5. Enhance `first-login-wizard.sh` to provide immediate pre-validation with clear error feedback before calling PAM.

---

## Decision

We adopt **Option 3**:

### 1. Initial Default Password
- Set `hsmutil` user password in `hsmutil-user.bb` to `password` using SHA-512 crypt:
  `$6$saltsalt$qFmFH.bQmmtXzyBY0s9v7Oicd2z4XSIecDzlB5KiA2/jctKu9YterLp8wwnSq.qc.eoxqOmSuNp2xS0ktL3nh/`

### 2. PAM `pwquality.conf` Configuration
```ini
# /etc/security/pwquality.conf
minlen = 10
dcredit = -1
ucredit = -1
ocredit = -1
maxrepeat = 1
maxsequence = 2
difok = 1
enforce_for_root
```

### 3. PAM Common Password (`/etc/pam.d/common-password`)
```text
password requisite pam_pwquality.so retry=3 enforce_for_root
password [success=1 default=ignore] pam_unix.so obscure sha512
password requisite pam_deny.so
password required pam_permit.so
```

### 4. 90-Day Expiration Policy
- `/etc/login.defs`: `PASS_MAX_DAYS 90`
- User record in `/etc/shadow`: Field 5 is set to `90`.

### 5. Wizard Interactive Validation
`first-login-wizard.sh` checks:
1. `length >= 10`
2. `[[ "$pwd" =~ [0-9] ]]`
3. `[[ "$pwd" =~ [A-Z] ]]`
4. `[[ "$pwd" =~ [^a-zA-Z0-9] ]]`
5. Adjacent repeat rejection: `([[ "$pwd" =~ (.)\1 ]])`
6. Monotonic 3-character sequence check across ASCII table.

---

## Consequences

- **Positive**:
  - The appliance meets regulatory cryptographic workstation security standards.
  - Operators cannot set weak or predictable passwords.
  - Initial password aligns with documented workspace default (`password`).
  - Mandatory 90-day password expiration is enforced across reboots in volatile `/etc`.
- **Negative / Neutral**:
  - Operators must select stronger passwords (e.g., `Hsm#92K!pL4`).
  - Inclusion of `pam` and `libpwquality` adds a minor footprint to the root filesystem (~250 KB), well within available flash and RAM budgets.
