---
date: '2025-08-02T07:15:44+09:00'
title: "peerswap premium alert"
---

# PeerSwap Premium Rate Sanity Checks - 設計提案

## 背景

[Issue #394](https://github.com/ElementsProject/peerswap/issues/394)では、プレミアムレートの設定ミスや悪用を防ぐためのサニティチェックが提案されています。現在の実装では任意の値（正負問わず）を設定可能であり、これは柔軟な価格戦略を可能にする一方で、設定ミスや裁定取引のリスクも存在します。

## 現在の実装分析

### プレミアムレートの仕組み

1. **デフォルト値** (`premium/premium.go`):
   - BTC SwapIn: 0 ppm
   - BTC SwapOut: 2000 ppm (0.2%)
   - LBTC SwapIn: 0 ppm
   - LBTC SwapOut: 1000 ppm (0.1%)

2. **レート設定時の検証**:
   - 現在は基本的な型チェックのみ
   - レート値の妥当性チェックなし

3. **スワップ実行時の処理**:
   - ユーザーは`premium_limit_rate_ppm`を指定
   - `CheckPremiumAmount`で制限を超えていないか確認
   - 超えていればスワップはキャンセル

## 設計方針

利用者がスワップ時にレートを確認して判断できるという市場原理を尊重しつつ、明らかなミスや悪用を防ぐバランスの取れた実装を目指します。

## 具体的な実装提案

### 1. ユーザー側の改善（レート表示の強化）

#### peerswaprpc/server.go に追加

```go
type RateWarning struct {
    Level   string // "info", "warning", "danger"
    Message string
}

func (p *PeerswapServer) analyzeRates(peerID string, asset string, operation string, amount uint64) (*RateWarning, error) {
    // ピアのレートを取得
    var premiumRate *premium.PremiumRate
    var err error
    
    if asset == "lbtc" {
        if operation == "out" {
            premiumRate, err = p.ps.GetRate(peerID, premium.LBTC, premium.SwapOut)
        } else {
            premiumRate, err = p.ps.GetRate(peerID, premium.LBTC, premium.SwapIn)
        }
    } else {
        if operation == "out" {
            premiumRate, err = p.ps.GetRate(peerID, premium.BTC, premium.SwapOut)
        } else {
            premiumRate, err = p.ps.GetRate(peerID, premium.BTC, premium.SwapIn)
        }
    }
    
    if err != nil {
        return nil, err
    }
    
    ratePPM := premiumRate.PremiumRatePPM().Value()
    
    // 警告レベルの判定
    if ratePPM < 0 && operation == "out" {
        return &RateWarning{
            Level: "danger",
            Message: fmt.Sprintf("⚠️ Negative swap-out rate (%d ppm) - You will PAY to send funds!", ratePPM),
        }, nil
    }
    
    if ratePPM > 10000 { // 1%以上
        return &RateWarning{
            Level: "warning", 
            Message: fmt.Sprintf("High premium rate: %.2f%% (%d ppm)", float64(ratePPM)/10000, ratePPM),
        }, nil
    }
    
    // クロスアセット裁定の検出
    if asset == "btc" && operation == "in" {
        lbtcOutRate, _ := p.ps.GetRate(peerID, premium.LBTC, premium.SwapOut)
        if lbtcOutRate != nil {
            totalRate := ratePPM + lbtcOutRate.PremiumRatePPM().Value()
            if totalRate <= 0 {
                return &RateWarning{
                    Level: "warning",
                    Message: fmt.Sprintf("Potential arbitrage: BTC_IN + LBTC_OUT = %d ppm", totalRate),
                }, nil
            }
        }
    }
    
    return nil, nil
}
```

### 2. オペレーター側の改善（設定時の警告）

#### premium/validator.go (新規ファイル)

```go
package premium

import (
    "fmt"
    "strings"
)

type RateValidationResult struct {
    IsValid  bool
    Warnings []string
    Errors   []string
}

// ValidateRateConfiguration はレート設定の妥当性を検証する
// 強制的なエラーではなく、警告として実装
func ValidateRateConfiguration(rates map[AssetType]map[OperationType]int64) *RateValidationResult {
    result := &RateValidationResult{IsValid: true}
    
    // Rule 1: 負のSWAP_OUTレートの警告
    for asset, ops := range rates {
        if swapOut, ok := ops[SwapOut]; ok && swapOut < 0 {
            result.Warnings = append(result.Warnings, 
                fmt.Sprintf("⚠️ %s SWAP_OUT has negative rate (%d ppm) - You will pay peers to take your outbound liquidity", 
                    asset, swapOut))
        }
    }
    
    // Rule 2: BTCの総レートチェック
    if btcRates, ok := rates[BTC]; ok {
        total := btcRates[SwapIn] + btcRates[SwapOut]
        if total <= 0 {
            result.Warnings = append(result.Warnings,
                fmt.Sprintf("⚠️ BTC total rate (IN+OUT) is %d ppm - Peers can profit by doing sequential swaps", total))
        }
    }
    
    // Rule 3 & 4: クロスアセット裁定の警告
    if btcIn, btcOk := rates[BTC][SwapIn]; btcOk {
        if lbtcOut, lbtcOk := rates[LBTC][SwapOut]; lbtcOk {
            crossRate := btcIn + lbtcOut
            if crossRate <= 0 {
                result.Warnings = append(result.Warnings,
                    fmt.Sprintf("⚠️ BTC_SWAP_IN + LBTC_SWAP_OUT = %d ppm - Potential arbitrage via peg-out", crossRate))
            }
        }
    }
    
    if lbtcIn, lbtcOk := rates[LBTC][SwapIn]; lbtcOk {
        if btcOut, btcOk := rates[BTC][SwapOut]; btcOk {
            crossRate := lbtcIn + btcOut  
            if crossRate <= 0 {
                result.Warnings = append(result.Warnings,
                    fmt.Sprintf("⚠️ LBTC_SWAP_IN + BTC_SWAP_OUT = %d ppm - Potential arbitrage via peg-in", crossRate))
            }
        }
    }
    
    // 極端に高いレートの警告
    for asset, ops := range rates {
        for op, rate := range ops {
            if rate > 50000 { // 5%以上
                result.Warnings = append(result.Warnings,
                    fmt.Sprintf("⚠️ %s %s has very high rate: %.2f%% (%d ppm)", 
                        asset, op, float64(rate)/10000, rate))
            }
        }
    }
    
    return result
}
```

#### premium/premium.go の修正

```go
func (p *Setting) SetRate(peerID string, rate *PremiumRate) error {
    // 既存のレートを取得して検証用のマップを作成
    allRates := make(map[AssetType]map[OperationType]int64)
    for asset := range DefaultPremiumRate {
        allRates[asset] = make(map[OperationType]int64)
        for op := range DefaultPremiumRate[asset] {
            existingRate, _ := p.GetRate(peerID, asset, op)
            if existingRate != nil {
                allRates[asset][op] = existingRate.PremiumRatePPM().Value()
            }
        }
    }
    
    // 新しいレートを適用
    allRates[rate.Asset()][rate.Operation()] = rate.PremiumRatePPM().Value()
    
    // 検証
    validation := ValidateRateConfiguration(allRates)
    if len(validation.Warnings) > 0 {
        log.Warnf("Premium rate configuration warnings for peer %s:\n%s", 
            peerID, strings.Join(validation.Warnings, "\n"))
    }
    
    err := p.store.SetRate(peerID, rate)
    if err == nil {
        p.notifyObservers()
    }
    return err
}
```

## 検証ルールの詳細

### Issue #394で提案されたルール

1. **基本的なレート方向の検証**
   - SWAP_INレートは負またはゼロ（受け取る側なので手数料を払わない）
   - SWAP_OUTレートは正またはゼロ（送る側なので手数料を取る）
   - 例：負のSWAP_OUTレートは「inbound liquidityを盗むために人々にお金を払う」ことになる

2. **BTCスワップの総レート検証**
   - BTC_SWAP_IN + BTC_SWAP_OUT > 0
   - 同じ金額で連続的にスワップして利益を得ることを防ぐ

3. **クロスアセット裁定取引の防止（BTC→LBTC）**
   - BTC_SWAP_IN_rate + LBTC_SWAP_OUT_rate > ベースラインレート
   - swap-out → peg-out → swap-inという戦略での裁定を防ぐ

4. **クロスアセット裁定取引の防止（LBTC→BTC）**
   - LBTC_SWAP_IN_rate + BTC_SWAP_OUT_rate > 0
   - 無料の変換ループを防ぐ

## 実装の利点

1. **柔軟性の維持**: 負のレートも戦略的に使用可能（liquidityバランスの調整など）
2. **ユーザー保護**: スワップ実行前に危険なレートを警告
3. **オペレーター支援**: 設定時に潜在的な問題を通知
4. **段階的導入**: 警告から始めて、将来的により厳格な制限も可能

## 今後の拡張案

1. **設定可能な閾値**: 警告レベルの閾値を設定ファイルで調整可能に
2. **統計情報**: レート設定の履歴や利用状況の追跡
3. **自動調整**: 市場状況に応じたレート推奨機能
4. **API拡張**: レート分析エンドポイントの追加

## まとめ

この設計は市場原理を尊重しながら、ユーザーとオペレーターの両方を保護するバランスの取れたアプローチです。強制的な制限ではなく、情報提供と警告によって、より安全なPeerSwapエコシステムの構築を目指します。