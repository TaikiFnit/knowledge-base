---
title: "Deoptimization: YJITはなぜ「遅くする」ことで速くなるのか"
date: 2025-04-16
tags:
  - ruby
  - yjit
  - jit-compiler
  - deoptimization
  - rubykaigi
---

RUBYKAIGI 2025 / SESSION NOTES
 # Deoptimization: YJITはなぜ「遅くする」ことで速くなるのか
 RubyKaigi 2025 セッションノート — JITコンパイラの脱最適化メカニズムを深掘りする

 
 
 📅
 2025-04-16
 
 
 🎤
 Takashi Kokubun (@k0kubun) — Shopify YJIT Team
 
 

 
 
 3x
 Ruby 2.7比
 
 
 92%
 インタプリタ比速度向上
 
 
 Ruby 4.0
 最新対応
 
 

 ↓ スクロールして詳細を確認
 

 
 
 
 
 
 01
 ## 前提知識 — JITコンパイラと脱最適化
 

 Rubyの高速化を理解するには、まずRubyの実行モデルを知る必要があります。Rubyはインタプリタ言語ですが、その内部構造は非常に洗練されています。

 ### Rubyの実行パイプライン
 Rubyのコードは、**YARV（Yet Another Ruby VM）**というバイトコード仮想マシンによって実行されます。YARVはバイトコードを逐次解釈して実行するため、ネイティブコードを直接実行するC言語と比較すると、本質的にオーバーヘッドが存在します。

 ここで登場するのが**JIT（Just-In-Time）コンパイラ**です。JITは実行時にバイトコードをx86-64やARM64などの機械語に変換し、それをキャッシュすることで大幅な高速化を実現します。

 ### 脱最適化（Deoptimization）とは
 JITコンパイラが最適化コードを生成する際、いくつかの**仮定**を基に最適化を行います。例えば「このメソッドの呼び出し先は常にこのクラスである」といった仮定です。しかし、Rubyは動的言語であるため、こうした仮定が実行時に破られることがあります。

 **脱最適化（Deoptimization）**は、この仮定が破られた際に、最適化済みのマシンコードを安全に放棄し、インタプリタにフォールバックするメカニズムです。これにより、正確性を保ちながら条件的な最適化が可能になります。

 ### 他のJIT処理系における脱最適化
 脱最適化はYJITに固有のメカニズムではありません。JavaScriptエンジンV8、JavaのHotSpotなど、多くの産業標準的なJIT実装が同様の仕組みを採用しています。

 - **V8（JavaScript）**：Maglevコンパイラが、SSA値からbytecodeレジスタへのマッピングメタデータを生成し、脱最適化時にこれを使用して状態を復元
- **HotSpot（Java）**：同期的脱最適化（uncommon traps）と非同期的脱最適化の2種類をサポート
- **YJIT（Ruby）**：ガードチェック（guard check）で仮定を検証し、失敗時にはブレークアウトポイント経由でインタプリタにフォールバック

 YJITのアプローチは特に優れており、ガードチェックの失敗時にはその直後の命令からインタプリタが再開できるため、コンパイル時間を最小化しつつ安全性を確保しています。

 

 
 
 
 02
 ## YJITアーキテクチャの要点
 

 YJIT（Yet Another Ruby JIT）は、Ruby 3.1で初めて導入され、Ruby 3.2で本番環境対応（production-ready）となったJITコンパイラです。その設計哲学は「段階的な最適化」にあります。

 ### Lazy Basic Block Versioning (LBBV)
 従来的なJIT実装では、メソッド全体をコンパイル前に解析し、事前にすべての最適化機会を特定しようとします。しかし、Rubyのような動的言語では、コンパイル時点では変数の型が不確定であることが多く、この全体的なアプローチはしばしば効果的ではありません。

 YJITが採用する**Lazy Basic Block Versioning (LBBV)**は、**Maxime Chevalier-Boisvert**が考案した革新的な手法です。この方式では、メソッド全体を事前にコンパイルするのではなく、実行時に観察された型情報に基づいて、基本ブロック（basic block）単位で遅延的にコンパイルします。

 
 **基本ブロック（Basic Block）**とは、制御フローの分岐を持たない命令列のことです。LBBVはこの粒度で型情報を追跡し、型が変わるたびに新しいバージョンをコンパイルします。
 

 ### パフォーマンスの実績
 YJITの効果は明確です：

 - **Ruby 2.7との比較**：約3倍のベンチマークパフォーマンス向上
