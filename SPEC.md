# SPEC.md: Ephemeral BLE Local Web Platform

> **App Name**: **BLEsite**  
> **Codename**: LocalRipple  
> **Version**: 0.1.0  
> **Stack**: Flutter (Dart) + Rust (via `flutter_rust_bridge v2`)  
> **Target**: Android-first (minSdk 31), Post-MVP: iOS/Desktop  
> **Inspiration**: BitChat Android (https://github.com/permissionlesstech/bitchat-android)  
> **Philosophy**: 「情報が、そこに居た者だけが持ち帰れる『体験の痕跡』である」

---

## 1. 🎯 Reality-Check Constraints (The "Sweet Spot")

物理的「すれ違い」の現実性と、バックグラウンド動作の技術的制限を両立させる設計値。

| 原則 | 実装値 | 根拠・計算式 |
|------|--------|-------------|
| **Payload Limit** | **Max 12KB (total)** | 自転車速度(15km/h)×BLE有効範囲(10m)=**約2.4秒**の接触時間。実効スループット20-30KB/s → 12KB転送に**約0.5秒**。マージン十分。 |
| **Data Format** | Strictly HTML/SVG Only | 画像バイナリ禁止。12KBを文字情報・軽量ベクタに全振り。レンダリング負荷・帯域・セキュリティを最適化。 |
| **State Storage** | RAM Only (Zero-Disk) | アプリキル=完全消去。永続化オーバーヘッドゼロ。究極の「儚さ」を実現。 |
| **Viewer** | Sandboxed WebView | `JavaScriptMode.disabled` + `NavigationDelegate` で全外部遷移ブロック + CSP `default-src 'none'` (§5.3 参照)。セキュリティと描画再現性の両立。 |
| **Discovery** | 1-Hop Passive Ripple | 接続不要の広告パケットで「存在の波紋」を伝播。詳細はタップ時のみ取得。 |
| **Host Battery** | **≥24h on 4000mAh** | 広告間隔・CPU制御・Foreground Service最適化で平均消費≤50mAを達成。 |
| **Vitality Lifetime** | **24 hours nominal** | 連続ホストで24時間維持。再検知でブースト、圏外離脱で減衰開始、vitality=0で完全消滅。 |

### 🚴 自転車速度での転送成功率計算
```
前提:
- 自転車速度: 15 km/h = 4.2 m/s
- BLE実効通信範囲: 10 m (障害物・干渉を考慮)
- 接触可能時間: 10 m / 4.2 m/s ≈ 2.4 秒

転送:
- データサイズ: 12,288 bytes
- チャンクサイズ: 480 bytes → 約26チャンク
- BLE通知間隔: 7.5 ms (Android最優値)
- 実効スループット: 20-30 KB/s (プロトコルオーバーヘッド含む)
- 転送所要時間: 12 KB / 25 KB/s ≈ 0.5 秒

結論: 2.4秒の接触窓に対し0.5秒転送 → **約5倍のマージン**。安定動作可能。
```

---

## 2. 📡 Background-Optimized Protocol

バックグラウンド制限・パケットロス・切断を前提とした「諦めの美学」プロトコル。

### 2.1 Advertising Payload (31 Bytes) — 「波紋」の設計
BLE Manufacturer Specific Data (AD type `0xFF`) に格納。接続不要で「存在だけ」を伝播させる最小メタデータ。
**エンディアン**: すべての多バイト整数フィールドは **little-endian (LE)**。`SiteID` / `PubKeyHint` は生バイト列 (endian 非依存)。

| Byte | Field | Size | Endian | Description |
|------|-------|------|--------|-------------|
| `0` | `Magic` | 1B | — | `0xB1` (Protocol identifier) |
| `1-6` | `SiteID` | 6B | bytes | `SHA256(host_pubkey ‖ timestamp_ms_le)[0..6]` |
| `7-8` | `CategoryCode` | 2B | LE u16 | `0x0001`=Food, `0x0002`=Art, `0x0003`=Event, `0x0004`=Info |
| `9-12` | `PubKeyHint` | 4B | bytes | `host_pubkey[0..4]` — relay/redetect authenticity check用 |
| `13` | `Vitality` | 1B | — | bit4–7 (upper nibble): vitality 0–15 (scaled from f32 · `(vitality*15).round() as u8`), bit0–3: reserved |
| `14` | `Flags` | 1B | — | bit0=`is_relay`, bit1=`has_update`, bit2–7=reserved |
| `15` | `Checksum` | 1B | — | CRC8 of bytes 0–14 (polynomial `0x07`, init `0x00`, no reflect, xor-out `0x00`) |
| `16-30` | `Reserved` | 15B | — | Zero padding / future extension (multi-hop TTL等) |

> ⚠️ **v0.0.1との変更点**: バイト9-12に`PubKeyHint`フィールドを追加。`TitleHash`フィールドを削除(Manifestから取得可能)。`is_authentic_redetect()`の実装と整合性確保のため。

### 2.2 GATT Service & Burst Transfer
```
Service UUID: 12345678-1234-5678-1234-56789ABCDEF0
```

> ⚠️ **UUID Placeholder**: 上記は開発用プレースホルダー。本番リリース前に `uuidgen` で v4 UUID を新規発行し、`rust/src/constants.rs::SERVICE_UUID` と Android/Dart 両側に反映すること (TODO 参照)。

| Char UUID | Permission | Name | Description |
|-----------|------------|------|-------------|
| `FFF1` | Read | `Manifest` | `SiteManifest` (postcard-serialized, max ~512B) |
| `FFF2` | Write/Notify | `DataChannel` | Client writes `RequestAll` → Host notifies `Chunk[]` burst |
| `FFF3` | Read | `HostHint` | Optional: `last_updated_ms` timestamp (cache invalidation) |

### 2.3 Fast Handshake Flow (ACK-less Design)
```text
1. [Passive] Client receives advertising packet (31B in Manufacturer Specific Data)
   ├─ Validate Magic byte (0xB1) + CRC8
   ├─ Parse SiteID, CategoryCode, PubKeyHint, Vitality
   ├─ Check Bloom filter (dedup by SiteID)
   └─ Upsert in-memory index (vitality boost if redetect)

2. [Active] User taps entry → GATT connect
   ├─ Discover services (target: 12345678-...)
   ├─ Request MTU: 512 bytes (timeout: 1000ms)
   │  ├─ Negotiated: chunk_size = negotiated_mtu - 3 (ATT header)
   │  └─ Fallback: chunk_size = 20 (default ATT MTU - 3)
   ├─ Read FFF1: Manifest (postcard deserialize → SiteManifest)
   │  ├─ Verify Ed25519 signature ← 失敗時は即時切断・ログ記録
   │  └─ Validate total_size ≤ 12288
   └─ Write FFF2: RequestAll (1-byte: 0x01)

3. [Burst] Host notifies Chunk[0..N] sequentially
   ├─ No ACK, no retransmission ("縁がなかった"設計)
   ├─ Client reassembles in memory by chunk index
   ├─ Timeout: 3000ms for full transfer (2× the 0.5s nominal)
   └─ If any chunk missing OR timeout → discard entire buffer

4. [Render] Client verifies total checksum → decompress if LZ4 header →
           inject CSP meta tag → blob: URL → WebView load
   └─ Disconnect immediately after render init

5. [Cleanup] On app background or vitality=0 → clear WebView + Blob URL
```

### 2.4 Power-Optimized Host Mode
4000mAhバッテリーで24時間動作を実現するための制御パラメータ。

```rust
// rust/src/power/config.rs
pub struct HostPowerConfig {
    pub advertise_interval_ms: u16,      // 500ms balanced / 2000ms power-save
    pub idle_cpu_threshold_ms: u64,      // 100ms 非活動でCPU freq yield
    pub burst_timeout_ms: u16,           // 3000ms (nominal 0.5s × 6 margin)
    pub notification_update_sec: u16,    // 60s (Foreground Service通知更新)
    pub connection_alive_max_ms: u16,    // 5000ms (コンテンツ転送後の最大保持)
}

// 消費電力見積もり (4000mAh, 3.7V = 14.8Wh)
// ─────────────────────────────────────────────
// 状態                  | 電流   | 時間割合 | 平均寄与
// ---------------------|--------|----------|----------
// BLE広告(500ms間隔)    | 8 mA   | 100%     | 8.0 mA
// GATTサーバー待機      | 3 mA   | 95%      | 2.9 mA
// バースト転送(活性)    | 40 mA  | 5%       | 2.0 mA
// CPU(バックグラウンド) | 25 mA  | 100%     | 25.0 mA  ← Foreground Service
// 画面オフ・その他      | 10 mA  | 100%     | 10.0 mA
// ─────────────────────────────────────────────
// 合計平均電流: ~48 mA
// 動作時間: 4000 mAh / 48 mA ≈ 83 hours (24h目標を大きく超過)
```

---

## 3. 📦 Data Structures (Rust Core)

### 3.1 Constants & Types
```rust
// rust/src/constants.rs
pub const MAX_TOTAL_SIZE: usize = 12_288;     // 12KB
// Conservative chunk size for BLE notifications.
// ATT MTU 512 → max notification payload = MTU - 3 (ATT opcode+handle) = 509 bytes.
// We use 480 to leave 29B headroom for the RipplePacket::Chunk postcard header
// (discriminant + index:u8 + total:u8 + Vec<u8> length varint). Measured overhead ≤ 6B;
// the remaining 23B is safety margin for OEM stacks that advertise smaller effective MTUs.
pub const CHUNK_SIZE_DEFAULT: usize = 480;
pub const CHUNK_SIZE_FALLBACK: usize = 17;    // Default ATT MTU 23 - 3 - 3 (postcard overhead)
pub const MAX_CHUNKS: usize = MAX_TOTAL_SIZE.div_ceil(CHUNK_SIZE_DEFAULT); // 26

pub type SiteID = [u8; 6];
pub type HostPubKey = [u8; 32];
pub type EdSignature = [u8; 64];
pub type PubKeyHint = [u8; 4];               // host_pubkey[0..4] for adv. packet

// rust/src/protocol/types.rs
#[derive(Serialize, Deserialize, Clone, Debug)]
pub struct SiteManifest {
    pub id: SiteID,
    pub title: String,              // Max 24 UTF-8 chars
    pub host_pubkey: HostPubKey,    // Ed25519 public key (32B)
    pub signature: EdSignature,     // Ed25519 over (title || timestamp_ms as LE u64)
    pub timestamp_ms: u64,          // Unix milliseconds — replay protection
    pub total_size: u16,            // Content byte count (≤12288)
    pub content_encoding: ContentEncoding,
    pub category: u16,              // Matches CategoryCode in adv. packet
    pub tags: Vec<String>,          // Max 3 tags, each ≤16 chars
}

#[derive(Serialize, Deserialize, Clone, Debug)]
pub enum ContentEncoding {
    Plain,   // Raw HTML/SVG
    Lz4,     // LZ4 frame compressed (only when content > 8KB)
}

#[derive(Serialize, Deserialize, Clone, Debug)]
pub enum RipplePacket {
    RequestAll,
    Chunk { index: u8, total: u8, data: Vec<u8> },
    Eof,
}
```

### 3.2 Advertising Packet Builder/Parser
```rust
// rust/src/protocol/advertising.rs
pub const MAGIC: u8 = 0xB1;

pub struct AdvPacket {
    pub site_id: SiteID,
    pub category: u16,
    pub pubkey_hint: PubKeyHint,
    pub vitality_scaled: u8,   // (vitality * 15.0) as u8, upper 4 bits
    pub is_relay: bool,
    pub has_update: bool,
}

impl AdvPacket {
    pub fn encode(&self) -> [u8; 31] { /* ... */ }
    pub fn decode(raw: &[u8; 31]) -> Result<Self, AdvError> {
        if raw[0] != MAGIC { return Err(AdvError::BadMagic); }
        let crc = crc8(&raw[0..15]);
        if crc != raw[15] { return Err(AdvError::CrcMismatch); }
        // parse fields...
    }
}
```

### 3.3 In-Memory Index (Zero Persistence)
```rust
// rust/src/index/ephemeral.rs
pub struct EphemeralIndex {
    entries: RwLock<HashMap<SiteID, IndexEntry>>,
    bloom: Mutex<BloomFilter>,   // 1-Hop dedup (capacity: 500, fp: 0.01)
    my_site_id: Option<SiteID>, // To exclude self from scan results
}

#[derive(Clone, Debug)]
pub struct IndexEntry {
    pub manifest: SiteManifest,
    pub content: Option<Vec<u8>>,      // None until GATT fetch
    pub last_seen_ms: u64,
    pub rssi: i16,
    pub vitality: f32,                 // 1.0 → 0.0 over 24h
    pub is_relay: bool,
    pub was_re_detected: bool,         // Flag for vitality boost
    pub hop_count: u8,                 // MVP: always 0
}
```

---

## 4. ⏳ Life Cycle & Decay (24-Hour Ephemeral)

### 4.1 Vitality Algorithm
```rust
// rust/src/vitality.rs
pub const VITALITY_FULL: f32 = 1.0;
pub const VITALITY_ZERO: f32 = 0.0;
pub const DECAY_PER_TICK: f32 = 1.0 / (24.0 * 120.0); // 30s ticks in 24h = 2880 ticks
pub const REDETECT_BOOST: f32 = 0.10;                  // +10% on genuine re-detect

pub fn apply_decay(entry: &mut IndexEntry) {
    entry.vitality = (entry.vitality - DECAY_PER_TICK).max(VITALITY_ZERO);

    if entry.was_re_detected {
        entry.vitality = (entry.vitality + REDETECT_BOOST).min(VITALITY_FULL);
        entry.was_re_detected = false;
    }
}

/// True re-detect: same SiteID + pubkey_hint matches + non-decreasing timestamp
pub fn is_authentic_redetect(existing: &IndexEntry, pkt: &AdvPacket) -> bool {
    existing.manifest.id == pkt.site_id
        && existing.manifest.host_pubkey[0..4] == pkt.pubkey_hint
}

pub fn should_purge(entry: &IndexEntry) -> bool {
    entry.vitality <= VITALITY_ZERO
}
```

### 4.2 Background Decay Task
```rust
pub async fn start_decay_scheduler(
    index: Arc<EphemeralIndex>,
    tx: mpsc::Sender<IndexEvent>,
) {
    let mut interval = tokio::time::interval(Duration::from_secs(30));
    loop {
        interval.tick().await;
        let mut entries = index.entries.write().await;
        let mut purged_ids = vec![];
        for (id, entry) in entries.iter_mut() {
            apply_decay(entry);
            if should_purge(entry) { purged_ids.push(*id); }
        }
        for id in &purged_ids { entries.remove(id); }
        if !purged_ids.is_empty() {
            let _ = tx.send(IndexEvent::Purged(purged_ids)).await;
        }
    }
}
```

---

## 5. 🔐 Security & Privacy Guardrails

### 5.1 Ed25519 Signature Verification (Pre-Burst)
```rust
// rust/src/crypto/ed25519.rs
use ed25519_dalek::{VerifyingKey, Signature, Verifier};

pub fn verify_manifest(manifest: &SiteManifest) -> Result<(), CryptoError> {
    let vk = VerifyingKey::from_bytes(&manifest.host_pubkey)
        .map_err(|_| CryptoError::InvalidKey)?;
    let msg = sign_message(&manifest.title, manifest.timestamp_ms);
    let sig = Signature::from_bytes(&manifest.signature);
    vk.verify(&msg, &sig).map_err(|_| CryptoError::BadSignature)
}

fn sign_message(title: &str, ts: u64) -> Vec<u8> {
    // Deterministic canonical form: title bytes + LE u64 timestamp
    let mut msg = title.as_bytes().to_vec();
    msg.extend_from_slice(&ts.to_le_bytes());
    msg
}
```

### 5.2 Content Validation Pipeline
```rust
// rust/src/protocol/validator.rs
pub fn validate_content(raw: &[u8], manifest: &SiteManifest) -> Result<Vec<u8>, ValidateError> {
    if raw.len() > MAX_TOTAL_SIZE {
        return Err(ValidateError::TooLarge(raw.len()));
    }
    let content = match manifest.content_encoding {
        ContentEncoding::Lz4 => lz4_flex::decompress_size_prepended(raw)
            .map_err(|_| ValidateError::DecompressFailed)?,
        ContentEncoding::Plain => raw.to_vec(),
    };
    // Verify decompressed size matches manifest
    if content.len() != manifest.total_size as usize {
        return Err(ValidateError::SizeMismatch);
    }
    // Must be valid UTF-8 (HTML/SVG is text)
    std::str::from_utf8(&content).map_err(|_| ValidateError::NotUtf8)?;
    Ok(content)
}
```

### 5.3 WebView Sandbox (Flutter / Dart)
```dart
// lib/ui/ephemeral_viewer.dart
//
// Inject CSP as first <meta> tag before rendering:
const _cspMeta = '<meta http-equiv="Content-Security-Policy" '
    'content="default-src \'none\'; style-src \'unsafe-inline\'; '
    'img-src \'self\' data:; font-src data:;">';

String _injectCsp(String html) {
  const head = '<head>';
  final idx = html.indexOf(head);
  if (idx < 0) return '$_cspMeta$html';
  return html.replaceFirst(head, '$head$_cspMeta');
}

// webview_flutter settings
WebViewController buildSecureController(String html) {
  return WebViewController()
    ..setJavaScriptMode(JavaScriptMode.disabled)
    ..setNavigationDelegate(NavigationDelegate(
      onNavigationRequest: (req) {
        // Block ALL navigations except the initial blob/data load
        if (req.url.startsWith('data:') || req.url == 'about:blank') {
          return NavigationDecision.navigate;
        }
        return NavigationDecision.prevent;
      },
    ))
    ..loadHtmlString(_injectCsp(html));
}
```

### 5.4 FLAG_SECURE & No-Link Enforcement
```dart
// android/app/src/main/kotlin/.../MainActivity.kt
// In onCreate(): window.addFlags(WindowManager.LayoutParams.FLAG_SECURE)

// lib/utils/share.dart — physical reference only, no routable URL
String formatPhysicalRef(SiteID id, String title) {
  final hex = id.map((b) => b.toRadixString(16).padLeft(2, '0')).join();
  return '"${title.substring(0, min(title.length, 12))}" [$hex] — physical only';
}
```

### 5.5 Android Network Security Config
```xml
<!-- android/app/src/main/res/xml/network_security_config.xml -->
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!-- Block all cleartext traffic; no remote connections ever needed -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

---

## 6. 📱 Android Implementation Requirements

### 6.1 Permissions (`AndroidManifest.xml`)
```xml
<!-- BLE permissions (API 31+) -->
<uses-permission android:name="android.permission.BLUETOOTH_SCAN"
                 android:usesPermissionFlags="neverForLocation" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE" />
<uses-feature android:name="android.hardware.bluetooth_le" android:required="true" />

<!-- Foreground Service -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_CONNECTED_DEVICE" />

<!-- No INTERNET permission — by design -->

<application
    android:usesCleartextTraffic="false"
    android:networkSecurityConfig="@xml/network_security_config">

    <service
        android:name=".BleForegroundService"
        android:foregroundServiceType="connectedDevice"
        android:exported="false"
        android:stopWithTask="false" />
</application>
```

### 6.2 Foreground Service States
```dart
// lib/ble/foreground_service.dart
enum BleState { idle, scanning, hosting, transferring }

String notificationText(BleState state, {String? title}) => switch (state) {
  BleState.idle         => '💤 BLEsite: 待機中',
  BleState.scanning     => '📡 BLEsite: 周囲をスキャン中...',
  BleState.hosting      => '🪧 掲示中: "${title ?? "無題"}" (24h)',
  BleState.transferring => '⚡ BLEsite: 転送中...',
};
```

### 6.3 App Startup & Permission Flow
```
App launch
  └─ Check Bluetooth adapter state
      ├─ Disabled → Show "Enable Bluetooth" prompt → System dialog
      └─ Enabled
          └─ Check BLUETOOTH_SCAN + BLUETOOTH_CONNECT + BLUETOOTH_ADVERTISE
              ├─ Not granted → Request permissions (rationale dialog first)
              └─ Granted
                  └─ Start BleForegroundService
                      ├─ [Scan mode] Start passive BLE scan → EphemeralIndex
                      └─ [Host mode] Start GATT server + advertising
```

---

## 7. 🎨 UI/UX Specification

### 7.1 Screen Map
```
┌─────────────────────────────────────┐
│  RadarScreen (home)                 │
│  ├─ SiteCard list (sorted vitality) │
│  ├─ Vitality pulse animation        │
│  └─ FAB: "+ Host Site"              │
├─────────────────────────────────────┤
│  EphemeralViewerScreen              │
│  ├─ Sandboxed WebView               │
│  ├─ Vitality bar (top)              │
│  └─ "Physical Only" share hint      │
├─────────────────────────────────────┤
│  HostEditorScreen                   │
│  ├─ Title input (max 24 chars)      │
│  ├─ Category picker                 │
│  ├─ Tags input (max 3)              │
│  ├─ HTML/SVG content editor         │
│  │   (Monaco-lite or plain textarea)│
│  ├─ Size meter (0–12KB live)        │
│  └─ "Start Broadcasting" button     │
└─────────────────────────────────────┘
```

### 7.2 SiteCard Visual Encoding
- **Vitality → opacity + pulse speed**: 1.0 = full opacity, fast pulse; 0.1 = 30% opacity, slow pulse
- **RSSI → distance indicator**: ≥-60dBm = "◎ Near", -60~-80dBm = "○ Medium", <-80dBm = "△ Far"
- **Category → accent color**: Food=amber, Art=violet, Event=teal, Info=slate
- **is_relay → badge**: "📶 Relayed" tag shown if hop_count > 0

### 7.3 State Management
- **Package**: `riverpod` (hooks_riverpod)  
- **Providers**: `ephemeralIndexProvider` (StateNotifier), `bleStateProvider` (StreamProvider), `hostSiteProvider` (StateNotifier)
- **Principle**: UI reads from Riverpod providers; Rust callbacks → Dart `Isolate` → StreamController → Riverpod

---

## 8. 🛠️ Flutter Package Dependencies

```yaml
# pubspec.yaml (key dependencies)
dependencies:
  flutter_rust_bridge: ^2.7.0
  flutter_blue_plus: ^1.33.0      # BLE scan/connect/GATT client
  webview_flutter: ^4.10.0        # Sandboxed viewer
  hooks_riverpod: ^2.5.1          # State management
  riverpod_annotation: ^2.3.5
  lz4: ^0.2.0                     # Content decompression (Dart side)
  permission_handler: ^11.3.0     # Runtime BLE permissions

dev_dependencies:
  build_runner: ^2.4.9
  riverpod_generator: ^2.4.0
  flutter_rust_bridge_codegen: ^2.7.0
```

```toml
# rust/Cargo.toml (key dependencies)
[dependencies]
flutter_rust_bridge = "2"
serde = { version = "1", features = ["derive"] }
postcard = { version = "1", features = ["alloc"] }
ed25519-dalek = { version = "2", features = ["rand_core"] }
lz4_flex = "0.11"
tokio = { version = "1", features = ["rt", "time", "sync", "macros"] }
bloomfilter = "1"
crc = "3"
```

---

## 9. 🧪 Testing Strategy

### 9.1 Rust Unit Tests
```rust
#[cfg(test)]
mod tests {
    #[test] fn test_adv_packet_roundtrip() { /* encode → decode → assert eq */ }
    #[test] fn test_burst_complete() { /* 26 chunks → assemble → 12288B */ }
    #[test] fn test_missing_chunk_discards_all() { /* skip chunk 1 → None */ }
    #[test] fn test_vitality_24h_decay() { /* 2880 ticks → should_purge */ }
    #[test] fn test_redetect_boost() { /* decay 12h → redetect → boost applied */ }
    #[test] fn test_signature_verify_valid() { /* sign → verify → Ok */ }
    #[test] fn test_signature_verify_tampered() { /* mutate title → Err */ }
    #[test] fn test_lz4_roundtrip() { /* compress → validate_content → match */ }
    #[test] fn test_crc8_mismatch_rejected() { /* flip bit in adv → CrcMismatch */ }
}
```

### 9.2 BLE Emulator (Mock)
```rust
// rust/src/mock/ble_emulator.rs
pub struct BleEmulator {
    pub tx: mpsc::Sender<EmulatorEvent>,
    pub rx: mpsc::Receiver<EmulatorEvent>,
}

pub enum EmulatorEvent {
    Advertise { payload: [u8; 31], rssi: i16 },
    Connect { site_id: SiteID },
    BurstChunks { chunks: Vec<(u8, Vec<u8>)> },
    PacketLoss { probability: f32 },
    SimulateBicyclePass { speed_kmh: f32, range_m: f32 },
}
```

### 9.3 E2E Physical Test Checklist
- [ ] Static 1m: connect-to-render ≤ 1.5s (12KB HTML)
- [ ] Static 5m: connect-to-render ≤ 2.0s
- [ ] Pass-by 15km/h at 10m closest: success rate ≥ 18/20 trials (90%)
- [ ] Host battery drain: ≤ 50mA average over 1h (measured with USB ammeter)
- [ ] Vitality decay: 24h later entry purged from index
- [ ] Signature tamper: reject and disconnect within 100ms

---

## 10. 🔮 Post-MVP Extension Hooks

```rust
// Multi-hop relay (Post-MVP)
pub struct IndexEntry {
    // ...existing...
    pub hop_count: u8,                        // MVP: 0 only
    pub relay_path: Option<Vec<SiteID>>,      // Loop detection
}
// Advertising byte 13 lower 4 bits: TTL (MVP: 0)

// Bloom filter scaling
pub struct BloomConfig {
    pub capacity: usize,   // MVP: 500; Post-MVP: 5000
    pub fp_rate: f64,      // 0.01
}

// Private channel stub (Noise protocol, BitChat-compatible)
// rust/src/crypto/noise_stub.rs
```

---

## 11. 🤖 Project Structure

```
blesite/
├── android/
│   └── app/
│       ├── src/main/
│       │   ├── AndroidManifest.xml
│       │   ├── kotlin/…/MainActivity.kt    # FLAG_SECURE
│       │   └── res/xml/network_security_config.xml
│       └── build.gradle                    # minSdk 31, ndk config
├── lib/
│   ├── main.dart
│   ├── ble/
│   │   ├── scanner.dart                    # flutter_blue_plus wrapper
│   │   ├── gatt_client.dart                # MTU nego + burst reassemble (Dart)
│   │   └── foreground_service.dart         # Android BLE Foreground Service
│   ├── ui/
│   │   ├── radar_screen.dart               # Home: vitality list + pulse
│   │   ├── ephemeral_viewer.dart           # Sandboxed WebView
│   │   └── host_editor_screen.dart         # Create & broadcast content
│   ├── providers/
│   │   ├── ephemeral_index_provider.dart   # Riverpod: index state
│   │   ├── ble_state_provider.dart         # Riverpod: BLE state stream
│   │   └── host_site_provider.dart         # Riverpod: current hosted site
│   ├── rust_bridge/
│   │   └── frb_generated.dart              # Auto-generated
│   └── utils/
│       ├── html_validator.dart             # ≤12KB, no external resources check
│       └── share.dart                      # Physical-ref format
├── rust/
│   ├── src/
│   │   ├── lib.rs                          # FRB entry points (pub fn)
│   │   ├── constants.rs
│   │   ├── protocol/
│   │   │   ├── types.rs                    # SiteManifest, RipplePacket, etc.
│   │   │   ├── advertising.rs              # AdvPacket encode/decode + CRC8
│   │   │   ├── gatt.rs                     # Manifest verify + burst handler
│   │   │   └── validator.rs                # Content size + encoding validation
│   │   ├── index/
│   │   │   └── ephemeral.rs                # EphemeralIndex + decay scheduler
│   │   ├── crypto/
│   │   │   └── ed25519.rs                  # Sign + Verify
│   │   ├── vitality.rs
│   │   ├── reassembler.rs                  # ChunkBuffer (index-based)
│   │   ├── power/
│   │   │   └── config.rs
│   │   └── mock/
│   │       └── ble_emulator.rs
│   ├── Cargo.toml
│   └── build.rs
├── pubspec.yaml
├── flutter_rust_bridge.yaml
├── SPEC.md
├── PLAN.md
└── TODO.md
```

### 11.1 Implementation Priorities
1. **Rust Core First**: `advertising.rs` (CRC8, encode/decode), `types.rs`, `crypto/ed25519.rs`, `reassembler.rs`, `vitality.rs` — all with unit tests.
2. **BLE Boundary Clarity**: All `flutter_blue_plus` calls in Dart. Rust receives `Uint8List`, returns `Result<T, String>`.
3. **Offline-First WebView**: `webview_flutter` with NavigationDelegate blocking all `http(s)://`. CSP injected into HTML before load.
4. **Power-Aware from Day 1**: Advertising interval + foreground service notification implemented before UI polish.
5. **No `INTERNET` permission** in `AndroidManifest.xml` — ensures OS-level network blocking.

---

> ## Appendix A: Name Candidates
> | Name | Rationale |
> |------|-----------|
> | **BLEsite** | Direct, descriptive (working title) |
> | **Ripple** | Wave propagation metaphor |
> | **PassBy** | "Surrender communication" |
> | **HereOnly** | Location-bound emphasis |
> | **Keshiki** (景色) | "Scenery" — ephemeral view |

> ## Appendix B: Known Constraints / Non-Goals (MVP)
> - ❌ No mesh relay (1-hop only in MVP)
> - ❌ No persistent storage of any kind
> - ❌ No images (binary data) in content
> - ❌ No JavaScript execution in viewer
> - ❌ No iOS / Desktop support in MVP
> - ❌ No content editing after broadcast start (create new site to update)

---

✅ **SPEC.md v0.1.0** — Reviewed & improved from v0.0.1.  
Key changes: PubKeyHint field added to adv. packet, sign_message canonical form fixed, LZ4 integrated, content validation pipeline, Flutter packages pinned, Riverpod state management specified, UI wireframes added.
