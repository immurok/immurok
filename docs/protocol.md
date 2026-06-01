# immurok BLE Protocol Specification

## Overview

immurok communicates over Bluetooth Low Energy (BLE) using two channels:

1. **HID Keyboard Service** — Standard BLE HID profile. Used for keystroke simulation (screen unlock, pre-auth Enter key). Also serves as a connection anchor so macOS maintains a persistent link.
2. **Custom GATT Service** — Proprietary service for commands, notifications, and data transfer.

All multi-byte integers are **little-endian** unless noted otherwise.

---

## GATT Services

### Custom immurok Service

All custom UUIDs are full-entropy random 128-bit (v4) UUIDs. They deliberately
avoid the Bluetooth Base UUID suffix (`-0000-1000-8000-00805f9b34fb`) and the
SIG Member Service range (`0xFE00–0xFFFF`) for qualification compliance.

| Attribute | UUID | Properties |
|-----------|------|------------|
| Service | `45529919-7668-48f9-b9fe-e4eabe6595d9` | — |
| Command Characteristic | `8a537e1f-3992-4b2c-8b77-8d4e778186e1` | Write (encrypted — bond required), max 64 B |
| Response Characteristic | `76a1660d-8cf6-44d1-b3fc-70486028e289` | Notify + Read (encrypted — bond required), max 64 B |

The command value uses `ENCRYPT_WRITE` and the response value `ENCRYPT_READ`,
with `ENCRYPT_WRITE` on the response CCCD — so only a bonded peer (encrypted
link) can drive the service or subscribe to notifications.

### Device Information Service (Standard)

| Attribute | UUID |
|-----------|------|
| Service | `0x180A` |
| Firmware Revision | `0x2A26` (Read) |

### Battery Service (Standard)

| Attribute | UUID |
|-----------|------|
| Service | `0x180F` |
| Battery Level | `0x2A19` (Read/Notify) |

### OTA Service

Also moved off the SIG Member range (`0xFEE0/0xFEE1`) to random 128-bit UUIDs.

| Attribute | UUID |
|-----------|------|
| Service | `d29005de-1391-4a54-8168-bf4e3c080430` |
| OTA Data Characteristic | `c75f4c30-9a2d-4445-92e0-0e034c53d092` (Read/Write) |

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
| `0x11` | WAIT_FP | Waiting for fingerprint verification (FP gate) |
| `0xF0` | WAIT_BUTTON | `PAIR_INIT` accepted; waiting for a physical button press |
| `0xF1` | NEEDS_RESET | `PAIR_INIT` rejected — device still holds enrolled fingerprints; factory reset required |
| `0xFD` | BUSY | Device busy (OTA in progress) or invalid state (e.g. pairing state machine) |
| `0xFE` | INVALID_PARAM | Invalid parameter |
| `0xFF` | ERROR | Unrecognized command or internal operation failure |

The status code in the second byte of a `PAIR_INIT` response is the pairing-specific value (`0xF0`/`0xF1`) rather than a generic response code.

---

## Commands

### System

| Command | Code | Direction | Payload | Response |
|---------|------|-----------|---------|----------|
| GET_STATUS | `0x01` | App -> Device | — | `[0x00][fp_bitmap:1B][paired:1B]` (+ optional pending-match tail) |
| GET_BATT_RAW | `0x02` | App -> Device | — | `[0x00][mv_lo:1B][mv_hi:1B][pct:1B][adc_lo:1B][adc_hi:1B]` |
| AUTH_REQUEST | `0x33` | App -> Device | — | `[0x11]` (wait for fingerprint) |
| FACTORY_RESET | `0x36` | App -> Device | — | `[0x00]` or `[0x11]` (FP-gated) |
| GATE_CANCEL | `0x37` | App -> Device | — | `[0x00]` (cancels a pending FP-gated command) |
| CHALLENGE | `0x38` | App -> Device | `[nonce:8B]` | `[0x38][hmac:8B]` — `HMAC-SHA256(shared_key, nonce)[0:8]` |

`CHALLENGE` is used after reconnection to verify the device identity without re-pairing (see [security.md](security.md)).

### Fingerprint

