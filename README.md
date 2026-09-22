# FIPS 互換性分析レポート

[jmcorgan/fips](https://github.com/jmcorgan/fips)(本家,「Free Internetworking Peering System」)と
[mmalmi 系 FIPS ライブラリ群](https://github.com/mmalmi/fips)(Nostr VPN フォーク)の
**真のワイヤー互換性**と、プロジェクトにおける mmalmi 系ライブラリの**採用可否**を、
ソースコードレベルで検証した包括的レポート群です。

分析基準日: 2026-09-20(各リポジトリの当該日時点の HEAD を対象)

## 対象リポジトリ

| # | リポジトリ | 役割 |
|---|-----------|------|
| 1 | [jmcorgan/fips](https://github.com/jmcorgan/fips) | 本家 Rust 実装(v0.6.0-dev,261★) |
| 2 | [jmcorgan/fips#140](https://github.com/jmcorgan/fips/issues/140) | 両実装の再統合を議論する issue(未回答) |
| 3 | [mmalmi/fips-tcp](https://github.com/mmalmi/fips-tcp) | FSP サービスデータグラム上の TCP スタック(Rust+TS, 0BSD) |
| 4 | [mmalmi/fips-ts](https://github.com/mmalmi/fips-ts) | ブラウザ向け TypeScript 実装(FMP/FSP/WebRTC/WSS) |
| 5 | [mmalmi/nostr-vpn](https://github.com/mmalmi/nostr-vpn) | フォークを利用する製品(Tailscale 型 VPN, 1159★) |
| 6 | [mmalmi/fips](https://github.com/mmalmi/fips) | データプレーン書き換えフォーク(`nvpn-fips-*` crate 群) |

## レポート一覧

| ファイル | 内容 |
|---------|------|
| [reports/01-ecosystem-landscape.md](reports/01-ecosystem-landscape.md) | エコシステム全体像・各リポジトリの役割・保守体制・配布形態 |
| [reports/02-wire-compatibility.md](reports/02-wire-compatibility.md) | FMP/FSP v0 ワイヤーフォーマットのコードレベル突合結果 |
| [reports/03-divergence-matrix.md](reports/03-divergence-matrix.md) | 本家 vs フォーク vs TS の機能・拡張差分マトリクス |
| [reports/04-compat-hazards.md](reports/04-compat-hazards.md) | 実害のある互換性ハザードと回避策のチェックリスト |
| [reports/05-adoption-verdict.md](reports/05-adoption-verdict.md) | mmalmi 系ライブラリ採用可否の判定・条件・推奨構成 |
| [reports/06-fips-big-picture.md](reports/06-fips-big-picture.md) | 【超大作】FIPS の本質・技術史的位置づけ・できること・限界・将来シナリオ |
| [reports/07-usecase-home-pc-and-phone.md](reports/07-usecase-home-pc-and-phone.md) | 実践ユースケース: 自宅PC × 外出スマホを FIPS(nostr-vpn)で繋ぐ方法と限界 |

## 結論の要約(TL;DR)

- **ワイヤー互換性は「ほぼ真」**。FMP/FSP v0 の全メッセージ型・ハンドシェイク・エンコード規則はコード上で一致。mmalmi 側の拡張(FSP msg 0x15–0x17、direct-FSP ヘッダフラグ 0x08、サービスポート 257、WebRTC/WebSocket トランスポート)はすべて後方互換にネゴシエートされ、本家ノードとの混在で通信は破壊されない。
- **ただし「同じ FIPS ではない」**。内部アーキテクチャ・API・トランスポートセットが大幅に異なり、fork は本家の上流追従ではなく独立進化(issue #140 で「再統合か並行維持か」が未決)。本家しか持たない機能(Nym/SOCKS5/ネイティブ API/gateway/OpenWrt)もある。
- **採用可否**: ライセンス(MIT/0BSD)・実績(nostr-vpn 製品で出荷中)・公開(crates.io に `nvpn-fips-*`)の面で利用は**可能**。ただし ①単一メンテナ+高頻度リリース、②npm 未公開(git ピン必須)、③ワイヤー上の拡張が本家と未調整、④API 不安定(0.x 系・exact pin 前提)のリスクを踏まえ、**バージョン固定 + 相互接続テストを CI に組み込む**ことが前提条件。

ライセンス: レポート本文は CC0 / 引用部分は各元プロジェクトのライセンスに従う。
