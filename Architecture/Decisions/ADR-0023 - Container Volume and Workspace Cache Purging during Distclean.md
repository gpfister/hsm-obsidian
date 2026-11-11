---
title: "ADR-0023: Container Volume and Workspace Cache Purging during Distclean"
status: Accepted
date: 2026-11-08
tags:
  - adr
  - architecture
  - build-automation
  - makefile
  - docker
  - podman
  - yocto
  - kas
  - idempotency
deciders:
  - Workspace System Architect
  - Security Software Lead
  - Build & CI/CD Engineer
references:
  - "[[Architecture/Build Lifecycle and Workspace Clean Routines|Build Lifecycle and Workspace Clean Routines]]"
  - "[[Architecture/07 - Air-Gapped RPi HSM OS Architecture|07 - Air-Gapped RPi HSM OS Architecture]]"
  - "[[Specification/Yocto 6.1 Blacksail Upgrade, HW-084 DS3231 RTC Integration and Workspace Distclean|Specification: Yocto 6.1 Blacksail Upgrade, HW-084 RTC & Distclean]]"
  - "[[00 - Index|Vault Index]]"
---

# ADR-0023: Container Volume and Workspace Cache Purging during Distclean

## Context & Problem Statement

The **air-gapped Raspberry Pi HSM OS** ([`hsm-yocto-offgrid-rpi`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi)) is cross-compiled using containerized Siemens **Kas** toolsets (`ghcr.io/siemens/kas/kas:latest` running on Docker or Podman). BitBake generates substantial disk artifacts across multiple locations:
1. `build/tmp`: Temporary build directories, sysroots, and compilation objects (~10–30 GB).
2. `sstate-cache`: BitBake shared-state binary cache objects (~15–40 GB).
3. `downloads`: Upstream tarballs, git clones, and cryptographic signatures (~10–25 GB).
4. Checked-out Kas repositories: External layer git clones (`openembedded-core`, `bitbake`, `meta-yocto`, `meta-openembedded`, `meta-raspberrypi`, `meta-rauc`).
5. **Container Volume (`rpi_yocto_hsm_tmp`)**: A dedicated Docker/Podman named volume mounted to `/tmp` inside the Kas container to accelerate build-time tmpfs operations.

Historically:
- The sub-repository provided only `make clean`, which simply deleted `build/tmp/`.
- Crucially, `make clean` left `sstate-cache`, `downloads`, Kas layer checkouts, and the container volume `rpi_yocto_hsm_tmp` completely untouched.
- When performing major Yocto branch upgrades (such as migrating from Scarthgap 5.0 to Blacksail 6.1, BitBake 2.20 syntax refactorings, or layer manifest adjustments), stale shared state signatures and outdated Kas layer checkouts produced cryptic build failures, checksum mismatches, and non-deterministic behavior.
- Furthermore, the root workspace [`Makefile`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/Makefile) had a `distclean` target that purged firmware, host CLI, and PKCS#11 provider build artifacts, but omitted the Yocto sub-repository entirely.

An atomic, total-reset mechanism (`distclean`) was necessary to restore the workspace to a pristine git clone state without leaving orphaned Docker volumes or corrupting future builds.

---

## Decision Drivers

- **Build Idempotency & Clean-Slate Guarantees**: Provide a deterministic command to completely purge all generated artifacts, caches, and cloned layers across the multi-repo workspace.
- **Orphaned Container Volume Elimination**: Ensure container runtime resources (Docker/Podman named volumes like `rpi_yocto_hsm_tmp`) are reclaimed cleanly.
- **Clear Operational Tiering (`clean` vs `distclean`)**: Preserve `clean` for fast iterative rebuilds (reusing downloaded tarballs and valid shared state) while designating `distclean` as the definitive nuclear reset.
- **Multi-Repository Orchestration Alignment**: Ensure root workspace `make distclean` cascades hierarchically down to all sub-repositories, including `hsm-yocto-offgrid-rpi`.
- **Fail-Safe Container Execution**: Support both Docker and Podman environments gracefully without failing if a volume does not exist.

---

## Considered Options

### Option 1: Maintain Status Quo (`clean` only, manual removal)
- **Description**: Document manual `rm -rf` and `docker volume rm` commands in troubleshooting guides.
- **Pros**: No changes to Makefiles.
- **Cons**: High failure rate; developers and CI runners regularly forget to delete Docker volumes or stale layer checkouts, leading to hours spent debugging cache pollution.
- **Outcome**: **Rejected**.

