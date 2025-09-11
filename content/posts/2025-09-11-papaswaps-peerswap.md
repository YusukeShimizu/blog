---
date: '2025-09-11T09:43:12+09:00'
title: "papaswaps-peerswap"
---

以下は **PeerSwap に PapaSwap（1tx Submarine Swap/Taproot）を統合するためのアーキテクチャ設計とプロトコル仕様** です。
目的は、既存のPeerSwapの「Swap Out」（LN→オンチェーン）フローに、PapaSwapの1取引・Taprootベースのスマートコントラクトを導入します。これにより、**手数料・コンファ時間・失敗時の運用コスト**を理論上半減できます。

> **方向の対応づけ**
> PapaSwap は *Lightning で支払い → オンチェーンで受け取り* のサブマリン・スワップです。
> これは PeerSwap の **Swap Out**（イニシエータ＝LN支払側、レスポンダ＝オンチェーン資金拠出側）に一致します。
> よって本仕様は Swap Out の「1tx/Taproot 変種」として定義します。

---

## 0. スコープと前提

* **スコープ**：Bitcoin（mainnet/testnet/signet）のTaproot（BIP340/341/342）を必須とし、PeerSwapの **Swap Out** にPapaSwapを統合。

  * Liquidについては、Elements/Liquid側のTaproot/Tapscript実装状況・互換性差分があるため、本ドラフトでは **将来拡張（オプション）** とします（セクション12参照）。
* **非スコープ**：Nostr/NWCは通信・資金ソースの一形態としては有用だが、PeerSwapのピア間通信は **BOLT#1 カスタムメッセージ**のまま据え置く。
* **用語**：RFC2119のMUST/SHOULD/MAYを使用する。

---

## 1. 目的とゴール

* **取引数削減**：既存PeerSwap（2取引：opening/claim）⇒ Papa変種（**1 取引：funding のみ**）。
* **コスト削減**：1取引＋Taprootにより **手数料・時間**の削減。
* **安全性**：ユーザ（LN支払側）が **前受け署名**（serverのtx3/tx4署名）と **事前バリデーション**を完了した後のみインボイスを支払う設計で、**信頼最小化**を保つ。
* **後退しない互換性**：ピアがPapaに未対応でも、**通常の Swap Out** へフォールバック可能。

---

## 2. 高レベル・アーキテクチャ

PeerSwapノードに新モジュール **`PapaEngine`** を追加する。

* **Contract Builder**：Taprootキー/ツリー生成、アドレス生成、PSBTv2（BIP370）作成。
* **Signer**：BIP340 Schnorr（x-only）署名器。
* **PSBT 協調**：tx1/tx2/tx3/tx4の **部分署名配布**（ANYONECANPAY等）を規定。
* **State Machine**：PapaSwap専用の状態遷移（セクション8）。
* **Watcher**：`tx1` 観測時の `tx4` 自動放流、CPFP/RBF支援。

**データフロー（概略）**

```
Initiator(User/Taker) ──SwapOut Req(papa)──▶ Responder(Server/Maker)
        │                                           │
        │◀─ SwapOut Agreement(+fee invoice) ────────│
        │───(fee paid)────────────────────────────▶ │
        │◀─ Papa Contract Offer(+swap invoice,      │
        │      funding UTXO, taptree, server sigs)──│
        │─── Papa User Sigs(tx1/tx2) ─────────────▶ │
        │◀─ opening_tx_broadcasted(tx0 broadcast)── │
        │───(tx0 confirm→pay swap invoice)────────▶ │
        │  (preimageでkeypath/leaf2経由花開く)       │
```

---

## 3. Taproot コントラクト（Papa の核）

### 3.1 Tapscript ポリシー（2 つの葉）

* **Leaf1（2-of-2 署名）**
  `user_pubkey CHECKSIGVERIFY server_pubkey CHECKSIG`
* **Leaf2（preimage + 2-of-2 署名）**
  `OP_SHA256 <payment_hash> OP_EQUALVERIFY user_pubkey CHECKSIGVERIFY server_pubkey CHECKSIG`

> Leaf version は `0xC0`（Tapscript v0）を使用。

### 3.2 Keypath（低コスト経路）

