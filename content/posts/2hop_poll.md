---
date: '2025-09-07T17:40:16+09:00'
title: '2hop_poll'
---

PeerSwap は 1 ホップ直結のピア間でオンチェーン資産を交換する。実ネットには 2 ホップ経路 `u → m → v` が多く、ここでもスワップ可能性を早期に見積もりたい。

現状の `poll` メッセージ（`docs/poll.md`／`poll/messages.go`）は、互換バージョン・サポート資産・ピア許可可否・プレミアムレートの共有を目的としており、トポロジ情報（隣接ピア・チャネル）は扱っていない。

`docs/2hop.md` では §6 にて「中継ノード m が近傍情報を任意で広告し、u と v が 2 ホップ候補を発見しやすくする」方針を提案している。本 ADR は、その正確なドメイン分割・スキーマ・互換性・移行を確定する。

## 決定（Decision）

まずは以下の最小実装を採用する。

- Option A（推奨）: 既存 `poll` の JSON に、オプショナル拡張セクション `ext.neighbors_ad` を追加する。
  - 未対応ノードは未知フィールドを無視でき、後方互換が保てる。
  - 送信頻度・ライフサイクルは `poll` と同調で十分なケースが多い。

将来的に運用上の要請（サイズ・頻度・イベント駆動等）が強まる場合は、以下に拡張する。

- Option B（拡張）: `CONNECTED_PEERS`／`REQUEST_CONNECTED_PEERS` の独立メッセージを追加し、`poll` から切り離して配信制御を最適化する。

## 根拠（Rationale）

- 後方互換性: Option A は既存 `poll` の未知フィールド無視で安全。
- 実装コスト: Option A は追加型・新サービスを最少に抑えられる。
- 拡張余地: `ext.*` セクションは将来の別ドメイン拡張にも使い回せる。

## スコープとドメイン分割（Domain Separation）

- poll（既存ドメイン）
  - 互換・資産・peer_allowed・premium レートの同期。
  - 変更点は `ext` セクションのエンコード／デコードと、拡張情報の委譲のみ。

- topology/neighbors（新規ドメイン）
  - 役割: 隣接ノード広告の送受信・保存・クリーンアップ、2hop 候補集計。
  - ストア: ピア別の近傍広告（LastSeen 付）を保持。
  - インデクサ: 受信ストアを走査して「u–m–v」候補を整形し、RPC で提供。
  - 注意: 受信情報は「未検証」。最終決定は 2hop discovery（`docs/2hop.md §4.1`）で行う。

## メッセージ設計（Option A, 推奨）

`poll` payload のオプショナル拡張として `ext.neighbors_ad` を定義する。スキーマはセクション単位でバージョニングする。

```jsonc
{
  "version": 5,
  "assets": ["btc", "lbtc"],
  "peer_allowed": true,
  "btc_swap_in_premium_rate_ppm": 100,
  "btc_swap_out_premium_rate_ppm": 200,
  "lbtc_swap_in_premium_rate_ppm": 50,
  "lbtc_swap_out_premium_rate_ppm": 150,
  "ext": {
    "neighbors_ad": {
      "v": 1,               // セクションのバージョン
      "public_only": true,   // 公開チャネルのみを広告
      "limit": 20,           // entries の上限
      "entries": [
        {
          "node_id": "<pubkey-of-neighbor>",
          "channels": [
            {
              "channel_id": 1234567890,
              "short_channel_id": "x:y:z",
              "active": true
              // オプション（既定は非送信）:
              // "local_balance": 0,
              // "remote_balance": 0
            }
          ]
        }
      ]
    }
  }
}
```

### 送信ポリシー

- 送信時には LN ノードのチャネル一覧から、公開・active な近傍を抽出。
- 既定では残高情報は送信しない（プライバシー保護）。設定で `approx`／`exact` を opt-in 可。
- `limit` を超える場合はランダム／スコア順で間引き。
- 送信トリガ: `poll` 周期・起動直後の `request_poll` 応答時。

### 受信処理

- `poll` のデコード時に `ext.neighbors_ad` を検出したら、新規 neighbors サービスに委譲して保存。
- 保存は「未検証情報」として扱い、UI/RPC ではその旨を表示可能にする。
- クリーンアップは LastSeen に基づき一定時間で削除（`poll` に準拠）。

## メッセージ設計（Option B, 代替）

- 新規メッセージ型を `messages/types.go` に追加（odd 番号）。
  - `CONNECTED_PEERS`／`REQUEST_CONNECTED_PEERS`
- ペイロードは Option A の `neighbors_ad` と同一。
- 送信トリガ: 起動／チャネル open/close/active 変化／低頻度の周期。
- 利点: `poll` のサイズ増を避け、配信頻度を独立制御できる。
- 欠点: 実装・運用要素が増える（型・サービス・ストア）。

