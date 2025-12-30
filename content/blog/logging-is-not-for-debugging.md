---
title: "ログはデバッグのためではない"
date: 2025-12-30
tags: ["SystemDesign", "TradingBot", "Logging", "Operation", "Reliability"]
toc: true
---

## はじめに
### なぜ「ログ＝デバッグ用」という考え方が危険なのか

多くの自動売買 Bot では、  
ログは次のように扱われがちです。

- エラーが出たときに見るもの
- 動かないときの原因調査用
- 開発中だけ必要なもの

しかし、  
**長期・無人運用を前提とするシステムにおいて、  
この考え方は非常に危険**です。

なぜなら、  
実運用中の Bot は次の状態に置かれるからです。

- 開発者は常に監視していない
- 異常はリアルタイムで気づけない
- 事故は「終わった後」にしか分からない

このとき、  
**唯一残る事実がログ**です。

本記事では、  
ログを「デバッグのためのもの」ではなく、  
**生き残るための設計要素**として  
どのように考えるべきかを整理します。

---

## ログが無いと何が起きるのか（実運用の現実）

ログ設計が弱い Bot は、  
必ず次の状態に陥ります。

- 負けているが、なぜ負けているのか分からない
- 止まっていたが、いつ止まったのか分からない
- 事故が起きたが、再現できない

これは、  
ロジックや戦略の問題ではありません。

**「何が起きたかを記録していない」**  
という、設計上の欠陥です。

---

## 「勝った／負けた」だけのログは意味を持たない

多くの Bot では、  
次のようなログしか残していません。

- エントリーした
- TP / SL で終わった
- 勝った / 負けた

しかし、  
これでは **運用改善に一切使えません**。

実運用で本当に知りたいのは、

- なぜその判断をしたのか
- そのとき市場はどういう状態だったのか
- その負けは「避けられなかった負け」か
- 別の判断は存在したのか

という点です。

---

## ログは「結果」ではなく「判断過程」を残すもの

長期運用を前提とする場合、  
ログで残すべきなのは **結果ではありません**。

重要なのは、

- どの条件が成立したのか
- どの条件は成立しなかったのか
- なぜその判断に至ったのか

という **判断の履歴** です。

ログとは、  
Bot が「なぜその行動を選んだのか」を  
後から再構築するための材料です。

---

## なぜ「無効シグナル」のログが重要なのか

多くの実装では、  
採用されたシグナルだけを記録します。

しかし実運用では、  
**採用されなかったシグナル**の方が  
はるかに重要です。

- なぜこのシグナルは無効だったのか
- どの条件が足りなかったのか
- 将来、条件を調整する余地はあるのか

これらは、  
無効シグナルのログが無ければ  
絶対に分かりません。

---

## シグナルログは「採用／却下」を分離して考える

ログ設計では、

- エントリーされたシグナル
- 却下されたシグナル

を **明確に分離** すべきです。

ここでは、

- 採用ログ：なぜ entry されたか
- 却下ログ：なぜ entry されなかったか

という視点でログを設計します。

```python 
def log_signal_decision(symbol, signal_type, accepted, reason, indicators):
    """
    シグナル判定結果を記録する

    symbol      : 通貨ペア
    signal_type : LONG / SHORT
    accepted    : True（採用） / False（却下）
    reason      : 採用または却下した理由
    indicators  : 判定時点のインジケータ情報
    """
    log = {
        "timestamp": datetime.utcnow().isoformat(),
        "symbol": symbol,
        "signal_type": signal_type,
        "accepted": accepted,
        "reason": reason,
        "indicators": indicators
    }

    # シグナル判定ログを CSV に追記
    with open("logs/signal_decisions.csv", "a", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=log.keys())

        # 初回のみヘッダーを書き込む
        if f.tell() == 0:
            writer.writeheader()

        writer.writerow(log)
```

---

## トレード結果ログで本当に見るべきもの

トレード結果ログで重要なのは、  
単純な勝敗ではありません。

- 最大含み益はどれくらいあったか
- 最大逆行はどれくらいだったか
- BE に到達したか
- 到達までにどれくらい時間がかかったか

これらは、  
**戦略の性質を理解するためのログ**です。

```python 
def log_trade_result(symbol, side, entry_price, exit_price,
                     max_profit_pct, max_drawdown_pct, reason):
    """
    トレード結果を記録する

    symbol             : 通貨ペア
    side               : LONG / SHORT
    entry_price        : エントリー価格
    exit_price         : クローズ価格
    max_profit_pct     : 最大含み益（％）
    max_drawdown_pct   : 最大逆行（％）
    reason             : クローズ理由（TP / SL / BE / TIMEOUT 等）
    """
    result = {
        "timestamp": datetime.utcnow().isoformat(),
        "symbol": symbol,
        "side": side,
        "entry_price": entry_price,
        "exit_price": exit_price,
        "max_profit_pct": round(max_profit_pct, 4),
        "max_drawdown_pct": round(max_drawdown_pct, 4),
        "exit_reason": reason
    }

    # トレード結果ログを CSV に保存
    with open("logs/trade_results.csv", "a", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=result.keys())

        # 初回のみヘッダーを書き込む
        if f.tell() == 0:
            writer.writeheader()

        writer.writerow(result)
```

---

## ログはリアルタイム監視のためのものではない

ログは、

- 常に眺めるもの
- ダッシュボード用途

ではありません。

ログの役割は、

- 事故が起きたあとに
- 冷静に振り返るための材料

です。

感情が入らない形で  
事実だけを残すことが重要です。

---

## CSVログが最初の選択として優れている理由

初期の長期運用 Bot においては、  
データベースよりも **CSV ログ** が適しています。

理由は、

- 書き込みが単純
- 壊れにくい
- Excel / Python / pandas で即分析できる

からです。

重要なのは、  
「高度な仕組み」ではなく  
**確実に残ること**です。

---

## ログが Bot を成長させる

ログがある Bot は、

- なぜ負けたかを説明できる
- 改善ポイントを特定できる
- 同じ失敗を繰り返さない

ログが無い Bot は、

- 勝った／負けたの感覚論になる
- 改善が属人的になる
- 長期的に進化しない

という差が生まれます。

---

## まとめ

ログは、

- デバッグのためのものではない
- 開発中だけのものではない
- おまけの機能ではない

**長期運用を支える中核設計要素**です。

自動売買 Bot を  
長期・無人で動かすためには、

- 何を判断したか
- なぜ判断したか
- 何が起きたか

を、  
人がいなくても残せる仕組みが必要です。

ログは、  
Bot が「生き残るための記憶」です。

これまでの記事で整理してきた、

- 観測モード
- Notional 管理
- State 管理
- WebSocket 設計

これらすべては、  
**ログによって初めて意味を持ちます。**

このログ設計まで含めて、  
自動売買システムは  
「長期運用可能なシステム」になります。
