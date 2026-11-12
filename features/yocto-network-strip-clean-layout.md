---
title: "Feature: Kernel Network Stripping, Clean Subrepo Architecture, Environment Conditioning, and gphsm Workflow"
status: Approved
date: 2026-11-12
tags:
  - feature
  - architecture
  - yocto
  - security
  - hardening
  - kas
  - gphsm
references:
  - "[[Architecture/Decisions/ADR-0028 - Complete Kernel Network Protocol Stripping|ADR-0028: Complete Kernel Network Protocol Stripping]]"
  - "[[Architecture/Decisions/ADR-0029 - Production Session History Suppression and Environment-Conditioned Hostnames|ADR-0029: Production Session History Suppression and Environment-Conditioned Hostnames]]"
  - "[[Architecture/Decisions/ADR-0030 - Sub-Repository Workspace Cleanliness via Encapsulated Build Directory|ADR-0030: Sub-Repository Workspace Cleanliness via Encapsulated Build Directory]]"
  - "[[Architecture/Decisions/ADR-0031 - gphsm Packaging Lifecycle and Complete Deprecation of hsmutil Symlink|ADR-0031: gphsm Packaging Lifecycle and Complete Deprecation of hsmutil Symlink]]"
  - "[[Specification/Kernel Network Stripping, Clean Subrepo Architecture, Environment Conditioning, and gphsm Workflow|Technical Specification]]"
  - "[[Plan/Kernel Network Stripping, Clean Subrepo Architecture, Environment Conditioning, and gphsm Workflow Implementation Plan|Implementation Plan]]"
  - "[[Architecture/07 - Air-Gapped RPi HSM OS Architecture|07 - Air-Gapped RPi HSM OS Architecture]]"
---

# Feature: Kernel Network Stripping, Clean Subrepo Architecture, Environment Conditioning, and gphsm Workflow

## 1. Feature Summary

This feature delivers deep-level OS hardening, workspace cleanliness encapsulation, anti-forensic environment conditioning, and tooling alignment across the **air-gapped Raspberry Pi HSM OS** ([`hsm-yocto-offgrid-rpi`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi)) and root workspace coordination:

1. **Complete Kernel Network Protocol Stripping**: Disables IPv4 (`CONFIG_INET`), IPv6 (`CONFIG_IPV6`), and Netfilter packet filtering (`CONFIG_NETFILTER`), while strictly preserving base socket infrastructure (`CONFIG_NET=y`) and UNIX domain sockets (`CONFIG_UNIX=y`) for `systemd`/`udev` IPC. Purges user-space `nftables` utilities.
2. **Production Session History Suppression**: In production builds (`HSM_DEV_MODE=0`), completely deactivates Bash command history (`HISTFILE=/dev/null`, `HISTSIZE=0`, `set +o history`) to protect operator privacy and prevent forensic key/credential leakage. In development mode (`HSM_DEV_MODE=1`), preserves standard Bash history for developer ergonomics.
3. **Environment-Conditioned Hostnames**: Automatically configures the system hostname as `piKMS` for production images and `piKMS-dev` for development images, eliminating operator ambiguity.
4. **Tidy Sub-Repository Layout**: Reconfigures Kas to clone all upstream layers (`openembedded-core`, `bitbake`, `meta-*`) into `build/repos/`, and places downloads and sstate cache in `build/downloads/` and `build/sstate-cache/`. Keeps the root of `hsm-yocto-offgrid-rpi` pristine.
5. **Canonical `gphsm` Packaging & Symlink Deprecation**: Packages `/usr/local/bin/gphsm` with dual dev/prod build workflows, permanently removes the legacy `/usr/local/bin/hsmutil` symlink, and preserves the unprivileged OS user `hsmutil` (`/home/hsmutil`).

---

## 2. User Stories & Acceptance Criteria

### User Story 1: True Air-Gap with Zero Network Protocols
> *As a security architect, I want the Linux kernel to compile out all IP networking stacks (IPv4, IPv6, Netfilter) so that the operating system has zero capability to process network packets, preventing any remote network exploitation even if a network dongle is attached.*

