# immurok — Wireless Fingerprint Authenticator

**One touch. No more passwords.**

immurok is a compact wireless Bluetooth fingerprint authenticator built for Mac and Linux desktop users. It lets you replace passwords with your fingerprint for everyday actions — screen unlock, sudo authorization, SSH signing, and more — with a single touch.

---

## Why immurok?

**How many times a day do you type your password?**

If you use a Mac mini, Mac Studio, Mac Pro, or a MacBook closed-lid with an external display, you don't have Touch ID. Apple's only answer is to spend ~$200 on a Magic Keyboard with Touch ID — and you have to use it as your main keyboard. If you're attached to your mechanical keyboard, that's not an option.

Linux users have it worse: there's almost no usable desktop fingerprint solution at all.

That's the problem immurok solves.

---

## Core Features

### Screen Unlock
While locked, just touch the sensor to unlock automatically. No opening the lid, no typing. Supported on both macOS and Linux.

### sudo and System Authorization
`sudo` prompting for a password in the terminal? System Settings asking you to authenticate? Touch your fingerprint to approve. Deep integration with the system authentication stack via a custom PAM module — not a simple password autofill.

### SSH Signing and Git Commit Signing
The device holds a built-in ECDSA P-256 keypair; the private key never leaves the hardware. You can sign Git commits and SSH login requests with your fingerprint — more convenient than a GPG key, more secure than a password.

### Long Battery Life
Idle current under 30 µA, USB-C charging, up to ~3 months of standby. The device sleeps automatically when idle and wakes the instant you touch it.

---

## New: A Human-in-the-Loop Authorization Gate Built for AI Coding Agents

Claude Code, Cursor, Codex, Aider, Cline, Gemini CLI — these AI agents increasingly **run commands on your machine on your behalf**: `sudo`, `git push`, reading API keys, calling remote SSH. But they don't have fingers, so they can't touch a sensor.

immurok offers a new approach: let the agent "hand a command to you for a fingerprint check."

```bash
# Wrap any privileged command, SSH command, or secret-reading command the agent is about to run
imk run --agent -- sudo systemctl restart api
imk run --agent -- git push origin main
imk run --agent --env-file .env -- python deploy.py
```

A HUD overlay pops up on screen and shows you the command verbatim. **Touch the device** and the subprocess runs; **close the overlay or stay idle for 30 seconds** and the subprocess is cleanly killed with `SIGTERM`, and the agent receives exit code 77 — signalling a user rejection.

### Why This Matters in the Age of AI Agents

| The pain of the traditional approach | The immurok solution |
|--------------------------------------|----------------------|
| Give the agent sudo NOPASSWD → effectively handing over root | Each privileged command needs one fingerprint; authorization is visible and refusable |
| Install ssh-agent + long-lived keys for the agent → your coding agent now also has prod push rights | SSH signing goes through the device's ECDSA private key; every signature needs a fingerprint |
| Put API keys directly in `.env` → the agent can read them for the entire conversation | `OPENAI_KEY=imk://api/openai` is injected into the subprocess only at the `exec()` moment; the agent never sees the plaintext |
| The agent stalls forever on a sudo password prompt | One fingerprint clears sudo + SSH + key reads at once; the whole subprocess runs uninterrupted within a 5-minute window |

### The Companion imk-skill

We maintain an open-source [imk-skill](https://github.com/immurok/imk-skill) repository — a single-file markdown skill that tells the agent: when to wrap a command, when not to, and how to exit cleanly when rejected.

- **Claude Code**: `/plugin marketplace add immurok/imk-skill && /plugin install imk-tools@imk`
- **Cursor / Cline / Continue / Windsurf**: vendor it into `.cursorrules` / `.clinerules` / `.continue/rules.md`
- **Codex / Aider / Gemini CLI**: paste it into `AGENTS.md` / `CONVENTIONS.md` / `GEMINI.md`

Once installed, the first time the agent wants to run a privileged command in your immurok project, it automatically wraps it with `imk run --agent --`. Your fingerprint is its sign-in sheet.

---

## What Lets It Do What Others Can't?

### The First True Wireless Bluetooth Fingerprint Authenticator

Fingerprint solutions on the market are either built into laptops or wired over USB. immurok is the first fingerprint authenticator that connects wirelessly over Bluetooth and supports both macOS and Linux. Put it anywhere on your desk — no cables.

### Mac and Linux at the Same Time

Not "supports Mac, Linux planned" — Linux is supported now. macOS has a native Swift menu bar app; Linux has a Rust daemon with a TUI management interface. Both platforms have full PAM integration.

### A Standalone Device, Not Tied to a Keyboard

Apple's Touch ID solution requires you to use the Magic Keyboard. immurok is a standalone device — keep using your favorite mechanical keyboard, HHKB, or whatever you like.

### Not Just Unlocking — Deep System Integration

Most biometric devices only do screen unlock. immurok achieves system-level integration via a PAM module: sudo, screen lock, and system authorization dialogs all work with your fingerprint. The SSH Agent feature is unique — sign code commits and remote logins with a fingerprint.

---

## Security Design

| Principle | Implementation |
|-----------|----------------|
| Biometrics never leave the device | Fingerprint templates are stored in the sensor's internal flash; even the host MCU cannot read them |
| End-to-end encrypted communication | ECDH P-256 key exchange for pairing; every BLE message is HMAC-SHA256 signed |
| No cloud, no account, no telemetry | Purely local Bluetooth communication; no internet required, no data collected |
| Encrypted firmware updates | OTA images are AES-128-CTR encrypted and verified with HMAC-SHA256 signatures |
| Open source and auditable | The app and PAM module are fully open source — don't trust us? Read the code |

---

## Comparison

| Feature | immurok | Magic Keyboard (Touch ID) | USB Fingerprint Reader |
|---------|---------|---------------------------|------------------------|
| Wireless | ✅ Bluetooth LE | ✅ Bluetooth | ❌ USB wired |
| Mac desktop support | ✅ | ✅ | ⚠️ Partial |
| Linux support | ✅ | ❌ | ⚠️ Limited |
| sudo / PAM integration | ✅ | ❌ | ❌ |
| SSH signing | ✅ | ❌ | ❌ |
| **AI agent command authorization** | ✅ `imk run --agent` | ❌ | ❌ |
| **FP-gated secret vault (API key / OTP)** | ✅ `imk://...` URI | ❌ | ❌ |
| Standalone (not tied to a keyboard) | ✅ | ❌ must be main keyboard | ✅ |
| Open source | ✅ | ❌ | ❌ |
| Price | Far below a Magic Keyboard | ~$200 | $15–45 |

---

## Hardware Specs

- **Connectivity**: Bluetooth LE 5.4
- **Fingerprint sensor**: Capacitive, 508 DPI, recognition time < 0.5 s
- **FAR / FRR**: < 0.001% / < 1%
- **Stored fingerprints**: up to 10
- **Charging**: USB-C
- **Standby life**: ~3 months
- **Dimensions**: 52 × 32 × 12 mm

---

## Who Is It For?

- **Developers who lean heavily on AI coding agents** — those who want the agent to run commands autonomously without handing over sudo / SSH / API keys
- Developers and pros on Mac mini / Mac Studio / Mac Pro
- MacBook users running closed-lid with an external display
- Linux users who need desktop fingerprint authentication
- Privacy-conscious, security-minded users who don't want to depend on cloud services
- Anyone who uses a mechanical keyboard but still wants fingerprint unlock

---

*Launching in 2026. Join the waitlist for early-bird access.*
