---
title: "ADR-0009: Kernel Driver llseek Version Guard and UNPACKDIR Migration"
status: Accepted
date: 2026-11-05
tags:
  - adr
  - architecture
  - yocto
  - blacksail
deciders:
  - Embedded Systems Architect
---

# ADR-0009: Kernel Driver llseek Version Guard and UNPACKDIR Migration

## Status
Accepted (plan approved; implementation pending). See [[Specification/Yocto 6.1 Blacksail Upgrade]] and [[Plan/Yocto 6.1 Blacksail Upgrade Implementation Plan]].

## Context
Yocto 6.1 requires `${UNPACKDIR}` instead of `${WORKDIR}` for local `file://` sources and removes the `debug-tweaks` image feature. Linux 6.12+ removes `noop_llseek` (used at `nrf52840_hsm.c:166`).

## Decision
Migrate the 8 affected recipes (kmod, usb-mount, rauc_%.bbappend, hsmutil, systemd-journald-airgap, volatile-storage, init-wizard, nftables-rules) to `${UNPACKDIR}` / `S = "${UNPACKDIR}"`. Replace `debug-tweaks` with `allow-root-login post-install-logging tools-debug` (`empty-root-password` only if the dev flow needs it); `kas-prod.yml` keeps an empty `EXTRA_IMAGE_FEATURES`. In the driver, guard `.llseek` with `LINUX_VERSION_CODE < KERNEL_VERSION(6,12,0)` (`noop_llseek` vs `no_llseek`) and include `<linux/version.h>`.

## Consequences
- Positive: clean build on 6.12+ with no QA warnings; air-gap invariants (read-only rootfs, nftables drop, `HSM_DEV_MODE=0`) are preserved.
- Negative: version guard adds minor driver conditional code; `hardened.cfg` options may need review against 6.12 renames.
