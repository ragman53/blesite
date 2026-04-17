# PLAN.md: BLEsite Development Strategy

> **Goal**: Ship a working Android MVP of BLEsite — an ephemeral BLE-delivered local web platform.  
> **Timeline**: 5 weeks (solo or small team)  
> **Stack**: Flutter 3.x + Rust (flutter_rust_bridge v2) + Android minSdk 31

---

## Guiding Principles

1. **Rust Core First** — Protocol correctness before any UI. All core logic (serialization, crypto, vitality, reassembly) must pass unit tests before Flutter integration.
2. **Offline Guarantee** — No `INTERNET` permission ever. Verify this at CI level.
3. **Fail Fast, Fail Silent** — ACK-less design means broken transfers are silently discarded. Don't retry; don't alert the user per-failure.
4. **Test on Physical Hardware Early** — BLE behavior on emulators is unreliable. Get two Android devices (minSdk 31) from Week 1.
5. **Power Budget is a First-Class Feature** — Measure mA drain from Week 2. Don't optimize later.

---

## Phase 0: Toolchain & Project Skeleton (Week 1)

**Goal**: A working flutter_rust_bridge v2 project on Android with BLE permissions granted at runtime.

### 0.1 Environment Setup
- Install Flutter stable, Rust stable + `cargo-ndk`, Android NDK r25+
- Verify `flutter_rust_bridge_codegen` generates Dart bindings from a trivial Rust `pub fn`
- Set up Android devices for USB debugging (minSdk 31 = Android 12+)

### 0.2 Project Scaffold
```bash
flutter create blesite --org com.blesite --platforms android
cd blesite
cargo init rust --lib
# Configure flutter_rust_bridge.yaml
# Configure android/app/build.gradle: minSdk 31, NDK, jniLibs
```

### 0.3 Android Baseline
- `AndroidManifest.xml`: BLE permissions (SCAN/CONNECT/ADVERTISE), Foreground Service (`connectedDevice` type)
- **NO** `INTERNET` permission (verify via `aapt dump permissions`)
- `network_security_config.xml`: cleartext blocked
- `MainActivity.kt`: `FLAG_SECURE` in `onCreate()`
- Runtime permission request flow using `permission_handler`

### 0.4 Rust Crate Baseline
- `Cargo.toml` with all dependencies pinned (postcard, ed25519-dalek, tokio, lz4_flex, bloomfilter, crc)
- Stub `lib.rs` with 3 FRB entry points: `init_engine()`, `get_version() -> String`, `parse_adv_packet(raw: Vec<u8>) -> Result<AdvPacketDto, BleError>`
- Run `flutter_rust_bridge_codegen generate` → verify Dart bindings compile

### 0.5 CI Foundation
- GitHub Actions: `cargo test`, `flutter analyze`, `flutter test`
- Lint: `cargo clippy -- -D warnings`, `flutter analyze --fatal-infos`

**Phase 0 Exit Criteria**:
- [ ] `flutter run` launches on physical Android 12+ device
- [ ] BLE permissions requested and granted at runtime
- [ ] FRB round-trip: Dart calls Rust `get_version()`, gets string back
- [ ] `cargo test` passes (even with stub tests)
- [ ] No `INTERNET` permission in APK

---

## Phase 1: Rust Core — Protocol, Crypto, Reassembly (Week 2)

**Goal**: All Rust logic unit-tested and FRB-exposed. Zero Flutter dependency.

### 1.1 Advertising Packet (`protocol/advertising.rs`)
- `AdvPacket::encode() -> [u8; 31]` with CRC8
- `AdvPacket::decode(&[u8; 31]) -> Result<AdvPacket, AdvError>`
- Validate: Magic byte, CRC8 checksum, field bounds
- FRB export: `parse_adv_packet(raw: Vec<u8>) -> Result<AdvPacketDto, String>`

### 1.2 Data Types & Serialization (`protocol/types.rs`)
- `SiteManifest` with `postcard` serialize/deserialize
- `RipplePacket` enum: `RequestAll`, `Chunk { index, total, data }`, `Eof`
- `ContentEncoding` enum: `Plain` / `Lz4`
- Round-trip test: serialize → deserialize → assert field equality

