# TODO.md: BLEsite — Task Units

> Status legend: `[ ]` = todo · `[~]` = in progress · `[x]` = done  
> Each task is a single atomic unit of work (1 PR / 1 commit scope).

---

## 🏗️ Phase 0: Toolchain & Project Skeleton

### Environment
- [ ] **P0-E1** Install Flutter stable, Rust stable, `cargo-ndk`, Android NDK r25c
- [ ] **P0-E2** Install `flutter_rust_bridge_codegen` (`cargo install flutter_rust_bridge_codegen`)
- [ ] **P0-E3** Enable USB debugging on two Android 12+ physical devices
- [ ] **P0-E4** Verify `flutter doctor` clean (no errors for Android toolchain)

### Project Scaffold
- [ ] **P0-S1** `flutter create blesite --org com.blesite --platforms android`
- [ ] **P0-S2** `cargo init rust --lib` inside `blesite/`, configure as cdylib
- [ ] **P0-S3** Create `flutter_rust_bridge.yaml` pointing to `rust/src/lib.rs`
- [ ] **P0-S4** Configure `android/app/build.gradle`: `minSdk 31`, NDK ABI filters (arm64-v8a, x86_64), jniLibs path
- [ ] **P0-S5** Add all `pubspec.yaml` dependencies (see SPEC.md §8)
- [ ] **P0-S6** Add all `rust/Cargo.toml` dependencies with pinned versions
- [ ] **P0-S7** Write stub `lib.rs` with `get_version() -> String` FRB export
- [ ] **P0-S8** Run `flutter_rust_bridge_codegen generate` → verify `frb_generated.dart` is created
- [ ] **P0-S9** `flutter run` on physical device → stub app launches without crash

### Android Baseline
- [ ] **P0-A1** Write `AndroidManifest.xml`: BLUETOOTH_SCAN (`neverForLocation`), BLUETOOTH_CONNECT, BLUETOOTH_ADVERTISE, FOREGROUND_SERVICE, FOREGROUND_SERVICE_CONNECTED_DEVICE
- [ ] **P0-A2** Verify **no** `INTERNET` or `ACCESS_FINE_LOCATION` permission: `aapt dump permissions app-debug.apk`
- [ ] **P0-A3** Write `res/xml/network_security_config.xml` (cleartext blocked)
- [ ] **P0-A4** Set `android:networkSecurityConfig` and `android:usesCleartextTraffic="false"` in `<application>`
- [ ] **P0-A5** Add `FLAG_SECURE` in `MainActivity.kt` `onCreate()`
- [ ] **P0-A6** Declare `BleForegroundService` in manifest (`foregroundServiceType="connectedDevice"`, `stopWithTask="false"`)
- [ ] **P0-A7** Implement runtime permission request flow using `permission_handler` (show rationale dialog first)
- [ ] **P0-A8** Handle Bluetooth adapter state: prompt user to enable if off on app start

### CI
- [ ] **P0-C1** Create `.github/workflows/ci.yml`: `cargo test`, `cargo clippy -- -D warnings`
- [ ] **P0-C2** Add `flutter analyze --fatal-infos` step to CI
- [ ] **P0-C3** Add `flutter test` step to CI (runs even with empty test/ initially)

---

## ⚙️ Phase 1: Rust Core

### Advertising Packet
- [ ] **P1-ADV1** Implement `AdvPacket` struct with all fields from SPEC §2.1
- [ ] **P1-ADV2** Implement `crc8()` (polynomial 0x07, init 0x00)
- [ ] **P1-ADV3** Implement `AdvPacket::encode() -> [u8; 31]` (Magic, SiteID, CategoryCode, PubKeyHint, Vitality nibble, Flags, CRC8, zero-pad)
- [ ] **P1-ADV4** Implement `AdvPacket::decode(&[u8; 31]) -> Result<AdvPacket, AdvError>`
- [ ] **P1-ADV5** Define `AdvError` enum: `BadMagic`, `CrcMismatch`, `InvalidField`
- [ ] **P1-ADV6** Test: encode → decode → field equality
- [ ] **P1-ADV7** Test: bit-flip in byte 5 → `CrcMismatch`
- [ ] **P1-ADV8** Test: wrong Magic → `BadMagic`
- [ ] **P1-ADV9** FRB export: `parse_adv_packet(raw: Vec<u8>) -> Result<AdvPacketDto, String>`
- [ ] **P1-ADV10** FRB export: `build_adv_packet(dto: AdvPacketDto) -> Vec<u8>`

