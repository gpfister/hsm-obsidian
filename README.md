# hsm-obsidian

Centralized engineering knowledge and documentation vault for the **Hardware Security Module (HSM)** project.

---

## Overview

[`hsm-obsidian`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian) serves as the single source of truth for architecture designs, technical specifications, Architecture Decision Records (ADRs), roadmap planning, verification telemetry, issue investigations, and session records across the entire HSM ecosystem.

The vault is maintained both for human navigation inside [Obsidian](https://obsidian.md) and automated indexing by workspace agentic workflows.

> [!NOTE]
> All documentation adheres to the generalized **Hardware Security Module (HSM)** and **HSM USB Key / Dongle** taxonomy formalized in [[Architecture/Decisions/ADR-0014 - Documentation Terminology Generalization to Generic HSM USB Key|ADR-0014]], while maintaining concrete hardware specifications for the initial microcontroller reference implementation ([`hsm-ncs-firmware`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-ncs-firmware) targeting the Nordic Semiconductor nRF52840 USB Dongle `PCA10059`).

---

## Vault Structure

The vault organizes engineering knowledge into dedicated top-level directories:

| Directory | Description | Key Notes & Artifacts |
| :--- | :--- | :--- |
| [`00 - Index.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/00%20-%20Index.md) | Vault master table of contents and navigation index. | Top-level links to all domains and specs. |
| [`Architecture/`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/Architecture) | High-level system architecture, hardware/firmware subsystem topology, and security models. | System Architecture, Dual-Stage Secure Boot, USB Composite Stack, Air-Gapped RPi OS. |
| [`Architecture/Decisions/`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/Architecture/Decisions) | Formal Architecture Decision Records (ADRs) capturing architectural context, decision drivers, considered options, and outcomes. | ADR-0001 through ADR-0018 (USB protocol tags, zeroization safeguards, U-Boot hardening, terminology, CLI restructuring, dual-mode auth, gphsm / libgphsm rebranding, and hsm-pkcs11 sub-repo renaming). |
| [`features/`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/features) | User journeys, feature-level acceptance criteria, and operational capabilities across subsystems. | Sub-repository renaming to hsm-pkcs11, dual-mode auth & gphsm/libgphsm refactoring, AT DFU chunk streaming, macOS CDC-ACM enumeration, factory reset confirmation, CLI restructuring & Bubble Tea UX, WIC/IMG outputs. |
| [`Specification/`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/Specification) | Deep technical specifications, protocol framing, hardware register mappings, and interface contracts. | Sub-repository renaming to hsm-pkcs11, dual-mode auth & gphsm/libgphsm, YubiHSM protocol, AT command syntax, DFU streaming framing, CLI restructuring architecture, USB descriptors. |
| [`Plan/`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/Plan) | Work breakdown structures, multi-repository implementation plans, task matrices, and verification strategies. | Multi-repo implementation plans, Project Roadmap. |
| [`Monitoring/`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/Monitoring) | Live hardware telemetry logs, SWD flashing traces, boot verification captures, and serial console surveillance. | J-Link flashing logs, USB enumeration verification, CDC-ACM surveillance. |
| [`Sessions/`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/Sessions) | Permanent archives of agentic engineering sessions, decisions, and development phases. | Phase-by-phase execution logs, genesis sessions, lifecycle transcripts. |
| [`Hardware/`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/Hardware) | Schematics, pinouts, USB hub integration, and debugger pin mappings. | nRF52840 Dongle PCA10059, Waveshare USB HAT (B), J-Link SWD headers. |
| [`Development/`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/Development) | Developer setup guides, toolchain instructions, and CLI/TUI reference documentation. | Environment setup, Build & Flash, `gphsm` / `hsmutil` CLI Reference, Yocto build guide. |
| [`investigations/`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/investigations) | Post-mortem root-cause analyses, error reproductions, and diagnostic assessments. | Device tree failures, BitBake git fetch errors, reset disambiguation. |
| [`git-status.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/git-status.md) | Multi-repository workspace git tracking, active branch topologies, and worktrees. | Multi-repo branch and commit status index. |
| [`Changelog.md`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/Changelog.md) | Comprehensive workspace changelog recording all versioned milestone releases. | Detailed release notes across all components. |

---

## Obsidian Conventions & Standards

All notes within this vault adhere to strict authoring conventions to ensure bidirectional traceability:

### 1. Internal Wikilinks
Notes reference related architectural concepts, decisions, and specifications using Obsidian internal wikilinks:
- Document links: `[[Note Name]]` or `[[Folder/Note Name]]`
- Aliased links: `[[Folder/Note Name|Custom Link Label]]`
- Sub-heading anchors: `[[Note Name#Section Heading|Anchor Text]]`

> [!TIP]
> Wikilinks enable Obsidian's interactive Graph View, revealing dependency topologies between ADRs, feature specs, and implementation plans.

### 2. YAML Frontmatter
Every note starts with structured YAML metadata enclosed by triple dashes (`---`), indexing core properties:

```yaml
---
title: "Document Title"
tags:
  - hsm
  - architecture
  - security
created: 2026-11-05
status: Approved
references:
  - "[[Architecture/System Architecture|System Architecture]]"
---
```

Standard frontmatter tags categorize notes for dataview queries and tag filtering (`#hsm`, `#architecture`, `#security`, `#adr`, `#plan`, `#specification`, `#monitoring`).

### 3. Mermaid Diagrams
Complex interactions, sequence flows, state machines, and component diagrams are defined inline using Mermaid fenced blocks:

```mermaid
graph LR
    Host["Host Tooling (CLI / PKCS#11)"] <-->|CDC-ACM AT Protocol| Dongle["HSM USB Key"]
    Dongle <-->|Crypto Operations| SecureEngine["CC310 Hardware Engine"]
```

Supported diagram types across the vault include:
- `graph TD` / `graph LR` (Block architecture & subsystem hierarchies)
- `sequenceDiagram` (Protocol handshakes, DFU chunk streaming, and zeroization sequences)
- `stateDiagram-v2` (HSM operational state machine: unconfigured, locked, authenticated)

---

## Role in the HSM Workspace

`hsm-obsidian` acts as the orchestrating documentation backbone for the workspace:
1. **Specification Before Implementation**: New features originate as specifications in [`Specification/`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/Specification) and plans in [`Plan/`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/Plan).
2. **Architecture Decision Records**: Architectural changes must record their rationale in [`Architecture/Decisions/`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-obsidian/Architecture/Decisions).
3. **Traceability**: Implementation sessions, hardware traces, and root-cause analyses are indexed to preserve an unbroken engineering audit trail.
