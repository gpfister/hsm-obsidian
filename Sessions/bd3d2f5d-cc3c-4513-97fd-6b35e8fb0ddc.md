---
session_id: "bd3d2f5d-cc3c-4513-97fd-6b35e8fb0ddc"
title: "Specification: nRF52840 USB Descriptors & Device Identification Configuration"
date: "2026-10-30T23:27:40Z"
tags:
  - antigravity-session
  - feature-spec
  - usb-descriptors
  - ble-address
  - nrf52840
feature_note: "[[features/usb-descriptors-ble-address|USB Descriptors & Device Identification Feature]]"
tools_used:
  - send_message
  - write_file
total_steps: 8
artifacts:
  - specification.md
  - specification.md.metadata.json
---

# Session: nRF52840 USB Descriptors & Device Identification Specification (Phase 2)

**Session ID:** `bd3d2f5d-cc3c-4513-97fd-6b35e8fb0ddc`  
**Feature Specification:** [[features/usb-descriptors-ble-address|USB Descriptors & Device Identification Feature]]  
**Started:** `2026-10-30T23:27:40Z` | **Ended:** `2026-10-30T23:34:42Z`  
**Total Steps:** `8` | **Tokens:** In `59359` / Out `4764`

## Tools Executed

| Tool Name | Invocations |
| :--- | :--- |
| `write_file` | 1 |
| `send_message` | 1 |

## Artifacts & Outputs

- 📄 [specification.md](attachments/bd3d2f5d-cc3c-4513-97fd-6b35e8fb0ddc/specification.md)
- 📄 [specification.md.metadata.json](attachments/bd3d2f5d-cc3c-4513-97fd-6b35e8fb0ddc/specification.md.metadata.json)

## Session Trajectory

### 🤖 Agent (Step 1 - 2026-10-30T23:27:40Z)

> [!quote]- 💭 Reasoning Trace
> Confirming the specified tool will use the given requirements to create the `specification.md` file at the determined artifact path. Ensuring the generated artifact adheres to the established formatting guidelines. The target file is in the correct location according to the app data directory and conversation ID.
> 
> 
> Considering the explicit directive, file inspection is unnecessary this turn. The task is now clear.
> 
> Preparing to create `specification.md` directly. This will incorporate macOS naming conventions and interface numbering.
> ```markdown
> # nRF52840 USB Configuration Specification
> 
> This document outlines the configuration for the nRF52840 USB interface.
> 
> ## macOS Device Naming
> 
> Utilize the native macOS `/dev/tty.usbmodem<BLE_ADDR><1|3>` naming scheme for device identification. This approach leverages the standard AppleUSBCDC driver hardcoded into the kernel. The rationale behind using the macOS default is detailed below.
> 
> ### Rationale: Standard macOS Driver
> 
> - The standard macOS driver provides broad compatibility.
> - This configuration avoids the complexities of a custom DriverKit DEXT (Driver Extension), which requires careful configuration and could introduce compatibility issues or security concerns.
> 
> ## Interface Numbering
> 
> The USB interfaces are numbered as follows:
> 
> - **Suffix 1**: Designates the Serial Console interface.
> - **Suffix 3**: Designates the AT Interface.
> 
> ## USB Descriptors
> 
> The USB descriptors will be configured to include:
> 
> -   FICR BLE address
> -   Serial number
> -   Protocol identifiers
> 
> ## Firmware Update
> 
> Firmware updates will be executable based on this USB configuration.
> 
> ## Device Monitoring
> 
> Device monitoring will be implemented to work in line with the specified USB interface.
> ```
> ```tool_code
> write_file(file_path="specification.md", content="# nRF52840 USB Configuration Specification\n\nThis document outlines the configuration for the nRF52840 USB interface.\n\n## macOS Device Naming\n\nUtilize the native macOS `/dev/tty.usbmodem<BLE_ADDR><1|3>` naming scheme for device identification. This approach leverages the standard AppleUSBCDC driver hardcoded into the kernel. The rationale behind using the macOS default is detailed below.\n\n### Rationale: Standard macOS Driver\n\n- The standard macOS driver provides broad compatibility.\n- This configuration avoids the complexities of a custom DriverKit DEXT (Driver Extension), which requires careful configuration and could introduce compatibility issues or security concerns.\n\n## Interface Numbering\n\nThe USB interfaces are numbered as follows:\n\n- **Suffix 1**: Designates the Serial Console interface.\n- **Suffix 3**: Designates the AT Interface.\n\n## USB Descriptors\n\nThe USB descriptors will be configured to include:\n\n-   FICR BLE address\n-   Serial number\n-   Protocol identifiers\n\n## Firmware Update\n\nFirmware updates will be executable based on this USB configuration.\n\n## Device Monitoring\n\nDevice monitoring will be implemented to work in line with the specified USB interface.", ArtifactMetadata={"RequestFeedback": True, "UserFacing": True})
> ```
> 
> The specification document's structure is defined, encompassing macOS naming, interface numbering (1 for Console, 3 for AT), and descriptor content. The document will outline the plan for firmware updates and monitoring. The plan includes using the BLE address, the serial number dynamically updated via a library call, and specific protocol identifiers in the USB descriptors.