* **内部鍵**：`X = P_user ⊕ Q_invoice`

  * `Q_invoice` は **「インボイスの preimage を私鍵とみなして得た公開鍵」**。
  * ユーザはインボイス支払い後にpreimage（= `q`）を得るので、`x = p + q` でkeypath署名可能。
* **メリット**：keypathはscriptpathより重量が軽く、**支出手数料を節約**可能（最適経路）。
* **フェイルセーフ**：万一 `Q_invoice` が不正でも、Leaf2 + tx3/tx4で資金回収可（サーバのプロトコル違反）。

---

## 4. 4 種の取引（tx0～tx4）と署名方針

* **tx0（funding）**：**唯一のオンチェーン送金**（Makerがブロードキャスト）。

  * 出力：Taproot（scriptPubKeyはkey＋taptree）。
* **tx1（re-lock）**：**サーバの競合用**。Leaf1で入力を消費し、**同一コントラクト**へ再ロック。

  * **ユーザが事前署名**（SIGHASH\_ALL/DEFAULT）→ サーバは後から自署名を足して完成。
* **tx2（server sweep after CSV）**：Leaf1で **2016ブロック相対ロック**後にサーバが回収。

  * **ユーザが事前署名**（SIGHASH\_ALL/DEFAULT, `nSequence=2016`）→ サーバは後署名で完成。
  * BIP68のrelative locktimeを利用（Trans. ver≥2）。
* **tx3（user spend from tx0）**：Leaf2（preimage必須）で **即時引き出し**。

  * **サーバが事前署名**（`SIGHASH_ANYONECANPAY|NONE`）→ ユーザがpreimageと自署名・出力を自由に付与。
* **tx4（user spend from tx1）**：tx1が通った場合の **救済用**（Leaf2、即時引き出し）。

  * **サーバが事前署名**（`SIGHASH_ANYONECANPAY|NONE`）。

> **SIGHASH 設計理由**
>
> * `tx1/tx2`：サーバがユーザの「汎用署名」を得ないように **SIGHASH\_ALL**（出力固定）でユーザが先署名。サーバはその 2 取引しか作れない。
> * `tx3/tx4`：ユーザが **任意出力**へ送れるように、サーバ署名は `ANYONECANPAY|NONE`（入力のみ拘束）。

---

## 5. 機能交渉と互換性

### 5.1 プロトコルバージョンと機能ビット

* `protocol_version` を **6**（例）に引き上げ、**Papa モード**機能を表すビット（またはフラグ）を導入。
* 同機能に未対応の相手には、**従来の Swap Out** へフォールバック。

### 5.2 既存メッセージの拡張 or 新規メッセージ

PeerSwapのカスタムメッセージ範囲（`42069–42085`）に合わせ、以下を定義する。

* 既存：`swap_out_request (42071)` と `swap_out_agreement (42075)` を **拡張**（Papa用フィールドを追加）。
* **新規**：

  * `papa_contract_offer (42083)`：サーバ→ユーザ（契約情報＋tx3/tx4署名）
  * `papa_user_sigs (42085)`：ユーザ→サーバ（tx1/tx2のPSBT部分署名）

> 既存の `opening_tx_broadcasted (42077)` は再利用（tx0 の txid/vout/インボイス再提示）。

---

## 6. メッセージ仕様（JSON／BOLT#1 カスタム）

### 6.1 `swap_out_request`（42071/拡張）

```json
{
  "protocol_version": 6,
  "swap_id": "string-32B",
  "asset": "",                      // Bitcoin の場合は空
  "network": "mainnet|testnet|signet",
  "scid": "string",
  "amount": "uint64",               // sats
  "variant": "taproot_papa",        // 新規: Papa を要求
  "pubkey_tap_xonly": "hex32",      // ユーザの x-only BIP340 公開鍵 (P_user)
  "csv_delay_blocks": 2016          // 推奨デフォルト。交渉可
}
```

**要件（抜粋）**

* `variant=taproot_papa` を必須。
* `pubkey_tap_xonly` は32B x-only（BIP340）。
* `amount>0 && amount<=channel_balance`。
* `csv_delay_blocks` は1008以上推奨（デフォ2016）。

### 6.2 `swap_out_agreement`（42075/拡張）

