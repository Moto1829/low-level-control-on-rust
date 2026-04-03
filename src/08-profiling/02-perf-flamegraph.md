# 02 - perf & flamegraph によるプロファイリング

## プロファイリングとは

マイクロベンチマークは「この関数は何ナノ秒かかるか」を答えますが、「プログラム全体のどこで時間を使っているか」は答えられません。プロファイリングはプログラムの実行中にサンプリングを行い、**CPU 時間の配分**を可視化します。

```
プロファイリングのアプローチ:

サンプリング型                   計装型
─────────────────                ─────────────────
定期的に実行スタックを記録       関数の入口/出口にフックを挿入
→ オーバーヘッド小               → オーバーヘッド大
→ 統計的な精度                   → 正確なカウント
→ 本番環境でも使える             → 開発・テスト向け

perf, Instruments はサンプリング型
```

---

## Linux: `perf` の基本

### インストール確認

```bash
# perf が使えるか確認
perf --version

# Ubuntu/Debian でインストール
sudo apt-get install linux-tools-common linux-tools-$(uname -r)

# Fedora/RHEL でインストール
sudo dnf install perf
```

### カーネルパラメータの設定（権限緩和）

```bash
# 一時的に権限を緩和（再起動で元に戻る）
sudo sysctl -w kernel.perf_event_paranoid=1
sudo sysctl -w kernel.kptr_restrict=0

# または ~/.cargo/config.toml に追加
```

### ビルド設定（デバッグシンボルを保持）

```toml
# Cargo.toml
[profile.release]
debug = true          # デバッグシンボルを保持（バイナリサイズは増えるが最適化はそのまま）
```

または `.cargo/config.toml` でプロジェクト全体に設定：

```toml
# .cargo/config.toml
[profile.release]
debug = true
```

---

### `perf record` - プロファイルの取得

```bash
# 基本的な使い方
cargo build --release
perf record --call-graph dwarf ./target/release/my-binary

# 引数がある場合
perf record --call-graph dwarf ./target/release/my-binary arg1 arg2

# cargo run 経由で実行
perf record --call-graph dwarf cargo run --release -- arg1

# サンプリング頻度を指定（デフォルト: 4000 Hz）
perf record -F 9999 --call-graph dwarf ./target/release/my-binary

# 結果は perf.data に保存される
ls -lh perf.data
```

`--call-graph dwarf` オプションの説明：

```
コールグラフの収集方法:
  fp (frame pointer)  → 高速だが精度低い（フレームポインタ省略最適化と競合）
  dwarf               → DWARF デバッグ情報を使用（推奨、精度高い）
  lbr                 → Last Branch Record（Intel CPU 限定、最も高精度）
```

---

### `perf report` - 結果の表示

```bash
# インタラクティブな TUI で表示
perf report

# テキスト形式で出力
perf report --stdio

# 特定のシンボルに絞り込み
perf report --stdio --symbol-filter my_function

# コールグラフ付きで表示
perf report --call-graph
```

`perf report` TUI の操作：

```
キー操作:
  Enter / →    展開（コールグラフを開く）
  ←            折りたたむ
  /            検索
  q            終了
  h            ヘルプ

表示例:
# Overhead  Command     Shared Object       Symbol
    45.23%  my-binary   my-binary           [.] my_project::hot_function
    23.11%  my-binary   libc.so.6           [.] malloc
    12.04%  my-binary   my-binary           [.] my_project::another_fn
```

---

### `perf stat` - 基本統計

```bash
# CPU カウンタの統計を表示
perf stat ./target/release/my-binary

# 出力例:
#  Performance counter stats for './target/release/my-binary':
#
#       1,234.56 msec task-clock           # 1.000 CPUs utilized
#              5      context-switches     # 0.004 K/sec
#              0      cpu-migrations       # 0.000 K/sec
#            789      page-faults          # 0.639 K/sec
#  3,456,789,012      cycles               # 2.799 GHz
#  5,678,901,234      instructions         # 1.64  insn per cycle
#  1,234,567,890      branches             # 1.000 G/sec
#      1,234,567      branch-misses        # 0.10% of all branches
```

キャッシュミスも確認できます：

```bash
perf stat -e cache-references,cache-misses,L1-dcache-loads,L1-dcache-load-misses \
    ./target/release/my-binary

# 出力例:
#    100,000,000    cache-references
#      5,000,000    cache-misses           # 5.00% of all cache refs
#    200,000,000    L1-dcache-loads
#     10,000,000    L1-dcache-load-misses  # 5.00% of all L1 dcache accesses
```

---

## `cargo-flamegraph` によるフレームグラフ

### インストール

```bash
# cargo-flamegraph をインストール
cargo install flamegraph

# Linux では perf が必要（インストール済みであること）
# macOS では DTrace または Instruments を使用

# 依存ツールの確認
which perf   # Linux
```

### フレームグラフの生成

