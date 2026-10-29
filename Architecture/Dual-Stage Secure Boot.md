---
title: Dual-Stage Secure Boot
tags:
  - architecture
  - security
  - bootloader
  - mcuboot
created: 2026-10-29
---

# Dual-Stage Secure Boot

The dongle enforces an unbroken cryptographic chain of trust from initial hardware power-on through application execution.

```mermaid
sequenceDiagram
    participant ROM as Cortex-M4 ROM / Reset
    participant B0 as Stage 1: B0 Immutable Bootloader
    participant MCUboot as Stage 2: Upgradable MCUboot
    participant App as Stage 3: HSM Application

    ROM->>B0: Jump to Flash Origin (0x00000000)
    Note over B0: Initializes ARM CryptoCell CC310
    B0->>B0: Computes SHA-256 of MCUboot partition
    B0->>B0: Verifies ECDSA P-256 signature with embedded root public key
    alt Signature Valid
        B0->>MCUboot: Chain of trust established, jump to MCUboot
    else Tampered / Corrupted
        B0->>B0: Halt / Reset
    end

    Note over MCUboot: Checks slot 0 vs slot 1 candidate
    MCUboot->>MCUboot: Validates active application signature
    MCUboot->>App: Jump to application entry point
```

---

## Flash Layout & Signing

- **Stage 1 (B0)**: Immutable first stage bootloader placed at address `0x00000000`. Uses hardware acceleration (Nordic CryptoCell CC310) for SHA-256 hashing and ECDSA P-256 signature verification.
- **Stage 2 (MCUboot)**: Upgradable bootloader located immediately following B0. Handles partition swap, image verification, and rollback protection.
- **Keys**: All signing keys are kept at root level in `keys/root-ec-p256.pem`. The public key `keys/root-ec-p256-pub.c` is compiled into the bootloaders.