```json
{
  "protocol_version": 6,
  "swap_id": "string-32B",
  "variant": "taproot_papa",
  "server_pubkey_tap_xonly": "hex32",  // Leaf用の server_pubkey
  "payreq_fee": "string-bolt11",       // 既存の fee インボイス
  "premium": "uint64",                 // 任意の上乗せ手数料
  "csv_delay_blocks": 2016,            // 決定値
  "min_conf_tx0": 3                    // tx0 の既定必要承認数（交渉可）
}
```

**要件（抜粋）**

* Responder（Maker）は各スワップで異なるエフェメラル鍵の使用を推奨する。
* `payreq_fee` は従来同様に扱い、**支払われなければ失効**でswapをfail。

### 6.3 `papa_contract_offer`（新規 42083：サーバ→ユーザ）

**タイミング**：`payreq_fee` 支払い完了後。

```json
{
  "protocol_version": 6,
  "swap_id": "string-32B",
  "variant": "taproot_papa",

  "payreq_swap": "string-bolt11",      // 本体インボイス（LN→Server）
  "payment_hash": "hex32",             // 便宜のため冗長掲載（BOLT11内と一致）

  "invoice_pubkey_xonly": "hex32",     // Q_invoice
  "user_pubkey_tap_xonly": "hex32",    // 回送（チェック用）
  "server_pubkey_tap_xonly": "hex32",  // 回送（チェック用）

  "taptree": {
    "leaf1": { "script_hex": "..." , "leaf_version": "c0" },
    "leaf2": { "script_hex": "...",  "leaf_version": "c0" }
  },
  "taproot_output_key_xonly": "hex32", // 出力鍵（内部鍵⊕tweak）
  "csv_delay_blocks": 2016,

  "funding_outpoint": { "txid": "hex32", "vout": 0 },
  "funding_value_sat": "uint64",        // = amount (+prem.) - tx0 fee

  "server_sigs": {
    "tx3": { "sighash": "ANYONECANPAY|NONE", "psbtv2_base64": "..." },
    "tx4": { "sighash": "ANYONECANPAY|NONE", "psbtv2_base64": "..." }
  },

  "tx1_template": {                     // ユーザが署名すべき完成形 tx1
    "psbtv2_base64": "...",             // 出力＝同一Taproot契約（手数料控除後）
    "locktime": 0,
    "sequence": 0
  },
  "tx2_template": {
    "psbtv2_base64": "...",             // 出力＝Serverのアドレス
    "locktime": 0,
    "sequence": 2016
  },

  "min_conf_tx0": 3,
  "expiry_unix_secs":  ...              // payreq_swap の失効時刻
}
```

**重要要件**

* `server_sigs.tx3/tx4` は **Leaf2**（preimage要求）の署名。`ANYONECANPAY|NONE` をMUST。
* `tx1_template/tx2_template` は **PSBTv2** で、**Fund 出力額**や **scriptPubKey**（Taproot）に正確にコミット。
* `funding_outpoint` はtx0の **予定インプット**。ユーザは **utxo 存在と額**を自ノードで検証MUST。
* `taproot_output_key_xonly` と `taptree` が **矛盾なく一致**すること（BIP341のタプル）をユーザ側で検証MUST。

> **PSBTv2 採用理由**：Tapscript の追加データ、部分署名、SIGHASH 指定、witnessUtxo 値などを確実に共有するため。

### 6.4 `papa_user_sigs`（新規 42085：ユーザ→サーバ）

```json
{
  "protocol_version": 6,
  "swap_id": "string-32B",
  "variant": "taproot_papa",

  "user_sigs": {
    "tx1": { "sighash": "DEFAULT", "psbtv2_base64": "..." }, // or SIGHASH_ALL
    "tx2": { "sighash": "DEFAULT", "psbtv2_base64": "..." }  // sequence=2016
  }
}
```

**要件**

* `tx1/tx2` のユーザ署名は **Leaf1** を前提にし、**出力固定**（サーバが任意改変不可）。
* サーバは `psbtv2` を検証し、署名がtx0の **funding 出力**に対して正しいことをMUST確認。

### 6.5 `opening_tx_broadcasted`（42077/再利用）

```json
{
  "swap_id": "string-32B",
  "payreq": "string-bolt11", // 再送（確認用）
  "tx_id": "hex32",          // = tx0
  "script_out": 0,           // vout
  "blinding_key": ""         // Bitcoin では空
}
```

---

## 7. コンセンサス・トランザクション仕様

