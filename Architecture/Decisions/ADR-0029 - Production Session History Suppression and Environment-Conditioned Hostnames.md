---
title: "ADR-0029: Production Session History Suppression and Environment-Conditioned Hostnames"
status: Accepted
date: 2026-11-12
tags:
  - adr
  - architecture
  - security
  - hardening
  - anti-forensics
  - hostname
  - yocto
  - rpi
deciders:
  - Workspace System Architect
  - Security Software Lead
  - Embedded Linux Engineer
references:
  - "[[Architecture/Decisions/ADR-0026 - HDMI Console Noise Suppression and Log Level Hardening|ADR-0026: HDMI Console Noise Suppression]]"
  - "[[Architecture/Decisions/ADR-0027 - Appliance Identity Standardization to piKMS and CLI Packaging Alignment to gphsm|ADR-0027: Appliance Identity Standardization]]"
  - "[[Architecture/07 - Air-Gapped RPi HSM OS Architecture|07 - Air-Gapped RPi HSM OS Architecture]]"
  - "[[00 - Index|Vault Index]]"
---

# ADR-0029: Production Session History Suppression and Environment-Conditioned Hostnames

## Context & Problem Statement

The **piKMS air-gapped cryptographic appliance** ([`hsm-yocto-offgrid-rpi`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi)) is deployed across two distinct operating environments:
1. **Production Mode (`HSM_DEV_MODE=0`)**: Used for high-assurance cryptographic ceremonies (Root CA generation, code signing, custodian share generation, physical master key storage).
2. **Development Mode (`HSM_DEV_MODE=1`)**: Used by firmware, CLI, and kernel developers for debugging, automated integration testing, and driver validation.

Two critical usability and operational security concerns were identified:

### 1. Forensic Risks from Interactive Shell History
In production mode, operators execute sensitive commands that may involve parameter flags, key slot indices, or custodian credentials. Standard Linux distributions write interactive bash command history to `~/.bash_history`. Even in an air-gapped system:
- Command history may persist across reboots if the home directory is mounted or backed up.
- In memory or core dumps, history buffers expose previously typed commands and arguments.
- Shoulder surfing or unauthorized access to an active physical terminal allows trivial recall of sensitive operations via the Up-Arrow key.
- Conversely, in development environments (`HSM_DEV_MODE=1`), developers strongly depend on command line history for debugging, rapid iterative testing, and command recall.

### 2. Environment Hostname Disambiguation
In [[Architecture/Decisions/ADR-0027 - Appliance Identity Standardization to piKMS and CLI Packaging Alignment to gphsm|ADR-0027]], the appliance hostname was standardized to `piKMS`. However:
- Having identical hostnames in production and development makes it impossible for operators and test runners to visually or programmatically differentiate between a secured production node (`piKMS`) and an unhardened development node (`piKMS-dev`) that allows root logins and interactive serial debugging.
- Operators risk confusing environments during critical key ceremonies.

---

## Decision Drivers

- **Zero-Footprint Session Anti-Forensics**: Completely eliminate command history logging in production sessions (`HSM_DEV_MODE=0`).
- **Developer Ergonomics**: Preserve full bash history functionality when running in development mode (`HSM_DEV_MODE=1`).
- **Visual & Programmatic Disambiguation**: Clearly differentiate production (`piKMS`) from development (`piKMS-dev`) at the terminal prompt and in system identity files (`/etc/hostname`, `/etc/hosts`).
- **Declarative BitBake Implementation**: Enforce the differentiation cleanly at image build time via Yocto recipes and profile configuration scripts.

---

## Considered Options

### Option 1: Global History Deactivation Across All Builds
- **Description**: Disable bash history universally in both development and production images.
- **Pros**: Uniform security stance across all builds.
- **Cons**: Severe degradation of developer experience during testing, debugging, and bring-up.

### Option 2: Rely Solely on Read-Only Filesystem for History Prevention
- **Description**: Rely on the read-only rootfs to prevent writes to disk, leaving in-memory bash history active.
- **Pros**: Requires no profile scripts.
- **Cons**: In-memory history remains active during the session; pressing Up-Arrow on the physical monitor displays all previous commands; tmpfs overlays mounted on `/home` will still capture the `.bash_history` file until unmounted.

