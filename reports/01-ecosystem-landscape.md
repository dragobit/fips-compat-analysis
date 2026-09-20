# 01. エコシステム全体像

## FIPS とは

**FIPS (Free Internetworking Peering System)** は、Nostr アイデンティティ
(secp256k1 / Schnorr 鍵対 = npub)をノードアドレスとして使う、自己組織化・
パーミッションレスの暗号化メッシュネットワーク。Johnathan Corgan
(@jmcorgan)が主導する本家実装が、NAT 越え・素の Ethernet/WiFi/Bluetooth
まで含む任意のトランスポート上で、IPv6 トンネリング(既存アプリ互換)と
ネイティブ API(公開鍵アドレッシング)の両方を提供する。

プロトコルスタックは 3 層:

```
アプリケーション(IPv6 パケット or サービスポートデータグラム)
  └─ FSP (FIPS Session Protocol) — Noise XK, エンドツーエンド暗号
       └─ FMP (FIPS Mesh Protocol) — Noise IK, ホップバイホップ暗号 + メッシュルーティング
            └─ トランスポート — UDP / TCP / Ethernet / Tor / Nym / BLE / WebSocket / WebRTC ...
```

## 系譜

```
jmcorgan/fips (本家, MIT, v0.6.0-dev)
  │
  ├─ mmalmi/fips ───────── データプレーン全面書き換えフォーク
  │    (nvpn-fips-core/endpoint/identity を crates.io 公開)
  │    └─ 使用: mmalmi/nostr-vpn (製品)
  │
  ├─ mmalmi/fips-ts ────── ブラウザ向け TS 再実装
  │    (interop 対象は mmalmi/fips;「original FIPS 0.4.1 との互換」を謳う)
  │
  └─ mmalmi/fips-tcp ──── FSP サービスデータグラム上の TCP スタック
       (nvpn-fips-tcp / nvpn-fips-tcp-endpoint を crates.io 公開)
```

### フォークの動機(両者の公称値)

mmalmi 側は nostr-vpn の高レート VPN トラフィック向けにデータプレーンを
「owner-affine・エンドツーエンドバッチ処理」へ再設計。issue #140 の
同一ハーネス計測で、TCP 3.2 Gbit/s vs 本家 1.1 Gbit/s、5 Gbit/s UDP 供給で
2.69 vs 1.60 Gbit/s、同時 ping p99 を 2,496 ms → 61 ms に改善、
CPU/Gbit を 38–55% 削減と報告(未検証だが両側合意の計測として提出)。

## 各リポジトリのプロフィール(2026-09-20 時点)

| 項目 | jmcorgan/fips | mmalmi/fips | mmalmi/fips-ts | mmalmi/fips-tcp | mmalmi/nostr-vpn |
|------|---------------|-------------|----------------|-----------------|------------------|
| バージョン | 0.6.0-dev | 0.3.0-dev(クレートは nvpn-fips-core 0.4.82) | @fips/core 0.0.42 | nvpn-fips-tcp 0.2.2 | nvpn(製品) |
| ライセンス | MIT | MIT | MIT | 0BSD | MIT |
| GitHub ★ | 261 | 3 | 0 | 0 | 1,159 |
| 直近 push | 2026-09-19 | 2026-09-15 | 2026-09-05 | 2026-09-15 | 2026-09-17 |
| 主要コミッタ | J. Corgan + 3名 | M. Malmi 単独 | M. Malmi(+自動生成) | M. Malmi(+自動生成) | M. Malmi(+自動生成) |
| crates.io | **未公開**(`fips`名は無関係クレートが占有) | `nvpn-fips-*` 公開済 | — | `nvpn-fips-tcp*` 公開済 | `nvpn` 公開 |
| npm | — | — | **未公開**(git ピン/dist 追跡) | `@fips/tcp` 未公開 | — |
| SECURITY.md | あり | なし | なし | なし | — |

## 本家の特徴(フォークに無いもの)

- **トランスポート**: Nym ミックスネット、SOCKS5 出力
- **ネイティブ・プロセス API** (`src/native/`): ローカルプロセス向け
  データグラム API(コントロールソケット経由)。「信頼性レイヤ無し」を明示し
  ROD(Reliable Object Delivery)は v2 予定
- **fips-gateway / fipsctl / fipstop** CLI 群、**OpenWrt パッケージング**
  (802.11s バックホール、open access SSID)
- **ポートレジストリの公式規則**: 0–255 プロトコル予約、256–1023 標準
  サービス(256=IPv6 shim)、1024–65535 アプリ

## フォーク(mmalmi)の特徴(本家に無いもの)

- **トランスポート**: WebSocket(WSS シード + 9バイトキーヒント)、
  WebRTC(RTCDataChannel)、シミュレーション輸送、`link_negotiation`
- **FSP 拡張**: `EndpointData`(0x15,ポート無しアプリペイロード)、
  `TraversalOffer/Answer`(0x16/0x17,FSP 内ホールパンチシグナリング)、
  SessionSetup body flag 0x04 → direct-FSP ヘッダフラグ 0x08
- **サービスポート 257**: 汎用リンクネゴシエーション(WebRTC offer/answer
  等を FSP DataPacket で運ぶ)
- **接続回復/ローミング**: パス回復・スパースセッション回復
  (CHANGELOG 0.4.81–0.4.82)
- **データプレーン**: オーナーアフィン・バッチ化・バッファ再利用
  (issue #140 のベンチマーク主題)

## ガバナンス状況

- [jmcorgan/fips#140](https://github.com/jmcorgan/fips/issues/140)
  「Reconciling the original FIPS implementation with the Nostr-VPN rewrite」:
  本家側からの提案で 3 案(①フォークを新ベースに採用、②2実装並行+ワイヤー仕様
  とサービスポートレジストリで調整、③最適化のみ個別移植)を提示し、
  キャパビリティ・ハローによる機能ネゴシエーションを提案。**2026-09-20 時点で
  コメント 0 件・未回答**。つまり公式な調整ルートは未確立。
- mmalmi 側は「FMP/FSP v0 ワイヤーフォーマットの維持」を公称し、issue #140 の
  ベンチも「相互運用する」ことを前提として計測されている。
- サービスポートレジストリ・FSP メッセージ型の番号空間は現状**非公式に
  mmalmi が先取り割当**している状態(257, 0x15–0x17)。本家が将来同じ番号を別用途に
  割り当てる衝突リスクが残る。