### Data Types & Serialization
- [ ] **P1-TY1** Define `SiteManifest` struct (all fields from SPEC §3.1)
- [ ] **P1-TY2** Define `RipplePacket` enum (`RequestAll`, `Chunk`, `Eof`)
- [ ] **P1-TY3** Define `ContentEncoding` enum (`Plain`, `Lz4`)
- [ ] **P1-TY4** Implement `postcard::to_vec(manifest)` + `postcard::from_bytes(bytes)` round-trip
- [ ] **P1-TY5** Test: SiteManifest serialize → deserialize → assert all fields
- [ ] **P1-TY6** Test: `SiteManifest.title` max 24 UTF-8 chars validated at construction
- [ ] **P1-TY7** Test: `SiteManifest.tags` max 3 entries, each ≤16 chars
- [ ] **P1-TY8** FRB export: `SiteManifestDto` (mirror of SiteManifest for Dart)

### Ed25519 Crypto
- [ ] **P1-CR1** Implement `generate_keypair() -> (HostPubKey, SecretKey)` using `ed25519-dalek`
- [ ] **P1-CR2** Implement `sign_message(title: &str, ts: u64) -> Vec<u8>` — canonical: `title.bytes() ++ ts.to_le_bytes()`
- [ ] **P1-CR3** Implement `sign_manifest(sk: &SecretKey, title, ts) -> EdSignature`
- [ ] **P1-CR4** Implement `verify_manifest(manifest: &SiteManifest) -> Result<(), CryptoError>`
- [ ] **P1-CR5** Test: sign → verify → `Ok(())`
- [ ] **P1-CR6** Test: title mutated → `Err(BadSignature)`
- [ ] **P1-CR7** Test: timestamp mutated → `Err(BadSignature)`
- [ ] **P1-CR8** Test: pubkey swapped → `Err(InvalidKey)` or `Err(BadSignature)`
- [ ] **P1-CR9** FRB export: `generate_keypair() -> KeypairDto`
- [ ] **P1-CR10** FRB export: `sign_manifest_frb(keypair: KeypairDto, title, ts) -> Vec<u8>`
- [ ] **P1-CR11** FRB export: `verify_manifest_frb(manifest: SiteManifestDto) -> Result<(), String>`

### Chunk Reassembler
- [ ] **P1-RE1** Define `ChunkBuffer { total: u8, chunks: Vec<Option<Vec<u8>>> }`
- [ ] **P1-RE2** Implement `ChunkBuffer::new(total: u8) -> Self`
- [ ] **P1-RE3** Implement `add_chunk(index: u8, data: &[u8])` — bounds check, no duplicate overwrite
- [ ] **P1-RE4** Implement `is_complete() -> bool` — all slots `Some`
- [ ] **P1-RE5** Implement `assemble() -> Option<Vec<u8>>` — None if any slot None; concatenate in order
- [ ] **P1-RE6** Test: 26 chunks in order → assemble → 12,480 bytes (26×480)
- [ ] **P1-RE7** Test: 26 chunks out of order → assemble succeeds
- [ ] **P1-RE8** Test: chunk 13 missing → assemble → None
- [ ] **P1-RE9** Test: duplicate chunk index → idempotent (second write ignored)
- [ ] **P1-RE10** Test: chunk index out of bounds → no panic (return Err or ignore)

### Content Validator
- [ ] **P1-VA1** Implement `validate_content(raw: &[u8], manifest: &SiteManifest) -> Result<Vec<u8>, ValidateError>`
- [ ] **P1-VA2** Size check: `raw.len() <= MAX_TOTAL_SIZE`
- [ ] **P1-VA3** LZ4 decompress path (only if `manifest.content_encoding == Lz4`)
- [ ] **P1-VA4** LZ4 decompress wrapped in `std::panic::catch_unwind` → `ValidateError::DecompressFailed`
- [ ] **P1-VA5** UTF-8 validation on decompressed content
- [ ] **P1-VA6** Decompressed length == `manifest.total_size` check
- [ ] **P1-VA7** Test: valid plain HTML → Ok
- [ ] **P1-VA8** Test: LZ4-compressed HTML → decompressed → Ok
- [ ] **P1-VA9** Test: oversized raw → `TooLarge`
- [ ] **P1-VA10** Test: malformed LZ4 → `DecompressFailed` (no panic)
- [ ] **P1-VA11** FRB export: `validate_content_frb(raw: Vec<u8>, manifest: SiteManifestDto) -> Result<String, String>`

