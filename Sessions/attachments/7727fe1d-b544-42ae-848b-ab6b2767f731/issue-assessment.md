# Issue Assessment: Raspberry Pi Zero 2 W First-Boot Device Tree Failure

## 1. Summary & Observed Behavior

During the first boot of the air-gapped embedded OS on the Raspberry Pi Zero 2 W (`raspberrypi0-2w-64`), U-Boot fails to boot the Linux kernel image and drops into the interactive `U-Boot>` shell.

### Observed Boot Failure Log
```text
Failed to load 'booti.dtn'
Moving Image from 0x80000 to 0x200000, end=0x1da000
ERROR: Did not find a cmdline Flattened Device Tree
Could not find a valid device tree
Boot failed (err=-14)
U-Boot>
```

### Symptoms
1. U-Boot fails with `Failed to load 'booti.dtn'` (corresponds to `booti.dtb`).
2. U-Boot successfully relocates the 64-bit kernel image (`Image`) from load address `0x80000` (`${kernel_addr_r}`) to `0x200000`.
3. U-Boot attempts to inspect memory for a Flattened Device Tree (FDT), fails header validation (`fdt_check_header`), reports `Did not find a cmdline Flattened Device Tree`, and halts with error code `-14` (`-EFAULT` / `-EINVAL`).

---

## 2. Root Cause Analysis

Thorough static and dynamic inspection across [rpi-z2w-yocto-hsm](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm), [meta-hsm-os](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-hsm-os), [meta-raspberrypi](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-raspberrypi), and [meta-rauc-raspberrypi](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-rauc-community/meta-rauc-raspberrypi) revealed three primary defects in the bootloader configuration.