```bash
# 基本的な使い方（--release ビルドで実行）
cargo flamegraph --bin my-binary

# 引数付きで実行
cargo flamegraph --bin my-binary -- arg1 arg2

# ベンチマークをプロファイル
cargo flamegraph --bench my_benchmark -- --bench

# 例: criterion ベンチマークのプロファイル
cargo flamegraph --bench my_benchmark -- sum_naive --bench

# 出力ファイルを指定
cargo flamegraph -o my_flamegraph.svg --bin my-binary

# root 権限なしで実行（Linux）
CARGO_PROFILE_RELEASE_DEBUG=true cargo flamegraph --bin my-binary
```

生成されたファイルを開く：

```bash
# Linux
xdg-open flamegraph.svg

# macOS
open flamegraph.svg

# または任意のブラウザで開く
firefox flamegraph.svg
```

---

## フレームグラフの読み方

```
フレームグラフの構造:

┌──────────────────────────────────────────────────────────┐
│                        main                              │  ← 最下段: エントリーポイント
│  ┌────────────────────────────────────────────────────┐  │
│  │                   process_data                     │  │
│  │  ┌──────────────────────────────┐  ┌────────────┐ │  │
│  │  │       sort_algorithm         │  │  serialize  │ │  │
│  │  │  ┌──────────┐  ┌──────────┐ │  └────────────┘ │  │
│  │  │  │  compare  │  │   swap   │ │                  │  │
│  │  │  └──────────┘  └──────────┘ │                  │  │
│  │  └──────────────────────────────┘                  │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
  ↑                                                        ↑
  左端                                                    右端
  (順序に意味はない、幅が重要)

幅 = CPU 時間の割合
  → 幅が広い関数ほど CPU を多く消費している

"プラトー"（平らな天井部分）= 最も多く時間を使っている関数
  → ここがホットスポット！
```

### フレームグラフの操作

SVG ファイルをブラウザで開くと、インタラクティブに操作できます：

```
操作方法:
  クリック     → その関数にズームイン
  Ctrl+F       → 関数名を検索
  右クリック   → 親のフレームに戻る
  マウスオーバー → その関数の詳細情報を表示

検索のコツ:
  - 自分のクレート名で検索して、ライブラリコードを除外
  - "alloc" で検索してメモリアロケーションのコストを確認
  - "[unknown]" が多い場合はデバッグシンボルの設定を確認
```

---

## ホットスポットの特定手順

### Step 1: 全体像の把握

```bash
# まず perf stat で全体の統計を確認
cargo build --release
perf stat ./target/release/my-binary

# instructions per cycle (IPC) に注目
# IPC > 2.0  → CPUバウンド（計算が多い）
# IPC < 1.0  → メモリバウンド（キャッシュミスが多い）
```

### Step 2: フレームグラフの生成と確認

```bash
cargo flamegraph --bin my-binary
open flamegraph.svg
```

### Step 3: ホットスポットのコードを確認

```bash
# ホットな関数のアセンブリを確認
cargo rustc --release -- --emit asm
# target/release/deps/*.s に出力される

# または objdump で確認
objdump -d target/release/my-binary | grep -A 50 "my_hot_function"
```

### Step 4: 最適化して再計測

```rust
// 最適化前
fn process_items(items: &[Item]) -> Vec<Result> {
    items.iter().map(|item| {
        let intermediate = expensive_allocation(item); // ← ホットスポット
        compute_result(&intermediate)
    }).collect()
}

// 最適化後: アロケーションを削減
fn process_items_optimized(items: &[Item]) -> Vec<Result> {
    let mut intermediate = IntermediateBuffer::new(); // 再利用可能なバッファ
    items.iter().map(|item| {
        intermediate.clear();
        fill_buffer(&mut intermediate, item); // アロケーションなし
        compute_result(&intermediate)
    }).collect()
}
```

---

## 実践例: ホットスポットを見つけて修正する

### 問題のあるコード

```rust
// src/main.rs
use std::collections::HashMap;

fn word_count(text: &str) -> HashMap<String, usize> {
    let mut counts = HashMap::new();
    for word in text.split_whitespace() {
        // 毎回 String アロケーションが発生！
        *counts.entry(word.to_string()).or_insert(0) += 1;
    }
    counts
}

fn main() {
    let text = include_str!("large_text_file.txt");
    for _ in 0..1000 {
        let _ = word_count(text);
    }
}
```

### フレームグラフで確認

```bash
cargo flamegraph --bin my-binary
# → "alloc::string::String::from" が大きな割合を占めていることを確認
```

```
フレームグラフのイメージ:
┌──────────────────────────────────────────────────────────┐
│ main                                                     │
│ ┌──────────────────────────────────────────────────────┐ │
│ │ word_count                                           │ │
│ │ ┌─────────────────────────┐ ┌──────────────────────┐ │ │
│ │ │  String::from (40%)     │ │  HashMap::insert(30%)│ │ │
│ │ │  ┌───────────────────┐  │ └──────────────────────┘ │ │
│ │ │  │  alloc (35%)      │  │                          │ │
│ │ └─────────────────────────┘                          │ │
│ └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
  → String::from と alloc が大半を占めている！
```

