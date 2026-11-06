---
title: "Technical Specification: Sub-Repository Renaming to hsm-pkcs11"
tags:
  - specification
  - technical-spec
  - pkcs11
  - libgphsm
  - hsm-pkcs11
  - multi-repo
  - naming-convention
  - abi-compatibility
created: 2026-11-06
updated: 2026-11-06
status: Phase 5 (Review Approved / Pre-Merge Verified)
adrs:
  - "[[Architecture/Decisions/ADR-0018 - Sub-Repository Renaming to hsm-pkcs11 with libgphsm ABI Invariance|ADR-0018: Sub-Repository Renaming to hsm-pkcs11 with libgphsm ABI Invariance]]"
  - "[[Architecture/Decisions/ADR-0017 - Host CLI Rebranding to gphsm and PKCS11 Refactoring to libgphsm|ADR-0017: Host CLI Rebranding to gphsm and PKCS#11 Refactoring to libgphsm]]"
references:
  - "[[Plan/Sub-Repository Renaming to hsm-pkcs11 Implementation Plan|Implementation Plan: Sub-Repository Renaming to hsm-pkcs11]]"
  - "[[features/subrepo-rename-hsm-pkcs11|Feature Specification: Sub-Repository Renaming to hsm-pkcs11]]"
  - "[[00 - Index|Vault Index]]"
---

# Technical Specification: Sub-Repository Renaming to `hsm-pkcs11`

> [!NOTE]
> **Specification Purpose**: Formalizes repository paths, git remote tracking contracts, build system definitions, and binary output invariance for renaming the PKCS#11 sub-repository to `hsm-pkcs11` while retaining `libgphsm` ABI compatibility.

---

## 1. Executive Summary & Problem Motivation

### 1.1 Context
The nRF52840 HSM workspace coordinates multiple sub-repositories following a uniform naming convention:
- [`hsm-ncs-firmware`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware) (Firmware)
- [`hsm-cli`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-cli) (Host CLI & Administration)
- [`hsm-obsidian`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian) (Documentation Vault)
- [`hsm-yocto-offgrid-rpi`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi) (Air-gapped appliance OS)

Historically, the PKCS#11 shared library provider repository has had naming discrepancies across different layers:
- Local directory name: `libgphsm`
- Remote GitHub repository: `gpfister/hsm-libpkcs11`
- References in `scripts/worktree.sh`: `hsm-libpkcs11`
- References in `Makefile`: `libgphsm`
- Output binary library: `libgphsm.dylib` (macOS), `libgphsm.so` (Linux), `libgphsm.dll` (Windows)

### 1.2 Objective
Harmonize the repository naming convention by renaming both the local workspace directory and the GitHub remote repository to `hsm-pkcs11`, while strictly preserving compiled binary library names, CMake project definitions, and ABI compatibility (`libgphsm`) so external tooling (OpenSC, OpenSSL 3, OpenSSH) and existing configurations remain functional without breaking changes.

---

## 2. Architectural Boundary & Invariance Principles

```
+-----------------------------------------------------------------------------------+
| Workspace Root: nrf52840-hsm                                                      |
|                                                                                   |
|   +-------------------+  Rename Local Dir   +---------------------------------+   |
|   | libgphsm/         | =================>  | hsm-pkcs11/                     |   |
|   +-------------------+                     +---------------------------------+   |
|                                                             |                     |
|                                                             v                     |
|                                             +---------------------------------+   |
|                                             | Compiled Artifacts (Unchanged)  |   |
|                                             | - libgphsm.dylib (macOS)        |   |
|                                             | - libgphsm.so (Linux)           |   |
|                                             | - libgphsm.dll (Windows)        |   |
|                                             +---------------------------------+   |
|                                                             ^                     |
|                                                             | ABI Compatibility   |
|   +---------------------------------------------------------+-----------------+   |
|   | External Consumers: OpenSC pkcs11-tool, OpenSSL pkcs11-provider, OpenSSH  |   |
+---+---------------------------------------------------------------------------+---+
                                      |
                                      | GitHub CLI rename
                                      v
+-----------------------------------------------------------------------------------+
| GitHub Remote: gpfister/hsm-libpkcs11 -> gpfister/hsm-pkcs11                      |
| Remote URL: git@github.com:gpfister/hsm-pkcs11.git                                 |
+-----------------------------------------------------------------------------------+
```

