---
title: "ADR-0008: Internalize Raspberry Pi RAUC Integration into meta-hsm-os"
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

# ADR-0008: Internalize Raspberry Pi RAUC Integration into meta-hsm-os

## Status
Accepted (plan approved; implementation pending). See [[Specification/Yocto 6.1 Blacksail Upgrade]] and [[Plan/Yocto 6.1 Blacksail Upgrade Implementation Plan]].

## Context
`meta-rauc-community` (`meta-rauc-raspberrypi`) lags behind Yocto 6.1 (`LAYERSERIES_COMPAT` limited to `scarthgap`, legacy `${WORKDIR}` assumptions).

## Decision
Drop `meta-rauc-community` and copy only the required pieces into `meta-hsm-os`: dual-slot WKS (`wic/sdimage-dual-raspberrypi.wks.in`), `rauc-integration.bbclass` (if referenced), `u-boot_%.bbappend` (`fw_env.config`, `uboot-rauc.cfg`), extended `rpi-u-boot-scr`, kernel `rauc.cfg` (SQUASHFS, BLK_DEV_LOOP), `/data` base-files and ubootenv udev rules. All use `${UNPACKDIR}`; existing `meta-hsm-os` recipes are merged, not duplicated.

## Consequences
- Positive: self-contained layer with no dependency on community layers (`grep -r rauc-community` must be empty).
- Negative: the project now owns maintenance of the A/B integration code.
