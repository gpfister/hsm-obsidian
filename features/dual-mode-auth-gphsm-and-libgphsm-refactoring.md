---
title: "Feature Specification: HSM Dual-Mode Authentication, CLI Renaming (gphsm), and PKCS#11 Refactoring (libgphsm)"
tags:
  - feature
  - specification
  - authentication
  - shamir
  - pin
  - gphsm
  - libgphsm
  - pkcs11
  - yocto
  - pwquality
created: 2026-11-05
updated: 2026-11-05
status: "Phase 6 (Post-Merge Complete)"
target_branches:
  root: main (merged from task/dual-mode-workspace-alignment)
  firmware: main (merged from feature/dual-mode-auth)
  cli: main (merged from feature/gphsm-dual-mode)
  libgphsm: main (merged from task/rename-libgphsm-dual-mode)
  obsidian: main
adrs:
  - "[[Architecture/Decisions/ADR-0016 - Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)|ADR-0016: Dual-Mode Authentication Architecture]]"
  - "[[Architecture/Decisions/ADR-0017 - Host CLI Rebranding to gphsm and PKCS11 Refactoring to libgphsm|ADR-0017: Host CLI Rebranding to gphsm and PKCS#11 Refactoring to libgphsm]]"
  - "[[Architecture/Decisions/ADR-0015 - HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS11 Equivalence|ADR-0015: HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS#11 Equivalence]]"
  - "[[Architecture/Decisions/ADR-0014 - Documentation Terminology Generalization to Generic HSM USB Key|ADR-0014: Documentation Terminology Generalization to Generic HSM USB Key]]"
  - "[[Architecture/Decisions/ADR-0012 - PAM pwquality Password Complexity and Expiration Policy|ADR-0012: PAM pwquality Password Complexity and Expiration Policy]]"
references:
  - "[[Plan/HSM Dual-Mode Authentication, CLI Renaming (gphsm), and PKCS11 Refactoring Implementation Plan|Implementation Plan: Dual-Mode Auth, CLI Renaming & PKCS#11 Refactoring]]"
  - "[[Specification/HSM Dual-Mode Authentication, CLI Renaming (gphsm), and PKCS11 Refactoring|Technical Specification: Dual-Mode Auth, CLI Renaming & PKCS#11 Refactoring]]"
  - "[[Development/CLI Reference|CLI Reference (gphsm)]]"
  - "[[00 - Index|Vault Index]]"
---

# Feature Specification: HSM Dual-Mode Authentication, CLI Renaming (gphsm), and PKCS#11 Refactoring (libgphsm)