### 最適化: `&str` キーを使う

```rust
use std::collections::HashMap;

// 最適化版: 借用スライスをキーにしてアロケーション削減
fn word_count_optimized<'a>(text: &'a str) -> HashMap<&'a str, usize> {
    let mut counts = HashMap::new();
    for word in text.split_whitespace() {
        *counts.entry(word).or_insert(0) += 1; // アロケーションなし！
    }
    counts
}

fn main() {
    let text = include_str!("large_text_file.txt");
    for _ in 0..1000 {
        let _ = word_count_optimized(text);
    }
}
```

### ベンチマークで効果を確認

```rust
// benches/word_count.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn bench_word_count(c: &mut Criterion) {
    let text = "the quick brown fox jumps over the lazy dog ".repeat(10000);

    let mut group = c.benchmark_group("word_count");

    group.bench_function("with_string_alloc", |b| {
        b.iter(|| word_count(black_box(&text)))
    });

    group.bench_function("with_str_borrow", |b| {
        b.iter(|| word_count_optimized(black_box(&text)))
    });

    group.finish();
}
```

結果の例：

```
word_count/with_string_alloc  time: [15.234 ms 15.267 ms 15.301 ms]
word_count/with_str_borrow    time:  [4.123 ms  4.145 ms  4.168 ms]
                              change: [-72.94% -72.85% -72.76%]
                              Performance has improved.
```

約 3.7 倍の高速化を達成。

---

## macOS: Instruments による代替

Linux の `perf` に相当するのが macOS の **Instruments** です。

### コマンドラインから使う（`xctrace`）

```bash
# Xcode Command Line Tools が必要
xcode-select --install

# Time Profiler でプロファイル取得
xcrun xctrace record \
    --template 'Time Profiler' \
    --output my_profile.trace \
    --launch -- ./target/release/my-binary

# 結果を Instruments.app で開く
open my_profile.trace
```

### `cargo-flamegraph` を macOS で使う

macOS では `dtrace` を使ってフレームグラフを生成できます：

```bash
# dtrace ベースで実行（sudo 必要）
sudo cargo flamegraph --bin my-binary

# または環境変数で dtrace を指定
CARGO_PROFILE_RELEASE_DEBUG=true sudo cargo flamegraph --bin my-binary
```

### `samply` - モダンなクロスプラットフォームプロファイラ

```bash
# samply のインストール
cargo install samply

# プロファイル取得（macOS / Linux 両対応）
samply record ./target/release/my-binary

# ブラウザで Firefox Profiler が自動的に開く
```

`samply` は Firefox Profiler の UI を使用するため、直感的に操作できます。

---

## `cargo-instruments` (macOS 専用)

```bash
# インストール
cargo install cargo-instruments

# Time Profiler で実行
cargo instruments --template "Time Profiler" --bin my-binary

# Allocations テンプレートでメモリプロファイル
cargo instruments --template "Allocations" --bin my-binary

# 使用可能なテンプレートを確認
cargo instruments --list-templates
```

---

## よくある問題と解決策

### フレームグラフに `[unknown]` が多い

```bash
# 原因: デバッグシンボルがない
# 解決: Cargo.toml に debug = true を追加

[profile.release]
debug = true
```

### フレームグラフがほぼ `kernel` で埋まっている

```
原因: システムコールや I/O 待ちが多い
解決策:
  - ファイル I/O をバッファリングする
  - 非同期 I/O (tokio) の検討
  - システムコールの回数を減らす
```

### 計測中にプログラムが終わってしまう

```bash
# 処理を繰り返すループを入れるか、大きなデータを使う
# または perf record --pid でバックグラウンドプロセスにアタッチ
./target/release/my-server &
SERVER_PID=$!
perf record -p $SERVER_PID sleep 30  # 30秒間プロファイル
kill $SERVER_PID
```

---

## まとめ

| ツール | 用途 | プラットフォーム |
|--------|------|-----------------|
| `perf record` + `perf report` | 詳細な CPU プロファイル | Linux |
| `cargo flamegraph` | フレームグラフ生成 | Linux / macOS |
| `perf stat` | CPU カウンタ統計 | Linux |
| `samply` | モダンな UI プロファイル | Linux / macOS |
| `cargo instruments` | Instruments 統合 | macOS |
| `xcrun xctrace` | Instruments コマンドライン | macOS |

**プロファイリングの流れ**:
1. `perf stat` で全体像を把握
2. `cargo flamegraph` でホットスポットを特定
3. コードを修正
4. `criterion` で改善を計測
5. 繰り返す

次節では、コンパイラへのヒントを与えてコード生成を制御する方法を学びます。

→ [03 - コンパイラヒント](./03-compiler-hints.md)