### Invariance Rules:
1. **ABI & Binary Invariance**: The compiled shared provider library must continue to output as `libgphsm.dylib` / `libgphsm.so` / `libgphsm.dll`. The CMake project name (`project(libgphsm C)`) and output target properties (`OUTPUT_NAME "gphsm"`, `PREFIX "lib"`) in [`CMakeLists.txt`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-pkcs11/CMakeLists.txt) must remain intact.
2. **Backward Compatibility in Git Ignore**: Existing `.gitignore` entries (`/libgphsm/`, `/hsm-libpkcs11/`) must be preserved alongside the new entry `/hsm-pkcs11/`.
3. **Workspace Tooling Parity**: All root Makefile targets (`make libpkcs11-build`, `make libpkcs11-test`, `make libpkcs11-clean`, `make status`, `make sync`) and integration tests must function seamlessly with the new folder path.

---

## 3. Affected Repositories & Files

| Target | Path | Type of Modification |
| :--- | :--- | :--- |
| **Local Directory** | [`libgphsm`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/libgphsm) $\rightarrow$ [`hsm-pkcs11`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-pkcs11) | Directory rename (filesystem move) |
| **Git Remote** | `gpfister/hsm-libpkcs11` $\rightarrow$ `gpfister/hsm-pkcs11` | GitHub repository rename & remote origin URL update |
| **Workspace Root** | [`Makefile`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/Makefile) | Update `SUBREPOS` and `LIBPKCS11_DIR` variables |
| **Workspace Root** | [`.gitignore`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/.gitignore) | Add `/hsm-pkcs11/` entry |
| **Workspace Scripts** | [`scripts/worktree.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts/worktree.sh) | Update `SUBREPOS` array replacing `hsm-libpkcs11` with `hsm-pkcs11` |
| **Workspace Scripts** | [`scripts/run_integration_tests.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts/run_integration_tests.sh) | Update `P11_DIR` variable to `${WORKSPACE_ROOT}/hsm-pkcs11` |
| **Workspace Docs** | [`README.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/README.md) | Update sub-repository listing, descriptions, and file links |
| **Workspace Docs** | [`AGENTS.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/AGENTS.md) | Update directory tree and section 2.4 heading and path |

---

## 4. Functional Requirements

### FR-1: Local Workspace Directory Relocation
- The directory `libgphsm` at `/Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/libgphsm` must be moved/renamed to `/Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-pkcs11`.
- Any existing build cache or git metadata within the sub-repository must remain intact during relocation.

### FR-2: GitHub Remote Repository Rename & Remote URL Reconfiguration
- The GitHub repository `gpfister/hsm-libpkcs11` must be renamed to `gpfister/hsm-pkcs11` using GitHub CLI (`gh repo rename hsm-pkcs11 --repo gpfister/hsm-libpkcs11 --yes`).
- The git remote `origin` inside the renamed `hsm-pkcs11` directory must be updated to `git@github.com:gpfister/hsm-pkcs11.git`.
- Verification of `git remote -v` inside `hsm-pkcs11` must confirm `origin` points to `git@github.com:gpfister/hsm-pkcs11.git`.

### FR-3: Shared Library Target & ABI Invariance
- Building `hsm-pkcs11` via CMake must continue to output dynamic libraries prefixed with `libgphsm`:
  - macOS: `build/libgphsm.dylib`
  - Linux: `build/libgphsm.so`
  - Windows: `build/libgphsm.dll`
- CMake configuration in `hsm-pkcs11/CMakeLists.txt` must preserve `OUTPUT_NAME "gphsm"` and `PREFIX "lib"`.
- Unit tests run via `ctest` in `hsm-pkcs11` must find and link against `libgphsm`.

