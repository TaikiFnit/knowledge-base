---
title: Ractor Local GC — 並行Rubyのためのガベージコレクション改善
date: 2025-04-18
tags:
  - ruby
  - analysis
  - research
---

# Ractor Local GC — 並行Rubyのためのガベージコレクション改善

RubyKaigi 2025 / セッションノート

**登壇者:** Koichi Sasada | **日付:** 2025年4月18日 | **所属:** STORES, Inc.

## 主な指標

- **4x** — マイクロベンチ改善
- **Local** — Ractor分離型GC
- **3.5** — 目標リリース (Dec 2025)

---

## 01 / 前提知識

### Ractorとは — Ruby 3.0の並行モデル

Ruby 3.0で導入された**Ractor**は、GVL (Global VM Lock) を突破する新しい並行実行モデルです。スレッドとは異なり、各Ractorは独立したメモリ空間を持ち、オブジェクトを共有しません。

#### Ractorの基本特性

- **GVL per Ractor:** 各Ractorが独自のGVLを持つため、複数Ractorで真の並列実行が可能
- **オブジェクト分離:** 異なるRactor間でオブジェクトを直接参照できない（メモリ安全性）
- **メッセージパッシング:** Ractor.yield/Ractor#takeやRactor::Portを介した通信
- **独立したスレッド:** 各Ractorが独自のスレッドを持ち、スレッドプール化も可能

#### Shareableオブジェクト

オブジェクト分離という原則の中で、特定のオブジェクトは複数のRactor間で**安全に共有可能**です。これらが**Shareable Objects**です：

```ruby
# Shareable Objects の例
Ractor.shareable?(42)           # => true (小整数)
Ractor.shareable?(true)         # => true (ブール値)
Ractor.shareable?(nil)          # => true (nil)
Ractor.shareable?(:symbol)      # => true (シンボル)
Ractor.shareable?("frozen")     # => true (frozen文字列)
Ractor.shareable?(Regexp)       # => true (クラス/モジュール)

# 非Shareable Objects
Ractor.shareable?([1, 2, 3])    # => false (配列)
Ractor.shareable?({"a" => 1})   # => false (ハッシュ)
```

非Shareableオブジェクトは、Ractor間で送受信される際に**ディープコピー**されます。これはメモリ安全性を保証しますが、パフォーマンスオーバーヘッドの原因にもなります。

---

## 02 / なぜLocal GCが必要か

### 現行Global GCの問題点と性能瓶颈

Ruby 3.0〜3.4では、複数Ractorが実行されていても**ガベージコレクション (GC) はグローバル**です。つまり、GC実行時には全てのRactorを一時停止する必要があります。

#### Global GCの弊害

- **全Ractor停止:** 一つのRactorでGCが発動すると、他の全てのRactorが一時停止
- **同期オーバーヘッド:** 全Ractorを同期させるためのロック機構が複雑化
- **予測不可能なレイテンシ:** マルチRactorアプリケーション下では、GC一度で全体が停止
- **スケーラビリティ低下:** Ractor数が増えるほどGC停止時間が相対的に長くなる

#### ベンチマークから見る影響

特に**String生成**のようなアロケーション頻繁な処理では、GCの同期待機がボトルネックになります。複数Ractor間での並列実行に対してGCが同期化されることで、理論値の並列スケールが損なわれます。

> **重要:** Ruby 3.4までは「Ractor便利だがGCが課題」という状態が続いており、プロダクション環境での導入が制限されていました。

---

## 03 / Shareable Objectsの壁

### 参照追跡の複雑性とレースコンディション

Ractor Local GC実装の最大の課題は、**Shareable Objects**の存在です。クラスやモジュール、フローズンオブジェクトは複数Racterから参照されるため、GC時の参照追跡が極めて複雑になります。

#### 問題の具体例

```ruby
class MyClass
  attr_accessor :data
end

# 複数Ractor間で共有されるクラス
r1 = Ractor.new { MyClass.new.data = "ractor1" }
r2 = Ractor.new { MyClass.new.data = "ractor2" }

# MyClassはShareable（クラスだから）
# しかしそのインスタンスは非-Shareableで、各Ractorローカル
# GC時には、各Ractorローカルオブジェクトから、
# 共有クラスへの参照チェーンを正確に追跡する必要がある
```