- **Ruby 3.3+YJIT**：92%のインタプリタ比速度向上
- **Rails 7.2以降**：YJITがデフォルトで有効化
- **Shopify本番環境**：Storefront Rendererで平均10%のレスポンスタイム改善

 これらの数字は、理論的な改善に留まらず、実際の本番環境で検証されたものです。Shopifyは数百万リクエスト/秒規模のeコマースプラットフォームでYJITを実行し、安定性とパフォーマンスの両立を確認しています。

 

 
 
 
 03
 ## Code Patching — 定数書き換え時に何が起きるか
 

 脱最適化の具体的なシナリオを理解するため、最も単純で有効な例として「定数の書き換え」を見てみましょう。

 ### 定数の最適化と無効化
 Rubyでは、定数は通常、変更されないことが前提されています。YJITはこの前提に基づき、定数参照を機械語の`mov`命令で直接埋め込みます。

 FOO = 1
def calculate
 FOO * 2 # YJITは FOO=1 を前提に mov 命令を生成
end

result = calculate # => 2

# ここで定数を再定義
FOO = 2 # 何が起きるか？

result = calculate # => 4 になるべきだが...

 上記のコードで定数`FOO`が再定義された場合、YJITが生成した最適化済みコードは古い値を参照し続けます。これは**正確性（correctness）の問題**です。

 YJITはこの問題を解決するため、定数が書き換わるたびに、その定数を参照していた最適化済みコードを**無効化**し、`jump`命令でインタプリタにフォールバックさせます。

 ### 細粒度の無効化メカニズム
 初期のYJIT実装では、単一の定数が変更されると、その定数に関連するすべてのコンパイル済みコードが一括で無効化されていました。しかし、Ruby 3.3以降は**Kevin Newton**がより細粒度の定数キャッシュ無効化メカニズムを実装しました。

 この改善により、定数変更時の無効化範囲を最小限に抑え、他の最適化済みコードへの影響を減らすことができるようになりました。

 
 **開発環境での工夫：**定数の頻繁な書き換えはYJITのコンパイルコストを増加させるため、開発環境でのみ定数を動的に変更し、本番環境では安定した定数値を使用することが推奨されます。
 
 

 
 
 
 04
 ## 動的変更への対応 — 限界と選択肢
 

 定数の書き換えに限りません。Rubyの動的性質により、YJITが事前に仮定できない多くのシナリオが存在します。

 ### TracePointによるローカル変数の書き換え
 `TracePoint`は、Rubyコードの実行をフックし、デバッガやプロファイラの実装に利用される強力なAPIです。しかし、この柔軟性には代償があります。

 one = 1
tp = TracePoint.new(:line) { |tp|
 # ローカル変数 one を外部から書き換え
 tp.binding.local_variable_set(:one, 5 * 10**15)
}
tp.enable
puts one # 元の値ではなく新しい値が出力される

 このようなシナリオでは、YJITがコンパイル時に「`one`は小さな整数」と仮定してしまうと、実際にはそうでない値が代入される可能性があります。

 ### binding.of_caller による動的アクセス
 `binding.of_caller(depth)`を使用することで、呼び出し元のスコープにアクセスし、ローカル変数を操作することも可能です。

 def modify_caller
 binding.of_caller(1).local_variable_set(:x, 999)
end

x = 10
modify_caller
puts x # 999 が出力される

 ### Singleton Classの動的変更
 特定のオブジェクトのクラスを動的に変更することも可能です。

 class Original; end
class Modified; end

obj = Original.new
# obj のクラスを動的に変更
class << obj
 prepend Modified
end

 ### メソッド再定義の複雑性
 以下の例では、`define`メソッド内で新しいメソッドが定義されることで、呼び出し対象が変わるシナリオを示します。

 def define(obj, flag)
 if flag
 def obj.method; 'modified'; end
 end
 obj
