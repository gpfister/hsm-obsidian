---
title: "ADR-0030: Sub-Repository Workspace Cleanliness via Encapsulated Build Directory"
status: Accepted
date: 2026-11-12
tags:
  - adr
  - architecture
  - yocto
  - build-system
  - kas
  - cleanliness
  - makefile
  - docker
deciders:
  - Workspace System Architect
  - Embedded Linux Engineer
  - Build & Release Engineer
references:
  - "[[Architecture/Decisions/ADR-0023 - Container Volume and Workspace Cache Purging during Distclean|ADR-0023: Container Volume and Workspace Cache Purging]]"
  - "[[Architecture/Build Lifecycle and Workspace Clean Routines|Build Lifecycle and Workspace Clean Routines]]"
  - "[[Architecture/07 - Air-Gapped RPi HSM OS Architecture|07 - Air-Gapped RPi HSM OS Architecture]]"
  - "[[00 - Index|Vault Index]]"
---

# ADR-0030: Sub-Repository Workspace Cleanliness via Encapsulated Build Directory

## Context & Problem Statement

The **air-gapped OS repository** ([`hsm-yocto-offgrid-rpi`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi)) uses **Kas** to manage BitBake layers, dependencies, and containerized builds. In the previous implementation:
1. Kas cloned upstream layer repositories (`openembedded-core`, `bitbake`, `meta-yocto`, `meta-openembedded`, `meta-raspberrypi`, `meta-rauc`) directly into the sub-repository's top-level root directory.
2. BitBake download archives (`downloads/`) and shared state cache (`sstate-cache/`) were also placed at the top level of the sub-repository.
3. As a result, the root directory of `hsm-yocto-offgrid-rpi` became heavily cluttered with 7+ external git clones, nested `.git` folders, and cache directories alongside project configuration files (`kas-*.yml`, `Makefile`, `README.md`, `meta-hsm-os/`).

This layout created multiple operational friction points:
- **Git Clutter & False Status Alerts**: Running `git status` at the root of `hsm-yocto-offgrid-rpi` required complex `.gitignore` exclusions to avoid tracking cloned layer repos. Submodules or embedded `.git` directories in cloned layers occasionally triggered false git index alerts.
- **Accidental Deletions & Partial Cleans**: Distinguishing project source files from build-generated artifacts required error-prone manual scripting.
- **Suboptimal Clean Routines**: The subrepo `clean` and `distclean` targets had to enumerate each cloned layer individually (`rm -rf openembedded-core meta-yocto ...`). If a new layer was introduced in Kas, clean scripts had to be manually updated to prevent residue.

---

## Decision Drivers

- **Pristine Sub-Repository Root**: Ensure that running `ls` or `git status` in `hsm-yocto-offgrid-rpi` reveals only version-controlled project assets (`meta-hsm-os/`, `scripts/`, `kas-*.yml`, `Makefile`, `README.md`).
- **Single-Root Encapsulation (`build/`)**: Confine all transient BitBake artifacts, downloaded archives, sstate caches, layer checkouts, and build configurations inside a single `build/` directory.
- **Atomic Cleanup Semantics**: Guarantee that `rm -rf build/` completely restores the repository to a clean, cloned state without residual caches or untracked directories.
- **Docker / Podman APFS Performance Invariance**: Maintain fast APFS/Docker I/O by keeping `build/tmp` mapped to the high-performance Linux ext4 container volume (`rpi_yocto_hsm_tmp`) while mapping `build/downloads` and `build/sstate-cache` from the host.

---

## Considered Options

### Option 1: Retain Top-Level Layer Clones with Expanded `.gitignore` (Status Quo)
- **Description**: Keep cloning layers and caches in the repository root and maintain a long list of ignore patterns in `.gitignore`.
- **Pros**: Zero changes to Kas configuration files.
- **Cons**: Sub-repository root remains cluttered; high risk of dirty git status; brittle clean routines requiring explicit lists of directories.

### Option 2: Place Cloned Layers in a Separate External Directory (Sibling Directory)
- **Description**: Clone layers into `../yocto-layers/` outside `hsm-yocto-offgrid-rpi`.
- **Pros**: Sub-repository root is clean.
- **Cons**: Pollutes the parent multi-repository workspace root (`ncs-nrf52840dongle-kms/`); breaks self-containment of the sub-repository; breaks CI/CD containers that expect isolated checkouts.

### Option 3: Encapsulate All Transient Artifacts inside `build/` (Selected)
- **Description**:
  - Reconfigure `kas-project.yml` to clone all external layers into `build/repos/<repo-name>`.
  - Place persistent host caches in `build/downloads/` and `build/sstate-cache/`.
  - Update `scripts/run-kas.sh` and `Makefile` to manage paths relative to `build/`.
  - Simplify `.gitignore` to ignore only `build/`.