### 1.3 Ed25519 Crypto (`crypto/ed25519.rs`)
- Key generation: `generate_keypair() -> (HostPubKey, SecretKey)`
- Signing: `sign_manifest(secret: &SecretKey, title: &str, ts: u64) -> EdSignature`
- Verification: `verify_manifest(manifest: &SiteManifest) -> Result<(), CryptoError>`
- `sign_message` canonical form: `title.as_bytes() ++ ts.to_le_bytes()`
- Tests: valid sig passes, title-tampered fails, ts-tampered fails

### 1.4 Chunk Reassembler (`reassembler.rs`)
- `ChunkBuffer::new(total_chunks: u8)` — bitmap-based presence tracking
- `add_chunk(index: u8, data: &[u8])` — out-of-order safe
- `is_complete() -> bool` — all bits set
- `assemble() -> Option<Vec<u8>>` — None if any chunk missing
- 3s timeout tracked externally (in Dart / GATT client)

### 1.5 Content Validator (`protocol/validator.rs`)
- Size check: `raw.len() <= MAX_TOTAL_SIZE`
- LZ4 decompress if `content_encoding == Lz4`
- UTF-8 validation
- FRB export: `validate_content(raw: Vec<u8>, manifest: SiteManifestDto) -> Result<String, String>`

### 1.6 Vitality Engine (`vitality.rs`)
- `apply_decay(entry: &mut IndexEntry)` — 30s tick, 2880 ticks to zero
- `is_authentic_redetect(existing, pkt) -> bool`
- `should_purge(entry) -> bool`

### 1.7 Ephemeral Index (`index/ephemeral.rs`)
- `upsert_from_adv(pkt: &AdvPacket, rssi: i16, now_ms: u64)`
- `get_all_entries() -> Vec<IndexEntry>` (sorted by vitality desc)
- `set_content(site_id: SiteID, content: Vec<u8>)`
- `start_decay_scheduler(tx: mpsc::Sender<IndexEvent>)` — async Tokio task
- FRB exports: `get_index_snapshot() -> Vec<IndexEntryDto>`, `mark_content_fetched(site_id, content)`

### 1.8 BLE Emulator Mock (`mock/ble_emulator.rs`)
- Simulates advertising packets at configurable RSSI
- `SimulateBicyclePass { speed_kmh, range_m }` → fires adv packet once within window
- Used for Rust-only integration tests without hardware

**Phase 1 Exit Criteria**:
- [ ] All Rust unit tests pass (`cargo test`)
- [ ] FRB codegen produces valid Dart for all exported functions
- [ ] `flutter test` passes (Dart-side unit tests for DTO parsing)
- [ ] Emulator-based integration test: 12KB HTML assembled correctly
- [ ] Emulator-based integration test: missing chunk → None

---

## Phase 2: BLE Layer — Android Foreground Service & GATT (Week 3)

**Goal**: Real BLE scan/advertise/GATT on physical devices. Rust index populated from live BLE.

### 2.1 BLE Scanner (`lib/ble/scanner.dart`)
- `flutter_blue_plus` scan with filter: Manufacturer Specific Data (`0xB1` magic)
- On advertisement received: extract 31-byte payload → call `parseAdvPacket()` FRB
- On success: call `upsertFromAdv()` FRB → update Riverpod `ephemeralIndexProvider`
- Handle: Bluetooth adapter off/on events, scan restart on error

### 2.2 GATT Client (`lib/ble/gatt_client.dart`)
```
connect(device) → discoverServices()
  → requestMtu(512) → read FFF1 (Manifest)
  → FRB: verify_manifest() [disconnect if error]
  → write FFF2 (0x01 = RequestAll)
  → listen to FFF2 notifications → ChunkBuffer
  → 3s timeout → assemble() → FRB: validate_content()
  → success: set content in index → render
  → always: disconnect()
```
- Error states: `ConnectionTimeout`, `MtuNegotiationFailed` (fallback chunk_size=17), `ManifestInvalid`, `TransferIncomplete`, `ValidationFailed`
- Log errors (debug) but never show per-failure UI to the user