### 7.1 tx0（funding）

* **出力**：`scriptPubKey = P2TR(X, taptree)`（bech32m）
* **署名**：サーバが任意UTXOから入力構成。RBF有効化推奨。
* **ブロードキャスト**：`papa_user_sigs` 受領後に実施MUST。

### 7.2 tx1（re-lock｜Leaf1）

* **入力**：`tx0` の出力
* **出力**：**再び同一 Taproot 契約**（funding\_value - fee）
* **署名**：ユーザ先署名（SIGHASH\_ALL/DEFAULT）＋サーバ後署名。
* **ユースケース**：ユーザが `tx3` を流した際、サーバが**手数料上げレース**を仕掛けるための競合取引。

### 7.3 tx2（server sweep after CSV｜Leaf1）

* **入力**：`tx1` の出力
* **nSequence**：`csv_delay_blocks`（デフォ2016）
* **出力**：サーバの任意アドレス（テンプレートで固定）
* **署名**：ユーザ先署名＋サーバ後署名
* **ユースケース**：ユーザがswapインボイスを払わない / 送金不能で終了時の**延滞回収**。

### 7.4 tx3（user spend from tx0｜Leaf2）

* **入力**：`tx0` の出力
* **Witness**：`<sig_user> <sig_server(ANYONECANPAY|NONE)> <preimage> <script_leaf2> <controlblock>`
* **出力**：ユーザ自由（keypathでも可）
* **署名**：サーバの事前署名＋ユーザの当座署名。
* **ユースケース**：**正規ルート**。tx0確認後、swapインボイス支払い→preimage取得→即時引き出し。

### 7.5 tx4（user spend from tx1｜Leaf2）

* **入力**：`tx1` の出力
* **用途**：`tx1` が先に確定した場合の**救済**。
* その他はtx3と同様。

---

## 8. 状態機械（イニシエータ＝ユーザ視点）

```
IDLE
  └─swap_out_request(variant=papa)→
NEGOTIATING
  └─swap_out_agreement(受理, fee invoice)→
FEE_INVOICE_SENT
  └─(fee paid)→
PAPA_CONTRACT_OFFERED (42083 受領)
  └─検証OK→ユーザ署名送信(42085)→
USER_SIGS_ACCEPTED
  └─サーバtx0放流→opening_tx_broadcasted→
TX0_SEEN
  └─(min_conf_tx0 達成)→swap invoice 支払→
PREIMAGE_KNOWN
  └─keypath もしくは tx3/tx4 で資金化→
COMPLETE
```

**失敗分岐**はセクション10。

---

## 9. セキュリティ要件（抜粋）

**ユーザ（Taker） MUST**

* `papa_contract_offer` の以下を**自前フルノード**で検証：

  1. `funding_outpoint` の **存在・金額**（`funding_value_sat`≧要求額）
  2. `taproot_output_key_xonly` と `taptree` が矛盾しないこと（BIP341計算）
  3. `server_sigs.tx3/tx4` が **当該入力**に対する **Leaf2** 署名として検証OK
  4. `tx1_template/tx2_template` の **出力・金額・sequence** が仕様通り
* mempool/chainで`tx1`を検出した際は、`tx4`を即座に放流できるよう監視する（Watcher推奨）。
* swapインボイスを **tx0 が `min_conf_tx0` を満たすまで支払わない**。

**サーバ（Maker） MUST**

* ユーザから `papa_user_sigs` を **検証**し、署名がLeaf1かつ指定の出力へ固定されていることを確認。
* `tx0` を **その後**にブロードキャスト。
* 競合（`tx1`）を使う場合でも、ユーザに `tx4` の余地を与える設計（＝本プロトコル）を守る。
* `csv_delay_blocks` は2016を推奨（運用方針で交渉可）。

---

## 10. フェイル・キャンセル処理

* **fee invoice 失効**：`swap_out_agreement` 再送（従来通り）→失効で **swap fail**。
* **`papa_contract_offer` 検証失敗**：ユーザは `cancel(42079)` MUST。
* **tx0 放流済み・swap 未完了**：

  * ユーザ未支払い：サーバは `tx2` により `csv_delay_blocks` 経過後に回収可能。
  * サーバが `tx1` を先に通した場合：ユーザは `tx4` を放流（**二者競合を回避**）。