### Option 3: Conditional Profile Script & Image-Specific Hostname Override (Selected)
- **Description**:
  - Deploy a conditional profile drop-in (`/etc/profile.d/99-hsm-history.sh`) in `init-wizard.bb` that sets `HISTFILE=/dev/null`, `HISTSIZE=0`, `HISTFILESIZE=0`, and `set +o history` exclusively when `HSM_DEV_MODE == "0"`.
  - Configure baseline hostname `piKMS` in `hsm-image-base.bb` / `hsm-image-prod.bb`, and override to `piKMS-dev` in `hsm-image-dev.bb` via `hostname:pn-base-files = "piKMS-dev"`.
- **Pros**:
  - Perfect defense-in-depth: production terminals have zero history in memory, zero history on disk, and disabled line-editing recall.
  - Development builds retain complete bash history for developer agility.
  - Console prompts instantly convey environment status (`hsmutil@piKMS:~$` vs `hsmutil@piKMS-dev:~$`).
- **Cons**: Minor recipe configuration branching between `prod` and `dev` images.

---

## Decision Outcome

**Accepted Option 3**. We implement environment-conditioned shell history suppression and environment-specific hostname definitions.

### Implementation Details

1. **Conditional Shell History Suppression (`meta-hsm-os/recipes-core/init-wizard/init-wizard.bb`)**:
   - Recipe checks `HSM_DEV_MODE ??= "0"`.
   - When building production (`HSM_DEV_MODE == "0"`), installs `/etc/profile.d/99-hsm-history.sh`:
     ```bash
     # Production mode: Disable bash history retention
     export HISTFILE=/dev/null
     export HISTSIZE=0
     export HISTFILESIZE=0
     set +o history 2>/dev/null || true
     ```
   - When building development (`HSM_DEV_MODE == "1"`), the profile script is omitted, leaving default Bash history enabled.

2. **Hostname Environment Specialization**:
   - **Production & Baseline** (`meta-hsm-os/recipes-core/images/hsm-image-base.bb`):
     ```bitbake
     hostname:pn-base-files = "piKMS"
     ```
     Yields prompt `hsmutil@piKMS:~$` and hostname `piKMS`.
   - **Development** (`meta-hsm-os/recipes-core/images/hsm-image-dev.bb`):
     ```bitbake
     hostname:pn-base-files = "piKMS-dev"
     ```
     Yields prompt `hsmutil@piKMS-dev:~$` and hostname `piKMS-dev`.

---

## Consequences

### Positive
- **Anti-Forensic Privacy**: In production, no command history exists in memory or on persistent/volatile storage. Key arguments and sensitive flags disappear the instant execution finishes.
- **Zero Ambiguity**: Terminal prompt headers clearly signal whether the node is a production air-gapped station or an unhardened development unit.
- **Uncompromised Developer Workflow**: Developers retain full command history and editing shortcuts in development images.

### Negative / Neutral
- **Production Operator Convenience**: Production operators cannot press Up-Arrow to re-run past commands. Operators should use scripted commands or execute management operations via `gphsm` menus.

---

## Verification & Test Assertions

1. **Production History Check**:
   - `echo $HISTFILE` in production shell &rarr; `/dev/null`.
   - `echo $HISTSIZE` &rarr; `0`.
   - `history` &rarr; empty or returns `history: history library has no entries`.
   - Verify `ls -la /home/hsmutil/.bash_history` &rarr; file does not exist.
2. **Development History Check**:
   - `echo $HISTFILE` in dev shell &rarr; `/home/hsmutil/.bash_history`.
   - Run a command, exit shell, inspect `.bash_history` &rarr; command is logged.
3. **Hostname Validation**:
   - On production image: `cat /etc/hostname` &rarr; `piKMS`.
   - On development image: `cat /etc/hostname` &rarr; `piKMS-dev`.
