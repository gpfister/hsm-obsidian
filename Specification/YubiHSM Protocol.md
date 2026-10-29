---
title: YubiHSM Protocol Specification
tags:
  - specification
  - yubihsm
  - protocol
created: 2026-10-29
---

# YubiHSM 2 Protocol Specification

The firmware implements the wire protocol and command syntax of the **YubiHSM 2** over USB CDC-ACM virtual serial port 1.

---

## 1. Binary Packet Framing

Each transaction follows a binary packet framing format:

```text
Request:
+-------------------+--------------------+------------------------+
| Command (1 byte)  | Length (2 bytes BE)| Payload (0..N bytes)   |
+-------------------+--------------------+------------------------+

Response:
+-------------------+--------------------+------------------------+
| Resp Code (1 byte)| Length (2 bytes BE)| Payload (0..N bytes)   |
+-------------------+--------------------+------------------------+
```

- When successful, `Resp Code = Command | 0x80`.
- In case of error, `Resp Code = 0x7F` (or `0xFF`), followed by a 1-byte error code.

---

## 2. Supported Command Opcodes

| Opcode | Name | Description |
| :--- | :--- | :--- |
| `0x01` | `YMS_CMD_ECHO` | Echoes back received payload (latency & round-trip verification) |
| `0x03` | `YMS_CMD_CREATE_SESSION` | Initiates session handshake |
| `0x04` | `YMS_CMD_AUTH_SESSION` | Authenticates session and unlocks device (turns LED green) |
| `0x06` | `YMS_CMD_DEVICE_INFO` | Returns device version, serial number, and supported algorithms |
| `0x07` | `YMS_CMD_RESET_DEVICE` | Resets configuration back to initial unconfigured state (turns LED blue) |
| `0x08` | `YMS_CMD_CLOSE_SESSION` | Closes active session and locks device (turns LED red) |
| `0x1f` | `YMS_CMD_GET_PSEUDO_RANDOM` | Requests hardware TRNG random bytes from device |
| `0x70` | `YMS_CMD_DFU_INFO` | Queries firmware version and active partition slot |
| `0x71` | `YMS_CMD_DFU_CHUNK` | Streams firmware image chunks into secondary candidate slot |
| `0x72` | `YMS_CMD_DFU_APPLY` | Marks candidate image for upgrade and schedules reboot |