end

str = ""
result = str + define(str, true)
# str.method が呼ばれる場合、動的に定義されたメソッドが実行される

 
 **重要な制限：**TracePointを有効にすると、YJITのすべてのコンパイル済みコードが無効化されます。パフォーマンスクリティカルな本番環境では、TracePointの使用は控えるべきです。開発環境やデバッグ時のみの使用を推奨します。
 

 YJITは、これらの動的な変更が発生しうると判定した場合、該当するコードの最適化を避けるか、脱最適化によって安全にフォールバックします。つまり、「安全でない最適化は行わない」という原則を厳格に守っています。

 

 
 
 
 05
 ## Ruby 3.4の改善 — より賢い脱最適化
 

 Ruby 3.4では、脱最適化メカニズムと全体的なJIT統合が大幅に改善されました。

 ### ローカル変数とメソッド呼び出しの最適化改善
 Ruby 3.3では必ずしも最適化されなかった、ローカル変数の型追跡とメソッド呼び出しのインライン化が、Ruby 3.4では大幅に改善されました。特に、ループ内のメソッド呼び出しやクロージャ内のローカル変数アクセスが、より効率的にコンパイルされるようになりました。

 ### 統一されたメモリ管理
 Ruby 3.4で導入されたのが、統一的なJITメモリ管理です。`--yjit-mem-size`フラグで、JITが使用できる最大メモリサイズを制御できます。

 # デフォルト：128 MiB
ruby --yjit --yjit-mem-size=256 app.rb

# メモリが枯渇した場合、古いコンパイル結果から優先度の低いものが削除される

 このメモリ管理により、長時間実行されるサーバプロセスでも、メモリ使用量が予測可能な範囲内に収まるようになりました。

 ### メタデータ圧縮とメモリ効率化
 Ruby 3.3比で約8%のメモリ削減を実現しました。これは、コンパイル済みコードとその関連メタデータの構造を最適化することで達成されています。

 ### コンパイルログとデバッグ機能
 `--yjit-log`フラグを使用することで、どのメソッドやブロックがコンパイルされているかを追跡できます。これは、パフォーマンス最適化時に非常に有用です。

 ruby --yjit --yjit-log app.rb > yjit.log 2>&1
# yjit.log にはコンパイルされたすべての命令とその統計情報が記録される

 ### ランタイム統計情報
 `RubyVM::YJIT.runtime_stats`を使用することで、実行時のJIT統計を取得できます。

 stats = RubyVM::YJIT.runtime_stats
puts stats[:total_insns_count] # コンパイルされた命令総数
puts stats[:total_invalidations] # 脱最適化された回数
puts stats[:total_inlined_call_count] # インライン化されたメソッド呼び出し数

 ### コンパイル閾値の動的調整
 `--jit-call-threshold`（デフォルト30）は、メソッドがコンパイル対象となるまでの呼び出し回数です。Ruby 3.3以降では、大量のコードをロード時に自動的にこの閾値を引き上げ、ブート時のみ使われるメソッドの不要なコンパイルを回避します。

 これにより、Rails アプリケーション起動時のメモリ消費を削減しつつ、ホットパス（実際によく実行される部分）の最適化を確保しています。

 
 **本番環境での設定例：**
 `ruby --yjit --yjit-mem-size=256 --jit-call-threshold=100 app.rb`
 これにより、より多くのメモリを確保し、慎重なコンパイル方針を採用できます。
 
 

 
 
 
 06
 ## なぜArray#eachがCより速いのか — 脱最適化の勝利
 

 この章では、YJITの脱最適化メカニズムが生む、一見すると矛盾した現象を説明します。

 ### 従来のC実装の問題
 Ruby 3.3までの`Array#each`はC言語で実装されていました。この実装は確かに高速ですが、以下のような呼び出しのオーバーヘッドが発生します：

 > Ruby → C実装(each) → Ruby(ブロック) → C → Ruby(ブロック) → ...

 つまり、Ruby-C間の遷移が何度も発生し、各遷移の際に以下の処理が必要です：

 - スタックフレームの構築と破棄
