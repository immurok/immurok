# Linux App Redesign — Rust Rewrite with macOS Feature Parity

## Overview

Rewrite the Linux app (`app-linux/`) in Rust, achieving full feature parity with the macOS app (excluding Quick Fill). The new codebase replaces the existing Python implementation with a Cargo workspace producing two binaries: `immurok-daemon` and `immurok-cli`.

## Decisions

| Decision | Choice |
|----------|--------|
| Feature scope | Full macOS parity (no Quick Fill) |
| Language | Rust |
| BLE library | bluer (BlueZ D-Bus) |
| Binary architecture | Separate: daemon + cli |
| CLI style | Subcommands + optional `tui` mode |
| Auth dialog | Keep existing GTK4 script |
| PAM module | Keep existing C implementation |
| OTA | CLI subcommand via daemon socket |
| Localization | None, English only |

## Architecture

```
┌───────────────────────────────────────────────────────┐
│              immurok-daemon (Rust, tokio)               │
│  ┌─────────┬──────────┬───────────┬────────┬────────┐  │
│  │  BLE    │  Socket  │ SSH Agent │ Screen │  OTA   │  │
│  │ (bluer) │  Server  │  Server   │Monitor │ Engine │  │
│  └────┬────┴────┬─────┴─────┬────┴───┬────┴───┬────┘  │
│       │         │           │        │        │        │
│  ┌────▼─────────▼───────────▼────────▼────────▼─────┐  │
│  │              Core Coordinator                     │  │
│  │  (event routing: FP match → PAM/unlock/pre-auth)  │  │
│  └──────────────────────────────────────────────────┘  │
├───────────────────────────────────────────────────────┤
│              immurok-common (lib crate)                │
│  protocol constants · ECDH/HKDF/HMAC · types · socket │
└───────────────────────────────────────────────────────┘

┌──────────────┐  ┌──────────────────┐  ┌──────────────┐
│ immurok-cli  │  │ pam_immurok.so   │  │ auth-dialog  │
│ (Rust, clap) │  │ (C, retained)    │  │ (GTK4, kept) │
│              │  │                  │  │              │
│ Unix Socket ─┼──┼─► daemon         │  │ daemon spawn │
└──────────────┘  └──────────────────┘  └──────────────┘
```

## Workspace Structure

```
app-linux/
├── Cargo.toml                    # [workspace]
├── crates/
│   ├── immurok-common/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── protocol.rs       # command codes, response codes, notification types
│   │       ├── security.rs       # ECDH P-256, HKDF-SHA256, HMAC
│   │       ├── socket_proto.rs   # socket text protocol parse/serialize
│   │       └── types.rs          # DeviceStatus, FpBitmap, KeyCategory, etc.
│   ├── immurok-daemon/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs           # tokio::main, signal handling, startup sequence
│   │       ├── ble.rs            # bluer GATT client, notification routing, reconnect
│   │       ├── socket.rs         # Unix socket server (PAM + CLI commands)
│   │       ├── ssh_agent.rs      # OpenSSH Agent Protocol
│   │       ├── screen.rs         # D-Bus screen lock detection
│   │       ├── keystore.rs       # device key cache (SSH/OTP/API)
│   │       ├── ota.rs            # OTA session management
│   │       ├── settings.rs       # user feature toggles
│   │       └── coordinator.rs    # event routing: FP match → action
│   └── immurok-cli/
│       ├── Cargo.toml
│       └── src/
│           ├── main.rs           # clap subcommand entry
│           ├── commands/
│           │   ├── mod.rs
│           │   ├── status.rs
│           │   ├── pair.rs
│           │   ├── fingerprint.rs
│           │   ├── keys.rs
│           │   ├── settings.rs
│           │   ├── ota.rs
│           │   └── info.rs
│           ├── tui/
│           │   ├── mod.rs
│           │   ├── app.rs
│           │   └── widgets.rs
│           └── socket_client.rs  # daemon socket client
├── scripts/
│   ├── immurok-auth-dialog       # GTK4 script (retained)
│   └── immurok-pam-helper        # PAM config helper (retained)
├── pam/
│   ├── pam_immurok.c             # PAM module (retained)
│   └── Makefile
├── immurok-daemon.service        # systemd user service
├── Makefile                      # top-level: cargo build + PAM + install
└── README.md
```