* **従来の `coop_close(42081)`**：Papa変種では基本不要（返金CSV経路がないため）。

  * tx0未放流なら `cancel` 同等で終了。
  * どうしても合意解消する場合は、**keypath 署名や tx3/tx4**による資金分離で代替。

---

## 11. 手数料・ポリシー

* **手前支払い**：従来同様 **fee invoice**（事務・失敗時のon-chain予備費用）。
* **tx0 fee**：サーバ負担。RBF有効、`min_conf_tx0` デフォ3（交渉可）。
* **tx3/tx4 fee**：ユーザ負担。

  * サーバ署名が `ANYONECANPAY|NONE` のため、ユーザは **CPFP・追加インプット**で自由に手数料調整可。
* **競合（tx1）**：RBF/手数料で上書きが起きうる想定。`tx4` の準備で安全化。
* **keypath 優先**：preimageがkeypath私鍵として妥当なら、**keypath 送金が最安**。

  * 不一致時は **Leaf2（tx3/tx4）** にフォールバックし、サーバは**違反**として評価。

---

## 12. Liquid/Elements 拡張（将来）

* Liquid側で **Taproot/Tapscript・Schnorr**が十分互換であることを前提に、

  * `asset`・`blinding_key` の取り扱い、
  * `min_conf_tx0`（例：2）,
  * 機密トランザクション（CT）・CAへの対応
    を追加定義する。
* 当面は **Bitcoin のみで有効化**し、`variant=taproot_papa` 受領時に `network` がLiquid系の場合は **fail** か **従来 Swap Out** にフォールバック。

---

## 13. 実装ガイド（PSBTv2 / 署名）

* **PSBTv2（BIP370）** を **全テンプレート共有**の標準とする。

  * `PSBT_IN_TAP_BIP32_DERIVATION`、`PSBT_IN_TAP_INTERNAL_KEY`、`PSBT_IN_TAP_MERKLE_ROOT` を設定する。
  * Taproot関連フィールドを正しく設定する。
  * `witnessUtxo` にfunding出力の `amount/scriptPubKey` を必ず含める（署名検証に必須）。
* **署名フラグ**：

  * ユーザ→tx1/tx2：`DEFAULT`（=ALL相当）推奨。
  * サーバ→tx3/tx4：`ANYONECANPAY|NONE`（固定）。
* **トランザクション Version** は **2** を強制（BIP68準拠）。
* **nLockTime**：0（特段の要件なし）。
* **Bech32m** アドレス（P2TR）で外部提示。

---

## 14. ノード間の機能検出とフォールバック

* **機能ビット**：`features.papa_swap=1` をBOLT#1カスタム・ハンドシェイクまたは各リクエストの `variant` で表明。
* 相手が未対応なら：

  * `swap_out_request` への `variant=standard` 再送、またはユーザUIで「Papa非対応。標準Swap Outに切替」と通知。

---

## 15. 監視・運用

* **ユーザ側 Watcher**：

  * `funding_outpoint` の **ダブルスペンド/ブロードキャスト**検知。
  * **`tx1` 検出時に `tx4` 送出**（手数料はCPFPも活用）。
* **サーバ側**：

  * `papa_user_sigs` 検証完了前は **tx0 を絶対に放流しない**。
  * `tx0` 放流後は **min\_conf\_tx0** 参照をユーザへ再通知（`opening_tx_broadcasted`）。

---

## 16. 具体的な要件まとめ（抜粋）

* **両者 MUST**：BIP340/341/342を実装。
* **ユーザ MUST**：

  * `papa_contract_offer` の全整合性検証（UTXO、ツリー、署名、PSBT）。
  * `min_conf_tx0` 充足前に `payreq_swap` を支払わない。
* **サーバ MUST**：

  * `tx3/tx4` の **ANYONECANPAY|NONE** 署名を提供。
  * `tx1/tx2` は **ユーザ署名に完全拘束**（サーバ恣意を排除）。
  * `csv_delay_blocks` は既定2016、運用方針で合意。
* **安全性**：

  * ユーザはpreimage入手直後に **keypath**（最安）または **Leaf2(tx3)** で資金化可能。
  * サーバが `tx1` を通しても、ユーザは **`tx4` で奪還**可能。
  * ユーザが未支払いなら、サーバは **`tx2` により回収**可能（相対2016後）。

