---
title: "Feature Specification: HSM Device Initialization and Dual-Mode Protocol Compatibility"
tags:
  - feature
  - specification
  - authentication
  - shamir
  - pin
  - dual-mode
  - gphsm
  - firmware
  - protocol
  - cme-errors
  - dfu
created: 2026-11-06
updated: 2026-11-06
status: "Phase 3 (Plan Approved & Formalized)"
target_branches:
  root: bugfix/init-dual-mode-protocol-compat
  cli: bugfix/init-dual-mode-protocol-compat
  firmware: bugfix/init-dual-mode-protocol-compat
  obsidian: main
adrs:
  - "[[Architecture/Decisions/ADR-0019 - HSM Device Initialization and Dual-Mode Protocol Compatibility|ADR-0019: HSM Device Initialization and Dual-Mode Protocol Compatibility]]"
  - "[[Architecture/Decisions/ADR-0016 - Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)|ADR-0016: Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)]]"
  - "[[Architecture/Decisions/ADR-0015 - HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS11 Equivalence|ADR-0015: HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS#11 Equivalence]]"
  - "[[Architecture/Decisions/ADR-0006 - AT-Protocol DFU Chunk Streaming and Deferred MCUboot Swap|ADR-0006: AT-Protocol DFU Chunk Streaming and Deferred MCUboot Swap]]"
references:
  - "[[Plan/HSM Device Initialization and Dual-Mode Protocol Compatibility Implementation Plan|Implementation Plan: HSM Device Initialization and Dual-Mode Protocol Compatibility]]"
  - "[[investigations/2026-11-06 - HSM Device Initialization Failure|Issue Assessment: HSM Device Initialization Failure (+CME ERROR: 8)]]"
  - "[[00 - Index|Vault Index]]"
---

# Feature Specification: HSM Device Initialization and Dual-Mode Protocol Compatibility

> [!NOTE]
> **Lifecycle Phase**: Phase 3 (Plan Formalized & Approved &rarr; Development Ready)  
> **Working Branch**: `bugfix/init-dual-mode-protocol-compat` across Root, `hsm-cli`, and `hsm-ncs-firmware`  
> **Related Architecture Decision Records**:
> - [[Architecture/Decisions/ADR-0019 - HSM Device Initialization and Dual-Mode Protocol Compatibility|ADR-0019: HSM Device Initialization and Dual-Mode Protocol Compatibility]]  
> - [[Architecture/Decisions/ADR-0016 - Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)|ADR-0016: Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)]]  
> **Source Issue Assessment**:
> - [[investigations/2026-11-06 - HSM Device Initialization Failure|Issue Assessment: 2026-11-06 - HSM Device Initialization Failure (+CME ERROR: 8)]]  
> **Target Repositories & Working Branches**:
> - Root Workspace: `bugfix/init-dual-mode-protocol-compat`
> - [`hsm-cli`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli): `bugfix/init-dual-mode-protocol-compat`
> - [`hsm-ncs-firmware`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware): `bugfix/init-dual-mode-protocol-compat`
> - Knowledge Vault: `hsm-obsidian` (`main`)

---

## 1. Feature Overview & Scope

This feature resolves device initialization failures and protocol mismatches across the dual-mode Hardware Security Module (HSM) ecosystem. It guarantees robust interoperability between the host administrative CLI (`gphsm` / `hsmutil`), the physical nRF52840 USB Dongle (`PCA10059`), and virtual test mock environments across both PIN mode and Shamir's Secret Sharing threshold mode.

---

## 2. Key Capabilities & Behavioral Contracts

### 2.1 Fallback Compatibility for Legacy Firmware (`AT+INIT=SHAMIR` &rarr; `AT+INIT=1`)
- When the host CLI dispatches `AT+INIT=SHAMIR` and receives `+CME ERROR: 8` (`ErrCommandUnsupported`), the client automatically intercepts the error and falls back to dispatching canonical legacy `AT+INIT=1`.
- Eliminates the previous erroneous fallback to `AT+INIT=THRESHOLD,3,5`.