## immurok-common

### protocol.rs

All constants mirror `docs/protocol.md` and firmware definitions:

```rust
// GATT UUIDs
pub const SERVICE_UUID: &str = "12340010-0000-1000-8000-00805f9b34fb";
pub const CMD_CHAR_UUID: &str = "12340011-0000-1000-8000-00805f9b34fb";
pub const RSP_CHAR_UUID: &str = "12340012-0000-1000-8000-00805f9b34fb";
pub const OTA_SERVICE_UUID: u16 = 0xFEE0;
pub const OTA_CHAR_UUID: u16 = 0xFEE1;
pub const DEVICE_INFO_UUID: u16 = 0x180A;
pub const FW_REVISION_UUID: u16 = 0x2A26;

// Command codes (App → Device)
pub const CMD_GET_STATUS: u8 = 0x01;
pub const CMD_ENROLL_START: u8 = 0x10;
pub const CMD_ENROLL_CANCEL: u8 = 0x11;
pub const CMD_DELETE_FP: u8 = 0x12;
pub const CMD_FP_LIST: u8 = 0x13;
pub const CMD_FP_MATCH_ACK: u8 = 0x22;
pub const CMD_PAIR_INIT: u8 = 0x30;
pub const CMD_PAIR_CONFIRM: u8 = 0x31;
pub const CMD_PAIR_STATUS: u8 = 0x32;
pub const CMD_AUTH_REQUEST: u8 = 0x33;
pub const CMD_FACTORY_RESET: u8 = 0x36;
pub const CMD_GATE_CANCEL: u8 = 0x37;
pub const CMD_CHALLENGE: u8 = 0x38;
pub const CMD_KEY_COUNT: u8 = 0x60;
pub const CMD_KEY_READ: u8 = 0x61;
pub const CMD_KEY_WRITE: u8 = 0x62;
pub const CMD_KEY_DELETE: u8 = 0x63;
pub const CMD_KEY_COMMIT: u8 = 0x64;
pub const CMD_KEY_SIGN: u8 = 0x65;
pub const CMD_KEY_GETPUB: u8 = 0x66;
pub const CMD_KEY_GENERATE: u8 = 0x67;
pub const CMD_KEY_RESULT: u8 = 0x68;
pub const CMD_KEY_OTP_GET: u8 = 0x69;

// Response codes
pub const RSP_OK: u8 = 0x00;
pub const RSP_ERR_TIMEOUT: u8 = 0x06;
pub const RSP_ERR_FP_NOT_MATCH: u8 = 0x07;
pub const RSP_WAIT_FP: u8 = 0x11;
pub const RSP_BUSY: u8 = 0xFD;
pub const RSP_INVALID_PARAM: u8 = 0xFE;
pub const RSP_ERROR: u8 = 0xFF;

// Notification types (Device → App)
pub const NOTIFY_FP_MATCH_SIGNED: u8 = 0x21;
pub const NOTIFY_ENROLL_PROGRESS: u8 = 0x11;
pub const NOTIFY_CONN_PARAM_UPDATE: u8 = 0xF0;

// Key categories
pub const KEY_CAT_SSH: u8 = 0;
pub const KEY_CAT_OTP: u8 = 1;
pub const KEY_CAT_API: u8 = 2;

// Packet format
pub const MAX_PACKET_SIZE: usize = 64;
pub const MAX_PAYLOAD_SIZE: usize = 62;
```

### security.rs

