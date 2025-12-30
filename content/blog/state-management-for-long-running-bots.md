---
title: "長期運用BotにおけるState管理設計"
date: 2025-12-30
tags: ["SystemDesign", "TradingBot", "StateManagement", "Operation"]
toc: true
---

## はじめに
### なぜ State 管理が重要なのか

Notional 管理を正しく設計し、  
注文が安定して通るようになっても、  
自動売買システムはまだ「完成」ではありません。

長期運用において、  
次に必ず直面する問題が **State 管理** です。

- 再起動したらどうなるのか
- 途中で落ちたら、どこから再開するのか
- 今この Bot は、何をしている状態なのか

これらに答えられないシステムは、  
**長期運用に耐えません。**

本記事では、  
長期間止めずに動かす Bot において  
**State をどのように設計・管理すべきか** を整理します。

---

## State とは何か

ここでいう State とは、  
「取引所の状態」そのものではありません。

Bot が内部的に認識している  
**運用上の状態** を指します。

例えば、

- ポジションを持っているか
- 注文を出した直後か
- 監視中か
- クローズ処理中か

これらは、  
API を叩かないと分からない情報ではなく、  
**Bot 自身が管理すべき状態** です。

---

## なぜ取引所の状態だけに依存してはいけないのか
```python
# State guard: 二重エントリー防止
if symbol in load_open_positions():
    return  # すでに State 上はポジションあり
```

この判断は、
取引所の状態よりも Bot の State を優先する という設計判断を示しています。

多くの実装では、  
次のような前提が暗黙に置かれています。

> 「取引所の API を見れば、すべて分かる」

しかし実運用では、

- API 応答の遅延
- 一時的な不整合
- WebSocket の切断
- 再起動直後の空白時間

が頻繁に発生します。

取引所の状態だけに依存すると、

- 二重エントリー
- クローズ漏れ
- 想定外の再注文

といった事故が起きます。

**State は外部に委ねるものではありません。**

---

## 長期運用 Bot に必要な State の例

長期運用を前提とする場合、  
最低限、以下のような State が必要になります。

- open position の有無
- 対象シンボル
- エントリー価格
- 現在のフェーズ（監視 / BE / クローズ待ち）
- 注文発行済みかどうか

重要なのは、  
**State を曖昧にしないこと**です。

「たぶん今はこの状態」という設計は、  
必ず破綻します。

---

## State をどこで管理するか

```python
# open_positions.json を使った簡易 State 管理
def load_open_positions():
    if not os.path.exists("open_positions.json"):
        return set()
    with open("open_positions.json", "r") as f:
        return set(json.load(f))

def save_open_positions(positions):
    with open("open_positions.json", "w") as f:
        json.dump(list(positions), f)
```

State 管理にはいくつか選択肢があります。

- メモリ上のみ
- ローカルファイル
- データベース

長期運用の初期段階では、  
**ローカルファイル管理** が現実的です。

理由は、

- 再起動後も復元できる
- 実装が単純
- 状態が可視化しやすい

からです。

重要なのは技術スタックではなく、  
**State が永続化されているかどうか** です。

---

## 再起動を前提とした設計
```python
# 起動時の State 復元
open_positions = load_open_positions()

for symbol in open_positions:
    # 起動時に exchange 状態と突き合わせる
    pos = get_position_from_exchange(symbol)
    if not pos:
        # State と実態がズレている場合は安全側に倒す
        open_positions.remove(symbol)

save_open_positions(open_positions)
```

長期運用では、  
Bot は必ず再起動されます。

- サーバー再起動
- デプロイ
- 想定外のクラッシュ

そのため、起動時に必ず、

- 現在の State を読み込む
- 取引所の状態と突き合わせる
- 矛盾があれば安全側に倒す

という処理が必要になります。

再起動 = 異常系  
ではなく、  
**通常の運用フローの一部** として扱うべきです。

---

## State 不整合が引き起こす典型的な事故

State 管理が甘いと、  
次のような事故が起きます。

- すでにポジションがあるのに再エントリー
- クローズ済みなのに監視を続ける
- 注文が出ていない前提で処理が進む

これらはすべて、  
ロジックの問題ではありません。

**State 設計の問題** です。

---

## State 管理の設計方針

長期運用 Bot では、  
次の方針が重要になります。

- State を明示的に定義する
- State 遷移を限定する
- State を永続化する
- State と実態がズレたら安全側に倒す

State 管理は、  
パフォーマンス向上のためのものではありません。

**事故を起こさないための設計** です。

---

## まとめ

長期運用 Bot において、  
State 管理は補助的な要素ではありません。

**中核となる設計要素** です。

- ロジックが正しくても
- Notional 管理が完璧でも

State が曖昧であれば、  
システムは長く動きません。

自動売買システムを  
「長期間、止めずに動かす」ためには、

- State を定義し
- State を保存し
- State を信用しすぎない

という設計が不可欠です。

次回は、  
State 管理と密接に関係する  
**WebSocket の信頼性と再接続設計** について整理します。
