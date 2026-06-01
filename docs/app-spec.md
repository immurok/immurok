# immurok macOS App — Technical Specification

## Overview

The immurok macOS app is a background-resident menu bar application (`LSUIElement`) that communicates with the CH592F fingerprint device over BLE. It provides screen unlock, system authentication, SSH key management, and related features.

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                    immurokApp (SwiftUI)               │
│  MenuBarExtra (status bar) + Settings Window (500x420)│
├────────────┬──────────────┬───────────┬───────────────┤
│ AppDelegate│ AppViewModel │ContentView│ QuickFillPanel│
│  lifecycle │  UI state    │ settings  │ global hotkey │
├────────────┴──────────────┴───────────┴───────────────┤
│                     BLEManager                        │
│       GATT comms / command queue / reconnect / keepalive
├──────────┬──────────┬──────────┬──────────────────────┤
│ PAMSocket│SSHAgent  │CLISocket │  ImmurokSecurity     │
│ Server   │Server    │Server    │  ECDH/HMAC/Keychain  │
└──────────┴──────────┴──────────┴──────────────────────┘
```

## Source File Layout

| File | Lines | Purpose |
|------|-------|---------|
| `immurokApp.swift` | ~190 | App entry point, MenuBarExtra + Settings Window |
| `AppDelegate.swift` | ~600 | Service initialization, fingerprint-match handling, screen unlock |
| `AppViewModel.swift` | ~420 | UI state management, device-operation wrappers |
| `BLEManager.swift` | ~1960 | BLE GATT core, command protocol, reconnect/keepalive |
| `ImmurokSecurity.swift` | ~230 | ECDH pairing, HMAC verification, Keychain storage |
| `PAMSocketServer.swift` | ~940 | Unix socket PAM authentication service |
| `SSHAgentServer.swift` | ~370 | OpenSSH Agent protocol implementation |
| `CLISocketServer.swift` | ~380 | CLI tool communication interface |
| `FingerprintView.swift` | ~630 | Fingerprint management UI and enrollment flow |
| `SettingsTabViews.swift` | ~2190 | Settings window tabs |
| `QuickFillPanel.swift` | ~820 | Quick Fill floating panel |
| `KeystoreViewModel.swift` | ~400 | Key browsing / import / export |
| `SSHKeyCache.swift` | ~270 | Local SSH public key cache |
| `KeyNameCache.swift` | ~105 | Unified key-name cache |
| `LocalizationManager.swift` | ~1110 | 8-language localization |
| `SetupManager.swift` | ~110 | Initial-state checks |
| `SetupWizardView.swift` | ~360 | First-launch wizard |
| `GlobalHotKey.swift` | — | Global hotkey registration |
| `LogManager.swift` | ~25 | Runtime log |

## BLE Communication

### GATT Services

All custom UUIDs are full-entropy random 128-bit (v4) UUIDs — they avoid the Bluetooth Base UUID suffix and the SIG Member range (`0xFE00–0xFFFF`) for compliance. The custom characteristics require a bonded (encrypted) link.

| Service / Characteristic | UUID | Purpose |
|--------------------------|------|---------|
| immurok custom service | `45529919-7668-48f9-b9fe-e4eabe6595d9` | Command / response channel |
| CMD characteristic | `8a537e1f-3992-4b2c-8b77-8d4e778186e1` | App → Device (encrypted Write) |
| RSP characteristic | `76a1660d-8cf6-44d1-b3fc-70486028e289` | Device → App (encrypted Notify) |
| OTA service / char | `d29005de-...0430` / `c75f4c30-...d092` | Firmware update |
| Device Info | `180A` / `2A26` | Firmware version read |
| Battery | `180F` / `2A19` | Battery level read/notify |

### Command Protocol

Packet format: `[cmd:1B][payloadLen:1B][payload...]`

| Command | ID | Description | FP-gated |
|---------|----|-------------|----------|
| GET_STATUS | 0x01 | Status / battery / version / pending match | No |
| GET_BATT_RAW | 0x02 | Raw battery voltage + percentage + ADC | No |
| ENROLL_START | 0x10 | Enroll fingerprint | Yes |
| ENROLL_CANCEL | 0x11 | Cancel enrollment | No |
| DELETE_FP | 0x12 | Delete fingerprint | Yes |
| FP_LIST | 0x13 | Fingerprint bitmap | No |
| FP_MATCH_ACK | 0x22 | Match acknowledgement | No |
| PAIR_INIT | 0x30 | ECDH pairing init (button-gated) | No |
| PAIR_CONFIRM | 0x31 | ECDH pairing confirm | No |
| PAIR_STATUS | 0x32 | Pairing status query | No |
| AUTH_REQUEST | 0x33 | PAM authentication request | Yes |
| FACTORY_RESET | 0x36 | Factory reset | Yes |
| GATE_CANCEL | 0x37 | Cancel pending FP gate | No |
| CHALLENGE | 0x38 | Device identity verification | No |
| KEY_COUNT | 0x60 | Key count | No |
| KEY_READ | 0x61 | Read key (public part) | No |
| KEY_WRITE | 0x62 | Write key | Yes |
| KEY_DELETE | 0x63 | Delete key | Yes |
| KEY_COMMIT | 0x64 | Commit to flash | Yes |
| KEY_SIGN | 0x65 | ECDSA signature | Yes |
| KEY_GETPUB | 0x66 | Get public key | No |
| KEY_GENERATE | 0x67 | Generate keypair | Yes |
| KEY_RESULT | 0x68 | Read result buffer | No |
| KEY_OTP_GET | 0x69 | TOTP compute | Yes |

Device → App notifications: `0x21` fingerprint match (HMAC-signed), `0x11` enrollment progress, `0x23` lock request, `0x34` pair-button event, `0xF0` connection-parameter update. See [protocol.md](protocol.md) for the wire format.

### Connection Management

- **Device discovery**: `retrieveConnectedPeripherals` finds the already-paired HID device.
- **Reconnect timer**: after a disconnect, retries `doConnect()` every 1 s.
- **Connection timeout**: a `.connecting` state lasting over 10 s auto-resets.
- **Sleep keepalive**: `beginActivity(.idleSystemSleepDisabled)` prevents deep power saving from dropping BLE.
- **Command queue**: serial dispatch, `commandInFlight` mutex, 5 s timeout.

### Device State Machine

```
disconnected → connecting → connected(name)
     ↑                           │
     └───── didDisconnect ───────┘