### Option 2: Overload `make clean` to Purge Everything
- **Description**: Modify existing `clean` target to delete `downloads/`, `sstate-cache/`, layers, and container volumes.
- **Pros**: Only one clean command.
- **Cons**: Catastrophic for developer velocity: every routine clean rebuild would force re-downloading tens of gigabytes of source tarballs over the Internet.
- **Outcome**: **Rejected**.

### Option 3: Two-Tier Clean Architecture with Container Volume Purging
- **Description**:
  1. Retain `clean` for fast workspace rebuilds (clearing `build/tmp/deploy` and compilation artifacts).
  2. Implement `distclean` in `hsm-yocto-offgrid-rpi/Makefile` that wipes:
     - BitBake build artifacts (`rm -rf $(BUILD_DIR)`)
     - Download archives (`rm -rf downloads`)
     - Shared state cache (`rm -rf sstate-cache`)
     - Cloned layer repositories (`rm -rf bitbake openembedded-core meta-yocto meta-openembedded meta-raspberrypi meta-rauc`)
     - Named container volume (`$(CONTAINER_ENGINE) volume rm -f rpi_yocto_hsm_tmp 2>/dev/null || true`)
  3. Expose `yocto-distclean` in the root workspace `Makefile` and hook it directly into root `distclean`.
- **Pros**:
  - Predictable standard semantics: `clean` for fast turnaround, `distclean` for clean slate.
  - Zero leftover Docker/Podman artifacts.
  - Root Makefile single-command orchestration (`make distclean` resets all 5 sub-repos).
  - Robust error handling (`2>/dev/null || true` handles missing volumes and engine discrepancies).
- **Outcome**: **Accepted**.

---

## Decision Outcome

### 1. Sub-Repository `distclean` Target (`hsm-yocto-offgrid-rpi/Makefile`)
Added to [`hsm-yocto-offgrid-rpi/Makefile`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi/Makefile):
```makefile
.PHONY: distclean

distclean: clean
	@echo "==> Purging Yocto workspace, caches, and Kas repositories..."
	@rm -rf $(BUILD_DIR)
	@rm -rf downloads sstate-cache
	@rm -rf bitbake openembedded-core meta-yocto meta-openembedded meta-raspberrypi meta-rauc
	@echo "==> Removing container volume rpi_yocto_hsm_tmp if present..."
	@$(CONTAINER_ENGINE) volume rm -f rpi_yocto_hsm_tmp 2>/dev/null || true
	@echo "Distclean completed."
```

### 2. Root Workspace Orchestration (`Makefile`)
Integrated in root [`Makefile`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/Makefile):
```makefile
.PHONY: yocto-distclean distclean

yocto-distclean:
	@$(MAKE) -C $(YOCTO_DIR) distclean

distclean: firmware-distclean cli-clean libpkcs11-clean yocto-distclean
```

### 3. Cleanup Tier Matrix

| Target | Sub-Repositories Purged | Disk Targets Purged | Docker / Podman Volumes Purged | Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **`make clean` (subrepo)** | `hsm-yocto-offgrid-rpi` | `build/tmp/` | None | Fast recipe re-compilation reusing sstate and downloaded sources |
| **`make distclean` (subrepo)** | `hsm-yocto-offgrid-rpi` | `build/`, `downloads/`, `sstate-cache/`, Kas git checkouts | `rpi_yocto_hsm_tmp` volume purged | Total sub-repository reset before major release, layer rebase, or Kas branch upgrade |
| **`make distclean` (root)** | All 5 sub-repos + Root | Firmware build, CLI binaries, PKCS#11 builds, Yocto caches & layers | `rpi_yocto_hsm_tmp` volume purged | Global workspace nuclear clean slate; brings entire multi-repo workspace to pristine git checkout state |

---

## Consequences & Verification

### Positive Consequences
- **Elimination of Phantom Cache Incompatibilities**: Layer upgrades (such as Yocto 6.1 Blacksail) start from a guaranteed pristine state without stale sstate signatures.
- **Docker/Podman Resource Reclamation**: Frees 50–100 GB of local disk and container volumes on demand.
- **Root-Level Workspace Uniformity**: Running `make distclean` from root orchestrates total teardown of firmware, CLI, PKCS#11, and Yocto artifacts in a single command.

### Negative / Neutral Consequences
- Running `distclean` deletes downloaded source tarballs in `downloads/`; the subsequent build will require network access to re-fetch all open-source packages.
