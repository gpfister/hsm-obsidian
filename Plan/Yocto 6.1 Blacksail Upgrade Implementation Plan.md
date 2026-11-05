---
title: "Implementation Plan - Yocto 6.1 (Blacksail) Upgrade"
tags:
  - plan
  - implementation
  - yocto
  - blacksail
  - rpi-z2w-yocto-hsm
  - rauc
  - kas
created: 2026-11-05
updated: 2026-11-05
---

# Implementation Plan: Upgrade `rpi-z2w-yocto-hsm` to Yocto 6.1 (Blacksail)

Spec: [[Specification/Yocto 6.1 Blacksail Upgrade]] · Architecture: [[Architecture/07 - Air-Gapped RPi HSM OS Architecture]]

## Repositories & Branches

| Repo | Branch (from `main`) | Scope |
| :--- | :--- | :--- |
| `rpi-z2w-yocto-hsm` | `task/upgrade-yocto-6.1-blacksail` | All build/layer/recipe/driver/doc changes |
| root `ncs-nrf52840dongle-kms` | `task/upgrade-yocto-6.1-blacksail` | `AGENTS.md`, `README.md` only (merged after sub-repo) |
| `nrf52840-hsm-obsidian` | n/a | Delegated to `@coding-document` after approval / pre-merge / post-merge |

> [!NOTE]
> Repo inspection findings that refine the spec:
> - `meta-rauc-community/meta-rauc-raspberrypi` currently provides: `classes/rauc-integration.bbclass`, `wic/sdimage-dual-raspberrypi.wks.in`, `recipes-bsp/{rpi-u-boot-scr,u-boot}`, `recipes-core/{base-files,bundles,images,rauc,udev}`, `recipes-kernel/linux`. Step 3 copies the required ones (read them before copying).
> - `meta-hsm-os` already has `recipes-bsp/rpi-u-boot-scr` (+`boot.cmd.in`), `recipes-kernel/linux/*.bbappend`, `recipes-support/rauc`, and `recipes-rauc/bundles`. There is **no** `u-boot_%.bbappend` or `rpi-cmdline.bbappend` today. Merge, don't duplicate.
> - `${WORKDIR}` usage is in: nrf52840-hsm-kmod, usb-mount, rauc_%.bbappend, hsmutil, systemd-journald-airgap, volatile-storage, init-wizard, nftables-rules. Paths in the spec differ in places (e.g. `recipes-support/usb-mount`, `recipes-core/systemd-journald-airgap`). Use the real paths.
> - `IMAGE_FEATURES += "debug-tweaks"` is in `kas-dev.yml` and `hsm-image-dev.bb`.
> - `noop_llseek` is at `nrf52840_hsm.c:166`.
> - `meta-lts-mixins/` is an untracked/ignored kas clone.

## Steps — `rpi-z2w-yocto-hsm` (`coding-develop`)

1. **Branch & clean**: `git checkout -b task/upgrade-yocto-6.1-blacksail`.
2. **`kas-project.yml`** (FR-1, AC-1):
   - Replace `poky` with `openembedded-core` (`blacksail`, layer `meta`), `bitbake` (`2.20`, `layers: {.: disabled}`) and `meta-yocto` (`blacksail`, layers `meta-poky`, `meta-yocto-bsp`).
   - Point `meta-openembedded` to `blacksail`.
   - Point `meta-raspberrypi` and `meta-rauc` to `master`. Use `blacksail` if that branch exists. Check with `git ls-remote --heads`.
   - Remove `meta-lts-mixins` and `meta-rauc-community`.
   - Add `LAYERSERIES_COMPAT_raspberrypi:append`/`_rauc:append = " blacksail"` in `local_conf_header` only if upstream lacks it.
   - Pin to commits only if the build is unstable.
   - Keep `distro: poky`, WKS_FILE, IMAGE_CMD:img. Re-verify the `IMGDEPLOYDIR`/`IMAGE_LINK_NAME` naming.
3. **Internalize RPi RAUC integration** (FR-2):
   - Create `meta-hsm-os/wic/sdimage-dual-raspberrypi.wks.in`.
   - Create `meta-hsm-os/classes/rauc-integration.bbclass` if it is referenced.
   - Add `recipes-bsp/u-boot/u-boot_%.bbappend` with `fw_env.config`/`uboot-rauc.cfg`.
   - Extend `recipes-bsp/rpi-u-boot-scr`.
   - Extend `recipes-kernel/linux/linux-raspberrypi_%.bbappend` with `rauc.cfg` (SQUASHFS, BLK_DEV_LOOP).
   - Add `recipes-core/base-files` (`/data` mount) and `recipes-core/udev` rules (ubootenv).
   - Merge `recipes-core/rauc`/`bundles`/`images` pieces only as needed.
   - All new/copied recipes use `${UNPACKDIR}`.
   - Verify there is zero dependency: `grep -r rauc-community` must be empty.
4. **UNPACKDIR migration** (FR-3, AC-3):
   - Replace `${WORKDIR}/<file>` with `${UNPACKDIR}/<file>` in the 8 recipes listed above.
   - Set `S = "${UNPACKDIR}"` where `S = "${WORKDIR}"` is used (kmod, hsmutil).
   - Add `S = "${UNPACKDIR}"` or `S = "${WORKDIR}/sources"` for init-wizard (compiles `hsm-set-date.c`).
   - Check `inherit go`/prebuilt-binary handling in `hsmutil.bb` for `hsmutil-arm64`.