- **AC-1.1**: Kernel config sets `# CONFIG_INET is not set` and `# CONFIG_IPV6 is not set`.
- **AC-1.2**: Kernel config unsets all Netfilter options (`# CONFIG_NETFILTER is not set`).
- **AC-1.3**: Base socket infrastructure (`CONFIG_NET=y`) and UNIX domain sockets (`CONFIG_UNIX=y`) remain enabled for local IPC.
- **AC-1.4**: Calling `socket(AF_INET, ...)` returns `EAFNOSUPPORT`.
- **AC-1.5**: `nftables` and `/etc/nftables.conf` are completely omitted from the rootfs.

### User Story 2: Anti-Forensic Session Security in Production
> *As a cryptographic custodian performing a Root CA ceremony, I want all typed commands, parameters, and key slot IDs to disappear from memory and disk the moment execution ends, so that no residual history can be inspected by unauthorized parties or recovered from dumps.*

- **AC-2.1**: In `hsm-image-prod` (`HSM_DEV_MODE=0`), logging in via console sets `HISTFILE=/dev/null`, `HISTSIZE=0`, `HISTFILESIZE=0`, and `set +o history`.
- **AC-2.2**: No `.bash_history` file is created in `/home/hsmutil` or `/root`.
- **AC-2.3**: In `hsm-image-dev` (`HSM_DEV_MODE=1`), standard Bash history is retained.

### User Story 3: Environment Hostname Disambiguation
> *As an appliance operator, I want the console prompt to clearly show whether I am connected to a production node or a development node, so that I never execute production operations in a development environment or vice versa.*

- **AC-3.1**: Production image displays prompt `hsmutil@piKMS:~$` and `hostname` returns `piKMS`.
- **AC-3.2**: Development image displays prompt `hsmutil@piKMS-dev:~$` and `hostname` returns `piKMS-dev`.

### User Story 4: Pristine Sub-Repository Workspace
> *As an embedded Linux developer, I want all Kas clones, caches, and build outputs encapsulated inside a single `build/` directory, so that git status at the subrepo root remains clean and teardown is completely atomic via make distclean.*

- **AC-4.1**: Running Kas creates layers in `build/repos/`, downloads in `build/downloads/`, and sstate in `build/sstate-cache/`.
- **AC-4.2**: The root of `hsm-yocto-offgrid-rpi` contains zero cloned layer repositories or cache directories.
- **AC-4.3**: `make distclean` deletes the entire `build/` directory and purges the Docker volume `rpi_yocto_hsm_tmp`.

### User Story 5: Canonical gphsm Management Tool
> *As a system administrator, I want to manage the HSM using the standardized gphsm command, without legacy hsmutil symlinks cluttering /usr/local/bin.*

- **AC-5.1**: `/usr/local/bin/gphsm` is installed and executable.
- **AC-5.2**: `/usr/local/bin/hsmutil` does not exist (`test ! -e /usr/local/bin/hsmutil`).
- **AC-5.3**: System user `hsmutil` remains intact with UID 1000 and home directory `/home/hsmutil`.

---

## 3. Architecture & Traceability Matrix

| Capability Track | Architecture Decision Record (ADR) | Target Components |
| :--- | :--- | :--- |
| **Kernel Network Stripping** | [[Architecture/Decisions/ADR-0028 - Complete Kernel Network Protocol Stripping\|ADR-0028]] | `hardened.cfg`, `hsm-image-base.bb`, `recipes-connectivity/nftables` |
| **History & Hostname Conditioning** | [[Architecture/Decisions/ADR-0029 - Production Session History Suppression and Environment-Conditioned Hostnames\|ADR-0029]] | `init-wizard.bb`, `hsm-image-base.bb`, `hsm-image-dev.bb`, `hsm-image-prod.bb` |
| **Clean Subrepo Layout** | [[Architecture/Decisions/ADR-0030 - Sub-Repository Workspace Cleanliness via Encapsulated Build Directory\|ADR-0030]] | `kas-project.yml`, `scripts/run-kas.sh`, `Makefile`, `.gitignore` |
| **gphsm Tooling Alignment** | [[Architecture/Decisions/ADR-0031 - gphsm Packaging Lifecycle and Complete Deprecation of hsmutil Symlink\|ADR-0031]] | `recipes-support/gphsm/gphsm.bb`, `AGENTS.md` |