#### 参照追跡の複雑性

- **クロスRactor参照:** Shareable Objectが他Ractorの非-Shareableオブジェクトを参照する可能性
- **マーク・スイープの同期:** Markフェーズで複数Ractor間の参照を正確に追跡
- **レースコンディション:** GC実行中にRactor間でオブジェクトが送受信される場合の安全性
- **参照チェーンの分岐:** Shareable ObjectsはRootのようにGCから扱われるべきだが、実装が複雑

#### 従来の全体・部分GC分離案の限界

もし「各RactorがLocal GC、共有オブジェクトはGlobal GC」という単純な分離を試みると、各Ractor LocalオブジェクトからSharedオブジェクトへのポインタ追跡で、事実上全Ractor を同期させなければならず、Local GCの利点が失われます。

> **核心:** 「完璧な分離」を求めると実装コストが極めて高い。この課題を解決する「現実的」なアプローチが必要でした。

---

## 04 / 提案手法

### 「Done is Better than Perfect」— Shareable Objectsを収集しない戦略

Koichi Sasadaが提案した革新的なアプローチは、**哲学的な転換**です。完璧な参照追跡を放棄し、Shareable Objectsを**GCのRootとして扱う**ことで、Local GCを現実可能にしました。

#### 提案手法の概要

```
┌─────────────────────────────────────────┐
│         Global GC (必要に応じて実行)    │
│  全Racterを停止し、Shareableオブジェクト  │
│  のみをマーク・スウィープ                 │
└─────────────────────────────────────────┘
         ↑ (稀な発動)
         │
┌────────┴───────┬──────────┬──────────┐
│ Ractor 1       │ Ractor 2 │ Ractor 3 │
│ Local GC       │ Local GC │ Local GC │
│ (並列実行)     │ (並列実行) │ (並列実行) │
└────────────────┴──────────┴──────────┘
```

#### アルゴリズムの詳細

1. **Shareable Objects を Root として固定:**
   - クラス、モジュール、フローズンオブジェクトなどは、GCの「Markフェーズ」で最初から「生存」状態に設定
   - 参照チェーンを追跡しない（つまり、Shareableオブジェクト自体の内部参照も収集対象外）

2. **Ractor Local GC:**
   - 各Ractorは、自分のローカルヒープに対して独立してGCを実行
   - 他のRactorを停止させない
   - Mark/Sweepフェーズは完全に並列

3. **Global GC (稀に実行):**
   - Shareable Objectsの参照カウントやアロケーション量が閾値を超えた場合のみ発動
   - その時のみ全Racterを停止

#### なぜこれが機能するのか

> **重要な視点:** Shareable Objectsの数は、通常のRubyプログラムにおいて**非常に少ない**です。クラス定義は初期化時に一度だけ行われ、フローズン文字列も設定数は限定的。つまり、「Global GCの不可避性」より「Local GCの利益」が圧倒的に勝ります。

この戦略により：

- ✓ 実装が大幅に簡潔化（参照チェーンの完全追跡を放棄）
- ✓ Shareable Objects周辺の複雑な同期処理が不要
- ✓ 99%のケースでLocal GCが動作、並列性を実現
- ✓ 完璧性を放棄することで、実用性を獲得

---

## 05 / パフォーマンス評価

### ベンチマーク結果 — 改善と課題

Ractor Local GCの実装では、マイクロベンチマークで顕著な改善が観測されています。同時に、特定のワークロードでの課題も明らかになっています。

#### パフォーマンス向上: String生成

| Ractor数 | Global GC (従来) | Local GC (提案) | 改善率 |
|----------|-----------------|-----------------|--------|
| 4 Ractor | 1.0x (基準) | 4.0x | **+300%** |
| 8 Ractor | 1.0x (基準) | 3.5x | **+250%** |
| 16 Ractor | 1.0x (基準) | 2.5x | **+150%** |

**洞察:** String生成は頻繁なアロケーションとGCの回数が増えます。Local GCにより各Ractorが独立してGCできるため、同期待機が消滅し、並列性が大幅に向上します。

