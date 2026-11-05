---
title: "Technical Specification: HSM Dual-Mode Authentication, CLI Renaming (gphsm), and PKCS#11 Refactoring (libgphsm)"
tags:
  - specification
  - technical-spec
  - authentication
  - shamir
  - pin
  - gphsm
  - libgphsm
  - pkcs11
  - keystore
  - protocol
created: 2026-11-05
updated: 2026-11-05
status: Approved (Phase 3 Complete)
adrs:
  - "[[Architecture/Decisions/ADR-0016 - Dual-Mode Authentication Architecture (PIN and Shamir Secret Sharing)|ADR-0016: Dual-Mode Authentication Architecture]]"
  - "[[Architecture/Decisions/ADR-0017 - Host CLI Rebranding to gphsm and PKCS11 Refactoring to libgphsm|ADR-0017: Host CLI Rebranding to gphsm and PKCS#11 Refactoring to libgphsm]]"
references:
  - "[[Plan/HSM Dual-Mode Authentication, CLI Renaming (gphsm), and PKCS11 Refactoring Implementation Plan|Implementation Plan: Dual-Mode Auth, CLI Renaming & PKCS#11 Refactoring]]"
  - "[[features/dual-mode-auth-gphsm-and-libgphsm-refactoring|Feature Specification: Dual-Mode Auth, gphsm Rebranding & libgphsm Refactoring]]"
  - "[[00 - Index|Vault Index]]"
---

# Technical Specification: HSM Dual-Mode Authentication, CLI Renaming (gphsm), and PKCS#11 Refactoring (libgphsm)

> [!NOTE]
> **Specification Purpose**: Formalizes wire protocols, data structures, error codes, and interface contracts for dual-mode authentication, host CLI rebranding (`gphsm`), and PKCS#11 middleware refactoring (`libgphsm`).

---

## 1. Firmware Keystore Schema (`hsm-ncs-firmware`)

### 1.1 Authentication Method Enumeration

In `firmware/include/keystore.h`:

```c
typedef enum {
    AUTH_METHOD_NONE   = 0,  /**< Device unconfigured / factory default */
    AUTH_METHOD_PIN    = 1,  /**< Single-user PIN (6-64 chars) */
    AUTH_METHOD_SHAMIR = 2,  /**< 3-of-5 Shamir Secret Sharing */
} auth_method_t;
```

### 1.2 Persistent Keystore Structure

Stored in Zephyr Non-Volatile Storage (NVS) under file ID `0x1000` / key `0x0001`:

```c
#define HSM_MAX_PIN_LEN 64
#define HSM_SHAMIR_THRESHOLD 3
#define HSM_SHAMIR_TOTAL 5

struct hsm_keystore_header {
    uint32_t magic;           /**< Magic constant: 0x48534D31 ('HSM1') */
    uint16_t version;         /**< Keystore version: 0x0002 */
    uint8_t  auth_method;     /**< auth_method_t */
    uint8_t  configured;      /**< 1 if initialized, 0 if factory clean */
    uint8_t  pin_len;         /**< Length of stored PIN (0 if Shamir or None) */
    char     pin[HSM_MAX_PIN_LEN + 1]; /**< Null-terminated PIN string */
    uint8_t  master_secret[32];        /**< Master secret (Shamir reconstructed or internal) */
    uint8_t  secret_valid;             /**< 1 if master secret initialized */
    uint8_t  retries_remaining;        /**< Failed PIN attempts remaining (default 3) */
    uint8_t  reserved[16];             /**< Reserved padding */
};
```

---

## 2. AT Command Wire Protocols

### 2.1 `AT+STATUS?`

- **Request**: `AT+STATUS?\r\n`
- **Response**:
  ```text
  \r\n+STATUS: AUTH=<auth>,CONFIGURED=<conf>,METHOD=<method>,RETRIES=<retries>,KEYCOUNT=<count>\r\nOK\r\n
  ```
- **Fields**:
  - `auth`: `1` if authenticated, `0` if locked.
  - `conf`: `1` if configured (`METHOD != NONE`), `0` if unconfigured.
  - `method`: `NONE`, `PIN`, or `SHAMIR`.
  - `retries`: Integer $\ge 0$ (remaining attempts).
  - `count`: Integer $\ge 0$ (number of occupied key slots).

### 2.2 `AT+INIT=PIN,<pin>`

- **Request**: `AT+INIT=PIN,<pin>\r\n`
- **Validation**:
  - Length of `<pin>` must be between 6 and 64 characters inclusive.
  - Keystore must have `configured == 0`.
- **Response**:
  - Success: `\r\nOK\r\n` (Sets `configured = 1`, `auth_method = AUTH_METHOD_PIN`, `retries_remaining = 3`, `auth = 0`).
  - Already Configured: `\r\n+CME ERROR: 4\r\n`
  - Invalid Parameters / Length: `\r\n+CME ERROR: 5\r\n`

### 2.3 `AT+INIT=SHAMIR`

