---
title: "ADR-0007: Decompose Poky into openembedded-core, meta-yocto and bitbake"
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

# ADR-0007: Decompose Poky into openembedded-core, meta-yocto and bitbake

## Status
Accepted (plan approved; implementation pending). See [[Specification/Yocto 6.1 Blacksail Upgrade]] and [[Plan/Yocto 6.1 Blacksail Upgrade Implementation Plan]].

## Context
Yocto 6.1 (`blacksail`) retires the monolithic `poky` repository. `meta-lts-mixins` (`scarthgap/u-boot`) existed only to backport U-Boot and is obsolete.

## Decision
`kas-project.yml` uses `openembedded-core` (`blacksail`, layer `meta`), `bitbake` (`2.20`, `.` layer disabled), `meta-yocto` (`blacksail`, `meta-poky`, `meta-yocto-bsp`) and `meta-openembedded` (`blacksail`). `meta-raspberrypi` and `meta-rauc` track `master` (or `blacksail` if the branch exists), with `LAYERSERIES_COMPAT_*:append = " blacksail"` overrides only if upstream lacks it. `meta-lts-mixins` is removed; `distro: poky` is kept.

## Consequences
- Positive: modern kernel (6.12+), U-Boot, systemd and toolchain; fewer external layers.
- Negative: tracking `master` of BSP layers may be unstable (pin commits if needed); stale `build/tmp` and legacy clones (`poky/`, `meta-lts-mixins/`) must be cleaned; `.gitignore` updated.