### 2.3 GATT Server / Host Mode
> **Note**: `flutter_blue_plus` supports GATT server on Android. Implement host-side here.
- Register GATT service UUID `12345678-...`
- Characteristic FFF1: Read → return serialized `SiteManifest` (postcard)
- Characteristic FFF2: Write (receive `RequestAll`) → start burst notification loop
- Characteristic FFF3: Read → return `last_updated_ms` as LE u64
- Burst loop: iterate chunks → `notifyValue(chunk)` every 7.5ms → `Eof` at end

### 2.4 Android Foreground Service (`lib/ble/foreground_service.dart`)
- `MethodChannel` to native Android `BleForegroundService.kt`
- Start/stop service, update notification text based on `BleState`
- `stopWithTask="false"` in manifest: survives app background
- Notification channel: `BLE_SERVICE` with `IMPORTANCE_LOW` (no sound)

### 2.5 Power Management
- Scan window: `scanWindow(100ms) / scanInterval(1100ms)` in background (duty cycle ~9%)
- Advertise interval: 500ms default; switch to 2000ms after 30min hosting
- `WakeLock.partial()` during burst transfer only (acquired + released in GATT client)

**Phase 2 Exit Criteria**:
- [ ] Device A advertises; Device B scanner sees it within 2s
- [ ] Device B taps → GATT connect → manifest verified → 12KB HTML received
- [ ] Transfer completes at 5m static distance in ≤ 2.0s
- [ ] Signature-invalid manifest → immediate disconnect
- [ ] Host app backgrounded → Foreground Service persists, advertising continues
- [ ] Measured battery drain: ≤ 55mA average (USB ammeter or BatteryStatsService)

---

## Phase 3: Flutter UI (Week 4)

**Goal**: Production-quality UI tied to Riverpod state. No placeholder screens.

### 3.1 RadarScreen (`lib/ui/radar_screen.dart`)
- `ConsumerWidget` watching `ephemeralIndexProvider`
- `ListView` of `SiteCard` widgets, sorted by vitality descending
- `SiteCard`: title, category color accent, RSSI distance badge, vitality opacity + pulse animation
- Pulse animation: `AnimationController` looping 0.5-2.0s based on vitality (higher = faster)
- Empty state: "No sites nearby" with BLE scanning indicator
- FAB: "+ Host Site" → `HostEditorScreen`

### 3.2 EphemeralViewerScreen (`lib/ui/ephemeral_viewer.dart`)
- Full-screen `WebViewController` (sandboxed, JS disabled)
- Top bar: vitality linear progress indicator (decays in real-time via `Timer.periodic`)
- CSP injection before `loadHtmlString()`
- Physical-only share button: shows `formatPhysicalRef()` in a bottom sheet (copyable text only)
- `WillPopScope`: clear WebView + revoke any blob reference on back

### 3.3 HostEditorScreen (`lib/ui/host_editor_screen.dart`)
- Form: Title (max 24 chars, live counter), Category picker (4 options), Tags (max 3, chip input)
- HTML content area: `TextField` multiline with monospace font
- Live size meter: `content.utf8.length` vs 12288 — red when >10KB, error when >12KB
- "Preview" button: open `EphemeralViewerScreen` inline without BLE
- "Start Broadcasting" button:
  1. Validate content (FRB `validate_content`)
  2. Generate Ed25519 keypair (FRB `generate_keypair`)
  3. Sign manifest (FRB `sign_manifest`)
  4. Start GATT server + advertising
  5. Navigate to `RadarScreen` with host banner

### 3.4 Riverpod Providers (`lib/providers/`)
```dart
// ephemeral_index_provider.dart
class EphemeralIndexNotifier extends StateNotifier<List<IndexEntryDto>> { ... }
// Updates from: BLE scanner callbacks, decay timer (30s)

// ble_state_provider.dart
final bleStateProvider = StreamProvider<BleState>((ref) => bleScannerStream());

// host_site_provider.dart
class HostSiteNotifier extends StateNotifier<HostSiteState?> { ... }
// HostSiteState: { manifest, contentBytes, startedAt }
```

