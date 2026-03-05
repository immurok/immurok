# immurok Security Architecture

## Threat Model

immurok is a wireless fingerprint authenticator. The primary threats are:

1. **BLE eavesdropping** — An attacker captures BLE packets to extract credentials.
2. **Notification forgery** — An attacker sends fake fingerprint-match notifications to the companion app.
3. **Replay attacks** — An attacker replays a previously captured valid notification.
4. **Credential theft** — An attacker extracts stored keys or passwords from the device or app.
5. **Unauthorized operations** — An attacker triggers privileged operations (key signing, factory reset) without biometric proof.

---

## ECDH Pairing

### Overview

Device and app establish a shared symmetric key via Elliptic Curve Diffie-Hellman (ECDH) on the **NIST P-256 (secp256r1)** curve. This key is used to authenticate all subsequent fingerprint-match notifications via HMAC.

### Flow

```
    App                                    Device (CH592F)
     │                                        │
     │──── PAIR_INIT (0x30) ─────────────────>│
     │                                        │ Generate ephemeral P-256 keypair
     │                                        │ (via TMOS deferred event, ~2 s)
     │<──── [0x30][device_compressed_pub:33B]──│
     │                                        │
     │ Generate ephemeral P-256 keypair       │
     │                                        │
     │──── PAIR_CONFIRM (0x31) ──────────────>│
     │     [app_compressed_pub:33B]           │
     │                                        │ Compute shared secret
     │                                        │ (via TMOS deferred event, ~2 s)
     │                                        │ Derive shared_key via HKDF
     │                                        │ Save to EEPROM
     │                                        │ Wipe private key from RAM
     │<──── [0x31][0x00] ─────────────────────│
     │                                        │
     │ Compute shared secret                  │
     │ Derive shared_key via HKDF             │
     │ Save to Keychain                       │
     │                                        │
```

### Deferred Computation

ECDH scalar multiplication takes ~2 s on CH592F (RISC-V, 60 MHz). To avoid stalling the BLE stack:

- Key generation and shared-secret computation are scheduled as **TMOS events** (the CH592 cooperative scheduler).
- BLE interrupts continue to fire during computation.
- The `uECC` library calls a watchdog callback (`WWDG_SetCounter(0)`) periodically to prevent hardware watchdog reset.
- BLE supervision timeout (6 s) exceeds the computation time (2 s), so the connection stays alive.

### Endianness

The `uECC` library operates in **little-endian** (native RISC-V). Apple CryptoKit uses **big-endian** (SEC1 standard). Coordinate bytes are reversed (`reverse_32()`) at the boundary:

- Device -> App: reverse x-coordinate before compressing and sending.
- App -> Device: firmware reverses x-coordinate after receiving and decompressing.
- Shared secret: reversed before HKDF derivation.

### Key Properties

- **Ephemeral keys** — A fresh keypair is generated for each pairing. No long-term private key is stored.
- **Forward secrecy** — Compromising a future pairing does not reveal past shared keys.
- **Private key erasure** — `memset(priv, 0, 32)` immediately after shared-secret computation.

---

## Key Derivation (HKDF)

The raw ECDH shared secret is never used directly. A symmetric key is derived via **HKDF-SHA256** (RFC 5869):

| Parameter | Value |
|-----------|-------|
| Hash | SHA-256 |
| Salt | `immurok-pairing-salt` (20 bytes) |
| IKM | ECDH shared secret (32 bytes, big-endian) |
| Info | `immurok-shared-key` (18 bytes) |
| Output | 32 bytes |

Implementation (single-block expand):

```
PRK = HMAC-SHA256(salt, IKM)             // Extract
OKM = HMAC-SHA256(PRK, info || 0x01)     // Expand (1 block)
```

Both firmware and app derive identical `shared_key` values from the same shared secret.

---

## HMAC-Signed Notifications

### Fingerprint Match (`0x21`)

When the sensor matches a stored template, the device sends:

```
[0x21][page_id:2B LE][hmac:8B]     (11 bytes)
```

HMAC computation:

```
message  = 0x21 || page_id          (3 bytes)
full_mac = HMAC-SHA256(shared_key, message)
hmac     = full_mac[0:8]            (first 8 bytes)
```

The app verifies the HMAC before accepting the notification. A mismatch causes the notification to be silently discarded.

### Truncation Rationale

8 bytes (64 bits) of HMAC provides sufficient security for an interactive protocol where:
- Each notification is tied to a physical fingerprint touch.
- BLE MTU constraints favor compact packets.
- Brute-forcing 2^64 is infeasible in practice.

---

## Fingerprint Gate