### 2.1 Buggy Device Tree Loading in `meta-hsm-os` Boot Script
In [meta-hsm-os/recipes-bsp/rpi-u-boot-scr/files/boot.cmd.in](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-hsm-os/recipes-bsp/rpi-u-boot-scr/files/boot.cmd.in#L35-L38):

```sh
35: setenv bootargs "${bootargs} root=${rootdev} rootwait rauc.slot=${rauc_slot}"
36: load mmc 0:1 ${kernel_addr_r} @@KERNEL_IMAGETYPE@@
37: load mmc 0:1 ${fdt_addr_r} @@KERNEL_BOOTCMD@@.dtb || true
38: @@KERNEL_BOOTCMD@@ ${kernel_addr_r} - ${fdt_addr_r}
```

During build compilation in [rpi-u-boot-scr.bb](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-raspberrypi/recipes-bsp/rpi-u-boot-scr/rpi-u-boot-scr.bb#L14-L20), `sed` replaces:
- `@@KERNEL_IMAGETYPE@@` $\rightarrow$ `Image`
- `@@KERNEL_BOOTCMD@@` $\rightarrow$ `booti`

This generated the following commands in the deployed `boot.scr`:
```text
load mmc 0:1 ${kernel_addr_r} Image
load mmc 0:1 ${fdt_addr_r} booti.dtb || true
booti ${kernel_addr_r} - ${fdt_addr_r}
```

#### The Two Consequential Faults:
1. **Invalid Filename**: Line 37 attempts to load `booti.dtb`. No file named `booti.dtb` exists on partition 1 (`mmc 0:1`). U-Boot reports `Failed to load 'booti.dtb'`. Because of `|| true`, script execution proceeds.
2. **Incorrect FDT Address (`${fdt_addr_r}` vs `${fdt_addr}`)**: Line 38 calls `booti ${kernel_addr_r} - ${fdt_addr_r}`. Because line 37 failed, `${fdt_addr_r}` contains uninitialized or stale memory. When `booti` inspects `${fdt_addr_r}`, it fails FDT header checks (`fdt_check_header`), producing:
   ```text
   ERROR: Did not find a cmdline Flattened Device Tree
   Could not find a valid device tree
   Boot failed (err=-14)
   ```

### 2.2 Raspberry Pi Device Tree Architecture & Firmware Handoff
On Raspberry Pi platforms (including `raspberrypi0-2w-64`), U-Boot **does not and must not** load the Device Tree Blob from the filesystem:
1. **VideoCore IV Firmware Pre-Loads DTB**: When the board powers on, VideoCore IV GPU firmware (`start.elf`) reads `config.txt`, loads the base device tree (`bcm2710-rpi-zero-2.dtb`), applies hardware overlays (`disable-bt`, `disable-wifi`, `dwc2`), patches board serial numbers, MAC addresses, and populates the `/chosen/bootargs` property with `cmdline.txt`.
2. **Firmware Passes DTB in Memory**: The GPU firmware places the completed FDT in RAM and passes its memory pointer in CPU register `x0` to U-Boot.
3. **U-Boot Saves Pointer in `${fdt_addr}`**: In U-Boot's board driver (`board/raspberrypi/rpi/rpi.c`), patched by [0001-rpi-always-set-fdt_addr-with-firmware-provided-FDT-address.patch](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-raspberrypi/recipes-bsp/u-boot/files/0001-rpi-always-set-fdt_addr-with-firmware-provided-FDT-address.patch), U-Boot sets the environment variable `${fdt_addr}` to this firmware pointer.
4. **Correct Boot Execution**: Standard [meta-raspberrypi](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-raspberrypi/recipes-bsp/rpi-u-boot-scr/files/boot.cmd.in) and [meta-rauc-raspberrypi](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-rauc-community/meta-rauc-raspberrypi/recipes-bsp/rpi-u-boot-scr/files/boot.cmd.in) boot scripts execute:
   ```sh
   fdt addr ${fdt_addr} && fdt get value bootargs /chosen bootargs
   ...
   booti ${kernel_addr_r} - ${fdt_addr}
   ```
   Passing `${fdt_addr}` directly boots using the firmware-prepared device tree.

### 2.3 Missing VideoCore Bootargs Extraction
In [meta-hsm-os](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-hsm-os)'s `boot.cmd.in`, the command:
```sh
fdt addr ${fdt_addr} && fdt get value bootargs /chosen bootargs
```
is completely omitted. Consequently, `${bootargs}` in U-Boot is never populated from `cmdline.txt` (which contains `console=serial0,115200`, `dwc_otg.lpm_enable=0`, etc.). Even if the kernel had booted, no serial or display console would have been initialized.

### 2.4 Secondary Architectural Findings
1. **Shared Kernel vs Atomic A/B Update**:
   - `meta-hsm-os` loads the kernel from `mmc 0:1` (`load mmc 0:1 ${kernel_addr_r} @@KERNEL_IMAGETYPE@@`).
   - In RAUC A/B systems, the kernel should be loaded from the active slot (`load ${BOOT_DEV} ${kernel_addr_r} boot/@@KERNEL_IMAGETYPE@@`) where `${BOOT_DEV}` is `mmc 0:2` (Slot A) or `mmc 0:3` (Slot B). We verified that `/boot/Image` is packaged inside `hsm-image-dev` and `hsm-image-prod`. Loading from `mmc 0:1` creates a single shared kernel across both slots, defeating kernel rollback on update failure and risking kernel/module version mismatches.
2. **Missing Dual-Slot WKS Layout**:
   - In [kas-project.yml](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/kas-project.yml), `WKS_FILE` is not set, defaulting to single-rootfs `sdimage-raspberrypi.wks`. Inspection of the built `.wic` image showed only 2 partitions (`boot` and `rootfs_A`). Slot B (`mmcblk0p3`) was never provisioned on the partition table.

---

## 3. Proposed Solutions

### Solution 1 (Recommended): Align `meta-hsm-os` Boot Script with Upstream RAUC & Fix Device Tree Pointer

Fix [meta-hsm-os/recipes-bsp/rpi-u-boot-scr/files/boot.cmd.in](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-hsm-os/recipes-bsp/rpi-u-boot-scr/files/boot.cmd.in) to properly extract firmware bootargs, remove the erroneous `booti.dtb` load, load the kernel from the selected slot (`boot/Image`), and pass `${fdt_addr}`:

```sh
# Set active FDT and extract bootargs from firmware /chosen node
fdt addr ${fdt_addr} && fdt get value bootargs /chosen bootargs

test -n "${BOOT_ORDER}" || setenv BOOT_ORDER "A B"
test -n "${BOOT_A_LEFT}" || setenv BOOT_A_LEFT 3
test -n "${BOOT_B_LEFT}" || setenv BOOT_B_LEFT 3
test -n "${BOOT_DEV}" || setenv BOOT_DEV "mmc 0:1"

setenv bootpart
setenv raucslot

for BOOT_SLOT in "${BOOT_ORDER}"; do
  if test "x${bootpart}" != "x"; then
    # Skip remaining slots
  elif test "x${BOOT_SLOT}" = "xA"; then
    if itest ${BOOT_A_LEFT} -gt 0; then
      setexpr BOOT_A_LEFT ${BOOT_A_LEFT} - 1
      echo "RAUC: Booting Slot A (Remaining attempts: ${BOOT_A_LEFT})"
      setenv bootpart "/dev/mmcblk0p2"
      setenv raucslot "A"
      setenv BOOT_DEV "mmc 0:2"
    fi
  elif test "x${BOOT_SLOT}" = "xB"; then
    if itest ${BOOT_B_LEFT} -gt 0; then
      setexpr BOOT_B_LEFT ${BOOT_B_LEFT} - 1
      echo "RAUC: Booting Slot B (Remaining attempts: ${BOOT_B_LEFT})"
      setenv bootpart "/dev/mmcblk0p3"
      setenv raucslot "B"
      setenv BOOT_DEV "mmc 0:3"
    fi
  fi
done

if test -n "${bootpart}"; then
  setenv bootargs "${bootargs} root=${bootpart} rootwait rauc.slot=${raucslot}"
  saveenv
else
  echo "RAUC: No valid boot slots remaining. Resetting tries to 3."
  setenv BOOT_A_LEFT 3
  setenv BOOT_B_LEFT 3
  saveenv
  reset
fi

# Load kernel from selected slot partition
load ${BOOT_DEV} ${kernel_addr_r} boot/@@KERNEL_IMAGETYPE@@
if test ! -e mmc 0:1 uboot.env; then saveenv; fi;
@@KERNEL_BOOTCMD@@ ${kernel_addr_r} - ${fdt_addr}
```

Additionally, in [kas-project.yml](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/kas-project.yml), set:
```yaml
local_conf_header:
  base_settings: |
    ...
    WKS_FILE = "sdimage-dual-raspberrypi.wks.in"
```
or define an air-gapped dual-slot WKS template containing partitions for `boot`, `rootfs_A`, and `rootfs_B`.

- **Pros**:
  - Resolves FDT lookup failure and boots successfully.
  - Restores serial console output via firmware `bootargs` extraction.
  - Ensures atomic kernel + rootfs updates and rollback support per RAUC slot.
  - Conforms to standard `meta-rauc-raspberrypi` patterns.
- **Cons**:
  - Requires regenerating the `.wic` image with the dual-slot layout.

---

### Solution 2: Minimal FDT Fix in `meta-hsm-os` (Retaining Shared Partition 1 Kernel)

If the team prefers to keep the kernel image strictly in the FAT partition (`mmc 0:1`) for now:
1. In [boot.cmd.in](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-hsm-os/recipes-bsp/rpi-u-boot-scr/files/boot.cmd.in):
   - Add `fdt addr ${fdt_addr} && fdt get value bootargs /chosen bootargs` at line 1.
   - Delete line 37 (`load mmc 0:1 ${fdt_addr_r} @@KERNEL_BOOTCMD@@.dtb || true`).
   - Change line 38 to `@@KERNEL_BOOTCMD@@ ${kernel_addr_r} - ${fdt_addr}`.
- **Pros**:
  - Minimal diff (2 lines added/modified, 1 line deleted).
  - Quickest path to fixing the immediate boot failure.
- **Cons**:
  - Does not support atomic kernel rollbacks in RAUC A/B updates.
  - Does not address the missing Slot B partition in single-rootfs WIC images.

---

### Solution 3: Remove Redundant `rpi-u-boot-scr.bbappend` from `meta-hsm-os`

Delete [meta-hsm-os/recipes-bsp/rpi-u-boot-scr](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-hsm-os/recipes-bsp/rpi-u-boot-scr) completely and rely directly on [meta-rauc-raspberrypi/recipes-bsp/rpi-u-boot-scr/files/boot.cmd.in](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/meta-rauc-community/meta-rauc-raspberrypi/recipes-bsp/rpi-u-boot-scr/files/boot.cmd.in).
- **Pros**:
  - Eliminates duplicated/divergent recipe code.
  - Relies entirely on upstream-maintained RAUC Raspberry Pi boot script.
- **Cons**:
  - Requires accepting `meta-rauc-raspberrypi` default behavior and partition expectations.

---

## 4. Verification & Validation Plan

1. **Script Compilation & Deployment Check**:
   - Rebuild `rpi-u-boot-scr` using Kas (`make yocto-shell` or `kas build kas-dev.yml -c compile rpi-u-boot-scr`).
   - Inspect the compiled `boot.scr` with `strings` to confirm that `booti.dtb` is gone and `booti ${kernel_addr_r} - ${fdt_addr}` is emitted.
2. **WIC Dual-Slot Partition Verification**:
   - Build `hsm-image-dev` and verify that the output `.wic.bz2` contains Partition 1 (`boot`), Partition 2 (`rootfs_A`), and Partition 3 (`rootfs_B`).
3. **Hardware Boot Test**:
   - Flash the updated image to an SD card.
   - Boot Raspberry Pi Zero 2 W with USB UART console attached (`115200 8N1`).
   - Verify that U-Boot locates the FDT at `${fdt_addr}`, loads the kernel, applies `bootargs`, and hands off execution to Linux without `-14` errors.
4. **RAUC State Verification**:
   - After Linux boots to the login prompt, log in and verify `rauc status` recognizes active Slot A and responds normally.