- レジスタの退避と復元
- メモリバリアの適用
- GC安全ポイントのチェック

 ### Pure Ruby再実装とYJITの相乗効果
 Ruby 3.4（Feature #20182）では、`Array#each`, `Array#map`, `Array#select`などのコア列挙メソッドが**Pure Ruby# Ruby 3.3以前（C実装）
ary.each { |x| process(x) } # Ruby → C → Ruby → C → Ruby ...

# Ruby 3.4以降（Pure Ruby実装）
def each
 i = 0
 while i < @size
 yield @items[i]
 i += 1
 end
end

ary.each { |x| process(x) } # Ruby → Ruby → Ruby ...

 この変更により、YJIT有効時の呼び出しパターンが大きく変わります。

 ### クロスISEQ最適化とインライン化
 Pure Ruby実装になることで、YJITは以下の最適化が可能になりました：

 - メソッド境界の越境インライン化**：`each`メソッド内のループを、呼び出し元にインライン化
- **ブロック展開**：ブロックパラメータの型が安定している場合、ブロック呼び出しもインライン化
- **メモリ順序の最適化**：配列の連続アクセスをキャッシュフレンドリーな順序に最適化

 結果として、YJITはこれをほぼ単一の機械語の塊に最適化でき、C実装よりも速い実行速度を実現します。

 ### YJIT無効時の後方互換性
 重要な点として、YJIT無効時（`--yjit-disable`）または古いRubyバージョンでは、従来のC実装にフォールバックされます。これにより、後方互換性を完全に保ちながら、YJIT有効時の最適化を享受できます。

 # YJIT有効時：Pure Ruby実装が使用され、高度に最適化される
ruby --yjit app.rb

# YJIT無効時：C実装にフォールバック、安定性を優先
ruby --yjit-disable app.rb

 この戦略は、YJITの脱最適化メカニズムの本質を示しています。「最適化の失敗をしっかり検出して安全にフォールバックできる」ことが前提だからこそ、より積極的で有効な最適化を試みることができるのです。

 
 **パフォーマンスへの影響：**YJITが有効な環境でのArray#eachの性能は、C実装比で15～40%高速化することが報告されています。これは、メモリ取得パターンやキャッシュ効率の改善による効果です。
 
 

 
 
 
 07
 ## Ruby 4.0とYJITの現在地（2025年12月〜）
 

 Ruby 4.0は2025年12月25日にリリースされました。この版では、JITコンパイラの領域で大きな変化が起きています。

 ### ZJITの導入と位置付け
 Ruby 4.0では新しいJIT実装**ZJIT**が導入されました。ZJITはYJITより高度な最適化を目指しており、特に複雑なデータ構造やメタプログラミングの多いコードに対する対応を強化しています。

 しかし、本番環境の安定性という観点では、YJITは引き続き推奨されています。ZJITはまだ実験的な段階であり、より広範なテストと検証を経た後での本番利用を想定しています。

 ### 統計情報の強化
 Ruby 4.0では、`RubyVM::YJIT.runtime_stats`に新しい統計メトリクスが追加されました。

 stats = RubyVM::YJIT.runtime_stats