### FR-4: Workspace Build Orchestration Updates
- [`Makefile`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/Makefile):
  - Line 19: `SUBREPOS := hsm-ncs-firmware hsm-cli hsm-obsidian hsm-pkcs11 hsm-yocto-offgrid-rpi`
  - Line 24: `LIBPKCS11_DIR := hsm-pkcs11`
- [`.gitignore`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/.gitignore):
  - Add `/hsm-pkcs11/` under `# Sub-repositories`.
- [`scripts/worktree.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts/worktree.sh):
  - Line 11: `SUBREPOS=(hsm-ncs-firmware hsm-cli hsm-obsidian hsm-pkcs11 hsm-yocto-offgrid-rpi)`
- [`scripts/run_integration_tests.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts/run_integration_tests.sh):
  - Line 9: `P11_DIR="${WORKSPACE_ROOT}/hsm-pkcs11"`

### FR-5: Workspace Documentation & Agent Guidelines Synchronization
- [`README.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/README.md):
  - Update sub-repo bullet point from `libgphsm/` to `hsm-pkcs11/`.
  - Maintain mention of compiled provider name `libgphsm` for clarity.
- [`AGENTS.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/AGENTS.md):
  - Update Section 1 directory layout tree: replace `libgphsm/` with `hsm-pkcs11/`.
  - Update Section 2.4 header: `2.4 Cryptoki PKCS#11 Provider Library (hsm-pkcs11 / libgphsm)`.

---

## 5. Non-Functional Requirements

### NFR-1: Safety & Zero ABI Breakage
- No external consumer depending on `libgphsm.dylib` or `libgphsm.so` (e.g., `pkcs11-tool`, OpenSSH `PKCS11Provider`, or Yocto appliance configurations) shall encounter missing symbol or naming discrepancies.

### NFR-2: Multi-Repository Synchronization & Worktree Compatibility
- Running `make status` or `make st` from workspace root must track all 5 sub-repositories cleanly without reporting `libgphsm` or `hsm-pkcs11` as untracked directories or missing sub-repositories.
- Running `make worktree-list` or creating a worktree with `make worktree-add` must correctly operate on `hsm-pkcs11`.

### NFR-3: Idempotence & Safe Rollback
- All operations must be idempotent. If a git remote update fails or network is unavailable, local operations must remain intact and recoverable.

---

## 6. Verifiable Acceptance Criteria

1. **Local Filesystem Validation**:
   - `test -d /Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-pkcs11` returns exit code 0.
   - `test ! -d /Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/libgphsm` returns exit code 0.

2. **Git Remote URL Validation**:
   - `git -C hsm-pkcs11 remote get-url origin` returns `git@github.com:gpfister/hsm-pkcs11.git`.

3. **Workspace Automation & Status Validation**:
   - `make status` completes without errors and lists `hsm-pkcs11` with its current git branch.
   - `git status --porcelain` at workspace root only reflects expected modified files (`Makefile`, `.gitignore`, `scripts/worktree.sh`, `scripts/run_integration_tests.sh`, `README.md`, `AGENTS.md`).

4. **Build & Test Verification**:
   - `make libpkcs11-build` compiles successfully from `hsm-pkcs11/` and outputs `hsm-pkcs11/build/libgphsm.dylib` (on macOS) or appropriate platform extension.
   - `make libpkcs11-test` passes all CTest test cases.
   - `make integration-test` runs end-to-end integration test harness with mock HSM PTY and passes without error.

---

## 7. Implementation & Pre-Merge Verification Status

- **Status**: Implemented and verified (Phase 5 Review Approved / Pre-Merge Ready)
- **Branch**: `task/rename-subrepo-hsm-pkcs11`
- **Root Commit**: [`e53ab09`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm) (`e53ab09`) — *chore(workspace): standardize PKCS#11 sub-repo to hsm-pkcs11*
- **Acceptance Criteria Verification**: All acceptance criteria 1 through 4 have been validated and confirmed.