```

## Security

### ECDH Pairing (Button-Gated)

```
App                          Device
 │  PAIR_INIT (0x30)            │
 │──────────────────────────────►│
 │  [0x30][0xF0] WAIT_BUTTON    │ ← device blinks LED, opens 30 s window
 │◄──────────────────────────────│
 │   (user short-presses button)│
 │  [0x34][0x01] confirmed      │
 │◄──────────────────────────────│ ← device generates P-256 keypair (~2 s)
 │  [0x30][device_pubkey:33B]   │
 │◄──────────────────────────────│
 │  PAIR_CONFIRM(app_pubkey)    │
 │──────────────────────────────►│ ← App generates its keypair
 │  [0x31][OK]                  │ ← device computes shared secret
 │◄──────────────────────────────│
 │                              │
 │  Both: ECDH → HKDF-SHA256   │
 │  salt: "immurok-pairing-salt"│
 │  info: "immurok-shared-key"  │
 │  → shared_key (32 bytes)     │
```

`PAIR_INIT` is rejected with `[0x30][0xF1]` (NEEDS_RESET) if the device still holds fingerprints. A 30 s timeout (`0x34 0x00`) or long press (`0x34 0x02`) aborts.

### Challenge-Response Verification

```
After connection:
  UUID in cache? → Yes → isDeviceVerified = true (skip challenge)
                → No  → App sends [0x38][nonce:8B]
                        Device replies [0x38][HMAC-SHA256(key, nonce)[0:8]]
                        App verifies → on success, caches UUID