### Vitality Engine
- [ ] **P1-VI1** Define constants: `DECAY_PER_TICK`, `REDETECT_BOOST`, `VITALITY_FULL`, `VITALITY_ZERO`
- [ ] **P1-VI2** Implement `apply_decay(entry: &mut IndexEntry)` — single 30s tick logic
- [ ] **P1-VI3** Implement `is_authentic_redetect(existing, pkt) -> bool` — SiteID + PubKeyHint match
- [ ] **P1-VI4** Implement `should_purge(entry) -> bool`
- [ ] **P1-VI5** Test: 2880 ticks → `vitality ≈ 0.0` → `should_purge = true`
- [ ] **P1-VI6** Test: 1440 ticks (12h) → `vitality ≈ 0.5`
- [ ] **P1-VI7** Test: 1440 ticks → redetect boost → vitality increased by 0.1 (capped at 1.0)
- [ ] **P1-VI8** Test: redetect with wrong pubkey_hint → `is_authentic_redetect = false`

### Ephemeral Index
- [ ] **P1-IX1** Implement `EphemeralIndex` struct (RwLock<HashMap>, Mutex<BloomFilter>, `my_site_id`)
- [ ] **P1-IX2** Implement `upsert_from_adv(pkt, rssi, now_ms)`:
  - [ ] Bloom filter check → skip if seen recently
  - [ ] If entry exists: check `is_authentic_redetect` → set `was_re_detected`
  - [ ] If new: insert with vitality=1.0
- [ ] **P1-IX3** Implement `get_all_entries() -> Vec<IndexEntry>` sorted by vitality desc, excluding `my_site_id`
- [ ] **P1-IX4** Implement `set_content(site_id, content: Vec<u8>)`
- [ ] **P1-IX5** Implement `start_decay_scheduler(tx)` async Tokio task (30s interval)
- [ ] **P1-IX6** `IndexEvent` enum: `Purged(Vec<SiteID>)`, `Updated(SiteID)`
- [ ] **P1-IX7** FRB export: `init_index()` — creates singleton Arc<EphemeralIndex>
- [ ] **P1-IX8** FRB export: `upsert_adv(raw_pkt: Vec<u8>, rssi: i16, now_ms: u64)`
- [ ] **P1-IX9** FRB export: `get_index_snapshot() -> Vec<IndexEntryDto>`
- [ ] **P1-IX10** FRB export: `set_fetched_content(site_id: Vec<u8>, content: Vec<u8>)`
- [ ] **P1-IX11** FRB export: `subscribe_index_events() -> Stream<IndexEventDto>` (DartStream)

### BLE Emulator Mock
- [ ] **P1-EM1** Implement `BleEmulator` with `mpsc::channel` pair
- [ ] **P1-EM2** `emit_adv(site_id, rssi)` — fires `EmulatorEvent::Advertise`
- [ ] **P1-EM3** `simulate_bicycle_pass(speed_kmh, range_m)` — calculates contact window, fires one adv
- [ ] **P1-EM4** Integration test: emulator → `upsert_from_adv()` → `get_all_entries()` → 1 entry
- [ ] **P1-EM5** Integration test: simulate 12KB burst → `ChunkBuffer::assemble()` → valid

---

## 📡 Phase 2: BLE Layer

### BLE Scanner (Dart)
- [ ] **P2-SC1** Create `lib/ble/scanner.dart` wrapping `flutter_blue_plus`
- [ ] **P2-SC2** `startScan()`: filter scan results by Manufacturer Specific Data magic byte `0xB1`
- [ ] **P2-SC3** On advertisement: extract 31B payload → call FRB `upsertAdv(raw, rssi, nowMs)`
- [ ] **P2-SC4** Handle `BluetoothAdapterState` stream → auto-restart scan on `turningOn`
- [ ] **P2-SC5** Background scan duty cycle: 100ms window / 1100ms interval (via `ScanSettings`)
- [ ] **P2-SC6** Expose `scanResultStream` as `Stream<void>` (signals index changed) for Riverpod