```rust
// ECDH P-256 pairing
pub fn ecdh_generate_keypair() -> (PrivateKey, CompressedPublicKey);
pub fn ecdh_compute_shared_secret(privkey: &PrivateKey, peer_pubkey: &[u8; 33]) -> [u8; 32];

// HKDF-SHA256: salt="immurok-pairing-salt", info="immurok-shared-key"
pub fn hkdf_derive(shared_secret: &[u8; 32]) -> [u8; 32];

// HMAC-SHA256 truncated to 8 bytes
pub fn hmac_truncated(key: &[u8; 32], data: &[u8]) -> [u8; 8];

// FP match verification: HMAC(key, 0x21 || page_id_le)
pub fn verify_fp_match(key: &[u8; 32], page_id: u16, hmac: &[u8; 8]) -> bool;

// Challenge-Response
pub fn compute_challenge(key: &[u8; 32], nonce: &[u8; 8]) -> [u8; 8];
pub fn verify_challenge(key: &[u8; 32], nonce: &[u8; 8], hmac: &[u8; 8]) -> bool;

// Pairing data persistence: ~/.immurok/pairing.json
pub fn load_pairing(path: &Path) -> Option<PairingData>;
pub fn save_pairing(path: &Path, data: &PairingData) -> Result<()>;
pub fn clear_pairing(path: &Path) -> Result<()>;
```

Dependencies: `p256`, `hkdf`, `sha2`, `hmac` — all RustCrypto, pure Rust.

### socket_proto.rs

Unified text protocol shared between daemon and cli:

```rust
pub enum Request {
    Auth { user: String, service: String },
    Status,
    FpList,
    FpEnroll { slot: u8 },
    FpEnrollCancel,
    FpDelete { slot: u8 },
    FpVerify,
    FpStatus,
    FpLastMatch,
    GateCancel,
    PairStatus,
    PairStart,
    PairReset,
    SetUnlockSudo(bool),
    SetUnlockPolkit(bool),
    SetUnlockScreen(bool),
    GetSettings,
    GetInfo,
    OtaStart { size: u32, version: String },
    OtaData(Vec<u8>),
    OtaFinish,
}

pub enum Response {
    Ok(String),
    Deny(String),
    Retry { remaining: u8 },
    Error(String),
    Status { connected: bool, name: String, battery: u8, version: String },
}

pub fn parse_request(line: &str) -> Result<Request>;
pub fn serialize_response(resp: &Response) -> String;
```

### types.rs

```rust
pub struct PairingData {
    pub device_uuid: String,
    pub shared_key: [u8; 32],
    pub paired_at: String,
}

pub struct DeviceStatus {
    pub fp_bitmap: u8,
    pub paired: bool,
    pub battery: u8,
    pub fw_version: String,
}

pub enum KeyCategory { Ssh = 0, Otp = 1, Api = 2 }

pub enum EnrollStatus {
    Waiting, Captured, Processing, LiftFinger, Complete, Failed,
}
```

## immurok-daemon

### Startup sequence (main.rs)

```
1. Initialize tracing (stderr, systemd journal)
2. Load pairing.json + settings.json
3. Create Arc<Coordinator> with shared state + channels
4. Concurrent launch via tokio::select!:
   - ble::run()
   - socket::serve()
   - ssh_agent::serve()
   - screen::monitor()
   - signal handler (SIGTERM/SIGINT)
5. Cleanup: remove socket files, disconnect BLE
```

### coordinator.rs — event routing hub

All modules communicate through the coordinator, no direct cross-references:

```rust
pub struct Coordinator {
    // Shared state
    pub pairing: RwLock<Option<PairingData>>,
    pub settings: RwLock<Settings>,
    pub device_status: RwLock<Option<DeviceStatus>>,
    pub is_device_verified: AtomicBool,
    pub screen_locked: AtomicBool,

    // Cross-module channels
    pub ble_cmd_tx: mpsc::Sender<BleCommand>,
    pub fp_match_tx: broadcast::Sender<FpMatchEvent>,
    pub enroll_tx: broadcast::Sender<EnrollEvent>,
    pub pam_request_tx: mpsc::Sender<PamRequest>,
    pub pam_response_rx: watch::Receiver<PamResponse>,

    // Pre-authorization
    pre_auth_deadline: RwLock<Option<Instant>>,
}
```

