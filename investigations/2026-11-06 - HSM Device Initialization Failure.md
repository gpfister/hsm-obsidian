---
title: "Issue Assessment: HSM Device Initialization Failure (+CME ERROR: 8)"
tags:
  - investigation
  - issue-assessment
  - hsm
  - hsm-cli
  - gphsm
  - hsmutil
  - firmware
  - at-commands
  - dual-mode
  - shamir
  - pin
  - dfu
created: 2026-11-06
status: Approved / Ready for Implementation
severity: High
investigation_id: e76a30f0-e6c3-45db-9e75-b1c35c6e57f5
source_artifact: "[[Sessions/attachments/e76a30f0-e6c3-45db-9e75-b1c35c6e57f5/issue-assessment.md|Sessions/attachments/e76a30f0-e6c3-45db-9e75-b1c35c6e57f5/issue-assessment.md]]"
affected_components:
  - hsm-cli
  - hsm-ncs-firmware
  - scripts
related_adrs:
  - "[[Architecture/Decisions/ADR-0019 - HSM Device Initialization and Dual-Mode Protocol Compatibility|ADR-0019: HSM Device Initialization and Dual-Mode Protocol Compatibility]]"
  - "[[Architecture/Decisions/ADR-0016 - Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)|ADR-0016: Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)]]"
  - "[[Architecture/Decisions/ADR-0015 - HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS11 Equivalence|ADR-0015: HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS#11 Equivalence]]"
  - "[[Architecture/Decisions/ADR-0006 - AT-Protocol DFU Chunk Streaming and Deferred MCUboot Swap|ADR-0006: AT-Protocol DFU Chunk Streaming & Deferred MCUboot Swap Architecture]]"
related_features:
  - "[[features/hsm-device-initialization-and-dual-mode-protocol-compatibility|HSM Device Initialization and Dual-Mode Protocol Compatibility]]"
  - "[[features/dual-mode-auth-gphsm-and-libgphsm-refactoring|Dual-Mode Authentication, CLI Rebranding (gphsm) & PKCS#11 Refactoring (libgphsm)]]"
  - "[[features/at-protocol-dfu-support|AT-Protocol DFU Support]]"
related_architecture:
  - "[[Architecture/HSM State Machine|HSM State Machine]]"
  - "[[Architecture/USB Composite Stack|USB Composite Stack]]"
---

# Issue Assessment: HSM Device Initialization Failure (+CME ERROR: 8)

> [!NOTE]
> **Investigation Source**: [`issue-assessment.md`](file:///Users/gpfister/.gemini/antigravity/brain/e76a30f0-e6c3-45db-9e75-b1c35c6e57f5/issue-assessment.md)  
> **Archived Artifact**: [[Sessions/attachments/e76a30f0-e6c3-45db-9e75-b1c35c6e57f5/issue-assessment.md|Attachments]]  
> **Investigation Session**: `e76a30f0-e6c3-45db-9e75-b1c35c6e57f5`  
> **Status**: Approved by User / Ready for Implementation  
> **Severity**: High (Blocks device initialization across physical hardware in both PIN and Shamir modes)  
> **Affected Components**: [`hsm-cli`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli), [`hsm-ncs-firmware`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware), [`scripts`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts)  
> **Architecture Linkages**: [[Architecture/Decisions/ADR-0016 - Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)|ADR-0016: Dual-Mode Authentication Architecture]], [[Architecture/Decisions/ADR-0015 - HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS11 Equivalence|ADR-0015: HSM CLI Restructuring]], [[Architecture/Decisions/ADR-0006 - AT-Protocol DFU Chunk Streaming and Deferred MCUboot Swap|ADR-0006: AT DFU Chunk Streaming]], [[features/dual-mode-auth-gphsm-and-libgphsm-refactoring|Dual-Mode Auth Feature Spec]], [[Architecture/HSM State Machine|HSM State Machine]]

---

## 1. Summary & Observed Behavior

When executing `hsmutil device init` (or `gphsm device init`) against a physical nRF52840 HSM dongle (`PCA10059`), the command fails immediately on both authentication modes with `+CME ERROR: 8 (Command unsupported)`:

1. **Shamir Secret Sharing Mode (Choice 2)**:
   ```text
   Select Authentication Mode:
     [1] PIN Mode (Password-Protected)
     [2] Shamir Secret Sharing (3-of-5 Threshold)
   Enter choice [1 or 2]: 2
   Error: threshold ceremony failed: threshold initialization failed on HSM: +CME ERROR: 8 (Command unsupported)
   ```
2. **PIN Mode (Choice 1)**:
   ```text
   Enter New HSM PIN (min 10 chars, uppercase, lowercase, digit, special):
   > ••••••••••
   Confirm HSM PIN:
   > ••••••••••
   Error: PIN initialization failed on HSM: +CME ERROR: 8 (Command unsupported)
   ```

Pre-initialization queries such as `hsmutil device info` and `hsmutil device status` succeed, but initial configuration fails in both modes.

---

## 2. Root Cause Analysis

Thorough investigation across [`hsm-cli`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli), [`hsm-ncs-firmware`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware), [`scripts`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts), and physical device diagnostics revealed **six interrelated root causes**, comprising one hardware deployment gap and five protocol contract/testing discrepancies:

```mermaid
flowchart TD
    A["hsmutil device init"] --> B{"Query AT+STATUS?"}
    B -->|"Legacy: +STATUS: AUTH=0,RETRIES=3,KEYCOUNT=0"| C["hsm-cli fallback infers unconfigured"]
    
    C --> D{"Selected Mode"}
    
    D -->|"PIN Mode"| E["Dispatch AT+INIT=PIN,&lt;pin&gt;"]
    E --> F["Dongle (Old FW) only recognizes AT+INIT=1"]
    F --> G["Firmware returns +CME ERROR: 8 (Command unsupported)"]
    
    D -->|"Shamir Mode"| H["Dispatch AT+INIT=SHAMIR"]
    H --> F
    G --> I["Fallback: Dispatch AT+INIT=THRESHOLD,3,5"]
    I --> F
    F --> J["Firmware returns +CME ERROR: 8 (Command unsupported)"]
    
    subgraph Latent Protocol Bugs (on Updated Firmware / Mock)
        K["Firmware emits +INIT: &lt;idx&gt;,&lt;hex&gt;"] -.->|Mismatch| L["at_client.go only parses +SHARE:"]
        L -.->|Empty shares error| M["Triggers broken AT+INIT=THRESHOLD fallback"]
        M -.->|Device already initialized| N["Device returns +CME ERROR: 41 (Already Configured)"]
        N -.->|Missing from at_errors.go| O["Displayed as Unknown hardware error"]
    end
```