#### パフォーマンス低下: Long-lived Objects

| ワークロード | Global GC (従来) | Local GC (提案) | 相対変化 |
|------------|-----------------|-----------------|---------|
| 長寿命オブジェクト | 1.0x (基準) | 0.25x | **-75% (悪化)** |

**原因分析:** 長寿命オブジェクトは、各Racterの独立したMarkフェーズで何度も参照され、追跡オーバーヘッドが蓄積します。また、Shareable Objectsを除外することで、一部の参照チェーンが「死んだ重み」として残る可能性があります。

#### 正規表現マッチング: 中程度の改善

複雑な正規表現マッチングは、String生成とLong-lived Objectsの中間的なパフォーマンスを示します：

```ruby
# 16 Ractor で正規表現処理
# Global GC:   基準 1.0x
# Local GC:    改善 2.0x (16Racterで)
```

#### ベンチマークの解釈

> **重要な注記:** これらはマイクロベンチマークであり、実際のアプリケーション（Web フレームワーク、データ処理など）での改善率は異なる可能性があります。ワークロード特性、Ractor数、オブジェクト生存期間に大きく依存します。

一般的には、以下の特性を持つアプリケーションで最大の効果が期待できます：

- ✓ 多数のRactorが並列で動作している
- ✓ アロケーション頻度が高い（String, Array など）
- ✓ 各Ractorが相対的に独立した作業を行っている
- ✗ 長寿命なオブジェクトの参照が複雑
- ✗ Shareable Objectsへの頻繁なアクセス

---

## 06 / Ractor::Port

### 新しい通信プリミティブへの移行

Ractor Local GCと同時に進行している施策の一つが、**Ractor::Port**の導入です。これは従来の`Ractor.yield`と`Ractor#take`に代わる、より効率的で安全な通信メカニズムです。

#### 従来のAPI: yield/take の問題

```ruby
# 従来の方式
r = Ractor.new do
  value = Ractor.yield("hello")  # 値を返す
  # ... 処理
end

result = r.take  # 値を受け取る
```

- **タグ衝突:** receive_ifを使う場合、複数メッセージ送受信時にタグ指定が複雑化
- **双方向ロック:** rendezvous同期により、送受信双方がロック必要（パフォーマンス低下）
- **複雑な合成:** 複数チャネルの組み合わせが難しい
- **非推奨化への流れ:** Ruby 4.0での削除が検討されている

#### 新API: Ractor::Port

```ruby
# Ruby 4.0以降の方式
port = Ractor::Port.new

r1 = Ractor.new(port) do |p|
  p.send("message1")
  p.send("message2")
end

r2 = Ractor.new(port) do |p|
  msg1 = p.receive
  msg2 = p.receive
  # ... 処理
end
```

#### Ractor::Portの利点

| 特性 | yield/take | Ractor::Port |
|------|-----------|--------------|
| ロック構造 | 双方向 (rendezvous) | 片方向 (受信側のみ) |
| パフォーマンス | 2回コピー (Sender → Channel → Receiver) | 1回コピー (Sender → Receiver) |
| メッセージ衝突 | タグシステムで対応 | Portインスタンスが一意 |
| 合成可能性 | 低い | 高い (複数Portを組み合わせ可能) |

#### Ractor Local GCとの相乗効果

Ractor::Portは、オブジェクトのコピー回数を削減することで、Local GCとの組み合わせで更なるパフォーマンス向上を実現します：

- コピー回数削減 → アロケーション圧力低下 → GC頻度削減
- Ractor Local GCの恩恵をより大きく受ける

> **推奨:** Ruby 3.5以降では、新規コード作成時にはRactor::Portの使用が強く推奨されます。yield/takeはdeprecated予定です。

---

## 07 / Ruby 3.5〜4.0での実装状況

### 進捗状況と課題

#### Ruby 3.5 (Dec 2025) の目標

RubyKaigi 2025の発表時点では、Ractor Local GCの**基本実装はほぼ完了**状態にあります。目標リリース予定は**Ruby 3.5 (2025年12月)**です。

**実装済み:**

