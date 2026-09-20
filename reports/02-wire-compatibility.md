# 02. ワイヤー互換性:コードレベル突合

## 検証方法

両実装のワイヤーフォーマット仕様書(`fips/docs/reference/wire-formats.md` vs
`mmalmi-fips/docs/design/fips-wire-formats.md`)を diff した上で、**実際のエンコード/
デコードコードの定数・サイズ・フラグ**を突合。ドキュメント差分と実コード差分を区別して報告する。

対象:
- `fips/src/proto/fmp/wire.rs`, `fips/src/proto/fsp/wire.rs`, `fips/src/proto/lookup/wire.rs`, `fips/src/proto/mmp/wire.rs`
- `mmalmi-fips/crates/fips-core/src/node/wire.rs`, `proto/fsp_wire.rs`, `proto/protocol/{session,discovery}.rs`, `proto/mmp/report.rs`
- `fips-ts/packages/core/src/{fmp,fsp}/wire.ts`, `docs/rust-compat.md`

## 結論: FMP/FSP v0 はバイト互換

両実装とも `FMP_VERSION = 0`、`FSP_VERSION = 0` で、以下が一致することを確認:

| 要素 | 本家 | mmalmi フォーク | 一致 |
|------|------|----------------|------|
| 共通プレフィックス `[ver+phase:1][flags:1][payload_len:2 LE]` | ✓ | ✓ | ✅ |
| FMP phase: 0x0 Established / 0x1 IK msg1 / 0x2 IK msg2 | ✓ | ✓ | ✅ |
| FMP 確立フレーム: 16B 外ヘッダ(AAD) + 5B 内ヘッダ + AEAD | ✓ | ✓ | ✅ |
| Noise IK msg1=114B 総長 / msg2=69B 総長 | ✓ | ✓ | ✅ |
| FMP msg_type 表(0x00/0x01/0x02/0x10/0x20/0x30/0x31/0x50/0x51) | ✓ | ✓ | ✅ |
| TreeAnnounce / FilterAnnounce / Lookup*/Disconnect/両Report | ✓ | ✓ | ✅ |
| LookupRequest = `46 + 16n` バイト | ✓(コード) | ✓(コード) | ✅※ |
| Session-layer SenderReport=46B / ReceiverReport=66B body | ✓ | ✓ | ✅※ |
| FSP phase: 0x1–0x3 = Noise XK msg1–3 | ✓ | ✓ | ✅ |
| FSP 確立ヘッダ 12B + 6B 内ヘッダ(ts+msg_type+inner_flags) | ✓ | ✓ | ✅ |
| FSP flags: CP=0x01, K=0x02, U=0x04 | ✓ | ✓ | ✅ |
| FSP msg_type: 0x10 DataPacket … 0x14 CoordsWarmup, 0x20–0x22 エラー | ✓ | ✓ | ✅ |
| SessionFlags: 0x01 request_ack, 0x02 bidirectional | ✓ | ✓ | ✅ |
| サービスポート 256 = IPv6 shim | ✓ | ✓ | ✅ |
| NodeAddr = SHA-256(x-only pubkey)[0..16] | ✓ | ✓ | ✅ |
| Nostr advert: kind 37195, `fips-overlay-v1`, v1 | ✓ | ✓ | ✅ |
| Ethernet データフレーム `0x00 || len:u16LE || FMP` / ビーコン `0x01||0x01||xonly32` | ✓ | ✓(+optional scope trailer) | ✅ |

※ それぞれ「ドキュメント誤記」があったがコードは一致(下記)。

## 「FIPS Noise」は独自方言 — 第3実装の注意点

fips-ts の `docs/rust-compat.md` が明示する通り、Rust 実装の Noise は
**Noise Protocol 仕様からの逸脱**があり、fips-ts はそれを忠実に再現している:

1. **Pre-message MixHash でレスポンダ static を 0x02 パリティに正規化**
   (initiator が x-only 公開鍵しか知らなくても一致するため)
2. **ハンドシェイク中の EncryptAndHash は空 AAD**(Noise 仕様は AAD=h)
3. **XK msg1 は末尾の空ペイロード AEAD タグを付けない**(33B;仕様なら49B)
4. **IK msg2 の `se` は実際には `DH(e_init, s_resp)` を計算**
   (仕様では `DH(s_init, e_resp)`;XK msg3 の `se` は標準通り)
5. ハンドシェイクペイロードは u64 起動エポック(8B)を AEAD 暗号化して同梱
   — ピア再起動検知に使用
6. Nonce = 4B ゼロ ‖ 8B LE カウンタ、リプレイ窓 2048(WireGuard 式)

**含意**: 「Noise_XK_secp256k1_ChaChaPoly_SHA256 と書けば互換」では**ない**。
新実装(または本家の別言語ポート)を作る場合はこの方言を踏襲する必要がある。
mmalmi フォークは本家から継承しているため問題ないが、独立実装では罠になる。

## mmalmi 側の拡張と後方互換性

| 拡張 | ワイヤー上の表現 | 本家ノードとの混在時 |
|------|----------------|---------------------|
| `direct_fsp_transport` ネゴシエーション | SessionSetup/SessionAck **body** flag 0x04 | 本家は body flag の未使用ビットを無視 → 安全に未対応扱い。両端が広告した時のみヘッダ flag 0x08 を使用する設計なので本家に 0x08 が飛ばない |
| `EndpointData` (FSP msg 0x15) | 不明 msg_type | 本家は未知 msg_type をドロップ → 機能しないが破壊的でない |
| `TraversalOffer/Answer` (0x16/0x17) | 同上 | 同上 |
| サービスポート 257 リンクネゴシエーション | 通常の DataPacket (0x10) | ポート未登録としてドロップ → 安全 |
| OverlayTransportKind `webrtc`/`websocket` | Nostr advert JSON の enum 値 | **本家は serde が未知バリアントで advert 全体を拒否** → 要フォロー(レポート 04) |
| Ethernet scoped beacon (末尾に scope trailer) | 34B 超のビーコン | 本家 `parse_beacon` は `len >= 34` だけを見る → 互換 |

## 発見したドキュメント誤記(実害なしだが仕様書として要注意)

1. **`mmalmi-fips/docs/design/fips-wire-formats.md`**: LookupRequest を
   `303 + 16n` バイトと記載。実コードは本家と同じ `46 + 16n`
   (`discovery.rs` の `Vec::with_capacity(46 + depth*16)` および encode 実装)。
   303 は別シリアライズ形(署名付き等)の混入か古い版の名残。
2. **同**: セッション SenderReport/ReceiverReport を「リンク層と同じボディ形式」
   と記載(=msg_type 込み 48/68B に読める)。実コードは本家と同じ 46/66B
   (`SESSION_SENDER_REPORT_SIZE = 46` 等、msg_type は FSP 内ヘッダにのみ)。
3. **`fips-ts/docs/rust-compat.md`**: FMP 共通プレフィックスを `ver=1` と記載。
   実コード(`fmp/wire.ts`、両 Rust 実装)は `version=0`。

## issue #140 との関係

issue #140 は「両実装は FMP/FSP v0 を共有し相互運用する」を前提に
ベンチマークを提示しており、本レポートのコード突合はそれと整合する。
提案されている「標準 FSP サービスポート上の暗号化キャパビリティ hello」は
現時点では未実装(両側のコードに該当メッセージ無し)。導入されれば
`EndpointData`・port 257・direct-FSP などの拡張検出が明示化できる。