- **Pros**:
  - 100% encapsulation: the repository root contains zero external clones or cache folders.
  - Trivial teardown: `rm -rf build/` completely purges all build artifacts and layer clones atomically.
  - Simplified `.gitignore`: a single entry `build/` covers all ephemeral artifacts.
  - Self-contained: does not spill outside `hsm-yocto-offgrid-rpi`.
- **Cons**: Initial build after migration must clone layers into `build/repos/`.

---

## Decision Outcome

**Accepted Option 3**. All external layers, downloads, sstate caches, and build outputs are strictly encapsulated inside the `build/` directory.

### Directory Layout Blueprint

```text
hsm-yocto-offgrid-rpi/
├── Makefile                      # Orchestration targets (clean, distclean, build)
├── kas-project.yml               # Base Kas configuration (repo paths: build/repos/<name>)
├── kas-dev.yml                   # Dev overlay
├── kas-prod.yml                  # Prod overlay
├── README.md                     # Documentation
├── .gitignore                    # Simplified to ignore build/
├── meta-hsm-os/                  # In-tree custom Yocto layer
├── scripts/                      # Build & verification scripts
└── build/                        # [GIT-IGNORED] All transient & cached artifacts
    ├── repos/                    # Kas cloned layer repositories
    │   ├── openembedded-core/
    │   ├── bitbake/
    │   ├── meta-yocto/
    │   ├── meta-openembedded/
    │   ├── meta-raspberrypi/
    │   └── meta-rauc/
    ├── downloads/                # BitBake source archives (host persistent)
    ├── sstate-cache/             # Shared state cache (host persistent)
    ├── conf/                     # Generated local.conf and bblayers.conf
    └── tmp/                      # Container volume mount (rpi_yocto_hsm_tmp)
```

### Configuration Specifications

1. **Kas Layer Pathing (`kas-project.yml`)**:
   ```yaml
   repos:
     openembedded-core:
       url: https://git.openembedded.org/openembedded-core
       branch: blacksail
       path: build/repos/openembedded-core
       layers:
         meta:

     bitbake:
       url: https://git.openembedded.org/bitbake
       branch: "2.20"
       path: build/repos/bitbake
       layers:
         .: disabled

     meta-yocto:
       url: https://git.yoctoproject.org/meta-yocto
       branch: blacksail
       path: build/repos/meta-yocto
       layers:
         meta-poky:
         meta-yocto-bsp:

     meta-openembedded:
       url: https://git.openembedded.org/meta-openembedded
       branch: blacksail
       path: build/repos/meta-openembedded
       layers:
         meta-oe:
         meta-python:
         meta-networking:
         meta-filesystems:

     meta-raspberrypi:
       url: https://github.com/agherzan/meta-raspberrypi.git
       branch: master
       path: build/repos/meta-raspberrypi

     meta-rauc:
       url: https://github.com/rauc/meta-rauc.git
       branch: master
       path: build/repos/meta-rauc

     meta-hsm-os:
       path: meta-hsm-os
   ```

2. **Container Volume Mounts (`scripts/run-kas.sh`)**:
   - `mkdir -p "${REPO_DIR}/build/downloads" "${REPO_DIR}/build/sstate-cache"`
   - Map volume `-v "${REPO_DIR}/build/downloads:/downloads:rw"`
   - Map volume `-v "${REPO_DIR}/build/sstate-cache:/sstate-cache:rw"`

3. **Subrepo Clean Targets (`Makefile`)**:
   - `clean`: `rm -rf $(BUILD_DIR)/tmp` (cleans compilation outputs; preserves downloads, sstate, and layer checkouts).
   - `distclean`: `rm -rf $(BUILD_DIR)` and `$(CONTAINER_ENGINE) volume rm -f rpi_yocto_hsm_tmp` (nuclear purge returning repository to pristine state).

---

## Consequences

### Positive
- **Pristine Source Tree**: Root of `hsm-yocto-offgrid-rpi` remains pristine; `git status` reports only versioned recipe and script changes.
- **Robust Cleanliness**: A single `rm -rf build/` removes all build products, caches, and clones, eliminating leftover layer artifacts.
- **Simplified Ignore Rules**: `.gitignore` is reduced to ignoring `build/` rather than tracking individual upstream repositories.

### Negative / Neutral
- **Layer Re-Cloning on Distclean**: Executing `make distclean` purges `build/repos/`, requiring Kas to re-clone upstream layers on the next build. This is the desired behavior for a true clean-slate distclean.

---

## Verification & Test Assertions

1. **Path Assertion**:
   - Validate that all external entries in `kas-project.yml` have `path: build/repos/<name>`.
2. **Directory Isolation Check**:
   - After running Kas, verify that `ls -d openembedded-core meta-* bitbake` in the subrepo root returns nothing (exit code non-zero).
   - Verify that `ls build/repos/` contains all cloned layers.
3. **Distclean Validation**:
   - Running `make distclean` completely removes `build/` and leaves the subrepo root with zero untracked directories.