- **Request**: `AT+INIT=SHAMIR\r\n`
- **Behavior**:
  - Generates 32 bytes of cryptographic randomness via CC310 TRNG.
  - Computes 3-of-5 Shamir polynomial shares over $GF(2^8)$.
  - Sets `configured = 1`, `auth_method = AUTH_METHOD_SHAMIR`, `auth = 0`.
- **Response**:
  ```text
  \r\n+INIT: 1,<hex_share_1>\r\n
  +INIT: 2,<hex_share_2>\r\n
  +INIT: 3,<hex_share_3>\r\n
  +INIT: 4,<hex_share_4>\r\n
  +INIT: 5,<hex_share_5>\r\n
  OK\r\n
  ```

### 2.4 `AT+AUTH=<pin>`

- **Request**: `AT+AUTH=<pin>\r\n`
- **Enforcement**:
  - If `auth_method != AUTH_METHOD_PIN`: Returns `\r\n+CME ERROR: 5\r\n` immediately.
  - If `retries_remaining <= 0`: Returns `\r\n+CME ERROR: 3\r\n` (PIN Blocked).
  - If `<pin>` matches stored PIN: Sets `auth = 1`, `retries_remaining = 3`, returns `\r\nOK\r\n`.
  - If `<pin>` does not match: Decrements `retries_remaining`. If 0, returns `\r\n+CME ERROR: 3\r\n`, else `\r\n+CME ERROR: 2\r\n`.

### 2.5 `AT+SHARE=<idx>,<share>`

- **Request**: `AT+SHARE=<idx>,<share>\r\n`
- **Enforcement**:
  - If `auth_method != AUTH_METHOD_SHAMIR`: Returns `\r\n+CME ERROR: 5\r\n` immediately.
  - `<idx>` must be 1 to 5.
  - `<share>` must be 64 hexadecimal characters.
- **Response**:
  - If fewer than 3 unique valid shares received: `\r\n+SHARE: OK,<shares_needed>\r\nOK\r\n`
  - If 3rd valid share received and secret verified: Sets `auth = 1`, returns `\r\nOK\r\n`.
  - Invalid Share / Polynomial Error: `\r\n+CME ERROR: 2\r\n`.

---

## 3. Host CLI (`gphsm`) Component Design

### 3.1 Binary Compilation & Static Packaging

In `hsm-cli/Makefile`:
```makefile
BINARY_NAME := gphsm
BUILD_DIR := bin
CGO_ENABLED := 0

build:
	@mkdir -p $(BUILD_DIR)
	CGO_ENABLED=$(CGO_ENABLED) go build -ldflags "-s -w" -o $(BUILD_DIR)/$(BINARY_NAME) ./cmd/gphsm
	@ln -sf $(BINARY_NAME) $(BUILD_DIR)/hsmutil
	@ln -sf $(BINARY_NAME) $(BUILD_DIR)/hsmutils
```

### 3.2 Pure-Go `pam_pwquality` Enforcement

```go
package security

type PWQualityConfig struct {
    MinLen    int
    MinDiff   int
    MaxRepeat int
}

func ValidatePIN(pin string) error {
    if len(pin) < 10 {
        return fmt.Errorf("PIN length (%d) is below minimum required (10)", len(pin))
    }
    var hasUpper, hasLower, hasDigit, hasSpecial bool
    for _, r := range pin {
        switch {
        case unicode.IsUpper(r): hasUpper = true
        case unicode.IsLower(r): hasLower = true
        case unicode.IsDigit(r): hasDigit = true
        default: hasSpecial = true
        }
    }
    classes := 0
    if hasUpper { classes++ }
    if hasLower { classes++ }
    if hasDigit { classes++ }
    if hasSpecial { classes++ }
    if classes < 3 {
        return fmt.Errorf("PIN must contain at least 3 character classes (upper, lower, digit, special)")
    }
    // Check max repeat and sequential substrings...
    return nil
}
```

---

## 4. PKCS#11 Provider (`libgphsm`) Integration

### 4.1 Token Flag Mapping

| Device State | `flags` in `CK_TOKEN_INFO` |
| :--- | :--- |
| `CONFIGURED == 0` | `0` (Uninitialized) |
| `CONFIGURED == 1`, `METHOD == PIN` | `CKF_TOKEN_INITIALIZED \| CKF_USER_PIN_INITIALIZED \| CKF_LOGIN_REQUIRED` |
| `CONFIGURED == 1`, `METHOD == SHAMIR` | `CKF_TOKEN_INITIALIZED \| CKF_USER_PIN_INITIALIZED \| CKF_PROTECTED_AUTHENTICATION_PATH` |

### 4.2 Error Code Mapping

| Condition | Cryptoki Return Value (`CK_RV`) |
| :--- | :--- |
| Pre-authenticated (`AUTH == 1`) | `CKR_OK` |
| Successful `AT+AUTH` in PIN mode | `CKR_OK` |
| Bad PIN in PIN mode | `CKR_PIN_INCORRECT` |
| 0 Retries Remaining (Blocked) | `CKR_PIN_LOCKED` |
| Locked in Shamir mode (no shares) | `CKR_PIN_LOCKED` (with diagnostic log) |
| Device disconnected / Transport error | `CKR_DEVICE_REMOVED` / `CKR_DEVICE_ERROR` |