---

## 17. 参考ペイロード例（要点のみ）

**swap\_out\_request**

```json
{
  "protocol_version": 6,
  "swap_id": "abc123...",
  "asset": "",
  "network": "signet",
  "scid": "123x456x7",
  "amount": 200000,
  "variant": "taproot_papa",
  "pubkey_tap_xonly": "e1b2...32bytes",
  "csv_delay_blocks": 2016
}
```

**papa\_contract\_offer（抜粋）**

```json
{
  "swap_id": "abc123...",
  "payreq_swap": "lnbc2u1p...",
  "payment_hash": "0f0e...32bytes",
  "invoice_pubkey_xonly": "aa11...32bytes",
  "taptree": {
    "leaf1": {"script_hex":"20<user>ac...20<server>ac", "leaf_version":"c0"},
    "leaf2": {"script_hex":"a8 <hash> 88 20<user>ac...20<server>ac", "leaf_version":"c0"}
  },
  "taproot_output_key_xonly": "bb22...32bytes",
  "funding_outpoint": {"txid":"de..ad", "vout":0},
  "funding_value_sat": 199200,
  "server_sigs": {
    "tx3":{"sighash":"ANYONECANPAY|NONE","psbtv2_base64":"cHNidP8B..."},
    "tx4":{"sighash":"ANYONECANPAY|NONE","psbtv2_base64":"cHNidP8B..."}
  },
  "tx1_template":{"psbtv2_base64":"...","sequence":0},
  "tx2_template":{"psbtv2_base64":"...","sequence":2016},
  "min_conf_tx0":3
}
```

---

## 18. 既存 PeerSwap 仕様との整合ポイント

* **メッセージレンジ**：既存レンジ内（42083/42085）で追加。
* **`opening_tx_broadcasted` 再利用**：UI/運用資産を流用し、差分実装を最小化。
* **`premium`**：従来通り `swap_out_agreement` で提示し、tx0額とインボイス額に反映。
* **「1 チャンネル 1 スワップ」制約**：従来同様に維持。

---

## 19. 実装メモ（落とし穴）

* **SIGHASH の組合せ**：`ANYONECANPAY|NONE` 署名と `DEFAULT/ALL` 署名を同一inputの2回のCHECKSIGで組み合わせるケースをテスト。
* **BIP68**：`tx2` は **version=2**、`nSequence` ビット扱いを厳密に（時間/ブロック単位混同に注意）。
* **RBF と CPFP**：`tx3/tx4` の手数料調整は **CPFP** 基本、`tx1` 競合時の救済は **`tx4` 早期放流**で対応。
* **Keypath 鍵の合成**：`x = p + q (mod n)` の実装は **符号・奇偶**の扱い（x-only）に注意。
* **PSBTv2**：Taproot専用フィールドを欠かすと署名検証が不可能になるため、テストベクトルを整備。

---

## 20. まとめ

本統合案は、PeerSwapの **Swap Out** をPapaSwap（Taproot 1tx）へ拡張します。実現する効果は次のとおりです。

* **オンチェーン 1 取引**化
* **手数料と時間の半減**
* **失敗パス時の明快な回収動線**（`tx2`/`tx4`）

プロトコルは **既存メッセージの拡張＋2 メッセージ追加**で最小侵襲的に導入可能です。
運用上は **keypath 最優先**・**Leaf2 署名の事前受領**・**Watcher 常駐**が要点です。

必要であれば、このドラフトを踏まえた **PSBT テンプレート/テストベクトル**や **コードスケルトン**（Go/Rust）も用意できます。


## TL;DR

* **親和性◎**：両者は *LNの支払いプリイメージ* をオンチェーン条件に結びつける。PeerSwapの **Swap Out**（LN→on-chain）に **PapaSwap（1tx/Taproot）** が適合する。メッセージ拡張だけで導入可能であり、手数料と完了時間を半減できる。
* **主要課題**：
  ① 事前署名に **tx0 のTXID確定**が要る（tx1/tx3/tx4がtx0に依存）。
  ② **SIGHASH設計**の細部（`ANYONECANPAY|NONE` 等）と検証コードの複雑化。
  ③ **preimage→秘密鍵**の変換（域分離・範囲チェック）。
  ④ **ウォッチャ/救済フロー**（tx1観測→tx4放流）を各ノードで確実化。
  ⑤ 既存PeerSwapの **CSV/expiryパラメータ**との整合
  ⑥ Taproot/PSBTv2/Tapscriptの **実装負荷** とLiquid拡張時の差分

