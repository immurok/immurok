# Linux App Rust Rewrite Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite the Linux app in Rust with full macOS feature parity (excluding Quick Fill).

**Architecture:** Cargo workspace with 3 crates: `immurok-common` (shared protocol/security/types), `immurok-daemon` (tokio async BLE + socket + SSH agent + screen monitor), `immurok-cli` (clap subcommands + ratatui TUI). Retained: C PAM module, GTK4 auth dialog script, bash PAM helper.

**Tech Stack:** Rust, tokio, bluer (BLE), zbus (D-Bus), p256/hkdf/sha2/hmac (RustCrypto), clap, ratatui

**Spec:** `docs/superpowers/specs/2026-03-16-linux-app-redesign.md`

**Existing Python code for reference:** `app-linux/immurok/` (config.py, security.py, ble.py, socket_server.py, screen.py, daemon.py, settings.py)

---

## File Structure

### New files to create

```
app-linux-rs/
├── Cargo.toml
├── crates/
│   ├── immurok-common/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── protocol.rs
│   │       ├── security.rs
│   │       ├── socket_proto.rs
│   │       └── types.rs
│   ├── immurok-daemon/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs
│   │       ├── ble.rs
│   │       ├── socket.rs
│   │       ├── ssh_agent.rs
│   │       ├── screen.rs
│   │       ├── keystore.rs
│   │       ├── ota.rs
│   │       ├── settings.rs
│   │       └── coordinator.rs
│   └── immurok-cli/
│       ├── Cargo.toml
│       └── src/
│           ├── main.rs
│           ├── socket_client.rs
│           ├── commands/
│           │   ├── mod.rs
│           │   ├── status.rs
│           │   ├── pair.rs
│           │   ├── fingerprint.rs
│           │   ├── keys.rs
│           │   ├── settings.rs
│           │   ├── ota.rs
│           │   └── info.rs
│           └── tui/
│               ├── mod.rs
│               ├── app.rs
│               └── widgets.rs
├── Makefile
└── immurok-daemon.service
```

### Retained from existing app-linux/ (copy into app-linux-rs/)

```
scripts/immurok-auth-dialog       # GTK4 Python script
scripts/immurok-pam-helper        # Bash PAM config helper
pam/pam_immurok.c                 # C PAM module
pam/Makefile                      # PAM compilation
```

---

## Chunk 1: Workspace + immurok-common

### Task 1.1: Workspace scaffolding

**Files:**
- Create: `app-linux-rs/Cargo.toml`
- Create: `app-linux-rs/crates/immurok-common/Cargo.toml`
- Create: `app-linux-rs/crates/immurok-common/src/lib.rs`
- Create: `app-linux-rs/crates/immurok-daemon/Cargo.toml`
- Create: `app-linux-rs/crates/immurok-daemon/src/main.rs`
- Create: `app-linux-rs/crates/immurok-cli/Cargo.toml`
- Create: `app-linux-rs/crates/immurok-cli/src/main.rs`

- [ ] **Step 1: Create workspace Cargo.toml**

```toml
# app-linux-rs/Cargo.toml
[workspace]
members = ["crates/*"]
resolver = "2"
```

- [ ] **Step 2: Create immurok-common Cargo.toml**

```toml
# app-linux-rs/crates/immurok-common/Cargo.toml
[package]
name = "immurok-common"
version = "0.1.0"
edition = "2021"

[dependencies]
p256 = { version = "0.13", features = ["ecdh"] }
hkdf = "0.12"
sha2 = "0.10"
hmac = "0.12"
subtle = "2"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "2"
rand = "0.8"
hex = "0.4"

[dev-dependencies]
tempfile = "3"
```

- [ ] **Step 3: Create immurok-daemon Cargo.toml**

```toml
# app-linux-rs/crates/immurok-daemon/Cargo.toml
[package]
name = "immurok-daemon"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "immurok-daemon"
path = "src/main.rs"

[dependencies]
immurok-common = { path = "../immurok-common" }
tokio = { version = "1", features = ["full"] }
bluer = "0.17"
zbus = "5"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
serde_json = "1"
```

- [ ] **Step 4: Create immurok-cli Cargo.toml**

```toml
# app-linux-rs/crates/immurok-cli/Cargo.toml
[package]
name = "immurok-cli"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "immurok-cli"
path = "src/main.rs"

[dependencies]
immurok-common = { path = "../immurok-common" }
clap = { version = "4", features = ["derive"] }
ratatui = "0.29"
crossterm = "0.28"
indicatif = "0.17"
tokio = { version = "1", features = ["net", "rt", "macros"] }
```

- [ ] **Step 5: Create stub entry points**

```rust
// crates/immurok-common/src/lib.rs
pub mod protocol;
pub mod security;
pub mod socket_proto;
pub mod types;
```

```rust
// crates/immurok-daemon/src/main.rs
fn main() {
    println!("immurok-daemon stub");
}
```

```rust
// crates/immurok-cli/src/main.rs
fn main() {
    println!("immurok-cli stub");
}
```

- [ ] **Step 6: Create stub module files**

Create empty files so `lib.rs` compiles:
- `crates/immurok-common/src/protocol.rs`
- `crates/immurok-common/src/security.rs`
- `crates/immurok-common/src/socket_proto.rs`
- `crates/immurok-common/src/types.rs`

- [ ] **Step 7: Verify workspace compiles**

Run: `cd app-linux-rs && cargo build`
Expected: Compiles with no errors

- [ ] **Step 8: Commit**

```bash
git add app-linux-rs/
git commit -m "feat(linux): scaffold Rust workspace with 3 crates"
```

---

### Task 1.2: protocol.rs — constants

**Files:**
- Modify: `app-linux-rs/crates/immurok-common/src/protocol.rs`
- Reference: `app-linux/immurok/config.py`, `docs/protocol.md`

- [ ] **Step 1: Write protocol constants**

Port all constants from `config.py` and spec. Every constant listed below:

```rust
// GATT UUIDs (string form for bluer)
pub const SERVICE_UUID_STR: &str = "12340010-0000-1000-8000-00805f9b34fb";
pub const CMD_CHAR_UUID_STR: &str = "12340011-0000-1000-8000-00805f9b34fb";
pub const RSP_CHAR_UUID_STR: &str = "12340012-0000-1000-8000-00805f9b34fb";
pub const OTA_CHAR_UUID_STR: &str = "0000fee1-0000-1000-8000-00805f9b34fb";
pub const FW_REVISION_UUID: u16 = 0x2A26;

pub const DEVICE_NAME_PREFIX: &str = "immurok";

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
pub const RSP_FP_GATE_APPROVED: u8 = 0x10;
pub const RSP_WAIT_FP: u8 = 0x11;
pub const RSP_BUSY: u8 = 0xFD;
pub const RSP_INVALID_PARAM: u8 = 0xFE;
pub const RSP_ERROR: u8 = 0xFF;

// Notification types
pub const NOTIFY_FP_MATCH_SIGNED: u8 = 0x21;
pub const NOTIFY_ENROLL_PROGRESS: u8 = 0x11;
pub const NOTIFY_CONN_PARAM_UPDATE: u8 = 0xF0;

// Key categories
pub const KEY_CAT_SSH: u8 = 0;
pub const KEY_CAT_OTP: u8 = 1;
pub const KEY_CAT_API: u8 = 2;

// Enroll status values
pub const ENROLL_WAITING: u8 = 0x00;
pub const ENROLL_CAPTURED: u8 = 0x01;
pub const ENROLL_PROCESSING: u8 = 0x02;
pub const ENROLL_LIFT_FINGER: u8 = 0x03;
pub const ENROLL_COMPLETE: u8 = 0x04;
pub const ENROLL_FAILED: u8 = 0xFF;

// Packet format
pub const MAX_PACKET_SIZE: usize = 64;
pub const MAX_PAYLOAD_SIZE: usize = 62;

// Timing
pub const BLE_COMMAND_TIMEOUT_SECS: u64 = 5;
pub const BLE_RECONNECT_INTERVAL_SECS: u64 = 1;
pub const BLE_CONNECTING_TIMEOUT_SECS: u64 = 10;
pub const BLE_FP_GATE_TIMEOUT_SECS: u64 = 30;
pub const BLE_AUTH_TIMEOUT_SECS: u64 = 30;
pub const PRE_AUTH_DURATION_SECS: u64 = 10;
pub const FP_GATE_MAX_FAILURES: u8 = 3;
pub const MAX_FINGERPRINT_SLOTS: u8 = 5;

// Security
pub const COMPRESSED_PUBKEY_LEN: usize = 33;
pub const SHARED_KEY_LEN: usize = 32;
pub const HMAC_TRUNCATED_LEN: usize = 8;
pub const HKDF_SALT: &[u8] = b"immurok-pairing-salt";
pub const HKDF_INFO: &[u8] = b"immurok-shared-key";

// OTA
pub const OTA_IMAGE_B_BLOCKS: u32 = 54;
pub const OTA_READ_POLL_INTERVAL_MS: u64 = 200;
pub const OTA_ERASE_TIMEOUT_SECS: u64 = 15;

// Paths
pub const IMMUROK_DIR: &str = ".immurok";
pub const PAIRING_FILE: &str = "pairing.json";
pub const SETTINGS_FILE: &str = "settings.json";
pub const SSH_KEYS_FILE: &str = "ssh_keys.json";
pub const KEY_NAMES_FILE: &str = "key_names.json";
pub const PAM_SOCKET_NAME: &str = "pam.sock";
pub const AGENT_SOCKET_NAME: &str = "agent.sock";
```

- [ ] **Step 2: Verify compiles**