puts stats[:invalidate_everything] # 全コンパイルコード無効化の発生回数
puts stats[:global_invalidations] # グローバル無効化の回数
puts stats[:code_gc_count] # コード領域GCの実行回数

 `invalidate_everything`は、TracePoint有効化など重大なイベント時にのみ発生します。この統計は、パフォーマンス問題を診断する際に有用です。

 ### ARM64サポートの継続改善
 Apple Silicon対応は継続的に改善されています。Ruby 4.0では、ARM64特有の命令（SIMD、条件付き分岐最適化など）を活用した最適化が追加されました。これにより、MacBook Pro等でのパフォーマンスが従来比10～20%向上しました。

 ### バックグラウンドコンパイルの検討
 Ruby 4.0以降の開発ロードマップでは、**バックグラウンドコンパイル**の導入が検討されています。これにより、メインスレッドをブロックしない非同期コンパイルが実現され、レスポンスタイムのばらつきをさらに削減できる見込みです。

 ### 互換性宣言
 Ruby 4.0移行時、YJITから徐々にZJITへ移行することが予想されていますが、少なくともRuby 5.0まではYJITはサポートされることが公式に宣言されています。

 

 
 
 
 08
 ## 開発者のためのベストプラクティス
 

 YJITの脱最適化メカニズムを理解した上で、実際の開発でどのような心がけが必要か、実践的なガイドラインを示します。

 ### 避けるべきパターンと推奨パターン

 
 
 
 避けるべきパターン
 推奨パターン
 理由
 
 
 
 
 ホットパスでTracePoint使用
 開発環境のみに限定
 全コンパイル済みコード無効化により大幅な性能低下
 
 
 基本演算子の再定義（Integer#+ など）
 通常のメソッドオーバーライド
 広範囲の脱最適化を引き起こす
 
 
 頻繁なモンキーパッチ
 継承やモジュールミックスイン
 コード安定性を損なう
 
 
 ホットパスでの定数書き換え
 初期化時に定数を固定
 型ガードの失敗が頻繁に発生
 
 
 binding.of_callerの濫用
 明示的なパラメータ渡し
 スコープの追跡が困難に
 
 
 

 ### モニタリングとプロファイリング

 #### 1. コンパイル統計の確認
 `--yjit-stats`フラグを使用して、コンパイル結果を確認します。

 `ruby --yjit --yjit-stats app.rb 2>&1 | grep -A 20 "exit"`

 出力される`exit`セクションの統計から、脱最適化の理由を特定できます：

 - `bmethod_arity`：ブロック引数の数が一致しない
- `mid_mismatch`：メソッド名が変わった
- `klass_megamorphic`：型が多すぎる（メガモーフィック）
- `constant_changed`：定数が変更された

 #### 2. ランタイム統計の取得
 プログラムの実行中に、詳細な統計を取得できます。

 class ApplicationController
 before_action :log_yjit_stats

 private

 def log_yjit_stats
 stats = RubyVM::YJIT.runtime_stats
 Rails.logger.info "YJIT: #{stats[:total_insns_count]} insns, " \
 "#{stats[:total_invalidations]} invalidations, " \
 "#{stats[:total_inlined_call_count]} inlined calls"
 end
end

 #### 3. コンパイルログの追跡
 `--yjit-log`を使用して、詳細なコンパイルログを出力します。

 ruby --yjit --yjit-log app.rb > /tmp/yjit.log 2>&1

# ログサイズが大きい場合は、特定のメソッドに絞り込み
grep "Array#each" /tmp/yjit.log | head -20

 ### 本番環境での設定

 本番環境では、安定性とパフォーマンスのバランスを取った設定が推奨されます。

 # Railsの config/puma.rb の例
pidfile ENV.fetch("PIDFILE") { "tmp/pids/server.pid" }

# YJIT設定
yjit_heap_mb = ENV.fetch("YJIT_HEAP_MB") { "256" }
yjit_call_threshold = ENV.fetch("YJIT_CALL_THRESHOLD") { "100" }

# Puma設定
if ENV.fetch("YJIT_ENABLE") { "true" } == "true"
 ENV["RUBY_YJIT_ENABLE"] = "1"
 ENV["RUBY_YJIT_HEAP_SIZE_MB"] = yjit_heap_mb
 ENV["RUBY_JIT_CALL_THRESHOLD"] = yjit_call_threshold
end

 
 **推奨値：**
 - YJIT_HEAP_MB：256～512（サーバーメモリに応じて）
- YJIT_CALL_THRESHOLD：100～200（スタートアップ時間vs最適化のバランス）

 

 ### デバッグ時の注意

 デバッグ中にTracePointやbinding.irb等を使用する場合、YJITが全コンパイルコードを無効化することを意識しましょう。

 # デバッグ中はYJITを無効化
RUBY_YJIT_ENABLE=0 ruby -r debug app.rb

# または、デバッグメソッドのみで使用
def debug_info
 puts RubyVM::YJIT.runtime_stats if defined?(RubyVM::YJIT)
end

 ### マイグレーション戦略

 既存のRubyプロジェクトをYJIT対応にする際の段階的なアプローチ：

 1. **開発環境で有効化：**`ruby --yjit app.rb`で基本動作確認