| Command | Code | Direction | Payload | Response |
|---------|------|-----------|---------|----------|
| ENROLL_START | `0x10` | App -> Device | `[slot_id:1B]` | `[0x00]` or `[0x11]` (FP-gated) |
| ENROLL_CANCEL | `0x11` | App -> Device | — | `[0x00]` |
| DELETE_FP | `0x12` | App -> Device | `[slot_id:1B]` | `[0x00]` or `[0x11]` (FP-gated) |
| FP_LIST | `0x13` | App -> Device | — | `[0x00][bitmap:1B]` |
| FP_MATCH_ACK | `0x22` | App -> Device | — | `[0x00]` |

The fingerprint slot bitmap covers slots `0–28` (29 templates max).

### ECDH Pairing

| Command | Code | Direction | Payload | Response |
|---------|------|-----------|---------|----------|
| PAIR_INIT | `0x30` | App -> Device | — | `[0x30][0xF0]` (WAIT_BUTTON) or `[0x30][0xF1]` (NEEDS_RESET) |
| PAIR_CONFIRM | `0x31` | App -> Device | `[app_pubkey:33B]` | Async: `[0x31][status:1B]` |
| PAIR_STATUS | `0x32` | App -> Device | — | `[0x32][paired:1B]` |

**Pairing is button-gated.** `PAIR_INIT` does not start ECDH immediately:

1. App sends `PAIR_INIT`. If the device already holds fingerprints it replies `[0x30][0xF1]` (a factory reset is required first).
2. Otherwise the device replies `[0x30][0xF0]` (WAIT_BUTTON), blinks its LED, and opens a 30-second window.
3. The user **short-presses the physical button** on the device to confirm. The device emits a `PAIR_BUTTON` notification (`0x34`, status `0x01`) and starts ECDH key generation (~2 s, deferred via TMOS events).
4. The device sends its compressed public key asynchronously as `[0x30][compressed_pubkey:33B]`.
5. App sends `PAIR_CONFIRM` with its own compressed public key; the device computes the shared secret (~2 s, deferred) and replies `[0x31][status]`.

A 30-second timeout or a long press aborts the wait (see `PAIR_BUTTON` below). See [security.md](security.md) for the full key-derivation flow.

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

**Twelve captures** are required per enrollment, prompted at guided angles (4 centered, 3 left, 3 right, 1 up, 1 down).

### Lock Request (`0x23`)

```
[0x23]    (1 byte, no payload)
```

Sent when the user long-presses the device button to request a screen lock. The app decides based on the current screen state: if already locked it is ignored (it may follow a `0x21` unlock notification); if unlocked it triggers a system lock.

### Pair Button (`0x34`)

```
[0x34][status:1B]    (2 bytes)
```

| Status | Meaning |
|--------|---------|
| `0x00` | 30-second wait timed out — pairing aborted |
| `0x01` | Button pressed — ECDH key exchange starting |
| `0x02` | Long press — user cancelled pairing |

### Connection Parameter Update (`0xF0`)

```
[0xF0][interval_hi:1B][interval_lo:1B][latency:1B][timeout_hi:1B][timeout_lo:1B]    (6 bytes)
```

Informational notification reporting the negotiated BLE connection parameters (interval in units of 1.25 ms, timeout in units of 10 ms). The multi-byte fields in this notification are **big-endian**.

---

## Fingerprint Gate (FP Gate)

Sensitive operations (delete fingerprint, factory reset, key signing/generation/deletion/commit, OTP generation) require biometric verification before execution:

1. App sends a command (e.g. `DELETE_FP`).
2. Device returns `0x11` (WAIT_FP).
3. User touches the sensor.
4. Device sends `0x21` notification (HMAC-signed).
5. App verifies HMAC, replies with `0x22` (FP_MATCH_ACK).
6. Device executes the buffered command.

A 10-second cooldown allows consecutive operations without re-verification. `GATE_CANCEL` (`0x37`) cancels a pending gated command before the touch.

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
| Device Name | `immurok IK-1` (in scan response) |
| Appearance | `0x03C1` (HID Keyboard) |
| Bonding | Enabled, No Input No Output |
| Services Advertised | HID (`0x1812`) and Battery (`0x180F`), as 16-bit UUIDs |

The custom 128-bit immurok service is **not** advertised; it is discovered via
GATT service discovery after connection.

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