**Phase 3 Exit Criteria**:
- [ ] RadarScreen shows live entries with decay animation
- [ ] Tapping a card: GATT fetch → viewer renders HTML
- [ ] HostEditorScreen: size meter works, preview works
- [ ] "Start Broadcasting": GATT server + advertising starts, Device B can fetch
- [ ] Viewer: back nav clears WebView
- [ ] No `print()` or `debugPrint()` in production paths (use `kDebugMode` guard)

---

## Phase 4: Integration, Polish & Hardening (Week 5)

**Goal**: App is robust under adverse conditions. Ready for beta.

### 4.1 Error Surface Audit
- All `try/catch` in GATT client produce typed error state (not generic String)
- `EphemeralViewerScreen`: shows "Transfer failed (縁がなかった)" on error — auto-dismiss 3s
- `HostEditorScreen`: warns if content > 10KB (yellow) or > 12KB (red, blocks broadcast)

### 4.2 Edge Cases
- Bluetooth toggled off mid-scan: restart scanner on `BluetoothAdapterState.on`
- Multiple simultaneous GATT connections (unlikely but handle): queue, one at a time
- SiteID collision (SHA256 truncation): bloom filter handles dedup; collisions vanish on next adv cycle
- App killed by OS: ForegroundService `stopWithTask="false"` keeps advertising; on relaunch, index is blank (by design)

### 4.3 E2E Bicycle Simulation Test
- Two devices on bicycle handles, 15km/h pass-by at 10m closest approach
- Record: did transfer start? did it complete? render time?
- Target: ≥9/10 successful renders in 20 trials

### 4.4 Battery Profiling Session
- Host mode, screen off, 1 hour
- Measure with `adb shell dumpsys batterystats` or USB power monitor
- Target: ≤50mA average → extrapolates to ≥80h on 4000mAh

### 4.5 Security Verification
- `aapt dump permissions app-debug.apk | grep INTERNET` → must be absent
- Tampered manifest packet → app disconnects within 200ms
- WebView loads external URL (injected in HTML) → NavigationDelegate blocks it
- `adb shell settings get global airplane_mode_on` → BLE still works (BLE is exempt from airplane mode on Android 12+)

### 4.6 Code Cleanup
- Remove all `TODO` / `FIXME` or file as GitHub Issues
- Add inline docs to all public Rust `pub fn` (FRB exports)
- `flutter pub publish --dry-run` equivalent check

**Phase 4 Exit Criteria**:
- [ ] Bicycle simulation: ≥9/10 successful transfers
- [ ] Battery: ≤50mA measured
- [ ] Security checklist: all items pass
- [ ] Zero `cargo clippy` warnings
- [ ] Zero `flutter analyze` issues

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| GATT server API unstable in `flutter_blue_plus` | Medium | High | Pin version; test on API 31, 33, 34 |
| BLE MTU negotiation fails on cheap devices | High | Medium | Fallback chunk_size=17 mandatory |
| Foreground Service killed by aggressive OEM (MIUI, etc.) | High | High | Test on Xiaomi/Samsung; use `stopWithTask=false` + restart receiver |
| Ed25519-dalek API changes in v2 | Low | Medium | Pin exact version in `Cargo.lock` |
| LZ4 decompression panic on malformed input | Medium | High | Wrap in `catch_unwind`; treat error as ValidateError |
| App Store rejection (FLAG_SECURE + no sharing) | Low | Medium | Document design rationale; target Play Store only for MVP |

---

## Milestones Summary

| Week | Phase | Deliverable |
|------|-------|-------------|
| 1 | Foundation | Flutter+Rust project builds, BLE permissions granted, FRB ping-pong works |
| 2 | Rust Core | All protocol/crypto/index logic unit-tested, FRB exported |
| 3 | BLE Layer | Two physical devices exchange 12KB HTML over BLE |
| 4 | Flutter UI | Full UI (radar + viewer + host editor) connected to live BLE |
| 5 | Integration | Bicycle test pass, battery target met, security checklist green |
