---
date: "2025-05-11T10:51:56+09:00"
title: "TuscanyLightningSummit2025"
---

# Tuscany Lightning Summit 2025
目的: [Tuscany Lightning Summit 2025（TLS2025）](https://tuscanysummit.com/)で示された Bitcoin／Lightning Network および関連 Layer2 技術の最新動向を体系的に整理し、今後の技術課題と展望を明確化する。  
結論: LSP 標準化、ノンカストディ型 UX 改善、複数 Layer2（Liquid・RGB・Taproot Asset 等）のハイブリッド運用が不可欠である。一方、開発リソースは限定的であり、コミュニティ協働と提案（BOLT／BLIP）への貢献が急務である。

---

## Day&nbsp;1&nbsp;概要

| セッション                         | 主題                                                                                    | 私見                                                                 |
| ---------------------------------- | --------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| Keynote by Giacomo Zucco           | LN を総合送金・ルーティング基盤へ再定義。異資産スワップとセキュリティモデル統合を強調。 | P2P&nbsp;atomic&nbsp;swapが鍵。Taproot Asset と RGB の共存性を注視。 |
| 1000 Shades of Trust               | 秘密鍵管理を二項対立ではなくスペクトルで捉えるべきと提言。                              | フェデレーション維持可能性が核心。中間形態への社会的理解が不足。     |
| Bitcoin's Fintech Revolution       | LN によるリアルタイム小額決済と規制対応。                                               | Stablecoin 実装は RGB／Taproot Asset／Liquid の選択が重要。          |
| Lightning Without the Storms       | Liquid（許可制フェデレーション）と LN（分散 P2P）の比較。                               | プロトコル選択の複雑化は必至。                                       |
| Ark Lightning Gateways             | Ark を L2 オプションとして紹介。                                                        | プロトコル詳細は未確定で追跡必要。                                   |
| Lightning and Digital Identity     | LN を SSI／ID 管理へ適用する PoC。                                                      | 拡張可能性は高いが時期尚早。                                         |
| Attributable Failures in Lightning | MPP 失敗時に責任ノードを特定する BOLTs #1044。                                          | ルート信頼性向上に必須だが合意形成が難航。                           |
| Keynote by Adam Back               | Compact Blocks によるブロック伝播最適化。                                               | 誤検出リスクに留意。swap 需要増加を予測。                            |
| The Bitcoin Reality Check          | 決済普及度の現実とマーケティング課題。                                                  | 技術と社会の認知ギャップが大。                                       |

---

## 交流・開発関連

- **Greenlight**: Christian Deckerと PR([#599](https://github.com/Blockstream/greenlight/pull/599)) をマージ。HTTP 経由プラグイン構想が進行。
- **PeerSwap**: Nepetとfeaturebidsによる2hop peerswapの可能性の検討。

## まとめ

TLS2025 は Bitcoin／Lightning の進化を俯瞰し、Liquid・RGB・Taproot Asset 等を含む多層技術併用が前提となる「複雑化時代」の到来を示した。開発リソースは限られ、BOLT／BLIP 提案への実装貢献が停滞を打破する最短経路である。ユーザーオンボーディング、LSP 標準化、カストディ形態最適化を段階的に進め、利便性と分散性を両立したエコシステムを構築すべきである。次回サミットまでに、自身もコードおよび議論へ継続的に貢献し、課題解決を加速させたい。