---

## 1) 親和性（うまく噛み合う点）

### A. 目的・ユースケース

* **Swap Out と一致**：PeerSwapのSwap Outは「イニシエータがLN支払い→オンチェーン受取」である。Papaは **LN→on-chain** のサブマリンスワップである。チャネルのインバウンドを増やしつつ、同額をオンチェーンで回収したい場面に適する。

### B. 暗号設計の一致

* **プリイメージ連動**：PeerSwapはHTLCのpreimageをP2WSHに組み込む。Papaは同じpreimageを **(1) Tapleaf条件** と **(2) keypathの秘密鍵合成（p+q）** に使う。同じ根（preimage）で原子性を担保する設計思想が共通である。

### C. 経路とメッセージ拡張の容易さ

* **BOLT#1カスタムの延長で足りる**：`swap_out_request`/`…_agreement`/`opening_tx_broadcasted` に **Papa専用フィールド** を追加する。さらに、契約提示とユーザ事前署名送付の2つの補助メッセージを追加すれば統合できる。既存の `premium` と `fee invoice` の経済モデルも流用可能である。

### D. 性能・コスト

* **オンチェーン1取引**化：PeerSwap(2tx)→Papa(1tx)で **手数料・完了時間ともに縮小**。
* **Taproot利点**：keypath決済が通れば **最小重量・匿名性↑**（script非公開）。

### E. 失敗時の回収動線の明快さ

* **tx2（相対2016）** と **tx4（救済）** により、**ユーザ／サーバ双方の最終回収ルート**が明確。PeerSwapの「claim\_by\_csv」「coop\_close」と同じ思想で、Papa流の回収ロジックに置き換えやすい。

---

## 2) 課題・リスク（技術）

### ① tx0 TXID前提問題（事前署名の根本）

* **問題**：ユーザは `tx1/tx3/tx4` を **tx0の出力**に対して事前署名するため、**tx0のTXIDとvoutが厳密に確定**している必要がある。
* **対策**：

  * **サーバが完成済みtx0（PSBT/原始tx）を事前に提示**してTXIDを固定（ユーザが早期送信できる設計に割り切る）。
  * または **TXIDコミットメント**をメッセージで約束し、逸脱したらユーザは支払わない（信頼性は落ちるが安全性は保てる）。
* **評価**：ここが **最大の実装/運用上の分岐**。*早期公開容認（堅実）* か *TXIDコミットのみ（運用簡易だがDoS耐性弱め）* を選ぶ必要。

### ② SIGHASH設計と検証

* **要件**：

  * `tx1/tx2` の **ユーザ署名**は基本 **`SIGHASH_ALL（=DEFAULT）`** を推奨（サーバが入出力を勝手に変えられない）。
  * `tx3/tx4` の **サーバ署名**は **`ANYONECANPAY|NONE`**（ユーザが自由に出力・追加入力・CPFPで手数料調整可）。
* **オプション**（Papaの「小さな改善」に対応）：

  * サーバが `tx1` に **追加入力**（例：お釣りUTXO）を混ぜられるようにしたい場合、ユーザ署名を **`ANYONECANPAY|ALL`** にする協定が必要（出力は固定のまま、入力だけ増やせる）。
* **リスク**：署名の **検証実装が難しくバグりやすい**。PSBTv2での明確コミットと包括的テストベクトルが必須。

### ③ preimage → 秘密鍵 変換の堅牢化

* **懸念**：preimageを「そのまま」secp256k1秘密鍵にすると、**0/≥n** 等の例外・再利用問題・ドメイン分離欠如が起こりうる。
* 推奨: `q = int(tagged_hash("PapaSwap/q", preimage)) mod n; if q==0 → retry` とする。タグ付きハッシュで域分離し、常に有効レンジに収める。仕様として規範化する。

### ④ 相対ロック（CSV）とインボイス期限の整合

* **整合規則**：PeerSwap由来の原則（**請求期限 ≤ CSV/2**）を継承。
* **Papa既定**：`csv_delay_blocks = 2016` を推奨（安全マージン）。PeerSwapのBTC既定（1008）と **差**があるため、**ネットワーク/運用方針で合意**を明示。