Run: `cargo build -p immurok-common`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git commit -am "feat(linux): add protocol constants to immurok-common"
```

---

### Task 1.3: types.rs — shared types

**Files:**
- Modify: `app-linux-rs/crates/immurok-common/src/types.rs`

- [ ] **Step 1: Write shared types**

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PairingData {
    pub device_uuid: String,
    pub shared_key: [u8; 32],
    pub paired_at: String,
}

#[derive(Debug, Clone, Default)]
pub struct DeviceStatus {
    pub fp_bitmap: u8,
    pub paired: bool,
    pub battery: u8,
    pub fw_version: String,
    pub pending_match: Option<PendingMatch>,
}

#[derive(Debug, Clone)]
pub struct PendingMatch {
    pub page_id: u16,
    pub hmac: [u8; 8],
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(u8)]
pub enum KeyCategory {
    Ssh = 0,
    Otp = 1,
    Api = 2,
}

impl KeyCategory {
    pub fn from_u8(v: u8) -> Option<Self> {
        match v {
            0 => Some(Self::Ssh),
            1 => Some(Self::Otp),
            2 => Some(Self::Api),
            _ => None,
        }
    }

    pub fn from_str(s: &str) -> Option<Self> {
        match s.to_lowercase().as_str() {
            "ssh" => Some(Self::Ssh),
            "otp" => Some(Self::Otp),
            "api" => Some(Self::Api),
            _ => None,
        }
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum EnrollEvent {
    Waiting,
    Captured { current: u8, total: u8 },
    Processing,
    LiftFinger,
    Complete,
    Failed,
}

impl EnrollEvent {
    pub fn from_notification(status: u8, current: u8, total: u8) -> Self {
        match status {
            0x00 => Self::Waiting,
            0x01 => Self::Captured { current, total },
            0x02 => Self::Processing,
            0x03 => Self::LiftFinger,
            0x04 => Self::Complete,
            _ => Self::Failed,
        }
    }
}

/// Helper to check fingerprint bitmap
pub fn fp_bitmap_slots(bitmap: u8) -> Vec<u8> {
    (0..5).filter(|i| bitmap & (1 << i) != 0).collect()
}

/// Helper to format fingerprint bitmap for display
pub fn fp_bitmap_display(bitmap: u8) -> String {
    (0..5)
        .map(|i| if bitmap & (1 << i) != 0 { "[■]" } else { "[ ]" })
        .collect::<Vec<_>>()
        .join(" ")
}
```

- [ ] **Step 2: Verify compiles**

Run: `cargo build -p immurok-common`

- [ ] **Step 3: Commit**

```bash
git commit -am "feat(linux): add shared types to immurok-common"
```

---

### Task 1.4: security.rs — ECDH / HKDF / HMAC

**Files:**
- Modify: `app-linux-rs/crates/immurok-common/src/security.rs`
- Reference: `app-linux/immurok/security.py`, `docs/security.md`

- [ ] **Step 1: Write test for HKDF derivation**

Add `#[cfg(test)]` module at end of `security.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_hkdf_derive_deterministic() {
        let secret = [0xABu8; 32];
        let k1 = hkdf_derive(&secret);
        let k2 = hkdf_derive(&secret);
        assert_eq!(k1, k2);
        assert_ne!(k1, [0u8; 32]);
    }

    #[test]
    fn test_hmac_truncated_length() {
        let key = [0x42u8; 32];
        let data = b"test data";
        let result = hmac_truncated(&key, data);
        assert_eq!(result.len(), 8);
    }

    #[test]
    fn test_verify_fp_match() {
        let key = [0x42u8; 32];
        let page_id: u16 = 0x0001;
        // compute expected
        let mut msg = vec![0x21];
        msg.extend_from_slice(&page_id.to_le_bytes());
        let expected = hmac_truncated(&key, &msg);
        assert!(verify_fp_match(&key, page_id, &expected));
        // wrong hmac
        assert!(!verify_fp_match(&key, page_id, &[0u8; 8]));
    }

    #[test]
    fn test_challenge_roundtrip() {
        let key = [0x42u8; 32];
        let nonce = [1u8, 2, 3, 4, 5, 6, 7, 8];
        let hmac = compute_challenge(&key, &nonce);
        assert!(verify_challenge(&key, &nonce, &hmac));
        assert!(!verify_challenge(&key, &nonce, &[0u8; 8]));
    }

    #[test]
    fn test_pairing_save_load_roundtrip() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("pairing.json");
        let data = PairingData {
            device_uuid: "test-uuid".to_string(),
            shared_key: [0xAB; 32],
            paired_at: "2026-01-01T00:00:00Z".to_string(),
        };
        save_pairing(&path, &data).unwrap();
        let loaded = load_pairing(&path).unwrap();
        assert_eq!(loaded.device_uuid, data.device_uuid);
        assert_eq!(loaded.shared_key, data.shared_key);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test -p immurok-common`
Expected: FAIL (functions not defined)

- [ ] **Step 3: Write security module implementation**

```rust
use hmac::{Hmac, Mac};
use sha2::Sha256;
use hkdf::Hkdf;
use p256::{
    ecdh::EphemeralSecret,
    PublicKey,
    EncodedPoint,
};
use rand::rngs::OsRng;
use std::path::Path;

use crate::protocol::{HKDF_SALT, HKDF_INFO, SHARED_KEY_LEN, HMAC_TRUNCATED_LEN};
use crate::types::PairingData;

type HmacSha256 = Hmac<Sha256>;

// ── ECDH P-256 ──────────────────────────────────────────────

/// Generate an ephemeral P-256 keypair.
/// Returns (secret, compressed_public_key_33B).
pub fn ecdh_generate_keypair() -> (EphemeralSecret, [u8; 33]) {
    let secret = EphemeralSecret::random(&mut OsRng);
    let pubkey = secret.public_key();
    let encoded = EncodedPoint::from(pubkey);
    let compressed = encoded.compress();
    let mut buf = [0u8; 33];
    buf.copy_from_slice(compressed.as_bytes());
    (secret, buf)
}

/// Compute ECDH shared secret from our secret and peer's compressed public key.
/// Note: takes ownership because p256 EphemeralSecret::diffie_hellman consumes self
pub fn ecdh_shared_secret(
    secret: EphemeralSecret,
    peer_compressed: &[u8; 33],
) -> Result<[u8; 32], SecurityError> {
    let point = EncodedPoint::from_bytes(peer_compressed)
        .map_err(|_| SecurityError::InvalidPublicKey)?;
    let peer_pubkey = PublicKey::from_encoded_point(&point)
        .into_option()
        .ok_or(SecurityError::InvalidPublicKey)?;
    let shared = secret.diffie_hellman(&peer_pubkey);
    let mut buf = [0u8; 32];
    buf.copy_from_slice(shared.raw_secret_bytes().as_slice());
    Ok(buf)
}

// ── HKDF-SHA256 ─────────────────────────────────────────────

/// Derive shared_key from ECDH shared secret using HKDF-SHA256.
/// Salt = "immurok-pairing-salt", Info = "immurok-shared-key"
pub fn hkdf_derive(ecdh_secret: &[u8; 32]) -> [u8; 32] {
    let hk = Hkdf::<Sha256>::new(Some(HKDF_SALT), ecdh_secret);
    let mut okm = [0u8; 32];
    hk.expand(HKDF_INFO, &mut okm)
        .expect("HKDF expand should not fail for 32 bytes");
    okm
}

// ── HMAC ────────────────────────────────────────────────────

/// HMAC-SHA256 truncated to 8 bytes.
pub fn hmac_truncated(key: &[u8; 32], data: &[u8]) -> [u8; 8] {
    let mut mac = HmacSha256::new_from_slice(key).expect("HMAC key length valid");
    mac.update(data);
    let result = mac.finalize().into_bytes();
    let mut truncated = [0u8; 8];
    truncated.copy_from_slice(&result[..HMAC_TRUNCATED_LEN]);
    truncated
}

/// Verify FP match notification: HMAC(key, 0x21 || page_id_le)
pub fn verify_fp_match(key: &[u8; 32], page_id: u16, received_hmac: &[u8; 8]) -> bool {
    let mut msg = vec![0x21u8];
    msg.extend_from_slice(&page_id.to_le_bytes());
    let expected = hmac_truncated(key, &msg);
    constant_time_eq(&expected, received_hmac)
}

/// Compute challenge response: HMAC(key, nonce)
pub fn compute_challenge(key: &[u8; 32], nonce: &[u8; 8]) -> [u8; 8] {
    hmac_truncated(key, nonce)
}

/// Verify challenge response
pub fn verify_challenge(key: &[u8; 32], nonce: &[u8; 8], received_hmac: &[u8; 8]) -> bool {
    let expected = compute_challenge(key, nonce);
    constant_time_eq(&expected, received_hmac)
}

/// Constant-time comparison
fn constant_time_eq(a: &[u8; 8], b: &[u8; 8]) -> bool {
    use subtle::ConstantTimeEq;
    a.ct_eq(b).into()
}

// ── Pairing persistence ─────────────────────────────────────

pub fn load_pairing(path: &Path) -> Option<PairingData> {
    let content = std::fs::read_to_string(path).ok()?;
    serde_json::from_str(&content).ok()
}

pub fn save_pairing(path: &Path, data: &PairingData) -> Result<(), SecurityError> {
    if let Some(parent) = path.parent() {
        std::fs::create_dir_all(parent).map_err(SecurityError::Io)?;
    }
    let json = serde_json::to_string_pretty(data).map_err(SecurityError::Serialize)?;
    let tmp = path.with_extension("tmp");
    std::fs::write(&tmp, &json).map_err(SecurityError::Io)?;
    std::fs::rename(&tmp, path).map_err(SecurityError::Io)?;
    // Set permissions 0o600
    #[cfg(unix)]
    {
        use std::os::unix::fs::PermissionsExt;
        std::fs::set_permissions(path, std::fs::Permissions::from_mode(0o600))
            .map_err(SecurityError::Io)?;
    }
    Ok(())
}

pub fn clear_pairing(path: &Path) -> Result<(), SecurityError> {
    if path.exists() {
        std::fs::remove_file(path).map_err(SecurityError::Io)?;
    }
    Ok(())
}

// ── Errors ──────────────────────────────────────────────────

#[derive(Debug, thiserror::Error)]
pub enum SecurityError {
    #[error("invalid public key")]
    InvalidPublicKey,
    #[error("IO error: {0}")]
    Io(std::io::Error),
    #[error("serialization error: {0}")]
    Serialize(serde_json::Error),
}
```