FP match routing logic:
1. Pending PAM request? → approve it
2. Screen locked && unlock_screen enabled? → `loginctl unlock-session`
3. Otherwise → set 10s pre-auth window

### ble.rs

- Scan for connected "immurok" devices via bluer
- Connect + discover GATT services
- Subscribe to RSP characteristic notifications
- Notification routing by priority:
  1. `0x21` — signed FP match → verify HMAC → coordinator.on_fp_match()
  2. `0x11` — enroll progress → broadcast
  3. `0xF0` — connection parameter update → log
  4. Other — command response → oneshot channel for send_command
- Serial command sending with 5s timeout
- FP-gate flow: send cmd → recv WAIT_FP → wait FP match → ACK → recv final response
- Pending FP match: on reconnect, GET_STATUS response may contain appended pending match data `[page_id:2B LE][hmac:8B]` (device holds pending for 30s after BLE disconnect). If present, verify HMAC and process as a normal FP match event
- Auto-reconnect loop on disconnect (1s interval, 10s connecting timeout)

### socket.rs

- Listen on `~/.immurok/pam.sock` (chmod 0o666 for PAM root access)
- Verify peer credentials via `SO_PEERCRED` (`ucred`): only accept connections from root (PAM) or the daemon's own UID (CLI)
- Parse text protocol requests, dispatch to handlers
- AUTH flow: check pre-auth → check verified → spawn auth-dialog → wait FP → respond
- AUTH supports 3 retry attempts: FP mismatch returns `RETRY:remaining`, 3rd failure returns `DENY`
- OTA: long-lived connection with progress lines

### ssh_agent.rs

- Listen on `~/.immurok/agent.sock` (chmod 0o600)
- OpenSSH Agent binary protocol: `[length:4B BE][type:1B][payload]`
- REQUEST_IDENTITIES (11): return all SSH public keys from keystore cache
- SIGN_REQUEST (13): BLE KEY_SIGN (FP-gated) → KEY_RESULT → ECDSA-SHA256-NISTP256 signature
- All other messages: AGENT_FAILURE

### screen.rs

- D-Bus session bus via zbus
- Try GNOME: `org.gnome.ScreenSaver` `/org/gnome/ScreenSaver`
- Fallback KDE: `org.freedesktop.ScreenSaver` `/ScreenSaver`
- Listen `ActiveChanged(bool)` → update coordinator.screen_locked
- Supported desktops: GNOME, KDE Plasma, and any DE exposing `org.freedesktop.ScreenSaver` on the session bus. Non-D-Bus screen lockers (i3lock, swaylock, etc.) are unsupported; screen unlock feature degrades gracefully (remains disabled, no error)

### keystore.rs

- Post-connection sync: KEY_COUNT → KEY_READ chain → local cache
- SSH: public keys → compute SHA256 fingerprint → `~/.immurok/ssh_keys.json`
- OTP/API: names → `~/.immurok/key_names.json`

### ota.rs

- Handle OTA session over socket long connection
- Parse .imfw file header (version, size, HMAC)
- Verify signature
- Write to OTA characteristic (0xFEE1): IAP commands
- Chunked transfer with progress reporting via socket
- Verify + switch boot image + device restart

### settings.rs

```rust
// ~/.immurok/settings.json
pub struct Settings {
    pub unlock_sudo: bool,    // default true
    pub unlock_polkit: bool,  // default true
    pub unlock_screen: bool,  // default true
}
```

## immurok-cli

### Subcommands