2. **テスト環境で有効化：**CI/CDパイプラインでYJITを有効化し、互換性を検証
3. **ステージング環境で試験：**本番同等の負荷でパフォーマンスを測定
4. **段階的なロールアウト：**本番環境の一部サーバーで有効化し、メトリクスを監視
5. **全体有効化：**十分な実績を積んだ後、全サーバーで有効化

 

 
 
 
 09
 ## まとめ — 脱最適化の意義
 

 本記事を通じて、YJITの脱最適化メカニズムがRubyの性能向上にいかに重要であるかを見てきました。

 ### 脱最適化は「失敗」ではなく「選択肢」

 脱最適化という概念は、一見すると「最適化が失敗した」という否定的なニュアンスを持つように思えます。しかし、実態は逆です。

 > 脱最適化があるからこそ、より積極的で有効な最適化を安全に試みることができる。

 YJITは、「仮定が破られたら安全に戻ればよい」という信念の下に、大胆な最適化を行います。この設計哲学により、Rubyは言語としての動的性を完全に保ちながら、C言語に匹敵する実行速度を手に入れたのです。

 ### 継続的な改善とロードマップ

 Ruby 3.2から始まったYJITの進化は、現在も継続しています：

 - Ruby 3.3：定数キャッシュの改善、メモリ使用量の削減
- Ruby 3.4：ローカル変数の最適化、Array#each等のPure Ruby化、メモリ管理の統一
- Ruby 4.0：ZJIT導入、ARM64対応の強化、バックグラウンドコンパイルの検討

 このロードマップは、YJITが単なる「実験的な機能」ではなく、Ruby言語の中核的なコンポーネントへと進化しつつあることを示しています。

 ### 開発者への期待

 YJITの脱最適化メカニズムを理解することで、開発者が意識すべきことは意外と少ないです。基本的には：

 - 定数は可能な限り変更しない
- 基本演算子の再定義は避ける
- TracePointはデバッグ環境に限定する
- 本番環境でのメトリクス収集を習慣化する

 これらの心がけは、YJITに限らず、一般的なプログラミングのベストプラクティスと合致しています。

 ### 結論

 Rubyは、動的言語としての柔軟性と、コンパイル言語に近い実行速度を両立させるという、従来は相反するものと考えられていた目標を達成しました。その鍵となるのが、YJITの脱最適化メカニズムです。

 「遅くする」選択肢があるからこそ、「速くする」ことができる。これはコンピュータサイエンスにおける深い真実であり、Rubyエコシステムがこれからも進化し続ける源動力となるでしょう。

 
 **参考：**本記事は、RubyKaigi 2025でTakashi Kokubun氏により発表された"Deoptimization: Why YJIT Makes Code Slower to Make It Faster"に基づいています。詳細は、公式スライドおよび参考資料をご覧ください。
 
 

 
 
 ### 参考資料 — Further Reading
 - [RubyKaigi 2025 - Deoptimization: Why YJIT Makes Code Slower to Make It Faster](https://rubykaigi.org/2025/presentations/k0kubun.html)
- [Takashi Kokubun - Presentation Slides (SpeakerDeck)](https://speakerdeck.com/k0kubun/rubykaigi-2025)
- [Shopify Engineering Blog - YJIT: The New Ruby JIT Compiler](https://shopify.engineering/yjit-just-in-time-compiler-cruby)
- [Rails at Scale - YJIT 3.4: Even Faster and More Memory Efficient](https://railsatscale.com/2025-01-10-yjit-3-4-even-faster-and-more-memory-efficient/)
- [Ruby Issue #20182 - Array#each and other enumerable methods in Ruby](https://bugs.ruby-lang.org/issues/20182)
- [YJIT Benchmark Dashboard - Real-time Performance Metrics](https://speed.yjit.org/)
- [Shopify Ruby GitHub Repository - YJIT Implementation](https://github.com/Shopify/ruby)
- [Ruby Documentation - FAQ on Performance](https://ruby-doc.org/docs/ruby-doc-bundle/FAQ/FAQ.html)