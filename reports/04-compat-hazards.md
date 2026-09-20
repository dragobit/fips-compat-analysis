# 04. 互換性ハザード一覧と回避策

本家 ↔ フォーク ↔ fips-ts を混在させた際に実害が出る/出得る点と回避策。

## H-1. mmalmi の Nostr advert が本家ノードに拒否される【中】

**事象**: フォークの `OverlayTransportKind` に `webrtc`/`websocket` があり、
これらのエンドポイントを含む kind 37195 advert を公開すると、本家実装は
serde enum の未知バリアントで **advert 全体のデシリアライズに失敗**
(`BootstrapError::InvalidAdvert`)。advert 内に本家が使える `udp`/`tcp`/`tor`
エンドポイントがあってもまとめて捨てられる。

**影響**: 本家ノード → mmalmi ノードへの Nostr 経由ディスカバリが成立しない
(直接到達/静的ピア設定があれば影響なし)。

**回避策**:
- 混在メッシュでは mmalmi 側の advert に `webrtc`/`websocket` エンドポイントを
  含めない、または discoverable に udp: エンドポイントだけの advert を別スコープで出す
- 静的ピア / 明示的アドレス設定で Nostr ディスカバリに依存しない
- 上流修正の方向性: 本家側で未知 transport を「unknown に潰す」寛容な
  デシリアライズにするか、`#[serde(other)]` 的フォールバックを提案

## H-2. 番号空間の未調整【中・将来リスク】

mmalmi が先取りした番号:
- FSP msg_type `0x15` (EndpointData), `0x16`/`0x17` (TraversalOffer/Answer)
- FSP ヘッダフラグ bit3 `0x08` (DIRECT_TRANSPORT)
- SessionSetup body flag bit2 `0x04` (direct_fsp_transport)
- サービスポート `257` (link negotiation)

**事象**: 本家が将来同じ番号を別用途に割り当てると衝突。
issue #140 で提案されたサービスポートレジストリ調整が成立するまで継続するリスク。

**回避策**: 混在環境でこれらの拡張に依存しない設計にする(特に EndpointData を
アプリの必須パスにしない;DataPacket + アプリポート 1024+ で代替可能)。

## H-3. direct-FSP(0x08)の見え方【低】

mmalmi 同士がネゴシエートした FSP 確立レコードは `flags=0x08` で**隣接トランス
ポート上に直接**載る(FMP カプセル化をバイパス)。本家ノードへの送出は
body flag 0x04 の相互広告がない限り発生しないため実害はほぼ無いが、
中継ノードが mmalmi→本家のパケットを「フラグ付きのまま」見る構成では
本家は bit3 を未解釈で通過(パース上は無害)。モニタリング・パケット
インスペクション系を自作する場合は両フラグを認識しておく。

## H-4. Noise 方言の再実装罠【低・新規実装のみ】

独自 Noise 実装を書く場合(非 Rust/TS)、レポート 02 の 4 つの仕様逸脱
(0x02 パリティ正規化、空 AAD、XK msg1 タグ無し、IK msg2 の se オペランド入替)
を踏襲しないと**ハンドシェイクが成立しない**。エラーメッセージではなく
単に失敗するためデバッグが難しい。

## H-5. ドキュメントと実装のズレ【低】

mmalmi 側ワイヤー仕様書に 2 件の誤記(LookupRequest サイズ、session report
ボディ)、fips-ts の rust-compat に ver=1 の誤記。**実装ではなく仕様書を
信じて再実装すると非互換コードができる**。常にコードと fixture
(`fips-ts/fixtures/rust-vectors/`、`fips-tcp/rust/fips-tcp/protocol/*.json`)を
正とする。

## H-6. API 非互換【高・ただし設計上当然】

本家 `fips` crate(単一 src/)とフォーク `nvpn-fips-*`(4 クレート)は
**API が完全に別物**。同じアプリコードは両者で動かない。`fips-ts` の
`FipsNode` も独自 API。どの実装を選ぶかは依存関係レベルの決定であり、
ワイヤー互換があっても実装を差し替えるにはアダプタ層が要る。

## H-7. fips-tcp は「FIPS の TCP」ではなく「FIPS 上の TCP」【情報】

`fips-tcp` は FSP サービスポート上に独自 TCP セグメントを運ぶ**上位層
プロトコル**(tcp-fips-v1)。FIPS のリンク TCP トランスポートとは無関係。
本家 FIPS 上でも理論上動作する(サービスポート API さえあれば)が、
公式にテストされているのは nvpn-fips-core 上のみ。`nvpn-fips-tcp` が
smoltcp との相互運用ベクタを持つのはセグメント互換性の保証。

## 混在時の推奨ガードレール

1. **混在メッシュでは DataPacket + アプリポート(1024–65535)のみを必須パスにする**
2. mmalmi advert には `webrtc`/`websocket` を載せない(or 別 scope で分離)
3. 相互接続テストを CI で回す: mmalmi↔本家、mmalmi↔fips-ts、本家↔fips-ts の
   ハンドシェイク + echo のみでよい(EndpointData 等は使わない)
4. バージョンは完全一致ピン(`=x.y.z`)。nvpn-fips 系はリリースが速く、
   nostr-vpn 自身も exact pin で防御している
5. 仕様は docs ではなくコード + JSON ベクタを正とする