### GATT Client (Dart)
- [ ] **P2-GC1** Create `lib/ble/gatt_client.dart`
- [ ] **P2-GC2** `connect(BluetoothDevice) -> GattSession` with 5s timeout
- [ ] **P2-GC3** `discoverServices()` → find service UUID `12345678-...`
- [ ] **P2-GC4** `requestMtu(512)` with fallback: compute `chunkSize = negotiatedMtu - 3`
- [ ] **P2-GC5** `readManifest()` → read FFF1 → call FRB `verifyManifestFrb()` → disconnect on error
- [ ] **P2-GC6** `requestBurst()` → write `0x01` to FFF2
- [ ] **P2-GC7** Subscribe to FFF2 notifications → feed bytes to `ChunkBuffer` in Dart (or via FRB)
- [ ] **P2-GC8** 3s timeout watchdog on burst transfer → discard on expire
- [ ] **P2-GC9** On complete: call FRB `validateContentFrb()` → call FRB `setFetchedContent()`
- [ ] **P2-GC10** Always `disconnect()` in `finally` block
- [ ] **P2-GC11** `GattError` enum: `Timeout`, `ServiceNotFound`, `ManifestInvalid`, `TransferIncomplete`, `ValidationFailed`

### GATT Server / Host (Dart)
- [ ] **P2-GS1** `lib/ble/gatt_server.dart`: register GATT service with `flutter_blue_plus` GATT server API
- [ ] **P2-GS2** FFF1 (Read handler): return postcard-serialized manifest bytes
- [ ] **P2-GS3** FFF2 (Write handler): on `0x01` received → start burst notification coroutine
- [ ] **P2-GS4** Burst loop: iterate content in 480B chunks → `notifyValue(chunk)` → `await Future.delayed(7.5ms)`
- [ ] **P2-GS5** Send `Eof` packet after last chunk
- [ ] **P2-GS6** FFF3 (Read handler): return `lastUpdatedMs` as LE u64 bytes
- [ ] **P2-GS7** Handle multiple concurrent client connections: queue or serve first-come-first-served

### BLE Advertiser (Dart)
- [ ] **P2-AD1** `lib/ble/advertiser.dart`: start BLE advertising with `flutter_blue_plus`
- [ ] **P2-AD2** Advertising data: Manufacturer Specific Data with encoded `AdvPacket` (31B)
- [ ] **P2-AD3** Include GATT service UUID in advertising payload
- [ ] **P2-AD4** `setAdvertiseInterval(ms)`: 500ms default, expose setter for power-save switch
- [ ] **P2-AD5** After 30min hosting → auto-switch to 2000ms interval

### Foreground Service (Android Native + Dart)
- [ ] **P2-FS1** Create `android/app/src/main/kotlin/…/BleForegroundService.kt`
- [ ] **P2-FS2** `startForeground()` with notification (channel: `BLE_SERVICE`, importance: LOW)
- [ ] **P2-FS3** Handle `START_SCANNING` / `START_HOSTING` / `STOP` intents
- [ ] **P2-FS4** Create `lib/ble/foreground_service.dart` with `MethodChannel` bridge
- [ ] **P2-FS5** `BleState` enum: `idle`, `scanning`, `hosting`, `transferring`
- [ ] **P2-FS6** `updateNotification(BleState, {String? title})` — calls native via MethodChannel
- [ ] **P2-FS7** Test: app backgrounded → notification visible → BLE scan continues

### Power Management
- [ ] **P2-PM1** `WakeLock.partial()` acquired only during GATT burst transfer
- [ ] **P2-PM2** Release WakeLock in GATT client `finally` block
- [ ] **P2-PM3** Validate: `dumpsys wakelocks` shows no stuck wakelock after transfer

---

## 🎨 Phase 3: Flutter UI

