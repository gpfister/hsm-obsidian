---
title: "Feature Specification: Sub-Repository Renaming to hsm-pkcs11"
tags:
  - feature
  - specification
  - pkcs11
  - libgphsm
  - hsm-pkcs11
  - multi-repo
  - naming-convention
  - abi-compatibility
created: 2026-11-06
updated: 2026-11-06
status: "Phase 3 (Plan Approved)"
target_branches:
  root: task/rename-subrepo-hsm-pkcs11
  pkcs11: main
  obsidian: main
adrs:
  - "[[Architecture/Decisions/ADR-0018 - Sub-Repository Renaming to hsm-pkcs11 with libgphsm ABI Invariance|ADR-0018: Sub-Repository Renaming to hsm-pkcs11 with libgphsm ABI Invariance]]"
  - "[[Architecture/Decisions/ADR-0017 - Host CLI Rebranding to gphsm and PKCS11 Refactoring to libgphsm|ADR-0017: Host CLI Rebranding to gphsm and PKCS#11 Refactoring to libgphsm]]"
  - "[[Architecture/Decisions/ADR-0015 - HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS11 Equivalence|ADR-0015: HSM CLI Restructuring, Static Compilation, Bubble Tea UX, and PKCS#11 Equivalence]]"
references:
  - "[[Plan/Sub-Repository Renaming to hsm-pkcs11 Implementation Plan|Implementation Plan: Sub-Repository Renaming to hsm-pkcs11]]"
  - "[[Specification/Sub-Repository Renaming to hsm-pkcs11|Technical Specification: Sub-Repository Renaming to hsm-pkcs11]]"
  - "[[00 - Index|Vault Index]]"
---

# Feature Specification: Sub-Repository Renaming to `hsm-pkcs11`

> [!NOTE]
> **Lifecycle Phase**: Phase 3 (Plan Approved — Advancing to Phase 4 Develop)  
> **Related Architecture Decision Records**:
> - [[Architecture/Decisions/ADR-0018 - Sub-Repository Renaming to hsm-pkcs11 with libgphsm ABI Invariance|ADR-0018: Sub-Repository Renaming to hsm-pkcs11 with libgphsm ABI Invariance]]  
> - [[Architecture/Decisions/ADR-0017 - Host CLI Rebranding to gphsm and PKCS11 Refactoring to libgphsm|ADR-0017: Host CLI Rebranding to gphsm and PKCS#11 Refactoring to libgphsm]]  
> **Target Repositories & Working Branches**:
> - Root Workspace: `task/rename-subrepo-hsm-pkcs11`
> - Sub-Repository: `hsm-pkcs11` (`main`)
> - Knowledge Vault: `hsm-obsidian` (`main`)

---

## 1. Feature Overview & Scope

This feature harmonizes the sub-repository naming across the multi-repository workspace by renaming the PKCS#11 provider repository to **`hsm-pkcs11`**, bringing it into complete alignment with the workspace's canonical `hsm-<component>` naming standard (`hsm-ncs-firmware`, `hsm-cli`, `hsm-obsidian`, `hsm-yocto-offgrid-rpi`, `hsm-pkcs11`).

Simultaneously, this feature establishes strict **ABI and binary invariance**: the compiled dynamic provider library remains **`libgphsm.dylib`** (macOS), **`libgphsm.so`** (Linux), and **`libgphsm.dll`** (Windows) with identical C symbol exports, preserving transparent compatibility with OpenSC `pkcs11-tool`, OpenSSL 3 `pkcs11-provider`, and OpenSSH.

---

## 2. Key Capabilities & Deliverables

1. **GitHub Remote Repository Harmonization**:
   - The remote GitHub repository is renamed from `gpfister/hsm-libpkcs11` to `gpfister/hsm-pkcs11` using `gh repo rename`.
   - The local sub-repository `origin` URL is updated to `git@github.com:gpfister/hsm-pkcs11.git`.

2. **Filesystem Relocation**:
   - The sub-repository local directory is moved from `libgphsm/` to `hsm-pkcs11/`.
   - Stale CMake build caches in `build/` are purged to prevent path cache poisoning.

3. **Workspace Build & Tooling Synchronization**:
   - Root [`Makefile`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/Makefile): `SUBREPOS` array and `LIBPKCS11_DIR` variable updated to `hsm-pkcs11`.
   - Root [`.gitignore`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/.gitignore): `/hsm-pkcs11/` added; backward compatibility entries (`/libgphsm/`, `/hsm-libpkcs11/`) preserved.
   - Worktree script [`scripts/worktree.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts/worktree.sh): `SUBREPOS` updated to `hsm-pkcs11`.
   - Test harness [`scripts/run_integration_tests.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/scripts/run_integration_tests.sh): `P11_DIR` path updated to `${WORKSPACE_ROOT}/hsm-pkcs11`.

4. **Documentation & Guidelines Parity**:
   - [`README.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/README.md) and [`AGENTS.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/AGENTS.md) updated to document `hsm-pkcs11/` while detailing the `libgphsm` ABI.

---

## 3. ABI & Binary Compatibility Matrix

| Property | Value | Compatibility Status |
| :--- | :--- | :--- |
| **Sub-Repository Path** | `hsm-pkcs11/` | Renamed (Was `libgphsm/`) |
| **Remote Repository** | `gpfister/hsm-pkcs11` | Renamed (Was `gpfister/hsm-libpkcs11`) |
| **Remote Git URL** | `git@github.com:gpfister/hsm-pkcs11.git` | Updated |
| **CMake Project Name** | `libgphsm` | **Unchanged** |
| **macOS Provider Binary** | `build/libgphsm.dylib` | **Unchanged** |
| **Linux Provider Binary** | `build/libgphsm.so` | **Unchanged** |
| **Windows Provider Binary** | `build/libgphsm.dll` | **Unchanged** |
| **OpenSC Compatibility** | `pkcs11-tool --module build/libgphsm.dylib` | **Fully Preserved** |
| **OpenSSL 3 Compatibility**| `pkcs11-provider` linking `libgphsm` | **Fully Preserved** |
| **OpenSSH Compatibility**  | `ssh -I build/libgphsm.dylib` | **Fully Preserved** |

---

## 4. Verification & Acceptance

- [ ] Directory `test -d hsm-pkcs11` exists and `test ! -d libgphsm` confirms clean move.
- [ ] Git remote `origin` confirms `git@github.com:gpfister/hsm-pkcs11.git`.
- [ ] `make status` successfully iterates over all 5 sub-repositories including `hsm-pkcs11`.
- [ ] `make libpkcs11-build` compiles `hsm-pkcs11/build/libgphsm.dylib`.
- [ ] `make libpkcs11-test` passes all Cryptoki test cases.
- [ ] `make integration-test` executes end-to-end against mock HSM PTY.