Sensitive device operations require biometric proof before execution:

| Operation | Command |
|-----------|---------|
| Delete fingerprint | `0x12` |
| Key signing (ECDSA) | `0x65` |
| Key generation | `0x67` |
| Key deletion | `0x63` |
| Key commit | `0x64` |
| OTP generation | `0x69` |
| Factory reset | `0x36` |

When a gated command arrives:

1. Device buffers the command and returns `WAIT_FP` (`0x11`).
2. User touches the fingerprint sensor.
3. Device verifies the fingerprint locally.
4. If matched, device sends HMAC-signed `0x21` notification.
5. App verifies HMAC and replies with `FP_MATCH_ACK` (`0x22`).
6. Device executes the buffered command.

A **10-second cooldown** allows consecutive gated operations without repeated biometric checks within the same session.

---

## Password Handling

### Storage

The macOS login password is stored **only** in the macOS Keychain:

| Keychain Item | Service ID | Access Level |
|---------------|-----------|--------------|
| Shared Key | `com.immurok.shared-key` | `kSecAttrAccessibleAfterFirstUnlock` |
| Password | `com.immurok.password` | `kSecAttrAccessibleAfterFirstUnlock` |

The password is **never transmitted over BLE** and **never stored on the device**.

### Screen Unlock Flow

1. Device detects fingerprint match -> sends HMAC-signed `0x21`.
2. App verifies HMAC.
3. App loads password from Keychain.
4. App simulates keyboard input via `CGEvent` (requires Accessibility permission).
5. Password cleared from memory after keystroke simulation.

---

## PAM Integration

### Architecture

```
PAM client (sudo, screensaver, etc.)
    │
    ▼
pam_immurok.so ──── Unix socket ────> immurok.app (PAMSocketServer)
                    ~/.immurok/pam.sock        │
                                               ▼
                                        BLE -> Device -> Fingerprint
```

### Socket Security

| Property | Value |
|----------|-------|
| Path | `~/.immurok/pam.sock` |
| Permissions | `0600` (owner only) |
| Peer verification | `getpeereid()` — only UID 0 (root) or current user accepted |
| Timeout | 40 seconds |

### Protocol

Request: `AUTH:<username>:<service>\0`

Response:
- `OK` — Authentication succeeded.
- `DENY` — Authentication failed.
- `TIMEOUT` — Device not connected or timed out.

### Pre-authorization

If the app receives a fingerprint match while no PAM request is pending, it stores a **pre-authorization token** for the service. When a PAM request arrives shortly after, the token is consumed immediately without requiring a second fingerprint touch.

---

## Fingerprint Template Security

| Property | Value |
|----------|-------|
| Sensor | ZW3021 (capacitive) |
| Storage | ZW3021 internal flash (not on host MCU) |
| Capacity | 29 templates |
| Enrollment | 6 captures merged into one template |
| Module password | Derived from device MAC address |

Templates never leave the sensor module. The host MCU (CH592F) only receives match/no-match results and template IDs. There is no API to extract raw template data over BLE.

---

## On-Device Key Storage

The device stores cryptographic keys in CH592F EEPROM (DataFlash):

| Category | Code | Entry Size | Max Entries | Read Privacy |
|----------|------|-----------|-------------|--------------|
| SSH | `0` | 112 B | 32 | Public key only (private key masked) |
| OTP | `1` | 92 B | 128 | Name + issuer only (secret masked) |
| API | `2` | 160 B | 50 | Secret region FP-gated (offset >= 32) |

SSH private keys are used for **on-device ECDSA signing** — the private key never leaves the device. The app sends a hash to be signed; the device returns the signature after fingerprint verification.

---

## OTA Update Security

| Property | Value |
|----------|-------|
| Encryption | AES-128-CTR (per-device key) |
| Integrity | HMAC-SHA256 over encrypted image |
| Verification | SHA256 hash of decrypted image |
| Key source | Generated per development machine (`generate_ota_keys.py`) |

The OTA image is encrypted and signed before transmission. The IAP bootloader verifies integrity before committing the image to flash.

---

## Summary

| Layer | Mechanism | Key Size |
|-------|-----------|----------|
| Pairing | ECDH P-256 | 256-bit curve |
| Key derivation | HKDF-SHA256 | 256-bit output |
| Notification auth | HMAC-SHA256 (truncated) | 64-bit tag |
| Password storage | macOS Keychain | OS-managed |
| Template storage | ZW3021 on-chip | Hardware-isolated |
| SSH key signing | On-device ECDSA P-256 | 256-bit curve |
| OTA integrity | AES-128-CTR + HMAC-SHA256 | 128-bit / 256-bit |
