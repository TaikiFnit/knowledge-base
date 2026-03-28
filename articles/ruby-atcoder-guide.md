# 競プロRubyの速度限界と高速化の全技法

AtCoderでRubyを使い続けるための実践的パフォーマンスガイド。

## AtCoderのRuby実行環境

- **Ruby 3.2.2 (MRI)** / `--jit --yjit-exec-mem-size=128` (YJIT有効)
- 実行時間制限は **C++と同一**（言語補正なし）
- **ac-library-rb** が利用可能（DSU, セグ木, 遅延セグ木, BIT, 優先キュー等）
- 2025年秋: Ruby 3.4 + **TruffleRuby** 追加予定

## ループ構文の速度比較

| 構文 | 実行時間 (10^6回) |
|------|-------------------|
| `while` | **0.031s** |
| `times` | 0.050s |
| `each` | 0.051s |
| `for` | 0.054s |
| `loop` | 0.070s |

**10^6超のループではwhileが必須。**

## 入出力の高速化

```ruby
# 高速入力
N, M, *AS = $stdin.read.split.map(&:to_i)

# 高速出力（バッファリング）
buf = ""
results.each { |r| buf << "#{r}\n" }
puts buf
```

## 文字列をbytes変換

```ruby
# 遅い
s = gets.chomp
s[i] == "#"

# 速い
s = gets.chomp.bytes
s[i] == 35
```

## 配列操作の罠

```ruby
# 遅い（一時配列生成）
ans = [ans, result].max

# 速い
ans = result if result > ans
```

BFSキューはインデックス管理方式:

```ruby
queue = [start]
head = 0
while head < queue.size
  v = queue[head]
  head += 1
  # ...
end
```

## 多倍長整数(Bignum)の回避

```ruby
# 致命的に遅い
power = (i ** j) % MOD

# 高速（組み込み繰り返し二乗法）
power = i.pow(j, MOD)
```

`# frozen_string_literal: true` でGC圧力を軽減。

## TLEになりやすいパターン

- **ダイクストラ**: ac-library-rbのPriorityQueueを使う
- **セグ木**: 非再帰型 or ac-library-rb
- **O(N²) (N≤5000)**: Rubyではほぼ通らない
- **再帰DFS**: 明示的スタックに書き換え
- **O(N√N)**: 実質不可能

## レーティング別Ruby適性

- **灰〜茶**: 問題なし
- **緑〜水色**: 定数倍高速化で対処可能
- **青**: Crystal/C++併用推奨
- **黄以上**: Rubyのみでは極めて厳しい

## 提出前チェックリスト

1. ループをwhileに書き換えたか
2. `$stdin.read`で一括読み込みしているか
3. 文字列を`.bytes`変換しているか
4. MOD演算でBignum回避しているか
5. ac-library-rbを活用しているか
6. 再帰DFSを非再帰化したか
7. `[x,y].min/max`を直接比較に置き換えたか
8. BFSキューをインデックス管理にしているか

## 参考

- [universato氏 Ruby競プロTips 108](https://zenn.dev/universato/articles/20201210-z-ruby)
- [kona0001氏 定数倍高速化手法](https://kona0001.hatenablog.com/entry/2021/06/07/170343)
- [ac-library-rb](https://github.com/universato/ac-library-rb)