- [ ] **Step 4: Run tests**

Run: `cargo test -p immurok-common`
Expected: All 5 tests PASS

- [ ] **Step 5: Commit**

```bash
git commit -am "feat(linux): add security module with ECDH/HKDF/HMAC"
```

---

### Task 1.5: socket_proto.rs — socket text protocol

**Files:**
- Modify: `app-linux-rs/crates/immurok-common/src/socket_proto.rs`
- Reference: `app-linux/immurok/socket_server.py`, spec socket_proto.rs section

- [ ] **Step 1: Write tests**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_parse_auth() {
        let req = parse_request("AUTH:alice:sudo").unwrap();
        assert!(matches!(req, Request::Auth { ref user, ref service }
            if user == "alice" && service == "sudo"));
    }

    #[test]
    fn test_parse_status() {
        assert!(matches!(parse_request("STATUS").unwrap(), Request::Status));
    }

    #[test]
    fn test_parse_fp_enroll() {
        let req = parse_request("FP:ENROLL:2").unwrap();
        assert!(matches!(req, Request::FpEnroll { slot: 2 }));
    }

    #[test]
    fn test_parse_set_toggle() {
        let req = parse_request("SET:UNLOCK_SUDO:1").unwrap();
        assert!(matches!(req, Request::SetUnlockSudo(true)));
        let req = parse_request("SET:UNLOCK_SUDO:0").unwrap();
        assert!(matches!(req, Request::SetUnlockSudo(false)));
    }

    #[test]
    fn test_serialize_ok() {
        let s = serialize_response(&Response::Ok("done".into()));
        assert_eq!(s, "OK:done");
    }

    #[test]
    fn test_serialize_retry() {
        let s = serialize_response(&Response::Retry { remaining: 2 });
        assert_eq!(s, "RETRY:2");
    }

    #[test]
    fn test_parse_invalid() {
        assert!(parse_request("GARBAGE").is_err());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cargo test -p immurok-common`
Expected: FAIL

- [ ] **Step 3: Write socket_proto implementation**

```rust
use thiserror::Error;

#[derive(Debug, Clone, PartialEq)]
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

#[derive(Debug, Clone, PartialEq)]
pub enum Response {
    Ok(String),
    Deny(String),
    Retry { remaining: u8 },
    Error(String),
    Status {
        connected: bool,
        name: String,
        battery: u8,
        version: String,
    },
}

#[derive(Debug, Error)]
#[error("protocol parse error: {0}")]
pub struct ParseError(pub String);

pub fn parse_request(line: &str) -> Result<Request, ParseError> {
    let line = line.trim();
    let parts: Vec<&str> = line.splitn(4, ':').collect();

    match parts[0] {
        "AUTH" if parts.len() >= 3 => Ok(Request::Auth {
            user: parts[1].to_string(),
            service: parts[2].to_string(),
        }),
        "STATUS" => Ok(Request::Status),
        "FP" if parts.len() >= 2 => parse_fp(&parts[1..]),
        "PAIR" if parts.len() >= 2 => parse_pair(parts[1]),
        "SET" if parts.len() >= 3 => parse_set(parts[1], parts[2]),
        "GET" if parts.len() >= 2 => parse_get(parts[1]),
        "GATE" if parts.len() >= 2 && parts[1] == "CANCEL" => Ok(Request::GateCancel),
        "OTA" if parts.len() >= 2 => parse_ota(&parts[1..]),
        _ => Err(ParseError(format!("unknown command: {}", line))),
    }
}

fn parse_fp(parts: &[&str]) -> Result<Request, ParseError> {
    match parts[0] {
        "LIST" => Ok(Request::FpList),
        "ENROLL" if parts.len() >= 2 => {
            let slot = parts[1].parse().map_err(|_| ParseError("invalid slot".into()))?;
            Ok(Request::FpEnroll { slot })
        }
        "ENROLL_CANCEL" => Ok(Request::FpEnrollCancel),
        "DELETE" if parts.len() >= 2 => {
            let slot = parts[1].parse().map_err(|_| ParseError("invalid slot".into()))?;
            Ok(Request::FpDelete { slot })
        }
        "VERIFY" => Ok(Request::FpVerify),
        "STATUS" => Ok(Request::FpStatus),
        "LAST_MATCH" => Ok(Request::FpLastMatch),
        _ => Err(ParseError(format!("unknown FP command: {}", parts[0]))),
    }
}

fn parse_pair(cmd: &str) -> Result<Request, ParseError> {
    match cmd {
        "STATUS" => Ok(Request::PairStatus),
        "START" => Ok(Request::PairStart),
        "RESET" => Ok(Request::PairReset),
        _ => Err(ParseError(format!("unknown PAIR command: {}", cmd))),
    }
}

fn parse_set(key: &str, value: &str) -> Result<Request, ParseError> {
    let b = match value {
        "1" | "true" | "on" => true,
        "0" | "false" | "off" => false,
        _ => return Err(ParseError(format!("invalid bool: {}", value))),
    };
    match key {
        "UNLOCK_SUDO" => Ok(Request::SetUnlockSudo(b)),
        "UNLOCK_POLKIT" => Ok(Request::SetUnlockPolkit(b)),
        "UNLOCK_SCREEN" => Ok(Request::SetUnlockScreen(b)),
        _ => Err(ParseError(format!("unknown setting: {}", key))),
    }
}

fn parse_get(key: &str) -> Result<Request, ParseError> {
    match key {
        "SETTINGS" => Ok(Request::GetSettings),
        "INFO" => Ok(Request::GetInfo),
        _ => Err(ParseError(format!("unknown GET key: {}", key))),
    }
}

fn parse_ota(parts: &[&str]) -> Result<Request, ParseError> {
    match parts[0] {
        "START" if parts.len() >= 3 => {
            let size = parts[1].parse().map_err(|_| ParseError("invalid size".into()))?;
            Ok(Request::OtaStart { size, version: parts[2].to_string() })
        }
        "DATA" if parts.len() >= 2 => {
            // Hex-encoded binary data
            let bytes = hex::decode(parts[1]).map_err(|_| ParseError("invalid hex".into()))?;
            Ok(Request::OtaData(bytes))
        }
        "FINISH" => Ok(Request::OtaFinish),
        _ => Err(ParseError(format!("unknown OTA command: {}", parts[0]))),
    }
}

pub fn serialize_response(resp: &Response) -> String {
    match resp {
        Response::Ok(msg) => format!("OK:{}", msg),
        Response::Deny(msg) => format!("DENY:{}", msg),
        Response::Retry { remaining } => format!("RETRY:{}", remaining),
        Response::Error(msg) => format!("ERROR:{}", msg),
        Response::Status { connected, name, battery, version } => {
            format!("STATUS:{}:{}:{}:{}", if *connected { "1" } else { "0" }, name, battery, version)
        }
    }
}
```

- [ ] **Step 4: Run tests**

Run: `cargo test -p immurok-common`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git commit -am "feat(linux): add socket text protocol parser"
```

---

## Chunk 2: immurok-daemon core

### Task 2.1: settings.rs — user settings

**Files:**
- Create: `app-linux-rs/crates/immurok-daemon/src/settings.rs`
- Reference: `app-linux/immurok/settings.py`

- [ ] **Step 1: Write settings module**

```rust
use serde::{Deserialize, Serialize};
use std::path::Path;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Settings {
    #[serde(default = "default_true")]
    pub unlock_sudo: bool,
    #[serde(default = "default_true")]
    pub unlock_polkit: bool,
    #[serde(default = "default_true")]
    pub unlock_screen: bool,
}

fn default_true() -> bool { true }

impl Default for Settings {
    fn default() -> Self {
        Self {
            unlock_sudo: true,
            unlock_polkit: true,
            unlock_screen: true,
        }
    }
}

impl Settings {
    pub fn load(path: &Path) -> Self {
        std::fs::read_to_string(path)
            .ok()
            .and_then(|s| serde_json::from_str(&s).ok())
            .unwrap_or_default()
    }

    pub fn save(&self, path: &Path) -> std::io::Result<()> {
        if let Some(parent) = path.parent() {
            std::fs::create_dir_all(parent)?;
        }
        let json = serde_json::to_string_pretty(self)
            .map_err(|e| std::io::Error::new(std::io::ErrorKind::Other, e))?;
        let tmp = path.with_extension("tmp");
        std::fs::write(&tmp, &json)?;
        std::fs::rename(&tmp, path)?;
        Ok(())
    }
}
```

- [ ] **Step 2: Verify compiles**

Run: `cargo build -p immurok-daemon`

- [ ] **Step 3: Commit**

```bash
git commit -am "feat(linux): add settings module"
```

---

### Task 2.2: coordinator.rs — event routing hub

**Files:**
- Create: `app-linux-rs/crates/immurok-daemon/src/coordinator.rs`

- [ ] **Step 1: Write coordinator with shared state and channels**

```rust
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;
use std::time::{Duration, Instant};
use tokio::sync::{broadcast, mpsc, RwLock, Notify};
use tracing::{info, warn};

use immurok_common::types::{DeviceStatus, EnrollEvent, PairingData};
use crate::settings::Settings;

/// Commands sent to ble.rs via channel
#[derive(Debug)]
pub enum BleCommand {
    SendCommand { cmd: u8, payload: Vec<u8>, reply: tokio::sync::oneshot::Sender<BleResult> },
    SendFpGated { cmd: u8, payload: Vec<u8>, reply: tokio::sync::oneshot::Sender<BleResult> },
    Pair { reply: tokio::sync::oneshot::Sender<BleResult> },
    Disconnect,
}

pub type BleResult = Result<(u8, Vec<u8>), String>;

#[derive(Debug, Clone)]
pub struct FpMatchEvent {
    pub page_id: u16,
}

pub struct Coordinator {
    // Shared state
    pub pairing: RwLock<Option<PairingData>>,
    pub settings: RwLock<Settings>,
    pub device_status: RwLock<Option<DeviceStatus>>,
    pub is_device_verified: AtomicBool,
    pub is_connected: AtomicBool,
    pub screen_locked: AtomicBool,

    // Cross-module channels
    pub ble_cmd_tx: mpsc::Sender<BleCommand>,
    pub fp_match_tx: broadcast::Sender<FpMatchEvent>,
    pub enroll_tx: broadcast::Sender<EnrollEvent>,

    // PAM pending auth
    pending_pam: RwLock<Option<tokio::sync::oneshot::Sender<bool>>>,
    pre_auth_deadline: RwLock<Option<Instant>>,

    // Notify for auth-dialog kill
    pub auth_dialog_cancel: Notify,

    // Paths
    pub immurok_dir: std::path::PathBuf,
}

impl Coordinator {
    pub fn new(
        ble_cmd_tx: mpsc::Sender<BleCommand>,
        immurok_dir: std::path::PathBuf,
    ) -> Arc<Self> {
        let (fp_match_tx, _) = broadcast::channel(16);
        let (enroll_tx, _) = broadcast::channel(16);

        Arc::new(Self {
            pairing: RwLock::new(None),
            settings: RwLock::new(Settings::default()),
            device_status: RwLock::new(None),
            is_device_verified: AtomicBool::new(false),
            is_connected: AtomicBool::new(false),
            screen_locked: AtomicBool::new(false),
            ble_cmd_tx,
            fp_match_tx,
            enroll_tx,
            pending_pam: RwLock::new(None),
            pre_auth_deadline: RwLock::new(None),
            auth_dialog_cancel: Notify::new(),
            immurok_dir,
        })
    }

    /// Core FP match routing logic
    pub async fn on_fp_match(&self, page_id: u16) {
        // Broadcast to all subscribers (TUI, socket status)
        let _ = self.fp_match_tx.send(FpMatchEvent { page_id });

        // 1. Pending PAM request? → approve
        if self.approve_pending_pam().await {
            info!("FP match → approved pending PAM request");
            return;
        }

        // 2. Screen locked + unlock enabled? → loginctl unlock-session
        let settings = self.settings.read().await;
        if self.screen_locked.load(Ordering::Relaxed) && settings.unlock_screen {
            drop(settings);
            info!("FP match → unlocking screen");
            self.unlock_screen().await;
            return;
        }

        // 3. Otherwise → set pre-auth
        drop(settings);
        info!("FP match → setting pre-auth window");
        self.set_pre_auth(Duration::from_secs(
            immurok_common::protocol::PRE_AUTH_DURATION_SECS,
        )).await;
    }

    pub async fn set_pre_auth(&self, duration: Duration) {
        let mut deadline = self.pre_auth_deadline.write().await;
        *deadline = Some(Instant::now() + duration);
    }

    pub async fn consume_pre_auth(&self) -> bool {
        let mut deadline = self.pre_auth_deadline.write().await;
        if let Some(dl) = *deadline {
            if Instant::now() < dl {
                *deadline = None;
                return true;
            }
            *deadline = None;
        }
        false
    }

    pub async fn set_pending_pam(&self, sender: tokio::sync::oneshot::Sender<bool>) {
        let mut pending = self.pending_pam.write().await;
        *pending = Some(sender);
    }

    async fn approve_pending_pam(&self) -> bool {
        let mut pending = self.pending_pam.write().await;
        if let Some(sender) = pending.take() {
            let _ = sender.send(true);
            return true;
        }
        false
    }

    pub async fn deny_pending_pam(&self) {
        let mut pending = self.pending_pam.write().await;
        if let Some(sender) = pending.take() {
            let _ = sender.send(false);
        }
    }

    async fn unlock_screen(&self) {
        let result = tokio::process::Command::new("loginctl")
            .arg("unlock-session")
            .output()
            .await;
        match result {
            Ok(output) if output.status.success() => {
                info!("Screen unlocked via loginctl");
                // Set pre-auth for PAM request that follows
                self.set_pre_auth(Duration::from_secs(
                    immurok_common::protocol::PRE_AUTH_DURATION_SECS,
                )).await;
            }
            Ok(output) => warn!("loginctl unlock-session failed: {:?}", output.status),
            Err(e) => warn!("Failed to run loginctl: {}", e),
        }
    }

    /// Send a BLE command via the channel
    pub async fn ble_send(&self, cmd: u8, payload: Vec<u8>) -> BleResult {
        let (tx, rx) = tokio::sync::oneshot::channel();
        self.ble_cmd_tx
            .send(BleCommand::SendCommand { cmd, payload, reply: tx })
            .await
            .map_err(|_| "BLE channel closed".to_string())?;
        rx.await.map_err(|_| "BLE reply dropped".to_string())?
    }

    /// Send an FP-gated BLE command
    pub async fn ble_send_fp_gated(&self, cmd: u8, payload: Vec<u8>) -> BleResult {
        let (tx, rx) = tokio::sync::oneshot::channel();
        self.ble_cmd_tx
            .send(BleCommand::SendFpGated { cmd, payload, reply: tx })
            .await
            .map_err(|_| "BLE channel closed".to_string())?;
        rx.await.map_err(|_| "BLE reply dropped".to_string())?
    }

    pub fn settings_path(&self) -> std::path::PathBuf {
        self.immurok_dir.join(immurok_common::protocol::SETTINGS_FILE)
    }

    pub fn pairing_path(&self) -> std::path::PathBuf {
        self.immurok_dir.join(immurok_common::protocol::PAIRING_FILE)
    }
}
```

- [ ] **Step 2: Verify compiles**

Run: `cargo build -p immurok-daemon`

- [ ] **Step 3: Commit**

```bash
git commit -am "feat(linux): add coordinator event routing hub"
```

---

### Task 2.3: ble.rs — BLE GATT client (structure + stubs)

**Files:**
- Create: `app-linux-rs/crates/immurok-daemon/src/ble.rs`
- Reference: `app-linux/immurok/ble.py`, `docs/protocol.md`

This is the largest module. Implement the core structure with connection/reconnect loop, notification routing, and command sending. Device-specific flows (pair, auth, FP-gate) are methods on the struct.

- [ ] **Step 1: Write BLE module structure**

Key functions to implement (see existing `ble.py` for detailed logic):

```rust
use std::sync::Arc;
use bluer::{gatt::remote::Characteristic, AdapterEvent, Device, Session};
use tokio::sync::mpsc;
use tracing::{info, warn, error};

use immurok_common::protocol::*;
use immurok_common::security;
use crate::coordinator::{BleCommand, BleResult, Coordinator, FpMatchEvent};

/// Main BLE run loop: scan → connect → serve commands → reconnect
pub async fn run(coordinator: Arc<Coordinator>, mut cmd_rx: mpsc::Receiver<BleCommand>) {
    loop {
        match connect_and_serve(&coordinator, &mut cmd_rx).await {
            Ok(()) => info!("BLE session ended normally"),
            Err(e) => warn!("BLE session error: {}", e),
        }
        coordinator.is_connected.store(false, std::sync::atomic::Ordering::Relaxed);
        coordinator.is_device_verified.store(false, std::sync::atomic::Ordering::Relaxed);
        tokio::time::sleep(std::time::Duration::from_secs(BLE_RECONNECT_INTERVAL_SECS)).await;
    }
}

async fn connect_and_serve(
    coordinator: &Arc<Coordinator>,
    cmd_rx: &mut mpsc::Receiver<BleCommand>,
) -> Result<(), Box<dyn std::error::Error>> {
    // 1. Get BlueZ session + adapter
    // 2. Scan for connected "immurok" devices
    // 3. Find GATT service + CMD/RSP characteristics
    // 4. Subscribe to RSP notifications
    // 5. Challenge-Response verification
    // 6. GET_STATUS (check pending match)
    // 7. Sync key cache
    // 8. Enter command/notification serve loop
    todo!("Implement BLE connection logic — port from ble.py")
}

// Internal helpers (port from ble.py):
// - send_command(cmd_char, rsp_stream, cmd, payload) → (status, data)
// - handle_notification(data, coordinator) — route by first byte
// - challenge_verify(coordinator, cmd_char, rsp_stream)
// - pair_flow(coordinator, cmd_char, rsp_stream)
// - fp_gate_flow(coordinator, cmd_char, rsp_stream, cmd, payload)
```

The full BLE implementation is complex (~800 lines in Python). The structure above defines the interface; implementation will be filled during execution following the patterns in `ble.py`.

- [ ] **Step 2: Verify compiles (with todo!())**

Run: `cargo build -p immurok-daemon`

- [ ] **Step 3: Commit**

```bash
git commit -am "feat(linux): add BLE module structure with stubs"
```

---

### Task 2.4: main.rs — daemon entry point

**Files:**
- Modify: `app-linux-rs/crates/immurok-daemon/src/main.rs`

- [ ] **Step 1: Write daemon main**

```rust
mod ble;
mod coordinator;
mod keystore;
mod ota;
mod screen;
mod settings;
mod socket;
mod ssh_agent;

use std::path::PathBuf;
use tokio::sync::mpsc;
use tracing::info;

use immurok_common::protocol;
use immurok_common::security;

#[tokio::main]
async fn main() {
    // Initialize tracing
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::from_default_env()
                .add_directive("immurok=info".parse().unwrap()),
        )
        .with_target(false)
        .init();

    info!("immurok-daemon starting");

    // Resolve paths
    let home = std::env::var("HOME").expect("HOME not set");
    let immurok_dir = PathBuf::from(&home).join(protocol::IMMUROK_DIR);
    std::fs::create_dir_all(&immurok_dir).expect("cannot create ~/.immurok");

    // Load pairing + settings
    let pairing = security::load_pairing(&immurok_dir.join(protocol::PAIRING_FILE));
    let user_settings = settings::Settings::load(&immurok_dir.join(protocol::SETTINGS_FILE));

    // Create coordinator
    let (ble_cmd_tx, ble_cmd_rx) = mpsc::channel(32);
    let coord = coordinator::Coordinator::new(ble_cmd_tx, immurok_dir.clone());
    {
        let mut p = coord.pairing.write().await;
        *p = pairing;
        let mut s = coord.settings.write().await;
        *s = user_settings;
    }

    let pam_sock = immurok_dir.join(protocol::PAM_SOCKET_NAME);
    let agent_sock = immurok_dir.join(protocol::AGENT_SOCKET_NAME);

    // Concurrent launch
    tokio::select! {
        _ = ble::run(coord.clone(), ble_cmd_rx) => {},
        _ = socket::serve(coord.clone(), &pam_sock) => {},
        _ = ssh_agent::serve(coord.clone(), &agent_sock) => {},
        _ = screen::monitor(coord.clone()) => {},
        _ = tokio::signal::ctrl_c() => {
            info!("Received shutdown signal");
        },
    }

    // Cleanup
    let _ = std::fs::remove_file(&pam_sock);
    let _ = std::fs::remove_file(&agent_sock);
    info!("immurok-daemon stopped");
}
```

- [ ] **Step 2: Create stub modules** so main.rs compiles

Each stub must match the exact signature used in main.rs:

```rust
// socket.rs
use std::path::Path;
use std::sync::Arc;
use crate::coordinator::Coordinator;
pub async fn serve(_coordinator: Arc<Coordinator>, _socket_path: &Path) {
    std::future::pending::<()>().await;
}

// ssh_agent.rs
use std::path::Path;
use std::sync::Arc;
use crate::coordinator::Coordinator;
pub async fn serve(_coordinator: Arc<Coordinator>, _socket_path: &Path) {
    std::future::pending::<()>().await;
}

// screen.rs
use std::sync::Arc;
use crate::coordinator::Coordinator;
pub async fn monitor(_coordinator: Arc<Coordinator>) {
    std::future::pending::<()>().await;
}

// keystore.rs
use std::sync::Arc;
use crate::coordinator::Coordinator;
pub async fn sync_keys(_coordinator: &Arc<Coordinator>) -> Result<(), String> { Ok(()) }

// ota.rs
use std::sync::Arc;
use tokio::net::UnixStream;
use crate::coordinator::Coordinator;
pub async fn handle_ota_session(
    _stream: &mut UnixStream,
    _coord: &Arc<Coordinator>,
    _fw_data: &[u8],
    _version: &str,
) -> Result<(), String> { Err("not implemented".into()) }
```

Note: stubs use `std::future::pending()` instead of `todo!()` so the daemon doesn't panic on startup.

- [ ] **Step 3: Verify compiles**

Run: `cargo build -p immurok-daemon`

- [ ] **Step 4: Commit**

```bash
git commit -am "feat(linux): add daemon entry point with module stubs"
```

---

## Chunk 3: immurok-daemon services

### Task 3.1: socket.rs — Unix Socket server

**Files:**
- Modify: `app-linux-rs/crates/immurok-daemon/src/socket.rs`
- Reference: `app-linux/immurok/socket_server.py`

- [ ] **Step 1: Write socket server**

Key structure (port from `socket_server.py`, ~800 lines):

```rust
use std::path::Path;
use std::sync::Arc;
use tokio::net::{UnixListener, UnixStream};
use tracing::{info, warn, error};

use immurok_common::protocol::*;
use immurok_common::socket_proto::{self, Request, Response};
use crate::coordinator::Coordinator;

pub async fn serve(coordinator: Arc<Coordinator>, socket_path: &Path) {
    // Remove stale socket
    let _ = std::fs::remove_file(socket_path);

    let listener = UnixListener::bind(socket_path).expect("cannot bind socket");

    // chmod 0o666 for PAM root access
    #[cfg(unix)]
    {
        use std::os::unix::fs::PermissionsExt;
        std::fs::set_permissions(socket_path, std::fs::Permissions::from_mode(0o666)).ok();
    }

    info!("Socket server listening on {:?}", socket_path);

    loop {
        match listener.accept().await {
            Ok((stream, _)) => {
                let coord = coordinator.clone();
                tokio::spawn(async move {
                    if let Err(e) = handle_client(stream, coord).await {
                        warn!("Client error: {}", e);
                    }
                });
            }
            Err(e) => error!("Accept error: {}", e),
        }
    }
}

async fn handle_client(
    mut stream: UnixStream,
    coord: Arc<Coordinator>,
) -> Result<(), Box<dyn std::error::Error>> {
    // 1. Verify peer credentials via SO_PEERCRED (libc::ucred)
    verify_peer_credentials(&stream)?;

    // 2. Read request line
    let mut buf = vec![0u8; 4096];
    let n = stream.read(&mut buf).await?;
    let line = std::str::from_utf8(&buf[..n])?.trim().to_string();

    // 3. Parse and dispatch
    let request = socket_proto::parse_request(&line)?;
    let response = match request {
        Request::Auth { user, service } => handle_auth(&coord, &user, &service).await,
        Request::Status => handle_status(&coord).await,
        Request::FpList => handle_fp_list(&coord).await,
        Request::FpEnroll { slot } => handle_fp_enroll(&coord, &mut stream, slot).await,
        Request::FpEnrollCancel => handle_fp_cancel(&coord).await,
        Request::FpDelete { slot } => handle_fp_delete(&coord, slot).await,
        Request::PairStart => handle_pair_start(&coord).await,
        Request::PairStatus => handle_pair_status(&coord).await,
        Request::PairReset => handle_pair_reset(&coord).await,
        Request::SetUnlockSudo(v) => handle_set_toggle(&coord, "sudo", v).await,
        Request::SetUnlockPolkit(v) => handle_set_toggle(&coord, "polkit", v).await,
        Request::SetUnlockScreen(v) => handle_set_toggle(&coord, "screen", v).await,
        Request::GetSettings => handle_get_settings(&coord).await,
        Request::GetInfo => handle_get_info(&coord).await,
        Request::OtaStart { size, version } => {
            return crate::ota::handle_ota_session(&mut stream, &coord, &[], &version).await
                .map_err(|e| e.into());
        }
        _ => Response::Error("not implemented".into()),
    };

    // 4. Write response
    let resp_str = socket_proto::serialize_response(&response);
    stream.write_all(resp_str.as_bytes()).await?;
    Ok(())
}

fn verify_peer_credentials(stream: &UnixStream) -> Result<(), Box<dyn std::error::Error>> {
    // Use libc::getsockopt with SO_PEERCRED to get ucred
    // Accept only root (uid 0) or current user (getuid())
    // Reject all others
    todo!("Implement SO_PEERCRED check")
}

async fn handle_auth(coord: &Arc<Coordinator>, user: &str, service: &str) -> Response {
    // 1. Check pre-auth window → instant OK
    if coord.consume_pre_auth().await {
        return Response::Ok("pre-authorized".into());
    }
    // 2. Check device verified
    if !coord.is_device_verified.load(std::sync::atomic::Ordering::Relaxed) {
        return Response::Deny("device not verified".into());
    }
    // 3. Check settings toggle (sudo/polkit)
    // 4. Spawn immurok-auth-dialog process
    // 5. Send AUTH_REQUEST (0x33) via BLE → WAIT_FP
    // 6. Wait up to 30s with 3 retry attempts:
    //    - FP match → OK
    //    - FP not match (attempts < 3) → RETRY:remaining
    //    - FP not match (attempts >= 3) → DENY
    //    - Timeout → DENY
    //    - Dialog closed (process exit) → DENY
    // 7. Kill auth-dialog process
    todo!("Implement AUTH flow — port from socket_server.py handle_auth()")
}

// Each handler: simple BLE command → format response. Port from socket_server.py.
async fn handle_status(coord: &Arc<Coordinator>) -> Response { todo!() }
async fn handle_fp_list(coord: &Arc<Coordinator>) -> Response { todo!() }
async fn handle_fp_enroll(coord: &Arc<Coordinator>, stream: &mut UnixStream, slot: u8) -> Response { todo!() }
async fn handle_fp_cancel(coord: &Arc<Coordinator>) -> Response { todo!() }
async fn handle_fp_delete(coord: &Arc<Coordinator>, slot: u8) -> Response { todo!() }
async fn handle_pair_start(coord: &Arc<Coordinator>) -> Response { todo!() }
async fn handle_pair_status(coord: &Arc<Coordinator>) -> Response { todo!() }
async fn handle_pair_reset(coord: &Arc<Coordinator>) -> Response { todo!() }
async fn handle_set_toggle(coord: &Arc<Coordinator>, key: &str, value: bool) -> Response { todo!() }
async fn handle_get_settings(coord: &Arc<Coordinator>) -> Response { todo!() }
async fn handle_get_info(coord: &Arc<Coordinator>) -> Response { todo!() }
```

Implementation follows the same request dispatch pattern as `socket_server.py`. Key differences:
- Use `tokio::net::UnixStream` instead of asyncio
- Use `libc::ucred` + `getsockopt(SO_PEERCRED)` for peer credential verification
- AUTH handler: spawn `immurok-auth-dialog` process, wait on FP match channel with 30s timeout, 3 retries

- [ ] **Step 2: Verify compiles**

- [ ] **Step 3: Commit**

```bash
git commit -am "feat(linux): add socket server structure"
```

---

### Task 3.2: screen.rs — D-Bus screen lock monitor

**Files:**
- Modify: `app-linux-rs/crates/immurok-daemon/src/screen.rs`
- Reference: `app-linux/immurok/screen.py`

- [ ] **Step 1: Write screen monitor**

```rust
use std::sync::Arc;
use zbus::Connection;
use tracing::{info, warn};
use crate::coordinator::Coordinator;

pub async fn monitor(coordinator: Arc<Coordinator>) {
    let conn = match Connection::session().await {
        Ok(c) => c,
        Err(e) => {
            warn!("Cannot connect to session D-Bus: {}. Screen unlock disabled.", e);
            // Graceful degradation: just sleep forever
            std::future::pending::<()>().await;
            return;
        }
    };

    // Try GNOME ScreenSaver first, then freedesktop (KDE)
    // Subscribe to ActiveChanged signal
    // Update coordinator.screen_locked on each signal

    // Use zbus::proxy or MessageStream to listen for signals on:
    // org.gnome.ScreenSaver /org/gnome/ScreenSaver → ActiveChanged(bool)
    // org.freedesktop.ScreenSaver /ScreenSaver → ActiveChanged(bool)

    todo!("Implement D-Bus signal listener — port from screen.py")
}
```

- [ ] **Step 2: Commit**

```bash
git commit -am "feat(linux): add screen monitor structure"
```

---

### Task 3.3: keystore.rs — device key cache

**Files:**
- Modify: `app-linux-rs/crates/immurok-daemon/src/keystore.rs`
- Reference: spec keystore.rs section, macOS `SSHKeyCache.swift`

- [ ] **Step 1: Write keystore module**

```rust
use std::path::Path;
use std::sync::Arc;
use serde::{Deserialize, Serialize};
use tracing::info;

use immurok_common::protocol::*;
use crate::coordinator::Coordinator;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SshKeyEntry {
    pub index: u8,
    pub name: String,
    pub public_key_blob: Vec<u8>,  // SSH wire format (uncompressed P-256)
    pub fingerprint: String,        // SHA256:...
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct KeyNameEntry {
    pub index: u8,
    pub name: String,
}

/// Sync all keys from device after connection.
/// Reads KEY_COUNT → KEY_READ chain for each category.
pub async fn sync_keys(coordinator: &Arc<Coordinator>) -> Result<(), String> {
    // SSH keys: read count → read each → compute SHA256 fingerprint → save
    // OTP/API: read count → read names only → save

    let ssh_path = coordinator.immurok_dir.join(SSH_KEYS_FILE);
    let names_path = coordinator.immurok_dir.join(KEY_NAMES_FILE);

    // SSH
    let ssh_count = read_key_count(coordinator, KEY_CAT_SSH).await?;
    let mut ssh_keys = Vec::new();
    for i in 0..ssh_count {
        if let Ok(entry) = read_ssh_key(coordinator, i).await {
            ssh_keys.push(entry);
        }
    }
    save_json(&ssh_path, &ssh_keys);

    // OTP + API names
    let mut names = Vec::new();
    for cat in [KEY_CAT_OTP, KEY_CAT_API] {
        let count = read_key_count(coordinator, cat).await.unwrap_or(0);
        for i in 0..count {
            if let Ok(name) = read_key_name(coordinator, cat, i).await {
                names.push(KeyNameEntry { index: i, name });
            }
        }
    }
    save_json(&names_path, &names);

    info!("Key cache synced: {} SSH, {} names", ssh_keys.len(), names.len());
    Ok(())
}

async fn read_key_count(coord: &Arc<Coordinator>, category: u8) -> Result<u8, String> {
    let (status, data) = coord.ble_send(CMD_KEY_COUNT, vec![category]).await?;
    if status == RSP_OK && !data.is_empty() {
        Ok(data[0])
    } else {
        Err(format!("KEY_COUNT failed: 0x{:02x}", status))
    }
}

async fn read_ssh_key(coord: &Arc<Coordinator>, index: u8) -> Result<SshKeyEntry, String> {
    // Multi-chunk read via KEY_READ with offset
    // Parse name (16B) + pubkey (64B, uncompressed P-256 x,y)
    // Compute SHA256 fingerprint of SSH wire-format blob
    todo!("Implement chunked KEY_READ for SSH keys")
}

async fn read_key_name(coord: &Arc<Coordinator>, category: u8, index: u8) -> Result<String, String> {
    // Read first chunk (offset 0) to get name field
    let (status, data) = coord.ble_send(CMD_KEY_READ, vec![category, index, 0]).await?;
    if status != RSP_OK || data.len() < 3 {
        return Err("KEY_READ failed".into());
    }
    // data[0] = total_size, data[1] = offset, data[2..] = key data
    // Name is at the start of key data, null-terminated
    let key_data = &data[2..];
    let name_end = key_data.iter().position(|&b| b == 0).unwrap_or(key_data.len().min(32));
    Ok(String::from_utf8_lossy(&key_data[..name_end]).to_string())
}

fn save_json<T: serde::Serialize>(path: &Path, data: &T) {
    if let Ok(json) = serde_json::to_string_pretty(data) {
        let _ = std::fs::write(path, json);
    }
}

/// Load SSH keys from cache (used by ssh_agent)
pub fn load_ssh_keys(path: &Path) -> Vec<SshKeyEntry> {
    std::fs::read_to_string(path)
        .ok()
        .and_then(|s| serde_json::from_str(&s).ok())
        .unwrap_or_default()
}
```

- [ ] **Step 2: Commit**

```bash
git commit -am "feat(linux): add keystore cache module"
```

---

### Task 3.4: ssh_agent.rs — OpenSSH Agent Protocol

**Files:**
- Modify: `app-linux-rs/crates/immurok-daemon/src/ssh_agent.rs`
- Reference: macOS `SSHAgentServer.swift`, OpenSSH agent protocol RFC

- [ ] **Step 1: Write SSH Agent server**

```rust
use std::path::Path;
use std::sync::Arc;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::{UnixListener, UnixStream};
use tracing::{info, warn};

use immurok_common::protocol::*;
use crate::coordinator::Coordinator;
use crate::keystore;

// SSH Agent Protocol message types
const SSH_AGENTC_REQUEST_IDENTITIES: u8 = 11;
const SSH_AGENTC_SIGN_REQUEST: u8 = 13;
const SSH_AGENT_IDENTITIES_ANSWER: u8 = 12;
const SSH_AGENT_SIGN_RESPONSE: u8 = 14;
const SSH_AGENT_FAILURE: u8 = 5;

pub async fn serve(coordinator: Arc<Coordinator>, socket_path: &Path) {
    let _ = std::fs::remove_file(socket_path);
    let listener = UnixListener::bind(socket_path).expect("cannot bind agent socket");

    #[cfg(unix)]
    {
        use std::os::unix::fs::PermissionsExt;
        std::fs::set_permissions(socket_path, std::fs::Permissions::from_mode(0o600)).ok();
    }

    info!("SSH Agent listening on {:?}", socket_path);

    loop {
        match listener.accept().await {
            Ok((stream, _)) => {
                let coord = coordinator.clone();
                tokio::spawn(handle_agent(stream, coord));
            }
            Err(e) => warn!("Agent accept error: {}", e),
        }
    }
}

async fn handle_agent(mut stream: UnixStream, coord: Arc<Coordinator>) {
    loop {
        // Read message: [length:4B BE][type:1B][payload]
        let mut len_buf = [0u8; 4];
        if stream.read_exact(&mut len_buf).await.is_err() { return; }
        let msg_len = u32::from_be_bytes(len_buf) as usize;
        if msg_len == 0 || msg_len > 256 * 1024 { return; }

        let mut msg = vec![0u8; msg_len];
        if stream.read_exact(&mut msg).await.is_err() { return; }

        let msg_type = msg[0];
        let payload = &msg[1..];

        let response = match msg_type {
            SSH_AGENTC_REQUEST_IDENTITIES => handle_identities(&coord).await,
            SSH_AGENTC_SIGN_REQUEST => handle_sign(payload, &coord).await,
            _ => vec![SSH_AGENT_FAILURE],
        };

        // Write response: [length:4B BE][response]
        let resp_len = (response.len() as u32).to_be_bytes();
        if stream.write_all(&resp_len).await.is_err() { return; }
        if stream.write_all(&response).await.is_err() { return; }
    }
}

async fn handle_identities(coord: &Arc<Coordinator>) -> Vec<u8> {
    let keys_path = coord.immurok_dir.join(SSH_KEYS_FILE);
    let keys = keystore::load_ssh_keys(&keys_path);

    // Build IDENTITIES_ANSWER: [type:1B][nkeys:4B BE][for each: blob_len:4B + blob + comment_len:4B + comment]
    let mut resp = vec![SSH_AGENT_IDENTITIES_ANSWER];
    resp.extend_from_slice(&(keys.len() as u32).to_be_bytes());
    for key in &keys {
        // Key blob
        resp.extend_from_slice(&(key.public_key_blob.len() as u32).to_be_bytes());
        resp.extend_from_slice(&key.public_key_blob);
        // Comment (key name)
        let comment = key.name.as_bytes();
        resp.extend_from_slice(&(comment.len() as u32).to_be_bytes());
        resp.extend_from_slice(comment);
    }
    resp
}

async fn handle_sign(payload: &[u8], coord: &Arc<Coordinator>) -> Vec<u8> {
    // Parse: key_blob_len:4B + key_blob + data_len:4B + data + flags:4B
    // Find matching key index
    // BLE KEY_SIGN (FP-gated) → KEY_RESULT → build ECDSA signature
    // Return SSH_AGENT_SIGN_RESPONSE or SSH_AGENT_FAILURE

    if !coord.is_device_verified.load(std::sync::atomic::Ordering::Relaxed) {
        return vec![SSH_AGENT_FAILURE];
    }

    todo!("Implement SSH signing — port from SSHAgentServer.swift")
}
```

- [ ] **Step 2: Commit**

```bash
git commit -am "feat(linux): add SSH Agent server structure"
```

---

### Task 3.5: ota.rs — OTA session management

**Files:**
- Modify: `app-linux-rs/crates/immurok-daemon/src/ota.rs`
- Reference: existing Python OTA code, `ota/ota-update.py`

- [ ] **Step 1: Write OTA module structure**

```rust
use std::sync::Arc;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::UnixStream;
use tracing::{info, warn};

use immurok_common::protocol::*;
use crate::coordinator::Coordinator;

/// Handle an OTA session over a socket long connection.
/// Reports progress via newline-separated lines: "PROGRESS:45.2"
pub async fn handle_ota_session(
    stream: &mut UnixStream,
    coord: &Arc<Coordinator>,
    fw_data: &[u8],
    version: &str,
) -> Result<(), String> {
    // 1. Parse .imfw header, verify signature
    // 2. Write IAP erase command via OTA characteristic
    // 3. Chunked data transfer with progress
    // 4. Verify
    // 5. Switch boot image
    // 6. Device restart

    info!("OTA session starting: {} bytes, version {}", fw_data.len(), version);

    // Port the OTA write logic from ota-update.py:
    // - Find OTA characteristic (0xFEE1)
    // - Send IAP commands as BLE writes
    // - Poll for completion after each block

    todo!("Implement OTA transfer — port from ota-update.py")
}
```

- [ ] **Step 2: Commit**

```bash
git commit -am "feat(linux): add OTA module structure"
```

---

## Chunk 4: immurok-cli

### Task 4.1: socket_client.rs — daemon client

**Files:**
- Create: `app-linux-rs/crates/immurok-cli/src/socket_client.rs`

- [ ] **Step 1: Write socket client**

```rust
use std::io::{BufRead, BufReader, Write};
use std::os::unix::net::UnixStream;
use std::path::Path;

use immurok_common::protocol;
use immurok_common::socket_proto::{self, Request, Response};

pub struct DaemonClient {
    stream: UnixStream,
    reader: BufReader<UnixStream>,
}

impl DaemonClient {
    pub fn connect() -> Result<Self, String> {
        let home = std::env::var("HOME").map_err(|_| "HOME not set".to_string())?;
        let path = format!("{}/{}/{}", home, protocol::IMMUROK_DIR, protocol::PAM_SOCKET_NAME);
        Self::connect_path(Path::new(&path))
    }

    pub fn connect_path(path: &Path) -> Result<Self, String> {
        let stream = UnixStream::connect(path)
            .map_err(|e| format!("Cannot connect to daemon at {:?}: {}\nIs immurok-daemon running?", path, e))?;
        stream.set_read_timeout(Some(std::time::Duration::from_secs(60))).ok();
        let reader = BufReader::new(stream.try_clone().map_err(|e| e.to_string())?);
        Ok(Self { stream, reader })
    }

    pub fn send(&mut self, request: &str) -> Result<String, String> {
        self.stream.write_all(request.as_bytes()).map_err(|e| e.to_string())?;
        let mut response = String::new();
        self.reader.read_line(&mut response).map_err(|e| e.to_string())?;
        Ok(response.trim().to_string())
    }

    /// For OTA: send data, read progress lines until completion
    pub fn send_ota(&mut self, request: &str, on_progress: impl Fn(&str)) -> Result<String, String> {
        self.stream.write_all(request.as_bytes()).map_err(|e| e.to_string())?;
        loop {
            let mut line = String::new();
            self.reader.read_line(&mut line).map_err(|e| e.to_string())?;
            let line = line.trim();
            if line.starts_with("OK:") || line.starts_with("ERROR:") {
                return Ok(line.to_string());
            }
            on_progress(line);
        }
    }
}
```

- [ ] **Step 2: Commit**

```bash
git commit -am "feat(linux): add CLI socket client"
```

---

### Task 4.2: CLI subcommands

**Files:**
- Modify: `app-linux-rs/crates/immurok-cli/src/main.rs`
- Create: `app-linux-rs/crates/immurok-cli/src/commands/mod.rs`
- Create: `app-linux-rs/crates/immurok-cli/src/commands/status.rs`
- Create: `app-linux-rs/crates/immurok-cli/src/commands/pair.rs`
- Create: `app-linux-rs/crates/immurok-cli/src/commands/fingerprint.rs`
- Create: `app-linux-rs/crates/immurok-cli/src/commands/keys.rs`
- Create: `app-linux-rs/crates/immurok-cli/src/commands/settings.rs`
- Create: `app-linux-rs/crates/immurok-cli/src/commands/ota.rs`
- Create: `app-linux-rs/crates/immurok-cli/src/commands/info.rs`

- [ ] **Step 1: Write main.rs with clap**

```rust
mod commands;
mod socket_client;

use clap::{Parser, Subcommand};
use commands::{FpCommands, KeyCommands, SetCommands, PamCommands};

#[derive(Parser)]
#[command(name = "immurok-cli", about = "immurok device management")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Connection status, pairing, battery, firmware version
    Status,
    /// Detailed device information
    Info,
    /// Start ECDH pairing
    Pair,
    /// Clear pairing + factory reset
    Unpair,
    /// Fingerprint operations
    #[command(subcommand)]
    Fp(FpCommands),
    /// Key operations
    #[command(subcommand)]
    Key(KeyCommands),
    /// Change settings
    #[command(subcommand)]
    Set(SetCommands),
    /// Show all settings
    Settings,
    /// OTA firmware upgrade
    Ota { path: String },
    /// Manage PAM configuration
    #[command(subcommand)]
    Pam(PamCommands),
    /// View daemon logs
    Logs,
    /// Interactive TUI panel
    Tui,
}

// FpCommands, KeyCommands, SetCommands, PamCommands are defined in commands/mod.rs
// and imported via `use commands::{...}` above

fn main() {
    let cli = Cli::parse();
    let result = match cli.command {
        Commands::Status => commands::status::run(),
        Commands::Info => commands::info::run(),
        Commands::Pair => commands::pair::run(),
        Commands::Unpair => commands::pair::run_unpair(),
        Commands::Fp(cmd) => commands::fingerprint::run(cmd),
        Commands::Key(cmd) => commands::keys::run(cmd),
        Commands::Set(cmd) => commands::settings::run_set(cmd),
        Commands::Settings => commands::settings::run_show(),
        Commands::Ota { path } => commands::ota::run(&path),
        Commands::Pam(cmd) => commands::pam_cmd(cmd),
        Commands::Logs => commands::run_logs(),
        Commands::Tui => commands::run_tui(),
    };
    if let Err(e) = result {
        eprintln!("Error: {}", e);
        std::process::exit(1);
    }
}
```

- [ ] **Step 2: Write commands/mod.rs with shared helpers and subcommand enums**

```rust
pub mod status;
pub mod info;
pub mod pair;
pub mod fingerprint;
pub mod keys;
pub mod settings;
pub mod ota;

use clap::Subcommand;

#[derive(Subcommand)]
pub enum FpCommands {
    List,
    Enroll { slot: u8 },
    Delete { slot: u8 },
    Verify,
}

#[derive(Subcommand)]
pub enum KeyCommands {
    List { category: String },
    Add { category: String },
    Delete { category: String, index: u8 },
    ExportSsh { index: u8 },
    GenerateSsh { name: String },
    Otp { index: u8 },
}

#[derive(Subcommand)]
pub enum SetCommands {
    Sudo { value: String },
    Polkit { value: String },
    Screen { value: String },
}

#[derive(Subcommand)]
pub enum PamCommands {
    Install { service: String },
    Remove { service: String },
}

use crate::socket_client::DaemonClient;

fn connect() -> Result<DaemonClient, String> {
    DaemonClient::connect()
}

pub fn pam_cmd(cmd: super::PamCommands) -> Result<(), String> {
    let (action, service) = match cmd {
        super::PamCommands::Install { service } => ("add", service),
        super::PamCommands::Remove { service } => ("remove", service),
    };
    let status = std::process::Command::new("pkexec")
        .arg("immurok-pam-helper")
        .arg(action)
        .arg(&service)
        .status()
        .map_err(|e| format!("pkexec failed: {}", e))?;
    if status.success() { Ok(()) } else { Err("PAM operation failed".into()) }
}

pub fn run_logs() -> Result<(), String> {
    std::process::Command::new("journalctl")
        .args(["--user", "-u", "immurok-daemon", "-f", "--no-pager"])
        .status()
        .map_err(|e| e.to_string())?;
    Ok(())
}

pub fn run_tui() -> Result<(), String> {
    // Will be implemented in Task 4.3
    todo!("TUI implementation")
}
```

- [ ] **Step 3: Write each subcommand file**

Each command file follows the same pattern: connect to daemon → send request → parse/format response → print. Example `status.rs`:

```rust
use super::connect;

pub fn run() -> Result<(), String> {
    let mut client = connect()?;
    let resp = client.send("STATUS")?;
    // Parse STATUS:connected:name:battery:version
    let parts: Vec<&str> = resp.splitn(5, ':').collect();
    if parts.len() >= 5 && parts[0] == "STATUS" {
        let connected = parts[1] == "1";
        println!("Device:    {}", if connected { parts[2] } else { "-" });
        println!("Status:    {}", if connected { "Connected" } else { "Disconnected" });
        println!("Battery:   {}%", parts[3]);
        println!("Firmware:  {}", parts[4]);
    }

    let resp = client.send("PAIR:STATUS")?;
    if resp.starts_with("OK:PAIRED:") {
        println!("Paired:    Yes ({})", &resp[10..]);
    } else {
        println!("Paired:    No");
    }

    let resp = client.send("FP:LIST")?;
    if resp.starts_with("OK:") {
        // Parse bitmap
        println!("Fingers:   {}", &resp[3..]);
    }
    Ok(())
}
```

Other command files (`pair.rs`, `fingerprint.rs`, `keys.rs`, `settings.rs`, `ota.rs`, `info.rs`) follow the same pattern with their specific socket commands. See spec CLI section for the full list.

- [ ] **Step 4: Verify compiles**

Run: `cargo build -p immurok-cli`

- [ ] **Step 5: Commit**

```bash
git commit -am "feat(linux): add CLI subcommands"
```

---

### Task 4.3: TUI panel

**Files:**
- Create: `app-linux-rs/crates/immurok-cli/src/tui/mod.rs`
- Create: `app-linux-rs/crates/immurok-cli/src/tui/app.rs`
- Create: `app-linux-rs/crates/immurok-cli/src/tui/widgets.rs`

- [ ] **Step 1: Write TUI app structure**

```rust
// tui/mod.rs
pub mod app;
pub mod widgets;

pub use app::run_tui;
```

```rust
// tui/app.rs
use std::time::Duration;
use crossterm::event::{self, Event, KeyCode};
use ratatui::prelude::*;
use crate::socket_client::DaemonClient;

pub struct App {
    client: DaemonClient,
    device_name: String,
    connected: bool,
    battery: u8,
    fw_version: String,
    paired: bool,
    fp_bitmap: u8,
    sudo_on: bool,
    polkit_on: bool,
    screen_on: bool,
    running: bool,
    status_line: String,
}

pub fn run_tui() -> Result<(), String> {
    let client = DaemonClient::connect()?;
    let mut app = App::new(client);

    // Setup terminal
    crossterm::terminal::enable_raw_mode().map_err(|e| e.to_string())?;
    let mut stdout = std::io::stdout();
    crossterm::execute!(stdout, crossterm::terminal::EnterAlternateScreen).ok();
    let backend = ratatui::backend::CrosstermBackend::new(stdout);
    let mut terminal = Terminal::new(backend).map_err(|e| e.to_string())?;

    // Main loop: poll status every 2s, handle keyboard input
    while app.running {
        terminal.draw(|f| widgets::draw(f, &app)).ok();

        if event::poll(Duration::from_secs(2)).unwrap_or(false) {
            if let Ok(Event::Key(key)) = event::read() {
                app.handle_key(key.code);
            }
        } else {
            app.refresh_status();
        }
    }

    // Restore terminal
    crossterm::terminal::disable_raw_mode().ok();
    crossterm::execute!(
        terminal.backend_mut(),
        crossterm::terminal::LeaveAlternateScreen
    ).ok();

    Ok(())
}

impl App {
    fn new(client: DaemonClient) -> Self {
        let mut app = Self {
            client, device_name: String::new(), connected: false,
            battery: 0, fw_version: String::new(), paired: false,
            fp_bitmap: 0, sudo_on: true, polkit_on: true, screen_on: true,
            running: true, status_line: String::new(),
        };
        app.refresh_status();
        app
    }

    fn refresh_status(&mut self) {
        // Send STATUS, PAIR:STATUS, FP:LIST, GET:SETTINGS via socket
        // Parse responses and update fields
    }

    fn handle_key(&mut self, key: KeyCode) {
        match key {
            KeyCode::Char('q') => self.running = false,
            KeyCode::Char('p') => { /* pair */ }
            KeyCode::Char('u') => { /* unpair */ }
            KeyCode::Char('e') => { /* enroll */ }
            KeyCode::Char('d') => { /* delete */ }
            KeyCode::Char('v') => { /* verify */ }
            KeyCode::Char('s') => { /* toggle sudo */ }
            KeyCode::Char('o') => { /* toggle polkit */ }
            KeyCode::Char('k') => { /* toggle screen */ }
            KeyCode::Char('i') => { /* show info */ }
            KeyCode::Char('l') => { /* show logs */ }
            _ => {}
        }
    }
}
```

```rust
// tui/widgets.rs
use ratatui::prelude::*;
use ratatui::widgets::*;
use super::app::App;

pub fn draw(f: &mut Frame, app: &App) {
    let area = f.area();
    let block = Block::default()
        .title(" immurok ")
        .borders(Borders::ALL)
        .border_type(BorderType::Rounded);

    // Layout: device info | fingers + settings | hotkeys | status line
    // See spec TUI panel mockup for exact layout
    // Use Layout::vertical with constraints

    let inner = block.inner(area);
    f.render_widget(block, area);

    // Render device info, fingerprint slots, settings toggles, hotkey bar
    // Use immurok_common::types::fp_bitmap_display() for finger display
}
```

- [ ] **Step 2: Wire TUI into commands/mod.rs**

Replace `todo!()` in `run_tui()` with `crate::tui::run_tui()`.

- [ ] **Step 3: Verify compiles**

Run: `cargo build -p immurok-cli`

- [ ] **Step 4: Commit**

```bash
git commit -am "feat(linux): add TUI panel structure"
```

---

## Chunk 5: Build, Install & Integration

### Task 5.1: Copy retained scripts

**Files:**
- Copy: `app-linux/immurok-auth-dialog` → `app-linux-rs/scripts/immurok-auth-dialog`
- Copy: `app-linux/immurok-pam-helper` → `app-linux-rs/scripts/immurok-pam-helper`
- Copy: `app-linux/pam_immurok.c` → `app-linux-rs/pam/pam_immurok.c`

- [ ] **Step 1: Copy files**

```bash
mkdir -p app-linux-rs/scripts app-linux-rs/pam
cp app-linux/immurok-auth-dialog app-linux-rs/scripts/
cp app-linux/immurok-pam-helper app-linux-rs/scripts/
cp app-linux/pam_immurok.c app-linux-rs/pam/
```

- [ ] **Step 2: Commit**

```bash
git commit -am "feat(linux): copy retained scripts (auth-dialog, pam-helper, pam module)"
```

---

### Task 5.2: Makefile

**Files:**
- Create: `app-linux-rs/Makefile`
- Create: `app-linux-rs/pam/Makefile`

- [ ] **Step 1: Write top-level Makefile**

```makefile
# immurok Linux (Rust) — Build + Install

CC ?= gcc
LOCAL_BIN = $(HOME)/.local/bin
SYSTEMD_DIR = $(HOME)/.config/systemd/user

PAM_DIR := $(shell \
    if [ -d /usr/lib64/security ]; then echo /usr/lib64/security; \
    elif [ -d /lib/aarch64-linux-gnu/security ]; then echo /lib/aarch64-linux-gnu/security; \
    elif [ -d /lib/x86_64-linux-gnu/security ]; then echo /lib/x86_64-linux-gnu/security; \
    elif [ -d /lib/security ]; then echo /lib/security; \
    else echo /usr/lib/security; fi)

.PHONY: all build pam install uninstall clean

all: build pam

build:
	cargo build --release --workspace

pam:
	$(MAKE) -C pam

install: all
	@echo "=== immurok install ==="
	install -Dm755 target/release/immurok-daemon $(LOCAL_BIN)/immurok-daemon
	install -Dm755 target/release/immurok-cli $(LOCAL_BIN)/immurok-cli
	install -Dm755 scripts/immurok-auth-dialog $(LOCAL_BIN)/immurok-auth-dialog
	install -Dm755 scripts/immurok-pam-helper $(LOCAL_BIN)/immurok-pam-helper
	sudo install -Dm755 pam/pam_immurok.so $(PAM_DIR)/pam_immurok.so
	install -Dm644 immurok-daemon.service $(SYSTEMD_DIR)/immurok-daemon.service
	systemctl --user daemon-reload
	systemctl --user enable --now immurok-daemon.service
	mkdir -p $(HOME)/.immurok
	@echo "=== Done ==="

uninstall:
	-systemctl --user disable --now immurok-daemon.service 2>/dev/null
	rm -f $(LOCAL_BIN)/immurok-daemon $(LOCAL_BIN)/immurok-cli
	rm -f $(LOCAL_BIN)/immurok-auth-dialog $(LOCAL_BIN)/immurok-pam-helper
	rm -f $(SYSTEMD_DIR)/immurok-daemon.service
	-sudo rm -f $(PAM_DIR)/pam_immurok.so 2>/dev/null
	systemctl --user daemon-reload

clean:
	cargo clean
	$(MAKE) -C pam clean
```

- [ ] **Step 2: Write pam/Makefile**

```makefile
CC ?= gcc
CFLAGS = -fPIC -Wall -Wextra -O2
TARGET = pam_immurok.so

all: $(TARGET)

$(TARGET): pam_immurok.c
	$(CC) $(CFLAGS) -shared -o $@ $< -lpam

clean:
	rm -f $(TARGET)
```

- [ ] **Step 3: Commit**

```bash
git commit -am "feat(linux): add Makefile and PAM build"
```

---

### Task 5.3: systemd service

**Files:**
- Create: `app-linux-rs/immurok-daemon.service`

- [ ] **Step 1: Write service file**

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

- [ ] **Step 2: Commit**

```bash
git commit -am "feat(linux): add systemd service file"
```

---

### Task 5.4: Full build verification

- [ ] **Step 1: Build entire workspace**

Run: `cd app-linux-rs && cargo build --release --workspace`
Expected: Both `immurok-daemon` and `immurok-cli` binaries built in `target/release/`

- [ ] **Step 2: Build PAM module**

Run: `make pam`
Expected: `pam/pam_immurok.so` created

- [ ] **Step 3: Run all tests**

Run: `cargo test --workspace`
Expected: All tests pass

- [ ] **Step 4: Verify binary sizes are reasonable**

Run: `ls -lh target/release/immurok-{daemon,cli}`
Expected: Each binary < 20MB (typical for Rust with bluer/zbus)

- [ ] **Step 5: Final commit**

```bash
git commit -am "feat(linux): verify full workspace build"
```