```
immurok-cli <command>

Status:
  status                 Connection status, pairing, battery, firmware version
  info                   Detailed device information

Pairing:
  pair                   Start ECDH pairing (press device button)
  unpair                 Clear pairing + factory reset device

Fingerprint:
  fp list                List enrolled fingerprints
  fp enroll <slot>       Enroll fingerprint to slot (0-4), live progress
  fp delete <slot>       Delete fingerprint (FP-gated)
  fp verify              Verify fingerprint (test)

Keys:
  key list <category>    List SSH/OTP/API keys
  key add <category>     Add key (interactive input)
  key delete <cat> <idx> Delete key (FP-gated)
  key export-ssh <idx>   Export SSH public key (OpenSSH format)
  key generate-ssh <name> Generate SSH keypair on device
  key otp <idx>          Get TOTP code (FP-gated)

Settings:
  set sudo <on|off>      Toggle sudo fingerprint auth
  set polkit <on|off>    Toggle polkit fingerprint auth
  set screen <on|off>    Toggle screen unlock
  settings               Show all settings

OTA:
  ota <firmware.imfw>    OTA firmware upgrade with progress bar

PAM:
  pam install <service>  Install PAM config (sudo/gdm-password/polkit-1)
  pam remove <service>   Remove PAM config

Other:
  logs                   View daemon logs (journalctl)
  tui                    Enter interactive TUI panel
```

### TUI panel (ratatui)

```
┌─ immurok ──────────────────────────────────┐
│  Device:  immurok IK-1       Battery: 85%  │
│  Status:  Connected          FW: v1.2.3    │
│  Paired:  Yes                Verified: Yes │
├────────────────────────────────────────────┤
│  Fingers: [■] [■] [■] [ ] [ ]             │
│                                            │
│  sudo:   ON    polkit: ON    screen: ON    │
├────────────────────────────────────────────┤
│  [p]air  [u]npair  [e]nroll  [d]elete     │
│  [v]erify  [s]udo  [o]polkit  [k]screen   │
│  [i]nfo  [l]ogs  [q]uit                   │
└────────────────────────────────────────────┘
```

Real-time status polling via socket. FP match slot flash animation. Hotkeys trigger socket commands.

### socket_client.rs

```rust
pub struct DaemonClient {
    stream: UnixStream,
}

impl DaemonClient {
    pub fn connect() -> Result<Self>;  // ~/.immurok/pam.sock
    pub fn send(&mut self, req: &Request) -> Result<Response>;
    pub fn send_ota_stream(&mut self, data: &[u8], on_progress: impl Fn(f32));
}
```

## Key Flows

### BLE connection + verification

```
daemon startup → ble::run()
  → scan for connected "immurok" devices
  → connect + discover GATT services
  → subscribe RSP notifications
  → challenge_verify():
      UUID matches cache? → skip
      else → send [0x38][nonce:8B] → verify HMAC → cache UUID
  → get_status(): status + pending match check
  → keystore::sync_keys(): SSH pubkeys + OTP/API names
```

### FP match event routing

```
Device → notification [0x21][page_id:2B][hmac:8B]
  → ble.rs: verify_fp_match(shared_key, page_id, hmac)
  → fail → discard + warn log
  → pass → FP_MATCH_ACK (0x22) → coordinator.on_fp_match(page_id):
      pending PAM? → approve → PAM returns OK
      screen locked + unlock enabled? → loginctl unlock-session
      otherwise → set_pre_auth(10s)
```

### PAM authentication

```
pam_immurok.so → connect(~/.immurok/pam.sock) → "AUTH:user:sudo"
daemon socket.rs:
  → check pre-auth → instant OK
  → check is_device_verified → DENY if false
  → check settings toggle → DENY if disabled
  → spawn immurok-auth-dialog
  → BLE AUTH_REQUEST (0x33) → WAIT_FP
  → wait (30s):
      fp_match → OK
      fp_not_match (attempts < 3) → RETRY:remaining
      fp_not_match (attempts >= 3) → DENY
      timeout → DENY
      dialog closed → DENY
```

### SSH Agent signing