### Riverpod Providers
- [ ] **P3-PR1** `lib/providers/ephemeral_index_provider.dart`: `StateNotifierProvider<List<IndexEntryDto>>`
  - Updates from: FRB `subscribeIndexEvents()` stream + manual poll every 30s
- [ ] **P3-PR2** `lib/providers/ble_state_provider.dart`: `StreamProvider<BleState>` from scanner state
- [ ] **P3-PR3** `lib/providers/host_site_provider.dart`: `StateNotifierProvider<HostSiteState?>`
  - `HostSiteState`: `{ manifest, contentBytes, startedAt, keypair }`
- [ ] **P3-PR4** Run `build_runner` to generate `.g.dart` files

### RadarScreen
- [ ] **P3-RA1** Create `lib/ui/radar_screen.dart` as `ConsumerWidget`
- [ ] **P3-RA2** `ListView.builder` watching `ephemeralIndexProvider` — rebuild on change
- [ ] **P3-RA3** `SiteCard` widget: title, category color bar (amber/violet/teal/slate), vitality opacity
- [ ] **P3-RA4** `SiteCard` RSSI badge: "◎ Near" / "○ Medium" / "△ Far" based on dBm
- [ ] **P3-RA5** Vitality pulse animation: `AnimationController` speed = `0.5 + vitality * 1.5` seconds period
- [ ] **P3-RA6** Empty state: scanning spinner + "No sites nearby"
- [ ] **P3-RA7** `FloatingActionButton` → push `HostEditorScreen`
- [ ] **P3-RA8** AppBar: BLE state indicator (scanning dot / hosting badge)

### EphemeralViewerScreen
- [ ] **P3-VI1** Create `lib/ui/ephemeral_viewer.dart`
- [ ] **P3-VI2** `buildSecureController(html)` — JS disabled, NavigationDelegate blocks all external URLs
- [ ] **P3-VI3** Inject CSP meta tag before `loadHtmlString()`
- [ ] **P3-VI4** Top bar: `LinearProgressIndicator` for vitality, updated every 5s via `Timer.periodic`
- [ ] **P3-VI5** When vitality reaches 0: show "この記録は消えました" overlay, pop after 2s
- [ ] **P3-VI6** Share bottom sheet: `formatPhysicalRef()` text only, copy-to-clipboard button, explanatory text
- [ ] **P3-VI7** `WillPopScope`: clear WebView, cancel timer on pop

### HostEditorScreen
- [ ] **P3-HE1** Create `lib/ui/host_editor_screen.dart`
- [ ] **P3-HE2** Title `TextField`: max 24 chars, live char counter, `TextInputAction.next`
- [ ] **P3-HE3** Category picker: `SegmentedButton` (Food / Art / Event / Info)
- [ ] **P3-HE4** Tags input: `Wrap` of `InputChip` + `TextField`, max 3, each ≤16 chars
- [ ] **P3-HE5** Content `TextField`: multiline, monospace font, expand to fill
- [ ] **P3-HE6** Live size meter: `(content.length) / 12288` → `LinearProgressIndicator`
  - Green: < 10KB, Yellow: 10-12KB, Red: > 12KB (disable broadcast button)
- [ ] **P3-HE7** "Preview" `TextButton` → push `EphemeralViewerScreen` with local HTML (no BLE)
- [ ] **P3-HE8** HTML validator (`lib/utils/html_validator.dart`): reject if contains `<script`, `src="http`, `href="http`
- [ ] **P3-HE9** "Start Broadcasting" `FilledButton`:
  1. Validate content (FRB)
  2. FRB `generateKeypair()`
  3. FRB `signManifestFrb(keypair, title, nowMs)`
  4. Update `hostSiteProvider`
  5. Start GATT server + advertiser
  6. Start Foreground Service in hosting mode
  7. Pop to RadarScreen
- [ ] **P3-HE10** Disable "Start Broadcasting" if already hosting (show "Stop Hosting" instead)

### HTML Validator Utility
- [ ] **P3-HV1** `lib/utils/html_validator.dart`: `validateHtml(String html) -> HtmlValidationResult`
- [ ] **P3-HV2** Check: `html.length <= 12288` (UTF-8 byte count)
- [ ] **P3-HV3** Check: no `<script` tag
- [ ] **P3-HV4** Check: no `src="http` or `href="http` (external resource references)
- [ ] **P3-HV5** Check: no `<img src="data:image/` with binary data (allow SVG `data:image/svg+xml`)
- [ ] **P3-HV6** Return: `{ isValid: bool, byteSize: int, warnings: List<String> }`