- ✓ Ractor ローカルMarkフェーズの並列化
- ✓ Shareableオブジェクトを Rootとして固定
- ✓ Global GCの必要性最小化
- ✓ マイクロベンチマークでの検証

**残課題:**

- △ 実Rubyアプリケーションでの動作確認
- △ Edge caseの対応 (GC tracer, weakref など)
- △ メモリリーク修正（残存issues対応）
- △ パフォーマンス最適化（Long-lived objectsの課題）

#### Ruby 3.4.0 以降の関連改善

Ruby 3.4.0 (Dec 2024) では、完全な Local GC実装には至っていませんが、基礎的な機構が改善されました：

```ruby
# Ruby 3.4 以降で利用可能な API
Ractor[key]              # Ractor ローカルストレージへの読み取り
Ractor[key] = value      # Ractor ローカルストレージへの書き込み
Ractor.store_if_absent   # スレッド安全な初期化
```

#### Ruby 4.0 への展望

Ruby 4.0では以下が予定されています：

- Ractor Local GCの安定化と最適化
- Ractor::Portの標準化
- yield/take のdeprecation / 削除の可能性
- Racter関連のAPI整理

> **留意点:** 現在（Ruby 3.4時点）、Ractor Local GCは**まだexperimental**な機能です。プロダクション環境での導入には、十分なテストが必須です。

---

## 08 / まとめ

### Ractor Local GCの意義と今後の展望

Koichi Sasada と Ruby コミュニティの取り組みにより、以下が達成されました：

1. **完璧性の放棄から現実性へ:**
   「全ての参照を完璧に追跡する」という理想を放棄し、「Shareable Objectsを除外する」という実用的アプローチを採用。これにより実装可能性が格段に向上しました。

2. **並列性の獲得:**
   複数Ractorが独立してGCを実行できるようになり、99%のアロケーションシナリオで全体の停止が不要になります。

3. **マイクロベンチでの4x改善:**
   String生成などのアロケーション頻繁な処理で、4倍のスループット向上が確認されました。

4. **API改善による相乗効果:**
   Ractor::Portの導入により、オブジェクトコピー回数が削減され、GCの負荷がさらに軽減されます。

### 実践への示唆

Rubyエンジニアにとっての含意：

- **Ruby 3.5以降での並行プログラミング:** Ractorの実用性が大幅に向上。CPU-boundな並列タスクがより効率的に実行可能
- **新規コード:** Ractor::Portの使用を前提に設計。yield/takeは段階的に廃止予定
- **メモリ最適化:** Long-lived objectsの構造化に注意。Shareableオブジェクトの参照を整理
- **ベンチマーク文化:** 自身のアプリケーションでの性能向上を実際に計測することが重要

### Ruby並行プログラミングの進化

> Ractor Local GCは、Ruby が真の「並行言語」へと進化していることを象徴しています。スレッドの時代から Ractor、そして Local GC へ — Ruby は段階的ながらも着実に、メモリ安全性と並列性の両立を実現してきました。

### 参考資料とリンク

- [RubyKaigi 2025 - Toward Ractor local GC](https://rubykaigi.org/2025/presentations/ko1.html)
- [Feature #17100 - Ractor: a proposal for a new concurrent abstraction](https://bugs.ruby-lang.org/issues/17100)
- [Bug #18733 - Ruby GC problems cause performance issue with Ractor](https://bugs.ruby-lang.org/issues/18733)
- [Feature #21262 - Ractor::Port](https://bugs.ruby-lang.org/issues/21262)
- [DEV Community - Ractor::Port: Revamping the Ractor API](https://dev.to/ko1/ractorport-revamping-the-ractor-api-98)
- [Ruby 3.4 Ractor Documentation](https://docs.ruby-lang.org/en/3.4/ractor_md.html)

---

**編集後記:** 本記事は RubyKaigi 2025 での Koichi Sasada の講演をもとに、Ractor Local GC の概念と実装アプローチを整理したセッションノートです。Ruby のバージョン進化に伴い、内容は更新される可能性があります。

Ractor Local GC — 並行Rubyのためのガベージコレクション改善 | RubyKaigi 2025 セッションノート | 2025年4月18日

このページは Ruby コミュニティの理解向上を目的に作成されました。