### 2.1 Primary Root Cause: Physical Dongle Running Stale Pre-Dual-Mode Firmware
- **Diagnostic Finding**: Direct raw serial probing of the physical dongle at [`/dev/cu.usbmodemF7820A1E3B163`](file:///dev/cu.usbmodemF7820A1E3B163) revealed the firmware currently running on the device:
  - `AT+STATUS?` returned: `+STATUS: AUTH=0,RETRIES=3,KEYCOUNT=0\r\nOK\r\n` (lacking `CONFIGURED` and `METHOD`).
  - `AT+INIT=1` returned: 5 shares and `OK` (the legacy Shamir syntax).
  - `AT+INIT=PIN,...` and `AT+INIT=SHAMIR` returned: `\r\n+CME ERROR: 8\r\n` (`CME_ERR_COMMAND_UNSUPPORTED`).
- **Explanation**: Dual-mode authentication was committed to [`hsm-ncs-firmware`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware) in commit `f30484f` ([`at_commands.c#L223-L285`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware/firmware/lib/src/at_commands.c#L223-L285)) conforming to [[Architecture/Decisions/ADR-0016 - Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)|ADR-0016]]. However, the physical USB dongle was never flashed with this updated firmware. It continues running the initial pre-dual-mode release that only recognizes `AT+INIT=1` and defaults all unrecognized commands to `CME_ERR_COMMAND_UNSUPPORTED` (code 8) at [`at_commands.c#L850`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware/firmware/lib/src/at_commands.c#L850).

### 2.2 Secondary Root Cause: Protocol Response Prefix Mismatch on Shamir Init (`+INIT:` vs `+SHARE:`)
- **Diagnostic Finding**:
  - In [`at_commands.c#L272`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware/firmware/lib/src/at_commands.c#L272) and [`mock_hsm_pty.py#L244`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts/mock_hsm_pty.py#L244), the HSM emits shares formatted as:
    ```text
    +INIT: 1,6CC765B8123E8D9E8D70A1FACA4E674FA5802110FC335C4546A2B639454A6BC4
    ```
  - In [`at_client.go#L581`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli/internal/domain/at_client.go#L581) and [`at_client.go#L615`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli/internal/domain/at_client.go#L615), `InitShamir()` and `ThresholdInit()` only parse lines prefixed with `+SHARE:`.
  - In [`parseShareLine` in at_client.go#L723`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli/internal/domain/at_client.go#L723), the parser strictly trims prefix `+SHARE:`.
- **Impact**: Even when running against updated firmware or the mock PTY, `InitShamir()` fails with `no custodian shares returned by HSM`.

### 2.3 Secondary Root Cause: Erroneous Fallback and Cascade Lockout (`AT+INIT=THRESHOLD,3,5`)
- In [`ceremony_service.go#L107`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli/internal/usecases/ceremony_service.go#L107) and [`at_client.go#L573`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli/internal/domain/at_client.go#L573), when `InitShamir()` fails, `hsm-cli` triggers a fallback to `client.ThresholdInit(3, 5)` which sends `AT+INIT=THRESHOLD,3,5`.
- `AT+INIT=THRESHOLD,...` is not implemented in either firmware or mock PTY.
- Furthermore, because `AT+INIT=SHAMIR` already succeeded and set `CONFIGURED=1` on the device before `hsm-cli` discarded the `+INIT:` responses, sending `AT+INIT=THRESHOLD,3,5` causes the HSM to reject the request with `+CME ERROR: 41` (`CME_ERR_ALREADY_CONFIGURED`).

### 2.4 Secondary Root Cause: Missing CME Error Mappings in CLI
- In [`at_errors.go#L64-L85`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli/internal/domain/at_errors.go#L64-L85), `ParseCMEError(code)` only maps codes 1 through 8.
- Dual-mode error codes defined in [`at_commands.h#L21-L23`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware/firmware/lib/include/hsm/at_commands.h#L21-L23):
  - `CME_ERR_ALREADY_CONFIGURED = 41`
  - `CME_ERR_INVALID_AUTH_MODE = 42`
  - `CME_ERR_DUPLICATE_SHARE = 103`
  are unmapped and displayed as generic `Unknown hardware error`.

### 2.5 Secondary Root Cause: Protocol Response Prefix Mismatch on Share Submission (`AT+SHARE=`)
- In [`at_commands.c#L362`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware/firmware/lib/src/at_commands.c#L362) and [`mock_hsm_pty.py#L292`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts/mock_hsm_pty.py#L292), `AT+SHARE=<idx>,<share>` emits:
  `+SHARE: <count>/3 ACCEPTED,WORD="<word>"`
- In [`at_client.go#L651`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli/internal/domain/at_client.go#L651), `SubmitShare()` looks exclusively for `+COMMIT:` or `+WORD:`, failing with `missing +COMMIT: line in device response`.

### 2.6 Secondary Root Cause: Integration Test Harness Gap
- In [`scripts/run_integration_tests.sh#L147-L159`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts/run_integration_tests.sh#L147-L159), tests configured the mock HSM by injecting raw AT commands over a direct Python serial socket (`b'AT+INIT=PIN,123456\r\n'` and `b'AT+INIT=SHAMIR\r\n'`) rather than invoking `gphsm device init`. This concealed the response prefix mismatches from automated CI verification.

---

## 3. Proposed Solutions

### Option 1: Full Protocol Harmonization & Automated DFU Device Upgrade (Recommended)

1. **Protocol Alignment in `hsm-cli`**:
   - In [`at_client.go`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli/internal/domain/at_client.go):
     - Update `InitShamir()` and `ThresholdInit()` to accept both `+INIT:` and `+SHARE:` line prefixes.
     - Update `parseShareLine()` to strip either `+SHARE:` or `+INIT:` prefixes.
     - Update `SubmitShare()` and `parseCommitLine()` to parse `+SHARE: ... WORD=...` in addition to `+COMMIT:` and `+WORD:`.
     - Remove the non-standard `AT+INIT=THRESHOLD,3,5` fallback and instead fall back to `AT+INIT=1` (the legacy Shamir syntax) if `AT+INIT=SHAMIR` returns `CME_ERR_COMMAND_UNSUPPORTED`.
   - In [`at_errors.go`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli/internal/domain/at_errors.go):
     - Add explicit mappings for `CMEError` codes 41 (`ErrAlreadyConfigured`), 42 (`ErrInvalidAuthMode`), and 103 (`ErrDuplicateShare`).
2. **Update Integration Test Harness**:
   - In [`scripts/run_integration_tests.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts/run_integration_tests.sh):
     - Add end-to-end tests for `gphsm device init --mode pin` and `gphsm device init --mode shamir`.
3. **Physical Dongle Upgrade via DFU**:
   - Rebuild production firmware: `make firmware-build`.
   - Stream the signed binary to the physical dongle over USB CDC-ACM DFU:
     `gphsm device firmware update --file hsm-ncs-firmware/build-prod/app/zephyr/zephyr.signed.bin`.
   - Verify active slot reports new hash and version.

**Pros**:
- Completely resolves the reported issue on both physical hardware and virtual test environments.
- Fixes latent protocol bugs that would otherwise block Shamir mode ceremony and unlock even after flashing.
- Enforces strict backward compatibility with legacy `AT+INIT=1`.
- Closes the CI testing gap so regressions are caught automatically.

**Cons**:
- Requires flashing the physical dongle via DFU (requires physical dongle to reboot/swap into MCUboot).

---

### Option 2: Flash Physical Dongle Only (Without CLI Fixes)

Flash the dongle with the newly compiled firmware from commit `f30484f`.

**Pros**:
- Minimal immediate intervention.
- Resolves PIN mode initialization immediately.

**Cons**:
- **Critical Failure**: Shamir mode will still fail because `hsm-cli` will reject the `+INIT:` response lines emitted by the new firmware, cascade into `AT+INIT=THRESHOLD`, and lock the device with `+CME ERROR: 41`.
- Shamir unlock / ceremony will still fail on `SubmitShare` due to `+COMMIT:` mismatch.
- Does not fix the integration test gap.

---

### Comparison & Recommended Approach

| Dimension | Option 1 (Full Harmonization & Upgrade) | Option 2 (Firmware Flash Only) |
| :--- | :--- | :--- |
| **PIN Mode Resolution** | Complete | Complete |
| **Shamir Mode Resolution** | Complete (Init + Custodian Unlock) | **Fails** (Latent response prefix bug) |
| **Backward Compatibility** | High (supports both `+INIT:`/`+SHARE:` and `AT+INIT=1`) | None |
| **CI / Test Parity** | Verified in integration tests | Broken |
| **Architectural Quality** | Complies with ADR-0016 and Clean Architecture | Leaves technical debt in protocol layer |

> [!TIP]
> **Recommendation**: Adopt **Option 1**. Remediate the protocol discrepancies in `hsm-cli`, add unit and integration test coverage, and deploy the signed firmware binary to the physical dongle via DFU.

---

## 4. Verification & Validation Plan

1. **Host CLI Unit Tests**:
   - Run `make cli-test` to ensure all unit tests pass with the aligned parser and error codes.
2. **Virtual Mock Integration Tests**:
   - Run `make integration-test` to verify full end-to-end initialization, PIN authentication, Shamir ceremony, key migration, and DFU against `mock_hsm_pty.py`.
3. **Physical Hardware Verification**:
   - Upgrade dongle firmware:
     ```bash
     hsmutil device firmware update --file hsm-ncs-firmware/build-prod/app/zephyr/zephyr.signed.bin
     ```
   - Query status post-upgrade:
     ```bash
     hsmutil device info
     hsmutil device status
     ```
   - Test PIN initialization:
     ```bash
     hsmutil device init --mode pin --pin ValidP@ssw0rd!
     hsmutil unlock --pin ValidP@ssw0rd!
     hsmutil device reset -y
     ```
   - Test Shamir initialization ceremony:
     ```bash
     hsmutil device init --mode shamir
     ```
     Verify all 5 custodian shares are parsed and displayed correctly turn-by-turn.