---

## 🔗 Phase 4: Integration & Hardening

### Error Handling Polish
- [ ] **P4-EH1** `EphemeralViewerScreen`: on fetch error → show `SnackBar("縁がなかった")` auto-dismiss 3s
- [ ] **P4-EH2** `GattClient`: typed error states propagated to UI via `AsyncValue`
- [ ] **P4-EH3** Scanner: handle `FlutterBluePlusException` → log debug, restart scan silently
- [ ] **P4-EH4** All `kDebugMode` guard on `debugPrint` calls

### Edge Cases
- [ ] **P4-EC1** Bluetooth toggled off mid-scan → UI shows "Bluetooth is off" banner
- [ ] **P4-EC2** Simultaneous GATT connection attempt → queue second, process serially
- [ ] **P4-EC3** Host content > 12KB in editor → block broadcast, show red warning
- [ ] **P4-EC4** App relaunched → index blank (verify: no persistence leaks)
- [ ] **P4-EC5** 100+ devices nearby → index handles large HashMap (test with emulator: 200 mock entries)

### Security Verification
- [ ] **P4-SV1** `aapt dump permissions app-release.apk | grep INTERNET` → empty
- [ ] **P4-SV2** Tampered manifest byte → GATT disconnect logged within 200ms
- [ ] **P4-SV3** WebView: inject `<a href="https://evil.com">click</a>` → `NavigationDecision.prevent` fires
- [ ] **P4-SV4** Verify `FLAG_SECURE`: screenshot attempt shows black screen
- [ ] **P4-SV5** `network_security_config` verified: no cleartext allowed

### E2E & Performance Tests
- [ ] **P4-ET1** Static 1m: connect-to-render ≤ 1.5s for 12KB HTML
- [ ] **P4-ET2** Static 5m: connect-to-render ≤ 2.0s
- [ ] **P4-ET3** Bicycle simulation (15km/h, 10m pass): 20 trials, ≥18 successful
- [ ] **P4-ET4** Battery drain: host mode, screen off, 1h → ≤ 50mA average
- [ ] **P4-ET5** Memory: run 24h host mode in emulator → no OOM, heap stable

### Code Quality
- [ ] **P4-CQ1** `cargo clippy -- -D warnings` → zero warnings
- [ ] **P4-CQ2** `flutter analyze --fatal-infos` → zero issues
- [ ] **P4-CQ3** All public Rust FRB exports have `///` doc comments
- [ ] **P4-CQ4** Remove all `TODO` / `FIXME` comments or file as GitHub Issues
- [ ] **P4-CQ5** `cargo test` → all tests pass
- [ ] **P4-CQ6** `flutter test` → all tests pass

---

## 🔮 Post-MVP Backlog (Not Scheduled)

- [ ] **BACK-1** iOS support: Swift BLE GATT server + CoreBluetooth advertiser
- [ ] **BACK-2** Multi-hop relay: TTL nibble in adv packet + relay mode in host
- [ ] **BACK-3** Desktop (macOS): BLE via CoreBluetooth; Windows via WinRT
- [ ] **BACK-4** Noise protocol private channels (BitChat-compatible)
- [ ] **BACK-5** LZ4 compression in HostEditorScreen for content > 8KB
- [ ] **BACK-6** Content update: `has_update` flag in adv packet → re-fetch on tap
- [ ] **BACK-7** Category filter in RadarScreen
- [ ] **BACK-8** RSSI-based distance animation on radar canvas

---

## Task Count Summary

| Phase | Tasks | Priority |
|-------|-------|----------|
| Phase 0: Toolchain | 20 | 🔴 Blocking |
| Phase 1: Rust Core | 56 | 🔴 Blocking |
| Phase 2: BLE Layer | 29 | 🟠 Core |
| Phase 3: Flutter UI | 34 | 🟠 Core |
| Phase 4: Integration | 21 | 🟡 Quality |
| Post-MVP Backlog | 8 | 🟢 Future |
| **Total** | **168** | |