## ストレージ設計

- bbolt 新バケット（例）: `neighbors-ad-list`
- 構造体（例）: `NeighborsAdInfo`
  - `Version uint64`
  - `PublicOnly bool`
  - `Limit uint32`
  - `Entries []NeighborEntry`（`node_id`, `channels[]`）
  - `LastSeen time.Time`
- ユーティリティ:
  - `GetAllAds() map[peerId]NeighborsAdInfo`
  - `RemoveUnseen(now time.Time, olderThan time.Duration)`

## 2hop 候補インデクサ（集計ビュー）

- 出力型（例）: `TwoHopCandidate`
  - `nodeid`（v）, `intermediary_nodeid`（m）
  - `outgoing_scid`（u–m）, `incoming_scid`（m–v）
  - `spendable_msat`/`receivable_msat`: 参考値（確定は 2hop discovery）
  - `as_sender/as_receiver/paid_fee`: 既存 swap 実績をマージ
  - `swap_in_premium_rate_ppm`/`swap_out_premium_rate_ppm`: poll ストアから互換時にマージ
- フィルタリング: `active == true` のみ、`public_only` を尊重。

## 設定項目（Config）

- `advertise_neighbors`: bool（既定: true）
- `neighbors_limit`: int（既定: 20）
- `public_only`: bool（既定: true）
- `expose_balances`: enum `none|approx|exact`（既定: `none`）

## 互換性・セキュリティ（Security & Privacy）

- 後方互換: Option A は未知フィールド無視。Option B は未知メッセージ無視（BOLT#1）。
- 真偽性: 虚偽広告の可能性あり。必ず 2hop discovery（`twohop` フィールド）で実行可能金額を最終確定。
- プライバシー: 既定では残高を送らない。非公開チャネルは広告しない（`public_only`）。

## RPC/UI への露出

- 新規 API（例）: `ListTwoHopCandidates` を gRPC/HTTP に追加。
- 返却スキーマは `docs/2hop.md` の「2hop nodes」例に準拠。
- 既存 `ListPeers` は 1hop を優先表示。2hop 候補は別 API/セクションで提供。

## 実装インパクト（Modules）

- poll: `poll/messages.go`, `poll/service.go`（`ext` セクションの取扱い追加）
- neighbors: 新規 `topology/neighbors/*`（messages/store/service/indexer）
- messages: Option B 選択時のみ `messages/types.go` に新型追加
- rpc: `peerswaprpc.proto` に `ListTwoHopCandidates`（およびサーバ実装）

## 移行プラン（Migration）

Phase 1（最小導入）
- Option A 実装。`ext.neighbors_ad` のエンコード/デコード、ストア、インデクサ、読み取り専用 RPC。
- 既定設定は安全側（`public_only=true`, `expose_balances=none`）。

Phase 2（運用最適化）
- チャネルイベントに反応した即時配信（必要なら Option B 併用）。
- 候補選別（成功率・実績スコア）を導入して広告の有用性を向上。

Phase 3（拡張）
- メッセージサイズ最適化（圧縮・ページング）。`ext` に別ドメインも追加可能。

## テスト方針（Test Plan）

- メッセージ: `ext.neighbors_ad` の JSON round-trip、未知フィールド無視の互換性。
- ストア: 保存・上書き・RemoveUnseen の境界テスト。
- インデクサ: 受信データからの 2hop 候補構築、active/公開フィルタの検証。
- RPC: `ListTwoHopCandidates` のスキーマ・フィールド整合。

## 未決事項（Open Questions）

- `neighbors_limit` の既定値・上限（DoS/サイズ対策としての妥当性）。
- 送出順序のポリシー（ランダム vs スコアリング）。
- 近傍残高の `approx` 公開の具体的粒度（binning/丸め幅）。

## 付録（2hop 候補表示例）

```json
[
  {
    "nodeid": "<v_pubkey>",
    "intermediary_nodeid": "<m_pubkey>",
    "outgoing_scid": "<ch1 u–m>",
    "incoming_scid": "<ch2 m–v>",
    "spendable_msat": 0,
    "receivable_msat": 0,
    "sent": {
      "total_swaps_out": 2,
      "total_swaps_in": 1,
      "total_sats_swapped_out": 5300000,
      "total_sats_swapped_in": 302938
    },
    "received": {
      "total_swaps_out": 1,
      "total_swaps_in": 0,
      "total_sats_swapped_out": 2400000,
      "total_sats_swapped_in": 0
    },
    "total_fee_paid": 6082,
    "swap_in_premium_rate_ppm": 100,
    "swap_out_premium_rate_ppm": 100
  }
]
```

