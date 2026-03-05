# immurok BLE Protocol Specification

## Overview

immurok communicates over Bluetooth Low Energy (BLE) using two channels:

1. **HID Keyboard Service** — Standard BLE HID profile. Used for keystroke simulation (screen unlock, pre-auth Enter key). Also serves as a connection anchor so macOS maintains a persistent link.
2. **Custom GATT Service** — Proprietary service for commands, notifications, and data transfer.

All multi-byte integers are **little-endian** unless noted otherwise.

---

## GATT Services

### Custom immurok Service

| Attribute | Value |
|-----------|-------|
| Service UUID | `12340010-0000-1000-8000-00805f9b34fb` |
| Command Characteristic | `12340011-...` (Write, max 64 B) |
| Response Characteristic | `12340012-...` (Notify, max 64 B) |

### Device Information Service (Standard)

| Attribute | Value |
|-----------|-------|
| Service UUID | `0x180A` |
| Firmware Revision | `0x2A26` (Read) |

### OTA Service

| Attribute | Value |
|-----------|-------|
| Service UUID | `0xFEE0` |
| OTA Data Characteristic | `0xFEE1` (Read/Write) |

---

## Packet Format

All commands and responses follow:

```
[cmd: 1B][payload_len: 1B][payload: 0–62 B]
```

Maximum packet size: 64 bytes.

---

## Response Codes

| Code | Name | Meaning |
|------|------|---------|
| `0x00` | OK | Success |
| `0x06` | ERR_TIMEOUT | Operation timed out |
| `0x07` | ERR_FP_NOT_MATCH | Fingerprint did not match |
| `0x11` | WAIT_FP | Waiting for fingerprint verification |
| `0xFD` | BUSY / INVALID_STATE | Device busy (OTA in progress) or invalid state (e.g. pairing state machine) |
| `0xFE` | INVALID_PARAM | Invalid parameter |
| `0xFF` | ERROR | Unrecognized command or internal operation failure |

---

## Commands

### System

| Command | Code | Direction | Payload | Response |
|---------|------|-----------|---------|----------|
| GET_STATUS | `0x01` | App -> Device | — | `[0x00][fp_bitmap:1B][paired:1B]` |
| FP_LIST | `0x13` | App -> Device | — | `[0x00][bitmap:1B]` |
| FP_MATCH_ACK | `0x22` | App -> Device | — | `[0x00]` |
| AUTH_REQUEST | `0x33` | App -> Device | — | `[0x11]` (wait for fingerprint) |
| FACTORY_RESET | `0x36` | App -> Device | — | `[0x00]` or `[0x11]` (FP-gated) |

### Fingerprint Enrollment

| Command | Code | Direction | Payload | Response |
|---------|------|-----------|---------|----------|
| ENROLL_START | `0x10` | App -> Device | `[slot_id:1B]` | `[0x00]` or `[0x11]` (FP-gated) |
| DELETE_FP | `0x12` | App -> Device | `[slot_id:1B]` | `[0x00]` or `[0x11]` (FP-gated) |

### ECDH Pairing

| Command | Code | Direction | Payload | Response |
|---------|------|-----------|---------|----------|
| PAIR_INIT | `0x30` | App -> Device | — | Async: `[0x30][compressed_pubkey:33B]` |
| PAIR_CONFIRM | `0x31` | App -> Device | `[app_pubkey:33B]` | Async: `[0x31][status:1B]` |
| PAIR_STATUS | `0x32` | App -> Device | — | `[0x32][paired:1B]` |

ECDH computation (~2 s on CH592F) is deferred via TMOS events to avoid blocking the BLE stack. See [security.md](security.md) for the full pairing flow.

### Key Storage (SSH / OTP / API)

Key categories: `0` = SSH, `1` = OTP, `2` = API.