### 2.2 Response Line Prefix Flexibility (`+INIT:` and `+SHARE:`)
- **Initialization Shares**:
  - `parseShareLine()` accepts response lines prefixed with either `+INIT:` or `+SHARE:`.
  - `InitShamir()` processes all 5 custodian shares regardless of whether the target firmware emits `+INIT: <idx>,<hex>` or `+SHARE: <idx>,<hex>`.
- **Custodian Share Acceptance**:
  - `SubmitShare()` recognizes `+SHARE:`, `+COMMIT:`, and `+WORD:` line prefixes.
  - `parseCommitLine()` parses status lines formatted as `+SHARE: <count>/3 ACCEPTED,WORD="<word>"` and extracts the active commitment word and custodian index.

### 2.3 CME Error Mapping
- `ParseCMEError()` translates the following hardware CME codes into typed domain error sentinels:
  - `Code 41`: `ErrAlreadyConfigured` (`+CME ERROR: 41 (Device already configured)`)
  - `Code 42`: `ErrInvalidAuthMode` (`+CME ERROR: 42 (Invalid authentication mode)`)
  - `Code 103`: `ErrDuplicateShare` (`+CME ERROR: 103 (Duplicate custodian share index)`)

### 2.4 End-to-End Integration Verification
- `scripts/run_integration_tests.sh` executes the public CLI commands `gphsm device init --mode pin` and `gphsm device init --mode shamir` against the mock PTY.

### 2.5 Production Firmware Build & Physical Dongle DFU Upgrade
- Embedded firmware in `hsm-ncs-firmware` is rebuilt for production (`make firmware-build`).
- The signed binary (`zephyr.signed.bin`) is streamed to physical dongles using `gphsm device firmware update --file ...`.

---

## 3. Protocol & State Machine Specifications

### Initialization Sequence State Transition

```mermaid
stateDiagram-v2
    [*] --> Unconfigured: AT+FACTORYRESET or Fresh Device
    Unconfigured --> ConfiguredPIN: AT+INIT=PIN,<pin>
    Unconfigured --> ConfiguredShamir: AT+INIT=SHAMIR (or fallback AT+INIT=1)
    
    state ConfiguredPIN {
        [*] --> LockedPIN: Boot / Autolock
        LockedPIN --> UnlockedPIN: AT+AUTH=<pin>
        UnlockedPIN --> LockedPIN: AT+LOCK / Timeout
        LockedPIN --> LockedPIN: Invalid PIN (retries--)
    }
    
    state ConfiguredShamir {
        [*] --> LockedShamir: Boot / Autolock
        LockedShamir --> UnlockedShamir: 3 Valid AT+SHARE submissions
        UnlockedShamir --> LockedShamir: AT+LOCK / Timeout
        LockedShamir --> LockedShamir: Duplicate Share (+CME ERROR: 103)
    }
    
    ConfiguredPIN --> Unconfigured: AT+RESET / AT+FACTORYRESET
    ConfiguredShamir --> Unconfigured: AT+RESET / AT+FACTORYRESET
```

---

## 4. Test & Verification Matrix

| Area | Test Suite | Command | Verification Target |
| :--- | :--- | :--- | :--- |
| **Unit Tests** | `hsm-cli/internal/domain` | `make cli-test` | `TestATClient_InitShamir` with `+INIT:`, fallback to `AT+INIT=1`, `parseCommitLine`, `ParseCMEError` |
| **Virtual Integration** | `scripts` & `mock_hsm_pty.py` | `make integration-test` | `gphsm device init --mode pin` and `gphsm device init --mode shamir` end-to-end |
| **Hardware Upgrade** | Physical Dongle DFU | `gphsm device firmware update --file ...` | Image chunk streaming, MCUboot swap, post-swap enumeration |
| **Hardware Ceremonies** | Physical Dongle | `gphsm device init --mode pin/shamir` | Complete initialization, PIN unlock, Shamir share emission, factory reset |