> [!NOTE]
> **Lifecycle Phase**: Phase 6 (Post-Merge Complete).  
> **Related Architecture Decision Records**:
> - [[Architecture/Decisions/ADR-0016 - Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)|ADR-0016: Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)]]
> - [[Architecture/Decisions/ADR-0017 - Host CLI Rebranding to gphsm and PKCS11 Refactoring to libgphsm|ADR-0017: Host CLI Rebranding to gphsm and PKCS#11 Refactoring to libgphsm]]  
> **Merged Repositories & Commits on `main`**:
> - [`hsm-ncs-firmware`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware): `feature/dual-mode-auth` &rarr; `main` ([`7f516bd`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware))
> - [`libgphsm`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/libgphsm): `task/rename-libgphsm-dual-mode` &rarr; `main` ([`d25adea`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/libgphsm))
> - [`hsm-cli`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli): `feature/gphsm-dual-mode` &rarr; `main` ([`14ea65f`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli))
> - Root Workspace: `task/dual-mode-workspace-alignment` &rarr; `main` ([`2848d50`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm))
> - Knowledge Vault: [`hsm-obsidian`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian) on `main`
> **Verification**: All unit tests and end-to-end integration test suites (`make integration-test`) passed 100% (Phase A, B, C). All working trees clean on `main`.

---

## 1. Feature Overview & Scope

This feature delivers end-to-end architectural alignment across all layers of the Hardware Security Module (HSM) ecosystem, formalizing dual-mode authentication, rebranding the host management CLI to **`gphsm`**, refactoring the Cryptoki provider library to **`libgphsm`**, and aligning password complexity policies with the air-gapped Yocto appliance:

1. **Dual-Mode Cryptographic Authentication**:
   - Explicit `auth_method_t` enumeration in firmware keystore (`NONE`, `PIN`, `SHAMIR`).
   - Extended PIN storage supporting up to 64 bytes.
   - Comprehensive status query (`AT+STATUS?`) exposing both `CONFIGURED` and `METHOD`.
   - Distinct initialization endpoints (`AT+INIT=PIN,<pin>` vs `AT+INIT=SHAMIR`).
   - Strict mutual exclusivity between `AT+AUTH` and `AT+SHARE`.
2. **Host CLI Rebranding to `gphsm`**:
   - Executable, package, and Cobra command hierarchy rebranded to `gphsm`.
   - Preserves `bin/hsmutil` and `bin/hsmutils` relative symlinks for 100% backward compatibility.
   - Embeds a pure-Go Yocto-compliant `pam_pwquality` password validator.
   - Features an interactive Bubble Tea initialization wizard (`gphsm device init`) and auto-detecting unlock (`gphsm unlock`).
   - Enforces homogeneous migration verification (`source.METHOD == target.METHOD`).
   - Purges redundant `p11` subcommands and cleans low-level AT protocol strings from help text.
3. **PKCS#11 Cryptoki Provider Refactoring (`libgphsm`)**:
   - Renames repository directory `hsm-libpkcs11` to `libgphsm`.
   - Renames CMake build targets to emit `libgphsm.dylib`/`.so`/`.dll`.
   - Advertises `CKF_PROTECTED_AUTHENTICATION_PATH` in token flags under Shamir mode.
   - Adapts `C_Login` to recognize external custodian pre-authentication (`AUTH == 1`).
4. **Workspace & Test Harness Alignment**:
   - Updates root `Makefile` orchestration targets and `SUBREPOS`.
   - Updates `mock_hsm_pty.py`, `test_at_client.py`, and `run_integration_tests.sh`.

---

## 2. Protocol & Wire Interface Specifications

### 2.1 AT Command Set Updates

| Command | Arguments | Direction | Expected Response | Description |
| :--- | :--- | :--- | :--- | :--- |
| `AT+STATUS?` | None | Query | `+STATUS: AUTH=<0\|1>,CONFIGURED=<0\|1>,METHOD=<NONE\|PIN\|SHAMIR>,RETRIES=<N>,KEYCOUNT=<K>`<br/>`OK` | Queries hardware authentication state, initialization status, method, and slot usage. |
| `AT+INIT=PIN,<pin>` | `pin`: 6 to 64 ASCII/UTF-8 chars | Action | `OK`<br/>or `+CME ERROR: 4` (Already Configured)<br/>or `+CME ERROR: 5` (Invalid PIN Length) | Initializes device in single-user PIN mode. |
| `AT+INIT=SHAMIR` | None | Action | `+INIT: 1,<share1>`<br/>...<br/>`+INIT: 5,<share5>`<br/>`OK`<br/>or `+CME ERROR: 4` (Already Configured) | Initializes device in 3-of-5 Shamir threshold mode. Emits 5 custodian shares. |
| `AT+AUTH=<pin>` | `pin`: matching initialized PIN | Action | `OK`<br/>or `+CME ERROR: 2` (Incorrect PIN)<br/>or `+CME ERROR: 3` (PIN Blocked)<br/>or `+CME ERROR: 5` (Invalid Method — device in Shamir mode) | Authenticates session in PIN mode. |
| `AT+SHARE=<idx>,<share>` | `idx`: 1..5<br/>`share`: 64-hex string | Action | `OK` (if threshold reached)<br/>or `+SHARE: OK,<remaining>`<br/>or `+CME ERROR: 5` (Invalid Method — device in PIN mode) | Submits custodian share in Shamir mode. |

---

## 3. Host CLI (`gphsm`) Architectural Specification

### 3.1 Command Hierarchy

```text
gphsm [flags]
├── device <subcommand>
│   ├── list (alias: devices)
│   ├── info
│   ├── init [--mode <pin|shamir>] [--pin <pin>]
│   ├── lock
│   ├── reset [--confirm]
│   ├── status
│   ├── migrate --target <serial>
│   └── firmware <info|update>
├── key <subcommand>
│   ├── list
│   ├── generate --slot <N>
│   ├── pubkey --slot <N>
│   ├── sign --slot <N>
│   └── migrate --slot <N> --target <serial>
├── unlock (alias: auth) [--pin <pin>] [--share <idx>,<hex>]
├── echo [message]
└── random [bytes]
```

### 3.2 Yocto-Compliant `pam_pwquality` Validation

To guarantee parity with the air-gapped `hsm-yocto-offgrid-rpi` appliance ([[Architecture/Decisions/ADR-0012 - PAM pwquality Password Complexity and Expiration Policy|ADR-0012]]), `gphsm` validates user passphrases during initialization:
- **Minimum Length**: 10 characters.
- **Character Classes**: Minimum 3 of 4 classes:
  - Upper case (`A-Z`)
  - Lower case (`a-z`)
  - Digits (`0-9`)
  - Symbols / Special (`!@#$%^&*()-_=+[]{}|;:,.<>?`)
- **Repeating Characters**: Maximum 3 consecutive identical characters (rejects `aaaa`, `1111`).
- **Sequences**: Rejects monotonic sequences (`123456`, `abcdef`).

---

## 4. PKCS#11 Middleware (`libgphsm`) Specification

### 4.1 Token Information Flags (`CK_TOKEN_INFO.flags`)

When queried via `C_GetTokenInfo(slotID, &tokenInfo)`:
- `flags |= CKF_TOKEN_INITIALIZED` (if `CONFIGURED == 1`)
- `flags |= CKF_USER_PIN_INITIALIZED` (if `CONFIGURED == 1`)
- If `METHOD == PIN`:
  - `flags |= CKF_LOGIN_REQUIRED`
  - `flags &= ~CKF_PROTECTED_AUTHENTICATION_PATH`
- If `METHOD == SHAMIR`:
  - `flags |= CKF_PROTECTED_AUTHENTICATION_PATH`
  - Standard PKCS#11 consumers know that authentication is executed via an external protected mechanism.

### 4.2 Login Logic (`C_Login`)

```c
CK_RV C_Login(CK_SESSION_HANDLE hSession, CK_USER_TYPE userType,
              CK_UTF8CHAR_PTR pPin, CK_ULONG ulPinLen)
{
    hsm_status_t status;
    if (hsm_get_status(&status) != HSM_OK) {
        return CKR_DEVICE_ERROR;
    }

    /* If device is already authenticated out-of-band, succeed immediately */
    if (status.authenticated) {
        return CKR_OK;
    }

    /* In Shamir mode, in-band PIN submission cannot unlock threshold shares */
    if (status.method == AUTH_METHOD_SHAMIR) {
        LOG_ERR("Device configured with Shamir Secret Sharing. "
                "Unlock device out-of-band using 'gphsm unlock'.");
        return CKR_PIN_LOCKED;
    }

    /* In PIN mode, dispatch AT+AUTH */
    return hsm_authenticate_pin(pPin, ulPinLen);
}
```

---

## 5. Migration Compatibility Matrix

When executing `gphsm device migrate --target <serial>` or `gphsm key migrate --slot <N> --target <serial>`, `gphsm` verifies authentication compatibility:

| Source Device Method | Target Device Method | Migration Permitted? | Behavior |
| :--- | :--- | :--- | :--- |
| `PIN` | `PIN` | **Yes** | Standard key unwrapping and re-encryption transfer. |
| `SHAMIR` | `SHAMIR` | **Yes** | Threshold secret reconciliation and key transfer. |
| `PIN` | `SHAMIR` | **No** | Immediate fail-fast: `Error: incompatible authentication method (source is PIN, target is SHAMIR)`. |
| `SHAMIR` | `PIN` | **No** | Immediate fail-fast: `Error: incompatible authentication method (source is SHAMIR, target is PIN)`. |
| Any | `NONE` (Unconfigured) | **No** | Target must be initialized with the matching authentication method before migration. |
