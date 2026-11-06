---
title: Workspace Release Changelog
tags:
  - release
  - changelog
  - history
  - milestones
created: 2026-10-31
updated: 2026-11-06
---

# Workspace Release Changelog

This document tracks all formal releases, milestone deliveries, and post-merge changelog notes across the orchestrated multi-repository workspace (`ncs-nrf52840dongle-kms`).

---

## [Merged] — 2026-11-06: Retirement of Backward-Compatible Binary Symlinks and Native macOS USB Discovery (Merged into main via CLI PR #2 & Root PR #3)

> [!NOTE]
> **Post-Merge Milestone**: Formal retirement of transitional build-time binary symlinks (`bin/gphsms`, `bin/hsmutil`, `bin/hsmutils`) in `hsm-cli/bin` and removal of legacy symlink references from workspace docs and test runner successfully merged into `main`. Establishes `gphsm` as the sole canonical binary executable across the workspace. Introduction of platform-tailored compilation: macOS builds with `CGO_ENABLED=1` linking Apple IOKit/CoreFoundation for native hardware USB discovery, while Linux and Windows builds remain 100% hermetic static binaries (`CGO_ENABLED=0`).
>
> **Lifecycle Phase**: Phase 6 Post-Merge Complete.
>
> **Pull Requests & Merged Commits**:
> - **Host Administration CLI ([`hsm-cli`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli))**: Merged PR [#2 (https://github.com/gpfister/hsm-cli/pull/2)](https://github.com/gpfister/hsm-cli/pull/2) into `main`
>   - Commit: [`7341531`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli) (`7341531a7836365f5a896587c6aa5673ec4976c6`) — *refactor(cli): remove build-time binary symlinks and retire fallback discovery (#2)*
> - **Workspace Root Orchestrator ([`nrf52840-hsm`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm))**: Merged PR [#3 (https://github.com/gpfister/hsm/pull/3)](https://github.com/gpfister/hsm/pull/3) into `main`
>   - Commit: [`48d1ad9`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm) (`48d1ad9f0868f1c8413b827cb6305a2f58e1cfec`) — *chore(workspace): remove legacy symlink references from docs and test runner (#3)*
> - **Knowledge Vault ([`hsm-obsidian`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian))**: Branch `main`
>   - Synchronized [[Architecture/Decisions/ADR-0020 - Retirement of Backward-Compatible Binary Symlinks and Native macOS USB Discovery|ADR-0020]], [[Development/CLI Reference|CLI Reference]], [[git-status|Workspace Git Status]], and [[Changelog|Release Changelog]].
>
> **Deliverable Scope & Changes**:
> 1. **Retirement of Build-Time Symlinks**:
>    - Removed `ln -sf` symlink creation (`gphsms`, `hsmutil`, `hsmutils`) from [`hsm-cli/Makefile`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli/Makefile).
>    - Updated `make clean` to remove compiled binaries cleanly.
> 2. **Native macOS USB Discovery & Platform Compilation Matrix**:
>    - macOS (Darwin): Built with `CGO_ENABLED=1` and `-ldflags="-w"` for dynamic linkage to `libSystem`, `IOKit`, and `CoreFoundation`. Enables direct, native USB hardware device filtering (`0x1209:0x0001`) and BLE serial number extraction via `go.bug.st/serial/enumerator`.
>    - Linux & Windows: Built with `CGO_ENABLED=0` and `-ldflags="-s -w"` guaranteeing 100% hermetic static binaries with zero external C dynamic library dependencies.
> 3. **Obsolete Serial Fallback Pruning**:
>    - Deleted heuristic serial fallback (`internal/infra/serial/discovery_fallback.go`) in `hsm-cli`, unifying enumeration through `discovery_enumerator.go`.
>    - Added unit test suite in `internal/infra/serial/discovery_test.go` validating VID/PID matching and sorting.
> 4. **In-Process Ergonomic Aliases**:
>    - Preserved operator muscle memory within `gphsm`: `gphsm devices`, `gphsm ls`, `gphsm auth`, and hidden root-level commands (`info`, `reset`, `lock`, `init`, `status`, `firmware`, `auth`).
> 5. **Root Integration Test & Documentation Alignment**:
>    - Updated [`scripts/run_integration_tests.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts/run_integration_tests.sh) to invoke `CLI_BIN="${CLI_DIR}/bin/gphsm"` exclusively, removing obsolete fallback checks for `bin/hsmutil`.
>    - Synchronized workspace root [`README.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/README.md), [`AGENTS.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/AGENTS.md), and `hsm-cli` documentation.
>
> **Associated Architecture & Decision Artifacts**:
> - [[Architecture/Decisions/ADR-0020 - Retirement of Backward-Compatible Binary Symlinks and Native macOS USB Discovery|ADR-0020: Retirement of Backward-Compatible Binary Symlinks and Native macOS USB Discovery]]
> - [[Development/CLI Reference|CLI Reference: gphsm]]

---

## [Merged] — 2026-11-06: HSM Device Initialization and Dual-Mode Protocol Compatibility (Merged into main via CLI PR #1 & Root PR #2)

> [!NOTE]
> **Post-Merge Milestone**: Interoperability hardening and protocol compatibility across the dual-mode HSM ecosystem successfully merged into `main`. Enables reliable device initialization in both modern and legacy firmware builds, line parser prefix unification (`+INIT:` and `+SHARE:`), typed hardware CME error code translations (41, 42, 103), end-to-end CLI integration testing in Phase C, and verified in-place streaming DFU deployment on physical hardware.
>
> **Lifecycle Phase**: Phase 6 Post-Merge Complete.
>
> **Pull Requests & Merged Commits**:
> - **Host Administration CLI ([`hsm-cli`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli))**: Merged PR [#1 (https://github.com/gpfister/hsm-cli/pull/1)](https://github.com/gpfister/hsm-cli/pull/1) into `main`
>   - Commit: [`7edc9ba`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli) (`7edc9ba9c0fa4c3bdba309f069c4f33baba0b4f9`) — *fix(cli): add AT+INIT=1 fallback, dual-prefix parsing, and CME errors 41/42/103 (#1)*
> - **Root Workspace Orchestrator ([`nrf52840-hsm`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm))**: Merged PR [#2 (https://github.com/gpfister/hsm/pull/2)](https://github.com/gpfister/hsm/pull/2) into `main`
>   - Commit: [`a963023`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm) (`a9630231e5ba5009b4c5727e4370da03c5d0ead8`) — *test(scripts): add end-to-end CLI dual-mode init, status, and exclusivity tests to Phase C (#2)*
> - **Embedded Firmware ([`hsm-ncs-firmware`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware))**: Verified on `main`
>   - Commit: [`7f516bd`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware) (`7f516bdd7bf080819e45e5bbbd2b39e58c80eca8`) — *docs(firmware): document dual-mode authentication in README*
>   - Production signed firmware rebuilt ([`zephyr.signed.bin`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware/build-prod/app/zephyr/zephyr.signed.bin)) and verified via in-place DFU streaming upgrade against connected physical PCA10059 dongle.
> - **Knowledge Vault ([`hsm-obsidian`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian))**: Branch `main`
>   - Synchronized [[Architecture/Decisions/ADR-0019 - HSM Device Initialization and Dual-Mode Protocol Compatibility|ADR-0019]], [[Plan/HSM Device Initialization and Dual-Mode Protocol Compatibility Implementation Plan|Implementation Plan]], [[Specification/HSM Device Initialization and Dual-Mode Protocol Compatibility|Technical Specification]], and [[features/hsm-device-initialization-and-dual-mode-protocol-compatibility|Feature Specification]].
>
> **Deliverable Scope & Changes**:
> 1. **Unsupported Command Fallback**:
>    - Intercepts `+CME ERROR: 8` (`ErrCommandUnsupported`) on `AT+INIT=SHAMIR` and transparently falls back to canonical legacy `AT+INIT=1`.
>    - Eliminates the non-standard `AT+INIT=THRESHOLD,3,5` fallback call.
> 2. **Multi-Prefix Line Parsing**:
>    - Initialization shares: accepts both `+INIT:` and `+SHARE:` line prefixes.
>    - Custodian share submissions: parses intermediate commitment status lines `+SHARE: <count>/3 ACCEPTED,WORD="..."` alongside `+COMMIT:` and `+WORD:`.
> 3. **Hardware CME Error Code Mapping**:
>    - Code 41 (`CME_ERR_ALREADY_CONFIGURED`) &rarr; `ErrAlreadyConfigured`.
>    - Code 42 (`CME_ERR_INVALID_AUTH_MODE`) &rarr; `ErrInvalidAuthMode`.
>    - Code 103 (`CME_ERR_DUPLICATE_SHARE`) &rarr; `ErrDuplicateShare`.
> 4. **End-to-End Test Suite Automation**:
>    - Added comprehensive Phase C CLI execution steps in `scripts/run_integration_tests.sh`: PIN init, duplicate init rejection, unlock, factory reset, Shamir init with share parsing, PIN unlock rejection in Shamir mode, and reset restoration.
>    - Test suite passing 100% across Phase A (16/16 AT checks), Phase B (PKCS#11 mock runner), and Phase C (CLI test suite).
> 5. **Physical Hardware Verification**:
>    - Verified physical PCA10059 dongle streaming DFU upgrade via `gphsm device firmware update --file ...` and confirmed dual-mode readiness on hardware.
>
> **Associated Architecture & Specification Artifacts**:
> - [[Architecture/Decisions/ADR-0019 - HSM Device Initialization and Dual-Mode Protocol Compatibility|ADR-0019: HSM Device Initialization and Dual-Mode Protocol Compatibility]]
> - [[features/hsm-device-initialization-and-dual-mode-protocol-compatibility|Feature Specification: HSM Device Initialization and Dual-Mode Protocol Compatibility]]
> - [[Specification/HSM Device Initialization and Dual-Mode Protocol Compatibility|Technical Specification: HSM Device Initialization and Dual-Mode Protocol Compatibility]]
> - [[Plan/HSM Device Initialization and Dual-Mode Protocol Compatibility Implementation Plan|Implementation Plan: HSM Device Initialization and Dual-Mode Protocol Compatibility]]
> - [[investigations/2026-11-06 - HSM Device Initialization Failure|Issue Assessment: 2026-11-06 - HSM Device Initialization Failure]]

---

## [Merged] — 2026-11-06: Sub-Repository Renaming to hsm-pkcs11 with libgphsm ABI Invariance (Merged into main via PR #1)

> [!NOTE]
> **Post-Merge Milestone**: Standardization and harmonization of the PKCS#11 sub-repository naming across the multi-repository workspace by renaming the repository to **`hsm-pkcs11`** (remote: `gpfister/hsm-pkcs11`) while strictly preserving dynamic provider library ABI compatibility (`libgphsm.dylib` / `.so` / `.dll`). Successfully merged into `main` via PR #1 (squashed commit: `81d2cb9`), with working tree clean on `main` and all test suites passing 100%.
>
> **Lifecycle Phase**: Phase 6 Post-Merge Complete.
>
> **Pull Request & Commits**:
> - **Workspace Root ([`nrf52840-hsm`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm))**: Merged PR [#1 (https://github.com/gpfister/hsm/pull/1)](https://github.com/gpfister/hsm/pull/1) from `task/rename-subrepo-hsm-pkcs11` &rarr; `main`
>   - Squashed Commit: [`81d2cb9`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm) (`81d2cb9`) — *chore(workspace): standardize PKCS#11 sub-repo to hsm-pkcs11 (#1)*
>   - Branch Cleanup: Local and remote `task/rename-subrepo-hsm-pkcs11` deleted and pruned.
> - **PKCS#11 Sub-Repo ([`hsm-pkcs11`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-pkcs11))**: Reconfigured git origin to `git@github.com:gpfister/hsm-pkcs11.git` tracking `main`.
> - **Documentation Vault ([`hsm-obsidian`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian))**: Post-merge synchronization across index, feature notes, specs, implementation plans, git status, and changelog.
>
> **Deliverable Scope & Changes**:
> 1. **GitHub Remote & Filesystem Alignment**:
>    - Renamed GitHub repository `gpfister/hsm-libpkcs11` to `gpfister/hsm-pkcs11`.
>    - Moved sub-repository directory `libgphsm/` to `hsm-pkcs11/` and updated git origin URL.
> 2. **Root Workspace Automation & Tooling**:
>    - [`Makefile`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/Makefile): Updated `SUBREPOS` and `LIBPKCS11_DIR` variable to `hsm-pkcs11`.
>    - [`.gitignore`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/.gitignore): Added `/hsm-pkcs11/` exclusion with backward-compatible legacy entries retained.
>    - [`scripts/worktree.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts/worktree.sh): Aligned `SUBREPOS` array to `hsm-pkcs11`.
>    - [`scripts/run_integration_tests.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts/run_integration_tests.sh): Aligned `P11_DIR` to `${WORKSPACE_ROOT}/hsm-pkcs11`.
> 3. **Workspace Documentation**:
>    - Root [`README.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/README.md) and [`AGENTS.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/AGENTS.md) updated to reference `hsm-pkcs11/`.
> 4. **Strict ABI & Binary Invariance**:
>    - Preserved `libgphsm.dylib`, `libgphsm.so`, `libgphsm.dll` binary names and CMake target properties (`OUTPUT_NAME "gphsm"`, `PREFIX "lib"`).
>    - 100% test pass rate across `make libpkcs11-build`, `make libpkcs11-test`, and `make integration-test`.
>
> **Associated Architecture & Specification Artifacts**:
> - [[Architecture/Decisions/ADR-0018 - Sub-Repository Renaming to hsm-pkcs11 with libgphsm ABI Invariance|ADR-0018: Sub-Repository Renaming to hsm-pkcs11 with libgphsm ABI Invariance]]
> - [[features/subrepo-rename-hsm-pkcs11|Feature Specification: Sub-Repository Renaming to hsm-pkcs11]]
> - [[Specification/Sub-Repository Renaming to hsm-pkcs11|Technical Specification: Sub-Repository Renaming to hsm-pkcs11]]
> - [[Plan/Sub-Repository Renaming to hsm-pkcs11 Implementation Plan|Implementation Plan: Sub-Repository Renaming to hsm-pkcs11]]

---

## [Merged] — 2026-11-05: HSM Dual-Mode Authentication, CLI Renaming (gphsm), and PKCS#11 Refactoring (libgphsm) (Merged into main)

> [!NOTE]
> **Post-Merge Milestone**: Multi-repository delivery of HSM dual-mode authentication, host CLI rebranding (`gphsm`), pure-Go PAM pwquality password validation, and PKCS#11 provider refactoring (`libgphsm`) successfully merged into `main` across all target repositories. All feature and task branches were merged, working trees clean on `main`, and 100% test passage achieved across firmware CTest, CLI unit tests, and the end-to-end multi-phase integration test suite (`make integration-test`).
>
> **Lifecycle Phase**: Phase 6 Post-Merge Complete.
>
> **Merge Details Across Repositories**:
> - **Firmware Subsystem ([`hsm-ncs-firmware`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware))**: `feature/dual-mode-auth` merged into `main` (`e03c9bb..7f516bd`). Working tree clean on `main`.
> - **PKCS#11 Provider Library ([`libgphsm`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/libgphsm))**: `task/rename-libgphsm-dual-mode` merged into `main` (`9c54759..d25adea`). Working tree clean on `main`.
> - **Host Management CLI ([`hsm-cli`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli))**: `feature/gphsm-dual-mode` merged into `main` (`6c8781c..14ea65f`). Working tree clean on `main`.
> - **Root Workspace Orchestrator ([`nrf52840-hsm`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm))**: `task/dual-mode-workspace-alignment` merged into `main` (`58b82c0..2848d50`). Working tree clean on `main`.
> - **Central Knowledge Vault ([`hsm-obsidian`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian))**: Post-merge status, branch tracking, and release records synchronized on `main`.
>
> **Verification & Integration Results**:
> - **Firmware Unit Tests**: `ctest` passed 100% across keystore dual-mode state, NVS persistence, and AT command dispatch.
> - **CLI Verification**: `go test ./...` passed cleanly across domain, infra, and UI packages.
> - **End-to-End Integration Tests (`make integration-test`)**:
>   - **Phase A**: 16/16 AT protocol & ceremony steps passed (probing, status queries, PIN/Shamir initialization, auth mutual exclusivity, key operations, factory reset zeroization).
>   - **Phase B**: `libgphsm` PKCS#11 mock suite passed 100%.
>   - **Phase C**: `gphsm` CLI test suite passed 100%.
>
> **Key Deliverables & Architectural Enhancements**:
> 1. **Firmware Dual-Mode Authentication Keystore (`auth_method_t`)**:
>    - Distinct keystore methods: `NONE` (unconfigured), `PIN` (passphrase mode), `SHAMIR` (3-of-5 threshold mode) with persistent NVS storage.
>    - Extended PIN storage supporting up to 64 bytes with 16-byte random salt and SHA-256 hash.
>    - Dedicated initialization endpoints: `AT+INIT=PIN,<pin>` and `AT+INIT=SHAMIR` with re-initialization guards (`+CME ERROR: 41`).
>    - Comprehensive status output via `AT+STATUS?` (`CONFIGURED`, `AUTH`, `METHOD`, `RETRIES`, `KEYCOUNT`).
>    - Mutual exclusivity enforcement between `AT+AUTH` and `AT+SHARE` (`+CME ERROR: 42`).
>    - Zeroization in `AT+RESET` / `AT+FACTORYRESET` resetting salt, hash, keys, and restoring unconfigured state.
> 2. **PKCS#11 Provider Library Refactoring (`libgphsm`)**:
>    - Repository and library renamed to `libgphsm` emitting `libgphsm.{dylib,so,dll}`.
>    - Dual-mode token flags: sets `CKF_PROTECTED_AUTHENTICATION_PATH` in Shamir mode.
>    - Adapted `C_Login` to recognize external pre-authentication (`AUTH == 1`) in Shamir mode without requiring in-band PIN entry.
>    - Clean CMake and Makefile integration with updated CTest mock suite.
> 3. **Host CLI Rebranding to `gphsm` (`hsm-cli`)**:
>    - Rebranded executable to `gphsm` with backward-compatible `hsmutil` and `hsmutils` symlinks.
>    - Dual-mode initialization wizard (`gphsm device init`) with interactive TTY prompts for PIN vs Shamir mode.
>    - Embedded pure-Go `pam_pwquality` validation enforcing Yocto appliance password complexity.
>    - Auto-detecting `gphsm unlock` (alias `auth`) handling PIN prompts and custodian share ceremonies.
>    - Purged legacy internal `p11` diagnostic commands and dynamic PKCS#11 loaders.
>    - Homogeneous migration validation ensuring `source.METHOD == target.METHOD`.
> 4. **Root Workspace & Multi-Repository Orchestration**:
>    - Makefile targets updated: `cli-build`, `libpkcs11-build`, `libpkcs11-test`, `make integration-test`.
>    - Updated `scripts/mock_hsm_pty.py` to support dual-mode initialization, mutual exclusivity, and extended status responses.
>    - Updated `scripts/run_integration_tests.sh` and `scripts/test_at_client.py` covering all dual-mode auth paths.
>
> ### Deliverables & Architecture Records
> - **Commits**:
>   - Firmware: [`7f516bd`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware) — *docs(firmware): document dual-mode authentication in README*
>   - PKCS#11: [`d25adea`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/libgphsm) — *docs(p11): align protocol ID reference with ADR-0013 in README*
>   - CLI: [`14ea65f`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli) — *feat(cli): rebrand to gphsm, add dual-mode auth, pwquality PIN validation, and purge p11*
>   - Root: [`2848d50`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm) — *docs(workspace): update root README and AGENTS.md for gphsm, libgphsm, and dual-mode auth*
> - **ADRs Formalized**:
>   - [[Architecture/Decisions/ADR-0016 - Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)|ADR-0016: Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)]]
>   - [[Architecture/Decisions/ADR-0017 - Host CLI Rebranding to gphsm and PKCS11 Refactoring to libgphsm|ADR-0017: Host CLI Rebranding to gphsm and PKCS#11 Refactoring to libgphsm]]
> - **Technical Specification**: [[Specification/HSM Dual-Mode Authentication, CLI Renaming (gphsm), and PKCS11 Refactoring|Technical Specification: HSM Dual-Mode Authentication, CLI Renaming (gphsm), and PKCS#11 Refactoring]]
> - **Feature Note**: [[features/dual-mode-auth-gphsm-and-libgphsm-refactoring|Dual-Mode Authentication, CLI Rebranding (gphsm) & PKCS#11 Refactoring (libgphsm)]]
> - **Implementation Plan**: [[Plan/HSM Dual-Mode Authentication, CLI Renaming (gphsm), and PKCS11 Refactoring Implementation Plan|Implementation Plan: HSM Dual-Mode Authentication, CLI Renaming (gphsm), and PKCS#11 Refactoring]]
> - **CLI Reference**: [[Development/CLI Reference|CLI Reference (gphsm)]]
> - **Status Note**: [[git-status|Workspace Git Status & Repository Topologies]]

---

## [Merged] — 2026-11-05: HSM CLI Restructuring, Static Compilation, Bubble Tea UX & PKCS#11 Equivalence (Merged into main)

> [!NOTE]
> **Post-Merge Milestone**: Comprehensive restructuring of the host management CLI (`hsmutil`) successfully merged into `main` in [`hsm-cli`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli). Feature branch `feature/cli-restructuring-bubbletea-pkcs11` was fast-forward merged (`189c7fd..6c8781c`), working tree clean, and all packages passed verification suites (`go test ./...`).
>
> **Lifecycle Phase**: Phase 6 Post-Merge Complete.
>
> **Merge Details**:
> - **Repository**: [`hsm-cli`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli)
> - **Source Branch**: `feature/cli-restructuring-bubbletea-pkcs11`
> - **Target Branch**: `main`
> - **Merge Status**: Fast-forward merged (`189c7fd..6c8781c`)
> - **Working Tree**: Clean across all workspace repositories
> - **Verification**: `go test ./...` passed cleanly 100% across all packages
>
> **Key Deliverables & Architectural Enhancements**:
> 1. **Resource Command Hierarchy**: Reorganized commands under `hsmutil device <subcommand>` (`list`, `info`, `init`, `lock`, `reset`, `status`, `migrate`, `firmware`) and `hsmutil key <subcommand>` (`list`, `generate`, `pubkey`, `sign`, `migrate`), with clean top-level utilities (`unlock`, `echo`, `random`, `pkcs11`).
> 2. **Non-Breaking Backward Compatibility**: Preserved legacy root commands (`info`, `reset`, `lock`, `init`, `status`, `firmware`, `auth`) as hidden aliases on `rootCmd`.
> 3. **Ergonomic Aliases & Pluralization**: Added visible alias `hsmutil devices -> hsmutil device list` and Makefile relative symlink `bin/hsmutils -> bin/hsmutil`.
> 4. **Pruned Deprecated Code**: Removed unmaintained monolithic full-screen dashboard (`hsmutil tui`, `internal/ui/tui/`) and redundant embedded test runner (`hsmutil test`).
> 5. **Renamed Auth to Unlock**: Renamed `auth` command to `unlock` with `auth` retained as convenience alias.
> 6. **Terminology Neutrality**: Replaced `nrf52840` references with generic HSM USB Key / device across all user-facing CLI text and help messages per [[Architecture/Decisions/ADR-0014 - Documentation Terminology Generalization to Generic HSM USB Key|ADR-0014]].
> 7. **Bubble Tea Micro-Components**: Integrated Charm's Bubble Tea micro-components (`bubbles/textinput` for masked password bullets `••••••` during `unlock`, `bubbles/progress` for animated gradient DFU transfer bar during `device firmware update`) with automatic TTY detection and clean headless fallback.
> 8. **Static Binary Compilation**: Enforced `CGO_ENABLED=0` across all Makefile targets for zero-dependency portability across host and cross-compilation platforms (`darwin/arm64`, `linux/arm64`, `linux/amd64`, `windows/arm64`, `windows/amd64`).
> 9. **PKCS#11 Parity & Matrix**: Formalized complete equivalence matrix and GAP analysis against OpenSC `pkcs11-tool`.
>
> ### Deliverables & Architecture Records
> - **Commit**: [`6c8781c`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli) (`6c8781c`) — *feat(cli): restructure commands under device, add static build, bubble tea UX, and pkcs11 parity*
> - **ADR Formalized**: [[Architecture/Decisions/ADR-0015 - HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS11 Equivalence|ADR-0015: HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS#11 Equivalence]]
> - **Technical Specification**: [[Specification/HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS11 Equivalence|Technical Specification: HSM CLI Restructuring, Static Compilation, Bubble Tea UX & PKCS#11 Equivalence]]
> - **Feature Note**: [[features/cli-restructuring-bubbletea-pkcs11|HSM CLI Restructuring, Static Compilation, Bubble Tea UX & PKCS#11 Equivalence]]
> - **Implementation Plan**: [[Plan/HSM CLI Restructuring, Static Compilation, Bubble Tea UX & PKCS11 Equivalence Implementation Plan|Implementation Plan: HSM CLI Restructuring, Static Compilation, Bubble Tea UX & PKCS#11 Equivalence]]
> - **CLI Reference**: [[Development/CLI Reference|CLI Reference (hsmutil)]]
> - **Status Note**: [[git-status|Workspace Git Status & Repository Topologies]]

---

## [Merged] — 2026-11-05: Documentation Terminology Generalization to Generic HSM USB Key (Merged into main)

> [!NOTE]
> **Post-Merge Milestone**: Documentation terminology generalization successfully merged into `main` across all 6 repositories. Feature branch `task/generalize-doc-terminology` merged and local task branches pruned across all repositories. All titles and high-level prose standardized on generic **Hardware Security Module (HSM)** and **HSM USB Key / Dongle** terminology per [[Architecture/Decisions/ADR-0014 - Documentation Terminology Generalization to Generic HSM USB Key|ADR-0014]], while explicitly preserving firmware target hardware scoping in [`hsm-ncs-firmware`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware) (Nordic Semiconductor nRF52840 USB Dongle `PCA10059`) and non-breaking machine-readable identifiers (`libnrf52840_pkcs11.*`, `NRF52840_HSM_*`, `nrf52840-hsm-kmod`).
>
> **Lifecycle Phase**: Phase 6 Post-Merge Complete.
>
> **Merge Details Across All 6 Repositories**:
> - **Status**: Merged
> - **Source Branch**: `task/generalize-doc-terminology` (pruned)
> - **Target Branch**: `main`
> - **Working Trees**: Clean across all 6 repositories
>
> **Delivered Commits**:
> 1. **Root Workspace ([`nrf52840-hsm`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm))**: Commit [`af2d24e`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm) (`af2d24e`) — Generalized root [`README.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/README.md) and [`AGENTS.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/AGENTS.md).
> 2. **Microcontroller Firmware ([`hsm-ncs-firmware`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware))**: Commit [`e03c9bb`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware) (`e03c9bb`) — Updated [`README.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware/README.md) with explicit scope note that firmware currently exclusively supports the nRF52840 USB Dongle (`PCA10059`).
> 3. **Host CLI & TUI ([`hsm-cli`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli))**: Commit [`189c7fd`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli) (`189c7fd`) — Generalized [`README.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli/README.md) to `hsm-cli (hsmutil)` for interacting with HSM USB key, retained environment variables.
> 4. **PKCS#11 Provider ([`hsm-libpkcs11`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-libpkcs11))**: Commit [`9c54759`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-libpkcs11) (`9c54759`) — Generalized [`README.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-libpkcs11/README.md) description, retained library names and environment variables.
> 5. **Air-Gapped OS ([`hsm-yocto-offgrid-rpi`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi))**: Commit [`28925aa`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi) (`28925aa`) — Generalized [`README.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi/README.md) to HSM USB key and appliance, retained driver name `nrf52840-hsm-kmod`.
> 6. **Central Knowledge Vault ([`hsm-obsidian`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian))**: Commits [`682fbfc`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian) (`682fbfc`), [`f1d5a7c`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian) (`f1d5a7c`), and [`e91555a`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian) (`e91555a`) — Authored [`README.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/README.md), generalized [`00 - Index.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/00%20-%20Index.md), updated [`Architecture/System Architecture.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/Architecture/System%20Architecture.md), recorded [[Architecture/Decisions/ADR-0014 - Documentation Terminology Generalization to Generic HSM USB Key|ADR-0014]], [[Plan/Documentation Terminology Generalization Implementation Plan|Implementation Plan]], [[features/documentation-terminology-generalization|Feature Specification]], and updated post-merge release tracking records.
>
> ### Deliverables & Architecture Records
> - **Commits**: [`af2d24e`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm) (Root), [`e03c9bb`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware) (Firmware), [`189c7fd`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli) (CLI), [`9c54759`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-libpkcs11) (PKCS#11), [`28925aa`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi) (Air-Gapped OS), [`e91555a`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian) (Knowledge Vault)
> - **ADR Formalized**: [[Architecture/Decisions/ADR-0014 - Documentation Terminology Generalization to Generic HSM USB Key|ADR-0014: Documentation Terminology Generalization to Generic HSM USB Key]]
> - **Feature Specification**: [[features/documentation-terminology-generalization|Documentation Terminology Generalization]]
> - **Implementation Plan**: [[Plan/Documentation Terminology Generalization Implementation Plan|Documentation Terminology Generalization Implementation Plan]]
> - **Status Note**: [[git-status|Workspace Git Status & Repository Topologies]]
>
> ---
>
## [Merged] — 2026-11-05: Fix macOS CDC-ACM Enumeration for Interface 2 (Merged into feature/at-dfu-support)

> [!NOTE]
> **Post-Merge Milestone**: Feature branch `bugfix/macos-cdc-acm-enumeration` successfully fast-forward merged into `feature/at-dfu-support` (`7aa1d2b..1d76b0d`) in `nrf52840-hsm-firmware`. Working tree clean, HEAD at [`1d76b0d`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware). Resolves macOS USB CDC-ACM enumeration failure on Interface 2 (`cdc_acm_uart1`) per [[Architecture/Decisions/ADR-0013 - Standard CDC-ACM AT Protocol Descriptors for macOS Dual Enumeration|ADR-0013]].
>
> **Lifecycle Phase**: Phase 6 Post-Merge Complete.
>
> **Merge Details**:
> - Repository: [`nrf52840-hsm-firmware`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware)
> - Source Branch: `bugfix/macos-cdc-acm-enumeration`
> - Target Branch: `feature/at-dfu-support`
> - Status: Fast-forward merged (`7aa1d2b..1d76b0d`)
> - Current Branch: `feature/at-dfu-support`
> - Working Tree: Clean
> - HEAD: [`1d76b0d`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware)
>
> **Merged Commits**:
> - [`0e7361b`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware) (`0e7361b269f56090f981bfbd786a9bdd92a6e00e`): *fix(usb): standard CDC-ACM AT protocol descriptors for macOS dual enumeration (ADR-0013)*
> - [`1d76b0d`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware) (`1d76b0db6ff676b763ecaa82a20ca8817343468d`): *docs(firmware): update README with CDC-ACM AT protocol descriptor standardization (ADR-0013)*
>
> **Files Changed**:
> - [`firmware/app/src/usb_serial.c`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware/firmware/app/src/usb_serial.c): Standardized Interface 2 descriptor `bInterfaceProtocol` and IAD `bFunctionProtocol` to `0x01` (ITU-T V.250 AT command protocol), restoring simultaneous `/dev/cu.usbmodemF7820A1E3B161` (Console) and `/dev/cu.usbmodemF7820A1E3B163` (HSM AT channel).
> - [`firmware/app/src/dfu_service.c`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware/firmware/app/src/dfu_service.c): Padded chunk writes to 4-byte flash write word boundary (with `0xFF`) satisfying internal flash controller alignment.
> - [`README.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware/README.md): Documented protocol descriptor standardization.
>
> **Physical Hardware Verification**:
> - Validated on primary PCA10059 dongle (`F7:82:0A:1E:3B:16`) via Segger J-Link probe `802007333`.
> - Validated on secondary PCA10059 dongle (`D7:4B:8A:9C:AB:28`, UID `EB53D74B8A9CAB28`) via J-Link probe `802007333`: full SWD flash (`merged.hex`), dual CDC-ACM enumeration (`/dev/cu.usbmodemD74B8A9CAB281` and `/dev/cu.usbmodemD74B8A9CAB283`), PIN authentication (`123456`), AT DFU chunk streaming upgrade (142,331 bytes) with MCUboot test swap and self-confirmation, and factory reset (`hsmutil reset -y`). 100% Passed.
> - Simultaneous console and HSM AT channel generated across both hardware units without host kernel errors.
> - Verified with `hsmutil --port /dev/cu.usbmodemD74B8A9CAB283 info`, `auth`, `firmware update`, and `reset`. 100% Passed.

### Deliverables & Architecture Records
- **Commits**: [`0e7361b`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware), [`1d76b0d`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware)
- **ADR Formalized**: [[Architecture/Decisions/ADR-0013 - Standard CDC-ACM AT Protocol Descriptors for macOS Dual Enumeration|ADR-0013: Standard CDC-ACM AT Protocol Descriptors for macOS Dual Enumeration]]
- **Updated ADR**: [[Architecture/Decisions/ADR-0002 - In-Tree Protocol Descriptor Patching vs macOS DriverKit Extension|ADR-0002]] (updated with note referencing ADR-0013)
- **Feature Specification**: [[features/fix-macos-cdc-acm-enumeration|Fix macOS CDC-ACM Enumeration for Interface 2]]
- **Technical Specification**: [[Specification/Fix macOS CDC-ACM Enumeration for Interface 2|Fix macOS CDC-ACM Enumeration for Interface 2]]
- **Implementation Plan**: [[Plan/Fix macOS CDC-ACM Enumeration for Interface 2 Implementation Plan|Fix macOS CDC-ACM Enumeration for Interface 2 Implementation Plan]]
- **Architecture Record**: [[Architecture/USB Composite Stack|USB Composite Stack Architecture]] (updated Interface 2 protocol allocation and macOS driver matching criteria)
- **Telemetry Records**:
  - [[Monitoring/2026-11-05 - Production Flashing, DFU Verification, and Factory Reset - Second Dongle|Telemetry: Production Flashing, DFU Verification, and Factory Reset - Second Dongle]]
  - [[Monitoring/2026-11-05 - Production Flashing and USB CDC-ACM Interface Anomaly|Telemetry: Production Flashing and USB CDC-ACM Interface Anomaly]]

---

## [v0.5.0-alpha] — 2026-11-05: Yocto 6.1 (Blacksail) Upgrade, Boot Visibility, U-Boot Hardening & Password Policy

> [!NOTE]
> **Release Highlights**: Major platform upgrade of `rpi-z2w-yocto-hsm` to **Yocto Project 6.1 (`blacksail`, BitBake 2.20)** with **Linux Kernel 6.12+**. Implements kernel boot console multiplexing (dual HDMI/UART broadcast in development vs. silent console in production), hardened non-interactive U-Boot (`CONFIG_BOOTDELAY=-2`, `silent=1`, null console), default operator account password initialized to `password`, system-wide PAM `pam_pwquality` password complexity enforcement (10+ chars, mixed case, digits, symbols, no repetitions, no sequences), and 90-day password expiration policy (`PASS_MAX_DAYS 90`).
>
> **Release Status**: **Merged & Released (Phase 6 Post-Merge Complete)**. Fast-forward merged branch `task/upgrade-yocto-6.1-blacksail` into `main` at head commit [`aee860a`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm) (`aee860a39385cb71554f2243e94621a843d411b4`). Cumulative diff across 7 commits: **42 files changed, 767 insertions(+), 83 deletions(-)**.
>
> **Verification Status**: **100% Passed** across all quality gates:
> - All acceptance criteria AC-1 through AC-6 verified and satisfied.
> - Password validator suite ([`scripts/test-password-validator.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/scripts/test-password-validator.sh)): **14/14 tests passed** (4 valid passwords accepted, 10 invalid passwords properly rejected).
> - Security configuration verifier ([`scripts/verify-security-configs.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/scripts/verify-security-configs.sh)): **19/19 checks passed** (zero errors across YAML kas configs, bbappends, Kconfigs, PAM rules, and shadow settings).
> - BitBake recipe syntax: **0 parse errors** across all recipes.

### 1. Summary of Changes

| Subsystem | Branch | Commit | Key Additions |
| :--- | :--- | :--- | :--- |
| **Air-Gapped OS** (`rpi-z2w-yocto-hsm`) | `main` | [`aee860a`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm) | Synchronized README directory layout, automated verification test suites, and ADR cross-references. |
| **Air-Gapped OS** (`rpi-z2w-yocto-hsm`) | `main` | [`5155f4c`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm) | Implemented boot log visibility specialization (`rpi-cmdline.bbappend`), hardened silent U-Boot (`silent-boot.cfg`, `squashfs.cfg`), default password `password`, PAM `pam_pwquality` complexity, 90-day password expiration (`PASS_MAX_DAYS 90`), `first-login-wizard.sh` pre-validator, and test suites (`test-password-validator.sh`, `verify-security-configs.sh`). |
| **Air-Gapped OS** (`rpi-z2w-yocto-hsm`) | `main` | [`3634d7a`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm) | Updated documentation and Makefile for Yocto 6.1 Blacksail. |
| **Air-Gapped OS** (`rpi-z2w-yocto-hsm`) | `main` | [`7d52c9b`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm) | Linux 6.12+ `.llseek = noop_llseek` version guard in `nrf52840_hsm.c`. |
| **Air-Gapped OS** (`rpi-z2w-yocto-hsm`) | `main` | [`fc8259a`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm) | Migrated recipes to `${UNPACKDIR}`; dropped legacy `debug-tweaks` in favor of `allow-root-login post-install-logging`. |
| **Air-Gapped OS** (`rpi-z2w-yocto-hsm`) | `main` | [`d15e5c5`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm) | Internalized Raspberry Pi RAUC A/B integration into `meta-hsm-os`; dropped `meta-rauc-community` and `meta-lts-mixins`; layer compat `blacksail`. |
| **Air-Gapped OS** (`rpi-z2w-yocto-hsm`) | `main` | [`515ff0c`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm) | Decomposed monolithic `poky` into `openembedded-core`, `bitbake` (2.20), and `meta-yocto` in `kas-project.yml`. |
| **Obsidian Vault** (`nrf52840-hsm-obsidian`) | `master` | Current | Synchronized `Architecture/07 - Air-Gapped RPi HSM OS Architecture.md`, `Development/Yocto Build and RAUC Update Guide.md`, `features/yocto-boot-visibility-and-password-policy.md`, `00 - Index.md`, `git-status.md`, and indexed ADRs (ADR-0007 through ADR-0012). |

---

### 2. Detailed Subsystem Changes

#### Boot Visibility & Console Specialization (ADR-0010)
- Created `recipes-bsp/bootfiles/rpi-cmdline.bbappend` specializing `CMDLINE` based on `HSM_DEV_MODE`:
  - **Development (`HSM_DEV_MODE="1"`)**: `CMDLINE_DEV = "console=serial0,115200 console=tty1 loglevel=7 debug earlyprintk"`. Kernel `printk` broadcasts to both the physical HDMI display (`tty1`) and UART serial console (`serial0`), while attaching `/dev/console` to the foreground display.
  - **Production (`HSM_DEV_MODE="0"`)**: `CMDLINE_PROD = "quiet loglevel=0 console=tty1 fsck.mode=skip"`. Suppresses serial UART output entirely and silences HDMI boot logs.
- Cleaned up obsolete BitBake `APPEND` references from `kas-dev.yml` and `kas-prod.yml`.

#### Hardened Non-Interactive U-Boot Bootloader (ADR-0011)
- Replaced fragile board-specific `rpi_arm64_defconfig.patch` with modular Kconfig fragments:
  - [`squashfs.cfg`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-hsm-os/recipes-bsp/u-boot/files/squashfs.cfg): enables `CONFIG_FS_SQUASHFS=y` and `CONFIG_CMD_SQUASHFS=y`.
  - [`silent-boot.cfg`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-hsm-os/recipes-bsp/u-boot/files/silent-boot.cfg): enables `CONFIG_BOOTDELAY=-2`, `CONFIG_SILENT_CONSOLE=y`, `CONFIG_SYS_DEVICE_NULLDEV=y`, and `CONFIG_SILENT_U_BOOT_ONLY=y`.
- Updated `rpi-u-boot-scr.bbappend` and `boot.cmd.in`: in production mode, injects `setenv silent 1` and comments out RAUC slot diagnostic `echo` statements, eliminating boot banners and abort countdowns.

#### Operator Identity, PAM Complexity & Password Expiration (ADR-0012)
- Reconfigured default operator password in `hsmutil-user.bb` to `password` (SHA-512 crypt hash `$6$saltsalt$qFmFH.bQmmtXzyBY0s9v7Oicd2z4XSIecDzlB5KiA2/jctKu9YterLp8wwnSq.qc.eoxqOmSuNp2xS0ktL3nh/`).
- Added system-wide PAM `pam_pwquality` enforcement via `recipes-extended/libpwquality/`:
  - Enforces: minimum 10 characters, at least 1 digit, 1 uppercase, 1 symbol, no 2 consecutive identical characters (`maxrepeat=1`), no 3 sequential characters (`maxsequence=2`), and strict root enforcement (`enforce_for_root`).
- Enforced 90-day password expiration policy (`PASS_MAX_DAYS 90`) via `recipes-extended/shadow/shadow_%.bbappend` and `shadow-sysroot_%.bbappend` in both target and sysroot `/etc/login.defs`.
- Enhanced `first-login-wizard.sh` with interactive pre-validation and actionable feedback for each complexity rule violation.

#### Automated Test & Verification Suites
- `scripts/test-password-validator.sh`: 14 comprehensive test cases verifying positive acceptance and rejection of short, low-complexity, repeating, and sequential candidate passwords.
- `scripts/verify-security-configs.sh`: 19 automated assertions inspecting YAML kas configurations, bbappend recipes, Kconfig fragments, shadow parameters, and PAM rule files.

---

## [v0.4.0-alpha] — 2026-11-05: Air-Gapped Raspberry Pi Embedded Linux OS (`rpi-z2w-yocto-hsm`)

> [!NOTE]
> **Release Highlights**: Addition and initialization of the `rpi-z2w-yocto-hsm` sub-repository delivering an ultra-hardened, air-gapped 64-bit embedded Linux operating system built with Yocto Project 5.0 LTS (`scarthgap`) and Linux Kernel 6.6 LTS. Features containerized Kas build workflows, complete physical network elimination (WiFi, BLE, Ethernet disabled at hardware/DTB and kernel level), `nftables` 100% total-drop firewall, out-of-tree `nrf52840-hsm-kmod` kernel driver, mandatory first-login wizard with clock prompt, read-only root filesystem, USB mass storage automount, and dual-slot RAUC A/B update verification using x509 PKI certificates.
>
> **Release Status**: Scaffolding complete; development OS image (`hsm-image-dev`) verified with 5,844 BitBake tasks successfully completed; uncompressed `.img` (`1.6 GB`, via `IMAGE_TYPEDEP:img = "wic"` and `IMAGE_CMD:img`), compressed `.wic.bz2` (`168 MB`), fast block map `.wic.bmap` (`8.0 KB`), and `.tar.bz2` (`73 MB`) deployed; cross-platform macOS/Linux SD card flashing (`scripts/flash-sd.sh`) aligned; root Makefile targets integrated; architecture documentation indexed in vault.

### 0. Milestone: Verified Yocto Development Build (5,844 Tasks Succeeded)

- **Target Image**: `hsm-image-dev` for `raspberrypi0-2w-64` (Raspberry Pi Zero 2 W 64-bit).
- **Build Execution**: Successfully executed `kas build kas-dev.yml` (via `make yocto-build-dev`): **5,844 tasks attempted and succeeded** with zero errors or task failures.
- **Deployed Artifacts** in [`rpi-z2w-yocto-hsm/build/deploy/images/raspberrypi0-2w-64/`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/build/deploy/images/raspberrypi0-2w-64):
  - `hsm-image-dev-raspberrypi0-2w-64.rootfs.img` (**1.6 GB**, symlink to `hsm-image-dev-raspberrypi0-2w-64.rootfs-20261105103859.img`): Uncompressed raw bootable SD disk image with dual-slot A/B partitions and U-Boot (`IMAGE_FSTYPES = "tar.bz2 img wic.bz2 wic.bmap"`).
  - `hsm-image-dev-raspberrypi0-2w-64.rootfs.wic.bz2` (**168 MB**): Bzip2-compressed raw bootable SD disk image with dual-slot A/B partitions and U-Boot.
  - `hsm-image-dev-raspberrypi0-2w-64.rootfs.wic.bmap` (**8.0 KB**): Block map file for accelerated flashing.
  - `hsm-image-dev-raspberrypi0-2w-64.rootfs.tar.bz2` (**73 MB**): Rootfs compressed tarball containing packages and `hsmutil`.
  - Kernel Device Tree Overlays & Boot Files: `disable-bt.dtbo` (1.0 KB), `disable-wifi.dtbo` (387 B), `dwc2.dtbo` (801 B), `bcm2710-rpi-zero-2.dtb` (33.7 KB), `boot.scr` (1.3 KB).
- **Flashing Procedures**:
  - Direct raw flasher: `./scripts/flash-sd.sh build/deploy/images/raspberrypi0-2w-64/hsm-image-dev-raspberrypi0-2w-64.rootfs.img <target_device>`.
  - Direct streaming flasher: `./scripts/flash-sd.sh build/deploy/images/raspberrypi0-2w-64/hsm-image-dev-raspberrypi0-2w-64.rootfs.wic.bz2 <target_device>`.
  - Fast block map copy: `sudo bmaptool copy --bmap build/deploy/images/raspberrypi0-2w-64/hsm-image-dev-raspberrypi0-2w-64.rootfs.wic.bmap build/deploy/images/raspberrypi0-2w-64/hsm-image-dev-raspberrypi0-2w-64.rootfs.img <target_device>`.

### 1. Summary of Changes

| Subsystem | Branch | Commit | Key Additions |
| :--- | :--- | :--- | :--- |
| **Air-Gapped OS** (`rpi-z2w-yocto-hsm`) | `main` | [`f4adf0b`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm) | Custom `img` image type (`IMAGE_TYPEDEP:img = "wic"`, `IMAGE_CMD:img`) outputting uncompressed `.img` (`IMAGE_FSTYPES = "tar.bz2 img wic.bz2 wic.bmap"`) with zero stale `.wic` files; aligned `scripts/flash-sd.sh`, `.gitignore`, and `README.md`. Previous: Uncompressed `.wic` & Darwin/Linux `dd` flags (`b99899f`), U-Boot device tree handoff (`91428c8`), packaging conflicts (`053bf75`, `cb4e8ca`), Yocto 5.0 layer (`1ce95a1`). |
| **Workspace Root** (`nrf52840-hsm`) | `master` | [`3713c98`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm) | Added `rpi-z2w-yocto-hsm` to `SUBREPOS` in root `Makefile`; added `yocto-build-dev`, `yocto-build-prod`, `yocto-bundle`, `yocto-shell`, and `yocto-clean` targets; updated root `AGENTS.md`. |
| **Obsidian Vault** (`nrf52840-hsm-obsidian`) | `master` | [`edf62cd`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-obsidian) | Authored `Architecture/07 - Air-Gapped RPi HSM OS Architecture.md`, `Hardware/Waveshare USB HAT (B) Integration.md`, `Development/Yocto Build and RAUC Update Guide.md`, `features/uncompressed-wic-image-and-sd-flash-tooling.md`, updated `00 - Index.md`, `git-status.md`, and `Changelog.md`. |

---

### 2. Detailed Subsystem Changes

#### Air-Gapped OS Subsystem (`rpi-z2w-yocto-hsm`)
- **Yocto Layer & Architecture**:
  - Initialized custom layer `meta-hsm-os` compatible with Yocto 5.0 (`scarthgap`).
  - Integrated `kas` orchestration with reproducible container runtime (`ghcr.io/siemens/kas/kas:latest`).
  - Configured `hsm-image-base.bb` with `read-only-rootfs` and common utilities.
  - Implemented `hsm-image-prod.bb` with locked root superuser (`usermod -L root`) and `hsm-image-dev.bb` with root password `root`, sudo escalation, and debugging packages.
- **Kernel & Hardware Lockdown**:
  - Implemented kernel config fragment (`hardened.cfg`) stripping wireless (`CONFIG_CFG80211=n`, `CONFIG_BT=n`), Ethernet, USB net drivers, and user GPIO probing (`CONFIG_GPIO_SYSFS=n`).
  - Enforced kernel memory hardening: `CONFIG_SECURITY=y`, `CONFIG_HARDENED_USERCOPY=y`, `CONFIG_FORTIFY_SOURCE=y`, `CONFIG_STRICT_KERNEL_RWX=y`, `CONFIG_RANDOMIZE_BASE=y`.
  - Configured `/boot/config.txt` via `rpi-config_%.bbappend`: disabled WiFi/Bluetooth (`disable-wifi`, `disable-bt`), forced DWC2 host mode (`dtoverlay=dwc2,dr_mode=host`) for Waveshare USB HAT (B), and enabled HDMI console force hotplug (`hdmi_force_hotplug=1`, `hdmi_drive=2`).
- **Out-of-Tree Driver (`nrf52840-hsm-kmod`)**:
  - Dedicated character driver matching USB `0x1209:0x0001` and extracting device BLE address from USB serial number.
  - Blacklisted generic Linux ACM driver (`blacklist-cdc-acm.conf`) for exclusive dongle claiming.
  - In production mode: binds and exposes AT command channel (`cdc_acm_uart1`) under `/dev/tty.hsm<ble><num>` with symlink `/dev/tty.hsm` while suppressing logger port (`cdc_acm_uart0`).
  - In development mode: exposes both console and AT command ports.
  - Installed udev rules (`99-nrf52840-hsm.rules`) setting permissions to `0660` with group `dialout`.
- **Volatile Storage & Logging**:
  - Implemented `volatile-storage` service and `hsm-setup-volatile.sh`:
    - Mounts `tmpfs` on `/media` for unprivileged USB storage mount points.
    - Mounts `overlayfs` on `/etc` (`/run/overlay/etc-upper`), allowing runtime password updates in volatile RAM without altering immutable flash.
    - Mounts `tmpfs` on `/home/hsmutil` populated from `/etc/skel`.
  - Implemented `systemd-journald-airgap`:
    - Production: `Storage=none`, zero console/syslog/kmsg forwarding (no disk or console data leakage).
    - Development: `Storage=volatile` with 64 MB RAM buffer limit.
- **Network Security & User Space**:
  - Enforced `nftables` default drop across input, output, and forward chains.
  - Configured dedicated unprivileged user `hsmutil` in groups `users`, `dialout`, `tty`, `storage` with SHA-512 crypt password.
  - Implemented mandatory first-login wizard (`first-login-wizard.sh`) enforcing user password change, physical keyboard layout selection (`loadkeys`), and UTC timezone standard.
  - Implemented every-login UTC clock synchronization prompt (`clock-setup.sh`) backed by setuid root C helper (`hsm-set-date`), allowing unprivileged time synchronization without sudo.
  - Implemented USB storage automount service under `/media/usb-*` for `hsmutil` user space key exports and backups.
- **Dual-Slot RAUC A/B Updates**:
  - Integrated `meta-rauc` and `meta-rauc-raspberrypi` with U-Boot boot counter and watchdog fallback.
  - Implemented ECDSA P-256 PKI key generation script (`generate-rauc-keys.sh`) and offline bundle generator (`create-rauc-bundle.sh`).
  - Automated USB update installer service (`rauc-usb-autoinstall`): detects `*.raucb` bundles on USB insertion via `usb-mount.sh` and verifies x509 CMS signatures before streaming into the inactive slot.

---

### 3. Investigation & Remediation: First-Boot Device Tree Failure (`91428c8`)

- **Root Cause Assessment**: [[investigations/01 - RPi Zero 2 W First Boot Device Tree Failure|01 - RPi Zero 2 W First Boot Device Tree Failure]] (Session `7727fe1d-b544-42ae-848b-ab6b2767f731`).
- **Defects Remediated**:
  1. **Device Tree Handoff (`boot.cmd.in`)**: Eliminated erroneous `load mmc 0:1 ${fdt_addr_r} booti.dtb` which halted with error `-14`. Passed firmware-prepared device tree pointer at `${fdt_addr}` directly to `booti ${kernel_addr_r} - ${fdt_addr}`, preserving VideoCore hardware overlays (`disable-bt`, `disable-wifi`, `dwc2`) and dynamic board configuration.
  2. **VideoCore Bootargs Extraction**: Added `fdt addr ${fdt_addr} && fdt get value bootargs /chosen bootargs` to initialize serial console and kernel parameters from `cmdline.txt`.
  3. **Dual-Slot Kernel Isolation**: Updated RAUC boot logic to load `/boot/Image` directly from the active slot (`mmc 0:2` for Slot A, `mmc 0:3` for Slot B) with fallback to `mmc 0:1`, ensuring atomic kernel updates and rollbacks.
  4. **Dual-Slot WIC Partitioning**: Set `WKS_FILE = "sdimage-dual-raspberrypi.wks.in"` in `kas-project.yml` ensuring Partition 1 (`boot`), Partition 2 (`rootfs_A`), and Partition 3 (`rootfs_B`) are provisioned in `.wic` images.

---

## [v0.3.0] — 2026-10-31: Factory Reset Safeguards & Multi-Device Auto-Discovery Disambiguation

> [!NOTE]
> **Release Highlights**: Implementation of irreversible hardware factory reset zeroization (`keystore_factory_reset`) on CryptoCell CC310 keys (slots 1..32) and Shamir master secret in NVS, interactive confirmation prompt and `-y`/`--yes` bypass in `hsmutil reset`, and strict auto-discovery disambiguation (`ErrMultipleDevicesDetected`) preventing accidental targeting of multiple connected HSM dongles.
>
> **Release Status**: Merged via fast-forward into `master` / `main` production branches across all repositories with feature branches pruned. Verified with 100% test pass rate across unit tests, production sysbuilds, and root integration test suites.

### 1. Summary of Changes

| Subsystem | Merged Branch | Target Branch | Commit | Key Additions |
| :--- | :--- | :--- | :--- | :--- |
| **Firmware** (`nrf52840-hsm-firmware`) | `feature/factory-reset-zeroization` (pruned) | `main` | [`ae087c1`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware) | `keystore_factory_reset()` wiping CC310 slots 1..32, SSS zeroization, factory defaults restore, NVS persistence, `AT+RESET` & `AT+FACTORYRESET` (Base commit: [`166c307`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware)). |
| **CLI** (`nrf52840-hsm-cli`) | `feature/reset-safeguard-and-disambiguation` (pruned) | `main` | [`c110845`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli) | Interactive reset safeguard prompt `[y/N]`, `-y/--yes` bypass, `ErrMultipleDevicesDetected` disambiguation error, `ResolvePortOrSerial()` (Base commit: [`29659c2`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli)). |
| **Workspace Root** (`nrf52840-hsm`) | `feature/reset-and-disambiguation-tests` (pruned) | `master` | [`61a400e`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm) | Mock PTY factory zeroization, AT client cryptographic reset test assertions, end-to-end integration test coverage (Base commit: [`c4fb7d6`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm)). |
| **Obsidian Vault** (`nrf52840-hsm-obsidian`) | `master` | `master` | [`6ad1077`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-obsidian) | ADR-0004, feature specification, implementation plan, pre-merge & post-merge release records. |

---

### 2. Detailed Subsystem Changes

#### Firmware Subsystem (`nrf52840-hsm-firmware`)
- **Factory Reset Hardware Zeroization**:
  - Implemented `keystore_factory_reset(void)` in `firmware/lib/src/keystore.c` and header prototype in `include/hsm/keystore.h`.
  - Iterates all 32 persistent slots and calls `crypto_cc310_destroy_key(slot_id)`.
  - Zeroizes RAM Shamir Secret Sharing master secret with `sss_zeroize()`.
  - Restores default master PIN (`"12345678"`) and resets PIN retry counter to `3`.
  - Persists zeroized state to Non-Volatile Storage (`save_to_nvs()`).
  - Emits asynchronous state notification `notify_state(HSM_STATE_UNCONFIGURED)` switching RGB LED to breathing BLUE.
- **AT Command Handler**:
  - Bound `AT+RESET` and added alias `AT+FACTORYRESET` to `keystore_factory_reset()`.

#### Host CLI Subsystem (`nrf52840-hsm-cli`)
- **Confirmation Safeguards (`hsmutil reset`)**:
  - Bound `-y, --yes` flag to bypass interactive prompt.
  - Interactive warning banner and explicit `(yes/no)` confirmation prompt on stdin.
  - Non-affirmative responses abort cleanly with exit code 0 and `Reset cancelled.`.
  - Success feedback informs operator that device is unconfigured and guides to `hsmutil init --threshold`.
- **Multi-Device Auto-Discovery Disambiguation**:
  - Defined domain error `ErrMultipleDevicesDetected` formatting a table of attached devices with BLE serials, AT Data ports, and Console ports.
  - Refactored `AutoDiscoverPortTargeting()` to halt with `ErrMultipleDevicesDetected` if `len(dongles) > 1` and no `--serial` or `--port` is supplied.
  - Implemented `ResolvePortOrSerial()` transparently parsing paths vs 12-digit hex BLE serials for `hsmutil key migrate`.

#### Workspace Root & Test Harness (`scripts`)
- **Mock HSM Serial Engine**:
  - Enhanced `scripts/mock_hsm_pty.py` to clear all key slots, master secret, and revert `shamir_initialized` to `False` on `AT+RESET`.
- **Automated Integration & Regression Tests**:
  - Added cryptographic reset verification in `scripts/test_at_client.py`.
  - Added reset safeguard prompt cancellation and `--yes` assertions to `scripts/run_integration_tests.sh`.

---

### 3. Architecture Decision Records

- [[Architecture/Decisions/ADR-0004 - Factory Reset Semantics and Safeguards|ADR-0004: Factory Reset Semantics and Safe Erasure Safeguards]]
- [[Architecture/Decisions/ADR-0003 - Host Port Discovery and Serial Targeting Strategy|ADR-0003: Host Port Discovery and Serial Targeting Strategy]]

---

## [v0.2.0] — 2026-10-31: USB Descriptors & Device Identification

> [!NOTE]
> **Release Highlights**: Complete implementation of deterministic hardware BLE address serial numbers, runtime USB protocol descriptor patching, macOS dual CDC-ACM device node predictability (`suffix 1` for Console, `suffix 3` for AT data), CLI and PKCS#11 multi-device autodiscovery and `--serial` targeting.

### 1. Summary of Changes

| Subsystem | Merged Branch | Merge Commit | Key Additions |
| :--- | :--- | :--- | :--- |
| **Firmware** (`nrf52840-hsm-firmware`) | `feature/usb-descriptors-ble-address` $\rightarrow$ `main` | [`adb9c1e`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware) | Dynamic FICR BLE serial number hook, runtime protocol patching (`0x0001`/`0x0002`), custom string descriptors. |
| **CLI** (`nrf52840-hsm-cli`) | `feature/usb-descriptors-ble-address` $\rightarrow$ `main` | [`c9f4590`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli) | Port autodiscovery heuristics (score suffix `3`), `hsmutil devices` (`list`) command, `--serial` (`-s`) flag, `NRF52840_HSM_SERIAL`. |
| **PKCS#11** (`nrf52840-hsm-libpkcs11`) | `feature/usb-descriptors-ble-address` $\rightarrow$ `main` | [`4673116`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-libpkcs11) | Port candidate scoring (`serial_score_candidate`), quicksort candidate prioritization, `NRF52840_HSM_SERIAL` support. |
| **Docs Vault** (`nrf52840-hsm-obsidian`) | `master` | [`c29b40a`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-obsidian) | Feature specification, implementation plan, ADR-0001, ADR-0002, ADR-0003, Phase 3-5 session logs. |
| **Workspace Root** (`nrf52840-hsm`) | `master` / `main` | [`4b6d55b`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm) | Root README update with deterministic macOS port mappings, interface tables, and device targeting instructions. |

---

### 2. Detailed Subsystem Changes

#### Firmware (`nrf52840-hsm-firmware`)
- **Dynamic BLE Serial Number**:
  - Implemented `compute_ble_sn()` reading `NRF_FICR->DEVICEADDR[1]` (with BLE Random Static bitmask `0xC000`) and `NRF_FICR->DEVICEADDR[0]`.
  - Formatted 12-character uppercase hex serial (`%04X%08X`, MSB to LSB).
  - Strong symbol override of `usb_update_sn_string_descriptor()` returning dynamic BLE serial string.
  - Set `CONFIG_USB_DEVICE_SN="000000000000"` and enabled `CONFIG_HWINFO=y` in `prj.conf`.
- **Protocol Identifiers & String Descriptors**:
  - Registered user string descriptors via `USBD_STRING_DESCR_USER_DEFINE(primary)`:
    - `"Zephyr CDC ACM Console"`
    - `"nRF52840 HSM AT Interface"`
  - Implemented `usb_serial_patch_descriptors()` traversing `__usb_descriptor_start` prior to `usb_enable(NULL)`:
    - Interface 0 (Console CDC Comm): `bInterfaceProtocol = 0x01`, `iInterface = str_idx_console`.
    - Interface 2 (HSM AT CDC Comm): `bInterfaceProtocol = 0x02`, `iInterface = str_idx_hsm`.
- **Signed Binary Artifacts**:
  - MCUboot signed application: `firmware/app/build-prod/app/zephyr/zephyr.signed.bin`
  - Merged flash image: `firmware/app/build-prod/merged.hex`
  - DFU package: `firmware/app/build-prod/dfu_application.zip`

#### CLI (`nrf52840-hsm-cli`)
- **Port Autodiscovery Heuristics**:
  - Implemented candidate port scoring assigning suffix `3` score `+100` and demoting suffix `1` by `-50`.
  - Added `hsmutil devices` (alias `hsmutil list`) with optional `--json` flag to inspect connected dongles with BLE serial, Console port, and AT data port.
- **Serial Device Targeting**:
  - Added persistent `--serial` (`-s`) flag to target specific dongles by 12-digit hex BLE address or device UID.
  - Supported `NRF52840_HSM_SERIAL` environment variable.
- **Cross-Platform Compilation**:
  - Built binaries across 5 host targets: `darwin-arm64`, `linux-amd64`, `linux-arm64`, `windows-amd64.exe`, `windows-arm64.exe`.
  - Verified 100% Go unit test coverage (`go test ./...`).

#### PKCS#11 Library Provider (`nrf52840-hsm-libpkcs11`)
- **Candidate Port Scoring**:
  - Implemented `serial_score_candidate()`: assigns score `1000` to `tty.usbmodem*3`, `900` to `cu.usbmodem*3`, demoting suffix `1` candidates.
  - Ordered candidates via `qsort` and `compare_port_candidates` before probing.
- **Direct Serial Targeting**:
  - Supports `NRF52840_HSM_SERIAL` environment variable for direct node synthesis `/dev/tty.usbmodem<SERIAL>3`.
- **Test Harness Verification**:
  - 100% CTest pass on `test_p11_mock` and hardware validation on physical dongles.

---

### 3. Verification & Validation Evidence

- **Dual-Device Isolation**: Tested simultaneously with two connected PCA10059 dongles:
  - Dongle 1: BLE `FD54E793A75540C9` $\rightarrow$ AT `/dev/tty.usbmodem31424103`, Console `/dev/tty.usbmodem31424101`
  - Dongle 2: BLE `3302311888499AC7` $\rightarrow$ AT `/dev/tty.usbmodem31424403`, Console `/dev/tty.usbmodem31424401`
- **Zero Crosstalk**: Targeted queries via `hsmutil info -s FD54E793A75540C9` and `hsmutil info -s 3302311888499AC7` confirmed complete channel separation.

---

### 4. Architecture Decision Records

- [[Architecture/Decisions/ADR-0001 - Hardware BLE Address for Deterministic USB Serial Number|ADR-0001: Hardware BLE Address for Deterministic USB Serial Number]]
- [[Architecture/Decisions/ADR-0002 - In-Tree Protocol Descriptor Patching vs macOS DriverKit Extension|ADR-0002: In-Tree Protocol Descriptor Patching vs macOS DriverKit Extension]]
- [[Architecture/Decisions/ADR-0003 - Host Port Discovery and Serial Targeting Strategy|ADR-0003: Host Port Discovery and Serial Targeting Strategy]]

---

## [v0.1.0] — 2026-10-29: Multi-Repository Genesis Architecture & Dual-Stage Secure Boot

> [!NOTE]
> **Genesis Release**: Initial multi-repository workspace initialization, dual-stage secure boot chain, dual CDC-ACM USB endpoints, cross-platform CLI, PKCS#11 shared library provider, and documentation vault.

### 1. Key Additions
- **Dual-Stage Secure Bootloader**: Stage 1 B0 immutable bootloader with CC310 hardware crypto validation $\rightarrow$ Stage 2 MCUboot with dual-slot partition manager $\rightarrow$ Signed Application using root ECDSA P-256 keys (`keys/root-ec-p256.pem`).
- **Dual CDC-ACM Endpoints**: Zephyr kernel logging and console on `cdc_acm_uart0`, HSM binary/AT commands on `cdc_acm_uart1`.
- **RGB LED State Engine**: Visual status indication for unconfigured (pulsing blue), locked (solid red), and unlocked (solid green) states.
- **`hsmutil` CLI / TUI**: Built with Go 1.24, Cobra, and BubbleTea for macOS, Linux, and Windows.
- **PKCS#11 Shared Library Provider**: Cryptoki v2.40 compliant shared library (`libnrf52840_pkcs11.dylib`/`.so`/`.dll`).
- **Root Automation**: Root `Makefile` coordinating `sync`, `status`, `firmware-build`, `firmware-flash`, `cli-build`, and `libpkcs11-build`.
- **Obsidian Documentation Vault**: Centralized vault in `nrf52840-hsm-obsidian/`.