### ⑤ ウォッチャ/自動救済

* **要件**：`tx1` をmempool/chainで検知したら **即`tx4` 放流**（手数料はCPFPで確実化）。
* **課題**：各実装（lnd/cln + PeerSwap Plugin）に **ウォッチャ常駐**と **自動ブロードキャスト**の実装が必要。

### ⑥ Taproot/PSBTv2/Tapscript 実装負荷

* **新規実装**：BIP340/341/342、PSBTv2のフィールド（Taproot派生情報、witnessUtxo、merkle root等）を正しく扱う必要。
* **テスト**：`ANYONECANPAY|NONE` を含む **SIGHASHの境界テスト**、`version=2`/BIP68相対ロックの検証が必須。

### ⑦ プライバシ

* **長所**：keypathで払えば **単独P2TRと不可識別**。
* **短所**：Leaf2（スクリプト公開）を使うと、**LNのpayment\_hashとサーバ鍵**がオンチェーン露出し、**LN支払いとUTXOを結びやすく**なる。可能ならkeypath優先。

### ⑧ Liquid拡張

* **差分**：Taproot/Tapscriptの互換、CT/ブラインド、最小承認数（例:2）など仕様差。
* **方針**：まず **Bitcoin限定**で成熟させ、Liquidは段階導入。

---

## 3) 課題・リスク（運用／経済）

* **早期公開の是非**：tx0を事前にユーザへ完全提示する運用にすると、ユーザは**先に放流**できる。サーバ側は嫌う可能性がある。ただし **安全性はむしろ上がる**。
* **手数料政策**：

  * サーバ：tx0はRBF前提、fee invoiceで初期コスト回収。
  * ユーザ：tx3/tx4は自前CPFPで確定性を担保。
* **失敗パスのコスト**：Papaは2tx→1tx化で**失敗時コストも抑制**される一方、**fee invoice 詐取**の懸念（小額）や \*\*DoS（TXID不一致）\*\*には運用での緩和が必要である。
* **準拠/運用監査**：事前署名管理、エフェメラル鍵、ログ最小化（preimageは秘匿）、再現性ある監査ログの両立。

---

## 4) どう設計すれば良いか（推奨）

1. **機能ビットとフォールバック**

   * `variant=taproot_papa` を明示。未対応ピアには自動で **標準Swap Out** に戻す。
2. **メッセージ追加（最小）**

   * `papa_contract_offer`（サーバ→ユーザ：taptree、tx0、tx3/tx4署名、tx1/tx2テンプレ）
   * `papa_user_sigs`（ユーザ→サーバ：tx1/tx2署名）
3. **TXID問題の解決**

   * **原則：完成済みtx0（PSBTv2 or raw）を事前共有**。TXID/TX体に**厳格コミット**。
4. **SIGHASH規約の固定**

   * 既定：`tx1/tx2 = ALL(=DEFAULT)`、`tx3/tx4 = ANYONECANPAY|NONE`。
   * オプション改善：`tx1` だけ `ANYONECANPAY|ALL` を **双方合意時のみ**許可。
5. **preimage→keyの規範化**

   * タグ付きハッシュで域分離し、`q∈[1,n-1]` を保証。
6. **タイミング規則**

   * `invoice_expiry ≤ csv_delay/2`、`min_conf_tx0`（BTC:既定3）到達までは **swap請求支払不可**。
7. **ウォッチャ常設**

   * `tx1` 検出→ `tx4` 自動放流、CPFP自動化、失敗パスの無人化。

---

## 5) 総合評価

* **親和性は高い**：PeerSwapのSwap OutにPapaSwapを嵌めると、**同じ安全モデル**（preimage原子性＋相対CSVペナルティ）を維持しつつ、**1tx/Taproot**で**速く安く**なる。
* **実装のカギは3点**：

  1. **tx0 TXIDを事前に確定・共有**する設計を採用できるか
  2. **SIGHASH/PSBTv2** の運用と検証をバグなく作れるか
  3. **ウォッチャ＆自動救済**まで含めた運用自動化
* これらを満たせば、PeerSwapの強み（ピア間調整・チャネル健全化）を保ちながら、**鎖上負荷と手数料**を現実的に引き下げられる。