```

### Fingerprint Match Signing

```
Device → App: [0x21][page_id:2B LE][HMAC-SHA256(key, 0x21||page_id)[0:8]]
App acts on the notification only after verifying the HMAC.
```

### Keychain Storage

| Service ID | Content | Access Level |
|------------|---------|--------------|
| `com.immurok.shared-key` | ECDH shared key (32 B) | AfterFirstUnlock |
| `com.immurok.password` | Screen-unlock password | AfterFirstUnlock |
| `com.immurok.verified-device` | Verified device UUID | AfterFirstUnlock |

### Degraded Mode

When `isDeviceVerified = false`, the following are disabled: screen unlock, PAM authentication, SSH signing, OTP, fingerprint enroll/delete, key write/delete/generate, factory reset.

Still allowed: BLE connection, read-only queries (GET_STATUS / FP_LIST / KEY_READ / KEY_GETPUB), and re-pairing.

## Screen Unlock

### Flow

```
1. User touches the fingerprint sensor.
2. Device sends an HID CTRL key → macOS wakes the screen.
3. Device searches → matches → sends 0x21 notification (HMAC-signed).
4. App receives → verifies HMAC → handleFingerprintMatch().
5. Detect lock window (loginwindow / ScreenSaverEngine).
6. fakeKeyStrokes() types the password + Enter.
7. After 2 s, check isScreensaverWindowVisible(); retry once on failure.
```

### Pending Match

When BLE disconnects during sleep, the `0x21` fingerprint-match notification is lost. The device marks the match `pending`; after reconnection, when the app sends GET_STATUS, the firmware appends the pending-match data to the response (expires after 30 s).

### Safeguards

- Before typing the password, a `CGEvent flagsChanged` clears all modifiers (prevents a stuck CTRL).
- `HID_KEY_RELEASE_EVT` failures auto-retry after 10 ms.
- A failed `HidDev_Report` press does not schedule a release.

## PAM Authentication

### Unix Socket Protocol

```
Socket: ~/.immurok/pam.sock (0o600)
Request:  AUTH:username:service
Response: OK | DENY | RETRY:remaining
```

### Handling Flow

```
1. PAM module connects → AUTH:user:service
2. Verify peer UID (root or current user)
3. Check whether the service is enabled (sudo / authorization)
4. Check pre-authorization (auto-approve within a 10 s window)
5. Check isDeviceVerified
6. Send AUTH_REQUEST → wait for fingerprint → 30 s timeout
7. Fingerprint match → OK | 3 failures → DENY
```

### Pre-authorization

When a fingerprint match arrives with no pending PAM request, a 10 s pre-authorization window is set. PAM requests received within the window are auto-approved. The window can be scoped per service.

### Supported PAM Services

| Service | Setting | Purpose |
|---------|---------|---------|
| sudo / sudo_local | immurok.sudoAuthEnabled | sudo commands |
| authorization | immurok.authorizationEnabled | System permission prompts |

## SSH Agent

### Socket Protocol

```
Socket: ~/.immurok/agent.sock (0o600)
Protocol: OpenSSH Agent Protocol
```

### Supported Messages

| Message | ID | Description |
|---------|----|-------------|
| REQUEST_IDENTITIES | 11 | Return all SSH public keys |
| SIGN_REQUEST | 13 | ECDSA-SHA256-NISTP256 signature (FP-gated) |
| Others | — | Return AGENT_FAILURE |

### Key Cache

```
~/.immurok/ssh_keys.json
[{index, name, publicKeyBlob, fingerprint}, ...]
```

Synced automatically on device connect: KEY_COUNT → KEY_READ chain → compute SHA256 fingerprint → write JSON.

## Key Management

### Storage Categories

| Category | ID | Entry Size | Max Count | Content |
|----------|----|-----------|-----------|---------|
| SSH | 0 | 112 B | 32 | name(16) + pubkey(64) + privkey(32) |
| OTP | 1 | 92 B | 128 | name(30) + service(30) + secret(32) |
| API | 2 | 160 B | 50 | name(32) + key(128) |

### Operation Permissions

| Operation | FP-gated | Notes |
|-----------|----------|-------|
| KEY_READ | No (except API secret) | Public key / name readable directly |
| KEY_WRITE + KEY_COMMIT | Yes | Writing requires fingerprint confirmation |
| KEY_DELETE | Yes | Deletion requires fingerprint confirmation |
| KEY_SIGN | Yes | ECDSA signing requires fingerprint |
| KEY_GENERATE | Yes | Keypair generation requires fingerprint |
| KEY_OTP_GET | Yes | TOTP compute requires fingerprint |

## Quick Fill

### Activation

- Global hotkey: `Ctrl+\` (customizable)
- Registered via `GlobalHotKey` using a Carbon Event / NSEvent monitor

### Panel Features

- Search box + entry list (up to 6 rows)
- Search prefix filters: `o ` OTP, `a ` API, `s ` SSH
- Selecting an entry → FP gate (OTP / API) → auto-paste into the focused window
- 30 s fingerprint verification timeout

## CLI Interface

```
Socket: ~/.immurok/cli.sock (0o600)
Protocol: line text, colon-separated

Commands:
  PING              → OK:immurok
  LIST:category     → OK:count\nname1\nname2\n\n
  GET:category:name → OK:value
  ERROR:reason
```

## Settings UI

### Tab Structure

| Tab | Content |
|-----|---------|
| Device | Connection status, battery, version, fingerprint management, pairing, password setup |
| Keys | SSH/OTP/API key browse, add, delete, import/export |
| Permissions | PAM feature toggles, Accessibility permission, login item |
| Status | System status checklist (connection / fingerprint / password / permission / PAM) |
| About | Version info, runtime log window |

## Localization

Supports 8 languages with automatic system-language detection:

zh-Hans (Simplified Chinese), zh-Hant (Traditional Chinese), en, ja, fr, es, pt, ru

External custom translations are supported via `~/.immurok/strings.json`.

## Filesystem Layout

```
~/.immurok/
  ├── pam.sock          Unix socket (PAM)
  ├── agent.sock        Unix socket (SSH Agent)
  ├── cli.sock          Unix socket (CLI)
  ├── ssh_keys.json     SSH public key cache
  └── strings.json      Custom translations (optional)

/usr/local/lib/pam/
  └── pam_immurok.so    PAM module
```

## Launch Sequence

```
applicationDidFinishLaunching
  → BLEManager.connect()
  → setupBLECallbacks()
  → PAMSocketServer()
  → SSHAgentServer()    (if enabled)
  → CLISocketServer()   (if enabled)
  → GlobalHotKey()      (Quick Fill)
  → SMAppService.mainApp.register()  (login item)

After a successful BLE connection:
  → performChallengeVerification()  (UUID cache / challenge)
  → onDeviceConnected
    → refreshDeviceStatus()         (GET_STATUS + pending match)
    → SSHKeyCache.sync()            (SSH key sync)
    → KeyNameCache.syncNonSSH()     (OTP / API name sync)
```