| Command | Code | Direction | Payload | Response |
|---------|------|-----------|---------|----------|
| KEY_COUNT | `0x60` | App -> Device | `[cat:1B]` | `[0x00][count:1B]` |
| KEY_READ | `0x61` | App -> Device | `[cat:1B][idx:1B][off:1B]` | `[0x00][total_lo:1B][off:1B][data:<=59B]` or `[0x11]` (FP-gated for API secret) |
| KEY_WRITE | `0x62` | App -> Device | `[cat:1B][idx:1B][off:1B][data:<=59B]` | `[0x00]` |
| KEY_DELETE | `0x63` | App -> Device | `[cat:1B][idx:1B]` | `[0x00]` or `[0x11]` (FP-gated) |
| KEY_COMMIT | `0x64` | App -> Device | `[cat:1B][idx:1B]` | `[0x00]` or `[0x11]` (FP-gated) |
| KEY_SIGN | `0x65` | App -> Device | `[cat(0):1B][idx:1B][hash_off:1B][hash_data]` | `[0x00][size:1B]` or `[0x11]` (FP-gated) |
| KEY_GETPUB | `0x66` | App -> Device | `[cat(0):1B][idx:1B]` | `[0x00][size:1B]` |
| KEY_GENERATE | `0x67` | App -> Device | `[cat(0):1B][name:16B]` | `[0x00][size:1B][idx:1B]` or `[0x11]` (FP-gated) |
| KEY_RESULT | `0x68` | App -> Device | `[off:1B]` | `[0x00][total:1B][off:1B][data:<=59B]` |
| KEY_OTP_GET | `0x69` | App -> Device | `[idx:1B][ts:4B LE]` | `[0x00][code:6B]` or `[0x11]` (FP-gated) |

Key read privacy: SSH keys return only 80 B (private key bytes masked); OTP keys return only 60 B (secret masked). API key reads at offset >= 32 (secret region) return `WAIT_FP` and require fingerprint verification before data is returned.

---

## Notifications (Device -> App)

### Fingerprint Match (`0x21`)

Sent when a stored fingerprint is matched on the sensor.

```
[0x21][page_id:2B LE][hmac:8B]    (11 bytes total)
```

- `page_id` — Matched template slot ID.
- `hmac` — First 8 bytes of `HMAC-SHA256(shared_key, 0x21 || page_id)`.

The app must verify the HMAC before acting on the notification, then reply with `FP_MATCH_ACK` (`0x22`).

### Enrollment Progress (`0x11`)

Sent during fingerprint enrollment to report status.

```
[0x11][status:1B][captured:1B][total:1B]    (4 bytes)
```

| Status | Value | Meaning |
|--------|-------|---------|
| WAITING | `0x00` | Waiting for finger |
| CAPTURED | `0x01` | Image captured |
| PROCESSING | `0x02` | Processing template |
| LIFT_FINGER | `0x03` | Lift and re-place finger |
| COMPLETE | `0x04` | Enrollment complete |
| FAILED | `0xFF` | Enrollment failed |

Six captures are required per enrollment.

---

## Fingerprint Gate (FP Gate)

Sensitive operations (delete fingerprint, factory reset, key signing, pairing) require biometric verification before execution:

1. App sends a command (e.g. `DELETE_FP`).
2. Device returns `0x11` (WAIT_FP).
3. User touches the sensor.
4. Device sends `0x21` notification (HMAC-signed).
5. App verifies HMAC, replies with `0x22` (FP_MATCH_ACK).
6. Device executes the buffered command.

A 10-second cooldown allows consecutive operations without re-verification.

---

## BLE Connection Parameters

| Parameter | Value | Unit |
|-----------|-------|------|
| Min Interval | 24 | × 1.25 ms = 30 ms |
| Max Interval | 40 | × 1.25 ms = 50 ms |
| Slave Latency | 29 | intervals |
| Supervision Timeout | 600 | × 10 ms = 6 s |
| Effective idle interval | ~1.5 s | 50 ms × 30 |

Apple Accessory Design Guidelines require:
```
Supervision Timeout > Interval Max × (Slave Latency + 1) × 3
6000 ms > 50 ms × 30 × 3 = 4500 ms  ✓
```

Parameter update is requested 30 s after connection (after macOS completes service discovery).

---

## BLE Advertising

| Property | Value |
|----------|-------|
| Device Name | `immurok IK-1` |
| Appearance | `0x03C1` (HID Keyboard) |
| Bonding | Enabled, No Input No Output |
| Services Advertised | HID (0x1812), Custom (0x12340010...) |

---

## HID Keyboard

Standard 8-byte HID input report:

```
[modifier:1B][reserved:1B][keycode0:1B]...[keycode5:1B]
```

Used for:
- **Screen unlock** — Simulates password input + Enter.
- **Pre-auth trigger** — Sends Ctrl key to wake authentication dialogs before fingerprint notification.

Keys are auto-released after 80 ms (> 1 connection interval).

---

## OTA Firmware Update

| Property | Value |
|----------|-------|
| Encryption | AES-128-CTR |
| Integrity | HMAC-SHA256 + SHA256 |
| Max image size | 216 KB |

Flash layout (WCH Method 1):

```
0x00000 – 0x01000  JumpIAP     (4 KB)
0x01000 – 0x37000  Image A     (216 KB, running firmware)
0x37000 – 0x6D000  Image B     (216 KB, OTA staging)
0x6D000 – 0x70000  IAP         (12 KB, bootloader)
```