5. **Remove `debug-tweaks`** (FR-3): in `kas-dev.yml` and `hsm-image-dev.bb`, use `allow-root-login post-install-logging tools-debug`, and add `empty-root-password` only if the current dev flow depends on an empty password. Check `EXTRA_USERS_PARAMS`/`hsmutil-user.bb`. Keep `kas-prod.yml` with `EXTRA_IMAGE_FEATURES = ""`.
6. **`layer.conf`**: `LAYERSERIES_COMPAT_meta-hsm-os = "blacksail"`. Review for removed variables.
7. **Kernel driver** (FR-4, AC-4): in `nrf52840_hsm.c`, add `#include <linux/version.h>` and guard `.llseek` with `LINUX_VERSION_CODE < KERNEL_VERSION(6,12,0)` (`noop_llseek` vs `no_llseek`). Check `class_create` (single-arg since 6.4), `MODULE_*` and `file_operations` usage. Check `files/Makefile` for kbuild changes.
8. **Housekeeping** (AC-2): `rm -rf poky meta-lts-mixins meta-rauc-community` (ignored clones). Update `.gitignore` to add `openembedded-core/`, `bitbake/`, `meta-yocto/` and remove the legacy entries. Run `make clean` to drop the stale `build/tmp` (and the `rpi_yocto_hsm_tmp` Docker volume if needed).
9. **Docs** (FR-5, AC-7): update `Makefile` banner (Yocto 6.1) and `README.md` (title, layer list, layout comment `kas-project.yml`, kernel version, and a migration note).
10. **Static verification**: `kas dump kas-dev.yml` and `kas dump kas-prod.yml` in the container, `bitbake -p` parse check, and greps for `WORKDIR`, `debug-tweaks`, `scarthgap`, `poky/`, `lts-mixins`, `rauc-community`.
11. **Commit** in logical commits: kas/layers, rauc internalization, UNPACKDIR, kernel driver, docs.

## Steps — root repo

12. Branch `task/upgrade-yocto-6.1-blacksail`. In `AGENTS.md` (lines 25, 71) and `README.md` (line 13), change "Yocto 5.0 (Scarthgap)" to "Yocto 6.1 (Blacksail)" and "Linux 6.6 LTS" to "Linux 6.12+" (kernel version per the actual resolved `linux-raspberrypi`). Commit.

## Local Deployment (`coding-deploy`) — `rpi-z2w-yocto-hsm`

> [!IMPORTANT]
> Local deployment instructions exist: the repo `Makefile` and `scripts/run-kas.sh` run Kas in a Docker/Podman container (`ghcr.io/siemens/kas/kas:latest`). The root `Makefile` forwards `make yocto-build-dev` and `make yocto-build-prod`. `coding-deploy` triggers immediately after this repo's `coding-develop` step concludes.

- **Method**: containerized Kas build (no runtime service).
- **Commands** (from the workspace root, long-running, logs to file, expect hours on a cold cache):
  1. `make yocto-clean`
  2. `make yocto-build-dev` (AC-5)
  3. `make yocto-build-prod` (AC-6)
  4. Optional: `make yocto-bundle`
- **Health verification**:
  - Both commands exit `0`.
  - No BitBake `ERROR` lines, and no QA warnings about `WORKDIR`/`debug-tweaks`/`LAYERSERIES_COMPAT`.
  - `nrf52840-hsm-kmod` compiles against the 6.12+ kernel.
  - Artifacts exist in `rpi-z2w-yocto-hsm/build/deploy/images/<machine>/`: `hsm-image-dev-*` and `hsm-image-prod-*` `.img`, `.wic.bz2`, `.wic.bmap`.
  - Spot-check air-gap invariants in the prod image: `read-only-rootfs`, nftables rules and `HSM_DEV_MODE=0`.
- **Failure handling**: loop back to `coding-develop` with the BitBake log excerpt (use `@coding-monitor` for log tailing if desired).
- Root repo (docs only): **Local Deployment: Not Required** (the root `Makefile` only forwards).

## Risks

- `meta-raspberrypi` or `meta-rauc` may not yet support blacksail (mitigation: the `LAYERSERIES_COMPAT` override, plus fixing recipe breakages).
- Linux 6.12 hardening config fragment options may be renamed or removed. Review `hardened.cfg` against build warnings.
- The prebuilt `hsmutil-arm64` binary must exist in `files/`.
- macOS Docker volume performance and disk size (≥100 GB).

## Documentation (`@coding-document`)

Invoke after this plan is approved, before merge, and after merge. Update the Obsidian notes for Yocto architecture and git status.


## Related ADRs
- [[Architecture/Decisions/ADR-0007 - Decompose Poky into openembedded-core meta-yocto and bitbake|ADR-0007]]
- [[Architecture/Decisions/ADR-0008 - Internalize RPi RAUC Integration into meta-hsm-os|ADR-0008]]
- [[Architecture/Decisions/ADR-0009 - Kernel Driver llseek Version Guard and UNPACKDIR Migration|ADR-0009]]
