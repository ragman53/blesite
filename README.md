# BLEsite 📡

> **Ephemeral BLE Local Web Platform** — *codename: LocalRipple*

「情報が、そこに居た者だけが持ち帰れる『体験の痕跡』である」

BLEsite lets you broadcast tiny websites (≤12KB HTML/SVG) over Bluetooth Low Energy.  
Nearby devices pick them up passively. No internet. No servers. No links to share.  
When you walk away — the content fades.

---

## ✨ What it does

- 📡 **Broadcast** a hand-crafted HTML/SVG page over BLE (≤12KB)
- 🚶 **Discover** nearby sites passively — no pairing, no connection needed to see them
- 👆 **Tap** to fetch the full content via GATT burst transfer (~0.5s for 12KB)
- ⏳ **Vitality decay** — content lives 24h then vanishes. No disk writes ever.
- 🚴 Works at bicycle speed (15 km/h pass-by, 10m range)

## 🔐 Security by design

- Ed25519 signature verification before any content transfer
- Sandboxed WebView: JS disabled, all network requests blocked
- No `INTERNET` permission in the APK — enforced at OS level
- `FLAG_SECURE`: no screenshots of viewed content
- RAM-only storage: app kill = complete erasure

## 🛠️ Stack

| Layer | Technology |
|-------|-----------|
| UI | Flutter 3.x (Dart) |
| Core logic | Rust via `flutter_rust_bridge v2` |
| BLE | `flutter_blue_plus` (scan + GATT client/server) |
| State | Riverpod (hooks_riverpod) |
| Serialization | `postcard` (Rust) |
| Crypto | `ed25519-dalek` v2 |
| Compression | `lz4_flex` (optional, content >8KB) |
| Platform | Android-first (minSdk 31 / Android 12+) |

## 📐 Protocol at a Glance

```
Advertising (31B, passive):
  [Magic:1][SiteID:6][Category:2][PubKeyHint:4][Vitality:1][Flags:1][CRC8:1][Reserved:15]

GATT Burst (on tap):
  FFF1 Read  → SiteManifest (postcard) + Ed25519 signature
  FFF2 Write → RequestAll (0x01)
  FFF2 Notify← Chunk[0..N] burst, no ACK
              → assemble in RAM → validate → render
```

Missing any chunk? Silent discard. *「縁がなかった」*

## 🗂️ Project Docs

| File | Description |
|------|-------------|
| [`SPEC.md`](SPEC.md) | Full technical specification (protocol, data structures, security) |
| [`PLAN.md`](PLAN.md) | 5-phase development strategy with milestones |
| [`TODO.md`](TODO.md) | 168 atomic task units across all phases |

## 🚀 Getting Started

> **Prerequisites**: Flutter stable, Rust stable, `cargo-ndk`, Android NDK r25c, Android 12+ device

```bash
# Clone
git clone https://github.com/ragman53/blesite
cd blesite

# Install Flutter deps
flutter pub get

# Generate Rust↔Dart bridge
cargo install flutter_rust_bridge_codegen
flutter_rust_bridge_codegen generate

# Build & run on Android device
flutter run
```

*Note: BLE GATT server requires a physical Android device. Emulators do not support BLE advertising.*

## 📅 Roadmap

- [x] SPEC v0.1.0 — protocol design, security model, UI spec
- [ ] Phase 0 — Flutter+Rust project scaffold
- [ ] Phase 1 — Rust core (protocol / crypto / index)
- [ ] Phase 2 — BLE layer (scan / GATT / Foreground Service)
- [ ] Phase 3 — Flutter UI (Radar / Viewer / HostEditor)
- [ ] Phase 4 — Integration & battery/security hardening
- [ ] Post-MVP — iOS, mesh relay, private channels (Noise protocol)

## 🎨 Name Candidates

| Name | Vibe |
|------|------|
| **BLEsite** | Direct, descriptive (current working title) |
| **Ripple** | Wave propagation |
| **PassBy** | Surrender communication |
| **Keshiki** 景色 | Ephemeral scenery |

## 📄 License

MIT

---

*Inspired by [BitChat Android](https://github.com/permissionlesstech/bitchat-android). Adapt, don't copy.*
