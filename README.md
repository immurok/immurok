# immurok

Wireless fingerprint authenticator for Mac, Windows, Linux, and your AI coding agent.

immurok is a hardware security key with a built-in fingerprint sensor, with all source code public on GitHub. It brings biometric authentication to Mac, Windows, and Linux machines — desktops, clamshell laptops, headless servers — and to AI coding agents that run commands on your behalf, all over Bluetooth LE.

**One touch to unlock your screen, authorize sudo, sign SSH commits, and gate the privileged commands your AI agent wants to run.**

## Features

- **Screen unlock** — Touch to unlock the macOS lock screen, Windows login (native Credential Provider), and Linux login
- **sudo / polkit** — Fingerprint replaces password for privilege escalation (macOS/Linux)
- **Dual-host** — Bind two computers at once; touch a dedicated switch fingerprint to move the device between them (firmware 1.7+)
- **SSH agent** — On-device ECDSA key generation and signing (private key never leaves the device)
- **Password manager unlock (macOS)** — Fingerprint unlocks 1Password / Bitwarden with a separately stored vault password; custom injection targets configurable
- **AI agent authorization** — Wrap an agent's subprocess with `imk run --agent --`. One fingerprint touch authorizes sudo + SSH signing + secret reads for the entire wrapped command. Reject and the subprocess gets `SIGTERM`. See [imk-skill](https://github.com/immurok/imk-skill).
- **Encrypted secret vault** — `imk://api/...`, `imk://otp/...`, `imk://ssh/...` URIs read by the device under fingerprint control. Inject into agent env via `--env-file` without touching disk.
- **TOTP** — Hardware-backed one-time password generation with Quick Fill (`Ctrl+\`)
- **API key vault** — Encrypted credential storage with biometric access
- **Mutual authentication** — ECDH P-256 pairing + HMAC-SHA256 signed notifications
- **All source public** — Companion apps are Apache 2.0; firmware and the rest are source-available under BSL 1.1

## How It Works

```
                 ┌──────────────────────┐     BLE      ┌──────────────────┐
                 │  Companion App       │◄────────────►│  immurok Device  │
                 │  (macOS / Windows /  │   Custom     │  CH592F + R559S  │
  ┌───────────┐  │   Linux)             │   GATT       │                  │
  │ AI agent  │─►│  - BLE manager       │              │  - BLE HID       │
  │ (Claude   │  │  - PAM socket server │              │  - Fingerprint   │
  │  Code,    │  │  - SSH agent         │              │  - Key storage   │
  │  Cursor,  │  │  - Screen unlocker   │              │  - Dual-host     │
  │  ...)     │  │  - Agent approval    │              │    slots         │
  └───────────┘  │    HUD overlay       │              │  - ECDH / HMAC   │
                 └──────────┬───────────┘              └──────────────────┘
                            │
              ┌─────────────▼──────────────┐
              │ PAM module (macOS / Linux) │
              │ Credential Prov. (Windows) │
              └────────────────────────────┘
```

1. The device pairs with the companion app via ECDH P-256 key exchange.
2. When you touch the fingerprint sensor, the device sends an HMAC-signed notification over BLE.
3. The companion app verifies the signature and performs the requested action: type a password, respond to a PAM challenge, sign an SSH request, or release a queued AI-agent subprocess.

### AI agent flow

```bash
# the agent calls
imk run --agent -- sudo systemctl restart api

# CLI talks to the companion app's AGENT_APPROVE socket
# overlay HUD pops, shows the verbatim command + any imk:// URIs in bold
# you touch the device → fingerprint match → 5-min sudo pre-auth opens
# subprocess execs; sudo's PAM module sees the pre-auth and skips its prompt
# SSH key sign / OTP read / API key read inside the same subprocess all
# satisfy themselves from the AUTH cooldown — one touch, whole subprocess

# rejected / 30 s timeout → subprocess gets SIGTERM, agent exits 77 (EX_NOPERM)
```

See [imk-skill/SKILL.md](https://github.com/immurok/imk-skill/blob/main/skills/using-imk/SKILL.md) for the canonical wrap pattern, when to wrap (and not), and troubleshooting.

## Repository Structure

```
firmware/              CH592F main firmware (C, Makefile)
ota/
  jumpapp/             OTA JumpIAP bootloader (4 KB)
  iap/                 OTA IAP bootloader (12 KB)
app-macos/             macOS companion app (Swift, SwiftPM)
  pam/                 macOS PAM module (C, Makefile)
app-linux-rs/          Linux companion daemon + CLI (Rust)
hardware/              Schematics, PCB layout, component selection
docs/
  protocol.md          BLE communication protocol
  security.md          Security architecture whitepaper
```

The Windows companion app (C#/.NET service + C++ Credential Provider) lives in
its own repository: [app-win](https://github.com/immurok/app-win).

## Hardware

| Component | Part | Notes |
|-----------|------|-------|
| MCU | WCH CH592F | RISC-V, BLE 5.4, 448 KB flash |
| Fingerprint sensor | R559S | Capacitive, 508 DPI, < 500 ms match; 5 auth slots + 1 host-switch slot |
| Connection | Bluetooth LE | HID keyboard + custom GATT |
| Power | Built-in battery | USB-C charging, ~40 µA idle, a month+ standby |

See [hardware/README.md](hardware/README.md) for detailed component selection rationale, GPIO pinout, and wiring diagram.

## Quick Start

### Prerequisites

- **Firmware**: [WCH CH592F SDK](http://www.wch-ic.com/) in `firmware/SDK/` (not included)
- **Flash tool**: [`wlink`](https://github.com/ch32-rs/wlink) (`cargo install --git https://github.com/ch32-rs/wlink`)
- **OTA keys**: `pip3 install cryptography && python3 ota/generate_ota_keys.py`
- **macOS app**: Xcode Command Line Tools, Swift 5.7+, macOS 13+
- **Linux app**: Rust 1.75+, BlueZ, D-Bus

### Build

```bash
# Firmware — compile + OTA package + flash
ota/upload-ota.sh release

# macOS app — compile + sign + deploy
app-macos/build-deploy.sh

# macOS PAM module (/usr/lib/pam is SIP-protected — use /usr/local)
cd app-macos/pam && make
sudo mkdir -p /usr/local/lib/pam
sudo cp pam_immurok.so /usr/local/lib/pam/

# Linux
cd app-linux-rs && make && sudo make install

# Windows — see https://github.com/immurok/app-win (Visual Studio 2022 / Inno Setup)
```

### First Run (macOS)

1. Open `/Applications/immurok.app`.
2. Grant Accessibility permission when prompted.
3. Open settings from the menu bar icon.
4. Pair with your immurok device.
5. Enroll at least one fingerprint.
6. Set your Mac login password (stored in Keychain, never sent over BLE).
7. Lock the screen, touch the sensor — done.

## On-Device Key Storage

The device stores cryptographic keys in CH592F flash. Private keys never leave the device.

| Category | Max Keys | Description |
|----------|----------|-------------|
| SSH | 32 | ECDSA P-256 keypairs, on-device signing |
| TOTP | 128 | HMAC-based one-time passwords |
| API | 50 | Credential strings, biometric-gated read |

## OTA Firmware Update

```bash
python3 ota/ota-update.py firmware/build/immurok_CH592F.imfw
```

OTA images are encrypted (AES-128-CTR) and signed (HMAC-SHA256). Keys are generated per development machine and not committed to the repository. See [ota/README.md](ota/README.md) for partition layout and update flow.

## Documentation

| Document | Description |
|----------|-------------|
| [docs/protocol.md](docs/protocol.md) | BLE GATT protocol — commands, notifications, packet formats, connection parameters |
| [docs/security.md](docs/security.md) | Security architecture — ECDH pairing, HMAC signing, PAM integration, threat model |
| [hardware/README.md](hardware/README.md) | Hardware design — component selection, GPIO pinout, wiring diagram |
| [ota/README.md](ota/README.md) | OTA update — flash layout, boot sequence, .imfw package format |
| [app-macos/README.md](app-macos/README.md) | macOS app — build instructions, architecture, source file guide |
| [app-win](https://github.com/immurok/app-win) | Windows app — .NET service, Credential Provider, installer |
| [app-linux-rs/README.md](app-linux-rs/README.md) | Linux app — per-distro dependencies, install steps, desktop-environment notes, troubleshooting |
| [imk-skill](https://github.com/immurok/imk-skill) | The AI-agent skill — how Claude Code / Cursor / Codex / Aider / Cline / Gemini CLI use `imk run --agent` to gate privileged commands |

## Security

immurok uses defense-in-depth:

- **ECDH P-256** pairing with ephemeral keys and private key erasure
- **HKDF-SHA256** key derivation (RFC 5869)
- **HMAC-SHA256** signed fingerprint notifications (8-byte truncated tag)
- **On-device key storage** — SSH private keys and TOTP secrets never leave the device
- **Fingerprint gating** — Sensitive operations (key signing, deletion, factory reset) require biometric proof with 10-second cooldown
- **PAM peer verification** — Unix socket accepts only root or current user (`getpeereid`)
- **OTA encryption** — AES-128-CTR + HMAC-SHA256 signed firmware images

Passwords are stored in macOS Keychain and never transmitted over BLE.

See [docs/security.md](docs/security.md) for the full security whitepaper.

## License

Licensing is split by component:

- **Companion apps — open source**: [app-macos](https://github.com/immurok/app-macos) (incl. the PAM module), [app-win](https://github.com/immurok/app-win), and [app-linux-rs](https://github.com/immurok/app-linux-rs) are licensed under the **Apache License 2.0** (see the `LICENSE` file in each repository).
- **Everything else** (firmware, OTA tooling, hardware, docs) is licensed under the [Business Source License 1.1](LICENSE):
  - **Permitted**: personal use, education, non-commercial research
  - **Not permitted**: commercial use (until the Change Date)
  - **Change Date**: 2030-03-05 — automatically converts to **Apache License 2.0**
