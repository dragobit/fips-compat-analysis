# 03. 機能・拡張差分マトリクス

凡例: ✅=実装あり / ❌=なし / △=部分的・別形

## トランスポート層

| トランスポート | jmcorgan/fips | mmalmi/fips | fips-ts | 備考 |
|----------------|:---:|:---:|:---:|------|
| UDP | ✅ | ✅ | ❌(ブラウザ不可) | NAT 越えパンチ対応は両側あり |
| TCP | ✅ | ✅ | ❌ | ストリームフレーミング共通(`payload_len`) |
| Raw Ethernet | ✅ | ✅ | △(仮想 Ethernet, EtherType 0x2121) | フォークは scope trailer 付きビーコン(後方互換) |
| Tor(onion) | ✅ | ✅ | ❌ | |
| Nym | ✅ | ❌ | ❌ | 本家のみ |
| SOCKS5 | ✅ | ❌ | ❌ | 本家のみ |
| BLE (L2CAP) | ✅ | ✅ | ❌ | フォークに BLE v2 設計文書あり |
| WebSocket | ❌ | ✅ | ✅(WSS シード + キーヒント) | フォーク/TS 専用拡張 |
| WebRTC | ❌ | ✅(`webrtc-transport` feature) | ✅ | フォーク/TS 専用拡張 |
| メモリ/シミュレーション | △(loopback) | ✅(`sim-transport`) | ✅(transport-memory) | テスト用 |

## FSP セッション層メッセージ型

| msg_type | 名称 | 本家 | フォーク | fips-ts |
|----------|------|:---:|:---:|:---:|
| 0x10 | DataPacket(ポート多重サービス) | ✅ | ✅ | ✅ |
| 0x11 / 0x12 | SenderReport / ReceiverReport | ✅ | ✅ | ✅ |
| 0x13 | PathMtuNotification | ✅ | ✅ | △ |
| 0x14 | CoordsWarmup | ✅ | ✅ | ✅ |
| 0x15 | EndpointData(ポート無しアプリデータ) | ❌ | ✅ | ✅ |
| 0x16 / 0x17 | TraversalOffer / TraversalAnswer | ❌ | ✅ | △(経由: link negotiation) |
| 0x20–0x22 | CoordsRequired / PathBroken / MtuExceeded | ✅ | ✅ | ✅ |

## サービスポートレジストリ

| ポート | 用途 | 割当主体 |
|--------|------|---------|
| 0–255 | プロトコル予約 | 本家規則 |
| 256 (0x100) | IPv6 shim(ヘッダ圧縮) | 本家規則・両実装共通 |
| 257 (0x101) | 汎用リンクネゴシエーション(JSON offer/answer) | **mmalmi が先取り**・本家未承認 |
| 258–1023 | 標準サービス予約(未割当) | — |
| 1024–65535 | アプリケーション | 両者共通(hashtree=7001 等) |

## ノード・API 面

| 機能 | 本家 | フォーク | 備考 |
|------|:---:|:---:|------|
| IPv6 アダプタ / TUN | ✅ | ✅(`FipsEndpoint` + `without_system_tun` も可) | |
| ネイティブ・データグラム API(ローカルプロセス向けソケット風) | ✅(`src/native`) | ❌(代わりに `FipsEndpoint` インスプロセス API + EndpointData) | issue #140 で「残る主要な差異」と指摘 |
| ゲートウェイ / fipsctl / fipstop | ✅ | △(fipsctl, fips-gateway は Cargo.toml に残存) | |
| OpenWrt パッケージング | ✅ | ❌ | |
| メッシュファイアウォール / host firewall | ✅ | ✅ | |
| 接続回復・ローミング・スパースセッション | △ | ✅(0.4.81/0.4.82 で強化) | nostr-vpn のモバイル用途由来 |
| reply_learned ルーティングモード | △ | ✅ | fips-ts が mirror |
| MMP メトリクス(Sender/ReceiverReport, spin bit, ECN) | ✅ | ✅ | |
| Nostr ディスカバリ(kind 37195 advert) | ✅ | ✅ | advert schema は後方互換的(フィールド差は下記) |
| UDP ホールパンチ | ✅(kind 21059 signal + punch magic) | ✅(nostr/traversal + FSP 内 0x16/0x17) | シグナリング経路が異なる |

## Nostr advert(kind 37195)スキーマ差分

```jsonc
// 共通
{ "identifier": "fips-overlay-v1", "version": 1,
  "endpoints": [{"transport": "udp|tcp|tor", "addr": "..."}],
  "stunServers": ["stun:..."] }

// 本家のみのフィールド: "signalRelays": [...]
// フォークのみの transport 値: "webrtc", "websocket"
```

- フォーク → 本家: 未知フィールドは serde が無視するので本家 advert は読める
  (ただし `signalRelays` を失う → 本家側の NAT traversal シグナリング経路が欠落)。
- 本家 → フォーク: `webrtc`/`websocket` を含む advert は本家側で
  enum デシリアライズ失敗 → **advert 全体が無効化**。udp:nat 等の
  使えるエンドポイントが同居していても捨てられる(レポート 04 の H-1)。

## ライフサイクル・リリース形態

| | 本家 | フォーク |
|---|------|---------|
| リリース | 0.2.x–0.5.x のリリースノート体系、master/maint/next ブランチ | 数値連番を高速回転(0.4.79→0.4.82 を数日で),セマンティクスは製品都合 |
| 配布 | git ソース(crates.io の `fips` 名は別物) | crates.io `nvpn-fips-*`、git.iris.to が正本・GitHub はミラー |
| 依存ピン | — | nostr-vpn は `=0.4.82` の完全一致ピン(速いリリースへの防御) |