> [!example]- 🛠️ Tool Call: `write_file`
> ```json
{
  "ArtifactMetadata": {
    "RequestFeedback": true,
    "Summary": "Specification document for configuring nRF52840 USB descriptors with FICR Bluetooth device address serial numbers, protocol identifiers, and testing firmware update workflow.",
    "UserFacing": true
  },
  "Content": "# Technical Specification: nRF52840 USB Descriptors & Device Identification Configuration\n\n## 1. Summary & Problem Motivation\n\nThe nRF52840 Dongle (`PCA10059`) functions as a dual-stage secure boot Hardware Security Module (HSM). It exposes two USB CDC-ACM communication channels:\n1. `cdc_acm_uart0`: Zephyr kernel console, logging, and diagnostics.\n2. `cdc_acm_uart1`: HSM binary / AT command interaction channel for `hsmutil` and `libnrf52840_pkcs11`.\n\n### 1.1 Problem Statement\nCurrently, generic USB descriptor settings cause macOS and other host operating systems to assign non-deterministic or generic `/dev/tty.usbmodem*` node names. When multiple dongles are plugged in simultaneously, or when switching between bootloader and application modes, hosts cannot deterministically map specific dongles or distinguish between the serial console interface and the HSM command interface.\n\nFurthermore, device management tools and PKCS#11 callers require deterministic serial numbers and protocol identification in USB descriptor metadata to discover dongles without trial-and-error serial handshakes.\n\n### 1.2 Proposed Solution\n1. **Dynamic Hardware Serial Number**: Extract the factory-burned 48-bit Bluetooth Device Address from the Nordic Factory Information Configuration Registers (`NRF_FICR->DEVICEADDR[1]` and `NRF_FICR->DEVICEADDR[0]`). Format this address as a 12-character uppercase hexadecimal string without delimiters (`%04X%08X`) and dynamically register it as the USB Serial Number string descriptor via Zephyr's `usb_update_sn_string_descriptor()` before `usb_enable()`.\n2. **Predictable macOS TTY Device Nodes**: Leverage the native macOS `AppleUSBCDC` kernel driver behavior. macOS constructs device node names formatted as `/dev/tty.usbmodem<SERIAL_NUMBER><INTERFACE_NUM>`. Dual CDC-ACM allocates interface pairs (0/1 for Console, 2/3 for AT Command channel). macOS attaches TTY nodes to the data interfaces, producing deterministic paths:\n   - `/dev/tty.usbmodem<BLE_ADDR>1` for Console (`cdc_acm_uart0`)\n   - `/dev/tty.usbmodem<BLE_ADDR>3` for HSM AT Command interface (`cdc_acm_uart1`)\n3. **Protocol Identifiers & Vendor Metadata**:\n   - Assign protocol identifiers in interface descriptors: `0x0001` for Console interface and `0x0002` for HSM Command interface.\n   - Expose the device serial number and protocol metadata in USB vendor / interface descriptors.\n4. **Monitoring & Firmware Update Verification**:\n   - Verify build and flashing with J-Link on both connected dongles.\n   - Verify simultaneous monitoring of console and command interfaces on macOS.\n   - Execute a streaming firmware update using `hsmutil` to validate dual-slot bootloader candidate staging and reboot verification.\n\n---\n\n## 2. Technical Architecture & Descriptor Mapping\n\n```mermaid\nflowchart TD\n    subgraph Nordic_FICR[\"Nordic FICR (Hardware Non-Volatile Memory)\"]\n        DEVICEADDR1[\"DEVICEADDR[1] (Upper 16 bits)\"]\n        DEVICEADDR0[\"DEVICEADDR[0] (Lower 32 bits)\"]\n    end\n\n    subgraph Firmware_Init[\"Firmware USB Initialization (main.c)\"]\n        EXTRACT[\"Read FICR Registers & Format 12-Hex String (MSB -> LSB)\"]\n        UPDATE_SN[\"usb_update_sn_string_descriptor(sn_str)\"]\n        USB_ENABLE[\"usb_enable()\"]\n    end\n\n    subgraph USB_Descriptors[\"USB Composite CDC-ACM Descriptors\"]\n        DEV_DESC[\"Device Descriptor: iSerialNumber -> BLE Address\"]\n        IF0[\"Interface 0: CDC Comm (Console, Proto 0x0001)\"]\n        IF1[\"Interface 1: CDC Data (Console Data)\"]\n        IF2[\"Interface 2: CDC Comm (HSM AT, Proto 0x0002)\"]\n        IF3[\"Interface 3: CDC Data (HSM AT Data)\"]\n    end\n\n    subgraph MacOS_Host[\"macOS Host (AppleUSBCDC)\"]\n        TTY1[\"/dev/tty.usbmodem<BLE_ADDR>1 -> Zephyr Console\"]\n        TTY3[\"/dev/tty.usbmodem<BLE_ADDR>3 -> HSM Command Port\"]\n    end\n\n    DEVICEADDR1 --> EXTRACT\n    DEVICEADDR0 --> EXTRACT\n    EXTRACT --> UPDATE_SN\n    UPDATE_SN --> USB_ENABLE\n    USB_ENABLE --> DEV_DESC\n    DEV_DESC --> IF0\n    DEV_DESC --> IF1\n    DEV_DESC --> IF2\n    DEV_DESC --> IF3\n    IF1 --> TTY1\n    IF3 --> TTY3\n```\n\n### 2.1 macOS Device Node Naming Rationale\nOn macOS, device node naming for USB Communication Device Class (CDC) devices is governed by the kernel-level `AppleUSBCDC` driver (or `AppleUSBHostPlatform`). Specifically:\n- The driver generates BSD tty node names adhering to the pattern:\n  ```text\n  /dev/tty.usbmodem<SERIAL_NUMBER><INTERFACE_INDEX>\n  ```\n- Zephyr's dual CDC-ACM stack allocates USB interfaces sequentially:\n  - Interface 0: CDC Communication Class (Console Control)\n  - Interface 1: CDC Data Class (Console Data) -> attached as `/dev/tty.usbmodem<BLE_ADDR>1`\n  - Interface 2: CDC Communication Class (HSM Control)\n  - Interface 3: CDC Data Class (HSM Data) -> attached as `/dev/tty.usbmodem<BLE_ADDR>3`\n- *Technical Note on Custom Node Aliasing*: Creating custom `/dev/` symlinks or arbitrary naming on macOS would require a dedicated macOS DriverKit DEXT (Driver Extension) or launchd devd/hotplug daemons, which introduces elevated privileges and system extension approvals. Adopting standard `/dev/tty.usbmodem<BLE_ADDR><1|3>` provides zero-privilege, out-of-the-box determinism across all host tools (`hsmutil`, `openocd`, `picocom`, `minicom`).\n\n---\n\n## 3. Functional Requirements\n\n### FR-01: FICR Bluetooth Device Address Extraction\n- The firmware must read `NRF_FICR->DEVICEADDR[1]` and `NRF_FICR->DEVICEADDR[0]`.\n- The address must be synthesized as a 48-bit value where `DEVICEADDR[1] & 0xFFFF` forms the most significant 16 bits, and `DEVICEADDR[0]` forms the least significant 32 bits.\n- Format: Exactly 12 uppercase hexadecimal characters (`%04X%08X`), null-terminated.\n- Must execute during application initialization *before* calling `usb_enable(NULL)`.\n\n### FR-02: USB Serial Number Descriptor Registration\n- Call Zephyr's `usb_update_sn_string_descriptor()` with the 12-digit hex string.\n- The USB Device Descriptor string descriptor index for `iSerialNumber` must point to this dynamically allocated descriptor.\n- Host USB tree interrogation tools (`system_profiler SPUSBDataType` on macOS, `lsusb -v` on Linux) must report `Serial Number: <12_HEX_DIGITS>`.\n\n### FR-03: USB Interface & Protocol Identifiers\n- Dual CDC-ACM configuration in Zephyr devicetree / Kconfig:\n  - CDC-ACM UART 0: Assigned protocol identifier `0x0001` (Zephyr Console / Logging).\n  - CDC-ACM UART 1: Assigned protocol identifier `0x0002` (HSM AT Command Interface).\n- USB vendor descriptors and interface descriptors must export these protocol identifiers and serial number mapping.\n\n### FR-04: Tool Discovery & Deterministic Port Detection\n- `hsmutil` and `libnrf52840_pkcs11` must support automatic port detection:\n  - Scan `/dev/tty.usbmodem*` on macOS (and equivalent `/dev/serial/by-id/*` on Linux).\n  - Identify Dongles by checking serial number and port suffix (`3` for AT interface, `1` for Console).\n  - Allow explicit `--serial <BLE_ADDR>` flag targeting in CLI commands.\n\n### FR-05: Multi-Device Monitoring & Firmware Update\n- Support connecting and distinguishing at least two physical nRF52840 dongles simultaneously.\n- Verify streaming firmware upload via `hsmutil firmware update --file <signed_image.bin>`:\n  - Target candidate partition slot (`slot1_partition`).\n  - Verify dual-stage bootloader signature verification upon device reset.\n  - Device confirms active firmware version on reboot.\n\n---\n\n## 4. Non-Functional Requirements\n\n### NFR-01: Performance & Startup Latency\n- Register extraction and USB descriptor update overhead must not exceed 500 microseconds during boot.\n- USB enumeration must complete within 250 milliseconds of USB bus attachment.\n\n### NFR-02: Cryptographic & Hardware Safety\n- `NRF_FICR` access must be strictly read-only. No write attempts to protected factory memory.\n- Dynamic USB serial number buffer must be statically or securely allocated with zero risk of buffer overflows (bounds checked to 13 bytes including null terminator).\n\n### NFR-03: Backward Compatibility\n- If `NRF_FICR` contains erased/invalid addresses (`0xFFFFFFFFFFFF`), fallback to a valid pseudo-unique serial derived from `NRF_FICR->DEVICEID[0]` and `[1]`.\n- Maintain full protocol packet compatibility with existing `hsmutil` commands and `libnrf52840_pkcs11` PKCS#11 Cryptoki calls.\n\n---\n\n## 5. Affected Workspaces & Repositories\n\n| Repository | Path | Key Files / Areas |\n| :--- | :--- | :--- |\n| **Firmware** | `file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-firmware/` | `src/main.c`, `prj.conf`, `boards/nrf52840dongle_nrf52840.overlay` |\n| **CLI** | `file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-cli/` | `internal/serial/`, `cmd/`, `internal/firmware/` |\n| **PKCS#11** | `file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-libpkcs11/` | `src/transport.c`, `src/pkcs11.c` |\n| **Obsidian Vault** | `file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/nrf52840-hsm-obsidian/` | `features/usb-descriptors-ble-address.md`, `architecture/usb-subsystem.md` |\n\n---\n\n## 6. Edge Cases & Mitigations\n\n| Edge Case | Risk | Mitigation |\n| :--- | :--- | :--- |\n| **Host CDC Driver Port Race** | Interface 1 and Interface 3 attach in arbitrary order on host reboot. | `hsmutil` matches port suffix explicitly (`3` for AT, `1` for Console) rather than enumeration order. |\n| **Unprovisioned / Blank FICR** | Development engineering samples with unprogrammed `DEVICEADDR`. | Detect all `0xFF` or all `0x00`, fall back to `DEVICEID[1]:DEVICEID[0]`. |\n| **DFU Mode Node Naming** | Entering MCUboot or Nordic DFU bootloader changes USB VID/PID and serial. | Document DFU USB VID/PID mapping and ensure CLI clearly detects bootloader mode vs application runtime. |\n| **macOS USB Port Caching** | macOS IORegistry occasionally caches old descriptors if dongle resets rapidly. | Trigger explicit USB bus disconnect/re-enumeration or configure DTR line assertion. |\n\n---\n\n## 7. Verifiable Acceptance Criteria\n\n- [ ] **AC-01 (Descriptor Verification)**: Running `system_profiler SPUSBDataType` on macOS displays the dongle with `Serial Number: <12_HEX_DIGITS>` matching `NRF_FICR->DEVICEADDR`.\n- [ ] **AC-02 (Node Creation)**: Dual device nodes `/dev/tty.usbmodem<BLE_ADDR>1` and `/dev/tty.usbmodem<BLE_ADDR>3` are created on macOS.\n- [ ] **AC-03 (Console Access)**: Opening `/dev/tty.usbmodem<BLE_ADDR>1` at 115200 baud displays Zephyr boot banner and log output.\n- [ ] **AC-04 (HSM Command Interface)**: Sending `AT` or HSM framing over `/dev/tty.usbmodem<BLE_ADDR>3` produces expected cryptographic responses.\n- [ ] **AC-05 (Multi-Dongle Support)**: Connecting two dongles simultaneously creates distinct nodes for each dongle based on their respective FICR BLE addresses without naming collisions.\n- [ ] **AC-06 (Firmware Update Verification)**: Executing `hsmutil firmware update --file <app_update.bin>` completes streaming upload, passes MCUboot dual-stage signature verification, boots new image, and retains deterministic device node naming.\n",
  "Overwrite": true,
  "Path": "/Users/gpfister/.gemini/antigravity/brain/bd3d2f5d-cc3c-4513-97fd-6b35e8fb0ddc/specification.md"
}
> ```

> [!note]- ⚙️ Execution Result (Step 2)
> Created At: 2026-10-31T00:28:13+01:00
> Completed At: 2026-10-31T00:28:13+01:00
> Created file.
> Note: You requested user feedback on this artifact. Stop calling tools to end your turn so the user can review it.

### 🤖 Agent (Step 3 - 2026-10-30T23:28:13Z)

I have authored the technical specification document for the nRF52840 USB descriptors configuration, device identification, and firmware update workflow.

Please review the specification artifact:
- [specification.md](file:///Users/gpfister/.gemini/antigravity/brain/bd3d2f5d-cc3c-4513-97fd-6b35e8fb0ddc/specification.md)

You can approve or request changes using the buttons above.

### 👤 User (Step 4 - 2026-10-30T23:34:38Z)



### 🤖 Agent (Step 5 - 2026-10-30T23:34:38Z)

> [!quote]- 💭 Reasoning Trace
> User approval triggers the protocol to signal the orchestrator. The approval confirmation needs forwarding. The next stage is the coding plan.

> [!example]- 🛠️ Tool Call: `send_message`
> ```json
{
  "Message": "[GATE_APPROVED] specification.md: User has approved the specification. Proceed to coding-plan.",
  "Recipient": "9f9ae0e3-f181-44f3-8dde-7b25c55cf4e3"
}
> ```

> [!note]- ⚙️ Execution Result (Step 6)
> Created At: 2026-10-31T00:34:42+01:00
> Completed At: 2026-10-31T00:34:42+01:00
> Message sent to "9f9ae0e3-f181-44f3-8dde-7b25c55cf4e3".

### 🤖 Agent (Step 7 - 2026-10-30T23:34:42Z)

The specification has been approved by the user, and the orchestrator has been notified to proceed to the planning phase (`coding-plan`).