```
ssh client → connect(~/.immurok/agent.sock)
REQUEST_IDENTITIES → return cached SSH public keys
SIGN_REQUEST → find device index → BLE KEY_SIGN (FP-gated) → KEY_RESULT → ECDSA signature
```

### ECDH pairing

```
cli pair → socket → daemon:
  → generate P-256 ephemeral keypair
  → BLE PAIR_INIT (0x30) → recv [0x30][device_pubkey:33B]
  → BLE PAIR_CONFIRM (0x31, app_pubkey:33B)
  → poll (500ms × 60): WAIT_BUTTON → continue | OK → success | TIMEOUT → fail
  → shared_secret = ECDH(app_privkey, device_pubkey)
  → shared_key = HKDF-SHA256(shared_secret)
  → save_pairing(device_uuid, shared_key)
```

### OTA firmware update

```
cli ota firmware.imfw → socket long connection → daemon:
  → parse .imfw header, verify signature
  → BLE write OTA characteristic (0xFEE1)
  → IAP_CMD_ERASE → chunked data transfer → progress lines → verify → switch → reboot
```

## Degraded Mode

When `is_device_verified = false`:

**Disabled:**
- Screen unlock, PAM authentication, SSH Agent signing
- Fingerprint enroll/delete
- Key write/delete/generate/sign/OTP
- Factory reset

**Allowed:**
- BLE connection maintained
- Read-only queries (STATUS, FP_LIST, KEY_READ public parts, KEY_GETPUB)
- Re-pairing
- OTA firmware upgrade

## File System Layout

```
~/.immurok/
├── pairing.json          # ECDH pairing data (0o600)
├── settings.json         # feature toggles
├── ssh_keys.json         # SSH public key cache
├── key_names.json        # OTP/API name cache
├── pam.sock              # PAM + CLI socket (0o666)
└── agent.sock            # SSH Agent socket (0o600)
```

## Dependencies

```toml
# immurok-common
p256 = "0.13"
hkdf = "0.12"
sha2 = "0.10"
hmac = "0.12"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "2"

# immurok-daemon
tokio = { version = "1", features = ["full"] }
bluer = "0.17"
zbus = "5"
tracing = "0.1"
tracing-subscriber = "0.3"

# immurok-cli
clap = { version = "4", features = ["derive"] }
ratatui = "0.29"
crossterm = "0.28"
indicatif = "0.17"
tokio = { version = "1", features = ["net", "rt"] }
```

## Build & Install

```makefile
all: build pam

build:
	cargo build --release --workspace

pam:
	$(CC) -fPIC -shared -o pam/pam_immurok.so pam/pam_immurok.c -lpam

install: all
	install -Dm755 target/release/immurok-daemon ~/.local/bin/
	install -Dm755 target/release/immurok-cli ~/.local/bin/
	install -Dm755 scripts/immurok-auth-dialog ~/.local/bin/
	install -Dm755 scripts/immurok-pam-helper ~/.local/bin/
	sudo install -Dm755 pam/pam_immurok.so $(PAM_DIR)/
	install -Dm644 immurok-daemon.service ~/.config/systemd/user/
	systemctl --user daemon-reload
	systemctl --user enable --now immurok-daemon.service
	mkdir -p ~/.immurok

uninstall:
	systemctl --user disable --now immurok-daemon.service || true
	rm -f ~/.local/bin/immurok-{daemon,cli,auth-dialog,pam-helper}
	rm -f ~/.config/systemd/user/immurok-daemon.service
	sudo rm -f $(PAM_DIR)/pam_immurok.so
	systemctl --user daemon-reload
```

### systemd service

```ini
[Unit]
Description=immurok BLE fingerprint authentication daemon
After=bluetooth.target dbus.service

[Service]
Type=simple
ExecStart=%h/.local/bin/immurok-daemon
Restart=on-failure
RestartSec=5
Environment=RUST_LOG=info

[Install]
WantedBy=default.target
```
