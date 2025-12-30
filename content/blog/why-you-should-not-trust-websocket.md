---
title: "WebSocketは信用してはいけない"
date: 2025-12-30
tags: ["SystemDesign", "TradingBot", "WebSocket", "Operation", "Reliability"]
toc: true
---

## はじめに
### なぜ WebSocket を「信用してはいけない」のか

自動売買 Bot を作ると、  
多くの場合 WebSocket は「理想的なデータソース」として扱われます。

- 遅延が少ない
- イベント駆動で効率的
- REST よりリアルタイム

しかし、長期運用を前提としたシステムにおいて、  
**WebSocket を前提条件として信用する設計は非常に危険**です。

本記事では、  
なぜ WebSocket を「信用してはいけない」のか、  
そして **どのように扱うべきか** を  
実運用視点で整理します。

---

## WebSocket は「不安定である」ことが前提

まず前提として、  
WebSocket は以下を **保証しません**。

- 常時接続されていること
- イベントをすべて受信できること
- 順序通りに届くこと
- 遅延が一定であること

実運用では、次のような事象が**日常的に発生**します。

- 一瞬の切断（検知されない場合もある）
- ネットワーク瞬断
- Binance 側のメンテナンス
- ping/pong の失敗
- ローカル環境の負荷による停止

つまり、

> WebSocket は「切れるもの」である

という前提で設計しなければなりません。

---

## 「WebSocket が来ない＝何も起きていない」は誤り

多くの Bot が暗黙に持っている危険な前提があります。

> WebSocket にイベントが来ない  
> → 相場が動いていない

これは **誤り** です。

- WebSocket が落ちているだけ
- 自分のプロセスが詰まっているだけ
- イベントを取りこぼしているだけ

という可能性が常に存在します。

WebSocket は  
**唯一の真実のソース（Single Source of Truth）にしてはいけません。**

---

## WebSocket は「トリガー」であって「真実」ではない

正しい設計では、  
WebSocket の役割は次のように限定されます。

- 処理を開始する「きっかけ」
- 最新データを取りに行くための通知

つまり、

> WebSocket = トリガー  
> 状態判断 = 内部 State + 必要に応じて REST

という分離が必要です。

---

## 切断を前提としたループ設計

WebSocket を使う以上、  
**再接続は例外ではなく通常系** です。

以下は、  
切断を前提とした最小構成の設計例です。

```python
def start_socket(symbols):
    while True:
        try:
            ws = websocket.WebSocketApp(
                build_stream_url(symbols),
                on_message=on_message,
                on_error=on_error,
                on_close=on_close
            )
            ws.run_forever(ping_interval=20, ping_timeout=10)
        except Exception as e:
            log_error(e)

        # 切断後は必ず待ってから再接続
        time.sleep(5)
```

## WebSocket は信用してはいけない

WebSocket はリアルタイム性に優れていますが、  
**信頼できる通信手段ではありません。**

実運用では、以下は必ず発生します。

- 予告なしの切断
- メッセージ欠落
- 遅延・順序の入れ替わり
- 無言で止まる（例外も出ない）

そのため、WebSocket を  
「正しい情報源」として扱う設計は危険です。

重要なのは、次の前提に立つことです。

- 例外でプロセスを落とさない
- 切断後は必ず復帰する
- 一度で諦めない

WebSocket は  
**壊れる前提で使うもの**です。

たとえば、以下のように  
WebSocket からイベントが来ても  
State が NG なら処理を進めません。

```python
# WebSocket から signal が来ても
# State が NG なら何もしない
if symbol in load_open_positions():
    return
```

この設計により、

- WebSocket が二重にイベントを送ってきても
- 再接続直後に過去イベントが流れてきても

**事故は起きません。**

WebSocket がどれだけ不安定でも、  
最終的な挙動は State によって制御されます。

つまり、  
**State が最後の防波堤**になります。

## 再接続直後に必ずやるべきこと

再接続直後は、  
**最も危険なタイミング**です。

理由は明確です。

- State と実態がズレている可能性が高い
- 切断中のイベントを取りこぼしている
- 内部 State が古いまま残っている

そのため、再接続後には必ず、

- 現在の position を REST API で確認する
- open order の有無を確認する
- State と取引所の実態を突き合わせる

という処理が必要になります。

再接続後は、  
「通常運転に戻る前の整合性確認フェーズ」  
として扱うべきです。

```python
def reconcile_state(symbol):
    pos = get_position_from_exchange(symbol)
    if not pos:
        remove_open_position(symbol)
```

## WebSocket を信用すると起きる典型的な事故

WebSocket を前提に設計すると、  
次のような事故が発生します。

- WebSocket 切断中に相場が動き、未検知のまま進行
- 再接続後に過去イベントを拾い、二重エントリー
- クローズイベントを逃し、State が残り続ける
- Bot が止まっているのに、それに気づかない

これらはすべて、  
**WebSocket を信用した設計の結果**です。

## WebSocket との正しい付き合い方

WebSocket は便利ですが、  
**信用してはいけません。**

正しい設計では、  
次の思想が必要になります。

- WebSocket は切れる前提で使う
- WebSocket はトリガー用途に限定する
- 真実は State と REST 側にある
- 再接続は異常系ではなく通常系
- State が最後の安全装置になる

## まとめ

- WebSocket は不安定である
- 信用すると事故が起きる
- State 管理と組み合わせて初めて使える
- 再接続は例外ではない
- 「イベントが来ない＝何も起きていない」は誤り

自動売買 Bot を  
**長期・無人で動かす**ためには、

> WebSocket を疑い、  
> State を信じる

という設計思想が不可欠です。

次回は、  
これらすべてを支える  
**ログ設計（生き残るための Logging）**  
について整理します。

