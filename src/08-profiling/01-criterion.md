# 01 - criterion によるマイクロベンチマーク

## なぜ `std` の計測では不十分なのか

素朴な時間計測には多くの落とし穴があります。

```rust
// NG: 信頼性の低い計測
use std::time::Instant;

fn naive_bench() {
    let start = Instant::now();
    let result = my_function(1000);
    let elapsed = start.elapsed();
    println!("elapsed: {:?}", elapsed);
}
```

この方法の問題点：

- **ウォームアップなし**: CPU キャッシュが冷えた状態で計測
- **1回計測**: ノイズの影響が排除できない
- **コンパイラ最適化**: 使われない結果は最適化で消去される可能性
- **統計なし**: 分散・外れ値が分からない

`criterion` はこれらすべての問題を解決します。

---

## セットアップ

### `Cargo.toml` の設定

```toml
[package]
name = "my-project"
version = "0.1.0"
edition = "2021"

[dependencies]
# プロダクションコードの依存関係

[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

# ベンチマークの自動検出を無効化（criterion は手動設定が必要）
[[bench]]
name = "my_benchmark"
harness = false
```

### ディレクトリ構造

```
my-project/
├── Cargo.toml
├── src/
│   └── lib.rs
└── benches/
    └── my_benchmark.rs   ← ベンチマークファイル
```

---

## 基本的なベンチマーク

### 計測対象の関数

```rust
// src/lib.rs

/// ナイーブな合計計算（最適化なし）
pub fn sum_naive(data: &[u64]) -> u64 {
    let mut total = 0u64;
    for &x in data {
        total = total.wrapping_add(x);
    }
    total
}

/// イテレータを使った合計計算
pub fn sum_iter(data: &[u64]) -> u64 {
    data.iter().copied().fold(0u64, u64::wrapping_add)
}

/// チャンク処理による合計計算（SIMD フレンドリー）
pub fn sum_chunked(data: &[u64]) -> u64 {
    let mut acc0 = 0u64;
    let mut acc1 = 0u64;
    let mut acc2 = 0u64;
    let mut acc3 = 0u64;

    let chunks = data.chunks_exact(4);
    let remainder = chunks.remainder();

    for chunk in chunks {
        acc0 = acc0.wrapping_add(chunk[0]);
        acc1 = acc1.wrapping_add(chunk[1]);
        acc2 = acc2.wrapping_add(chunk[2]);
        acc3 = acc3.wrapping_add(chunk[3]);
    }

    let mut total = acc0.wrapping_add(acc1).wrapping_add(acc2).wrapping_add(acc3);
    for &x in remainder {
        total = total.wrapping_add(x);
    }
    total
}
```

### ベンチマークの実装

```rust
// benches/my_benchmark.rs

use criterion::{black_box, criterion_group, criterion_main, Criterion};
use my_project::{sum_naive, sum_iter, sum_chunked};

fn bench_sum(c: &mut Criterion) {
    // テストデータの準備
    let data: Vec<u64> = (0..1024).collect();

    // 各実装を比較
    c.bench_function("sum_naive", |b| {
        b.iter(|| sum_naive(black_box(&data)))
    });

    c.bench_function("sum_iter", |b| {
        b.iter(|| sum_iter(black_box(&data)))
    });

    c.bench_function("sum_chunked", |b| {
        b.iter(|| sum_chunked(black_box(&data)))
    });
}

criterion_group!(benches, bench_sum);
criterion_main!(benches);
```

### 実行

```bash
# リリースビルドでベンチマーク実行（必ず --release を付ける）
cargo bench

# 特定のベンチマークのみ実行
cargo bench -- sum_naive

# パターンマッチで絞り込み
cargo bench -- "sum"
```

---

## `black_box` の必要性

`black_box` はコンパイラの最適化を意図的に阻害するためのバリアです。

```rust
// NG: コンパイラが結果を事前計算して最適化してしまう可能性
c.bench_function("sum_bad", |b| {
    let data = vec![1u64, 2, 3, 4, 5];
    b.iter(|| sum_naive(&data))  // コンパイラが定数畳み込みで消去するかもしれない
});

// OK: black_box で最適化バリアを設置
c.bench_function("sum_good", |b| {
    let data = vec![1u64, 2, 3, 4, 5];
    b.iter(|| sum_naive(black_box(&data)))  // 入力を「不透明」にする
});

// 出力側も black_box が必要な場合
c.bench_function("sum_output", |b| {
    let data: Vec<u64> = (0..1024).collect();
    b.iter(|| {
        let result = sum_naive(black_box(&data));
        black_box(result)  // 使われない戻り値の消去を防ぐ
    })
});
```

`black_box` の仕組み：

```
通常のコンパイル:
  result = sum_naive(&data)
  // result が使われなければコンパイラが sum_naive 自体を消去

black_box あり:
  result = sum_naive(black_box(&data))
  black_box(result)
  // コンパイラは &data の内容が何かを「知らないふり」をする
  // → 最適化で消去できない
```

---

## `BenchmarkGroup` による比較

複数の実装を同じグループでまとめて比較できます。

```rust
use criterion::{
    black_box, criterion_group, criterion_main,
    BenchmarkId, Criterion, Throughput,
};
use my_project::{sum_naive, sum_iter, sum_chunked};

fn bench_sum_sizes(c: &mut Criterion) {
    let mut group = c.benchmark_group("sum_comparison");

    // スループットの単位を設定（バイト数で表示）
    // → criterion が MB/s などの単位で表示してくれる

    for size in [64usize, 256, 1024, 4096, 16384] {
        let data: Vec<u64> = (0..size as u64).collect();

        // スループットをバイト数で設定
        group.throughput(Throughput::Bytes((size * 8) as u64));

        group.bench_with_input(
            BenchmarkId::new("naive", size),
            &data,
            |b, data| b.iter(|| sum_naive(black_box(data))),
        );

        group.bench_with_input(
            BenchmarkId::new("iter", size),
            &data,
            |b, data| b.iter(|| sum_iter(black_box(data))),
        );

        group.bench_with_input(
            BenchmarkId::new("chunked", size),
            &data,
            |b, data| b.iter(|| sum_chunked(black_box(data))),
        );
    }

    group.finish();
}

criterion_group!(benches, bench_sum_sizes);
criterion_main!(benches);
```

---

## セットアップとティアダウン

ベンチマーク対象外の処理（データ生成など）をループ外に出す方法：

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn bench_sort(c: &mut Criterion) {
    let mut group = c.benchmark_group("sorting");

    // iter_batched: 毎回新鮮なデータを用意しつつ、生成コストを計測外にする
    group.bench_function("vec_sort", |b| {
        b.iter_batched(
            // セットアップ: 毎回実行されるが計測対象外
            || {
                let mut v: Vec<u32> = (0..1000).rev().collect();
                v
            },
            // ベンチマーク本体: ここだけ計測される
            |mut v| {
                v.sort();
                black_box(v)
            },
            // バッチサイズのポリシー
            criterion::BatchSize::SmallInput,
        )
    });

    group.finish();
}

criterion_group!(benches, bench_sort);
criterion_main!(benches);
```

### `iter_batched` のバッチサイズポリシー

| ポリシー | 説明 | 用途 |
|----------|------|------|
| `SmallInput` | 小さいバッチ（最大1万要素） | 少量データ |
| `LargeInput` | 大きいバッチ | 大量データ |
| `PerIteration` | 毎イテレーション1個用意 | ドロップコストを計測外にしたい場合 |
| `NumBatches(n)` | n バッチ固定 | バッチ数を制御したい場合 |
| `NumIterations(n)` | n イテレーション固定 | イテレーション数を制御したい場合 |

---

## 設定のカスタマイズ

```rust
use criterion::{
    black_box, criterion_group, criterion_main,
    Criterion, SamplingMode,
};
use std::time::Duration;

fn configured_bench(c: &mut Criterion) {
    let mut criterion = Criterion::default()
        // ウォームアップ時間（デフォルト: 3秒）
        .warm_up_time(Duration::from_secs(5))
        // 計測時間（デフォルト: 5秒）
        .measurement_time(Duration::from_secs(10))
        // サンプル数（デフォルト: 100）
        .sample_size(200)
        // ノイズ許容閾値（デフォルト: 0.01 = 1%）
        .noise_threshold(0.05);

    // 低速な関数には FlatSampling を使う
    let mut group = criterion.benchmark_group("slow_functions");
    group.sampling_mode(SamplingMode::Flat);
    group.sample_size(10);  // サンプル数を減らす

    group.bench_function("expensive", |b| {
        b.iter(|| {
            // 重い処理
            black_box(expensive_computation())
        })
    });

    group.finish();
}

fn expensive_computation() -> u64 {
    // 仮の重い処理
    (0u64..100_000).sum()
}

criterion_group!(benches, configured_bench);
criterion_main!(benches);
```

---

## 統計的有意性の読み方

`criterion` は各ベンチマークの実行後に以下の統計情報を出力します。

```
sum_naive               time:   [1.2345 µs 1.2356 µs 1.2368 µs]
                        thrpt:  [6.4689 GiB/s 6.4750 GiB/s 6.4814 GiB/s]
Found 3 outliers among 100 measurements (3.00%)
  3 (3.00%) high mild
```

各数値の意味：

```
[下限  中央値  上限]
 ↑      ↑      ↑
 95% 信頼区間の下限
        推定の中心値（通常これを参照）
               95% 信頼区間の上限
```

### 変更検出レポート

前回のベンチマーク結果と比較したレポートも表示されます。

```
sum_naive               time:   [1.2345 µs 1.2356 µs 1.2368 µs]
                        change: [-3.0523% -2.5312% -1.9867%] (p = 0.00 < 0.05)
                        Performance has improved.
```

| 表示 | 意味 |
|------|------|
| `Performance has improved.` | 統計的に有意な改善 |
| `Performance has regressed.` | 統計的に有意な劣化 |
| `No change in performance detected.` | 変化なし（または誤差範囲内） |
| `p = 0.00 < 0.05` | p 値が 0.05 未満 → 統計的に有意 |

### 外れ値（Outliers）

```
Found 5 outliers among 100 measurements (5.00%)
  2 (2.00%) low mild       ← わずかに低速な外れ値
  1 (1.00%) low severe     ← 大幅に低速な外れ値（問題の可能性）
  1 (1.00%) high mild      ← わずかに高速な外れ値
  1 (1.00%) high severe    ← 大幅に高速な外れ値（キャッシュ等の影響）
```

外れ値が多い場合は、システムの負荷を下げるか、`--bench` で専用実行することを検討してください。

---

## HTML レポートの見方

```bash
# ベンチマーク実行後、HTML レポートが生成される
cargo bench

# レポートの場所
open target/criterion/report/index.html
```

HTML レポートに含まれるもの：

```
target/criterion/
├── report/
│   └── index.html          ← 全ベンチマークのサマリー
└── sum_comparison/
    ├── naive/1024/
    │   ├── report/
    │   │   ├── index.html  ← 個別ベンチマークのレポート
    │   │   ├── pdf.svg     ← 確率密度関数グラフ
    │   │   ├── regression.svg  ← 回帰分析グラフ
    │   │   └── ...
    │   └── ...
    └── report/
        └── index.html      ← グループ比較レポート
```

グループ比較レポートには**バイオリンプロット**が含まれており、各実装の分布を視覚的に比較できます。

```
Violin plot の見方:

  naive   ████████████████░░░░░░   ← 広い分布 = 不安定
  iter    ██████████░░░░           ← 中程度
  chunked █████░░                  ← 狭い分布 = 安定

  幅が広い → 計測にばらつきがある
  幅が狭い → 安定した計測
  位置が左 → 高速
  位置が右 → 低速
```

---

## 実践例: ソートアルゴリズムの比較

```rust
// benches/sorting.rs
use criterion::{black_box, criterion_group, criterion_main, BenchmarkId, Criterion};

fn insertion_sort(arr: &mut Vec<u32>) {
    let n = arr.len();
    for i in 1..n {
        let key = arr[i];
        let mut j = i;
        while j > 0 && arr[j - 1] > key {
            arr[j] = arr[j - 1];
            j -= 1;
        }
        arr[j] = key;
    }
}

fn bench_sorting(c: &mut Criterion) {
    let mut group = c.benchmark_group("sorting");

    for size in [10usize, 100, 1000] {
        // ランダムなデータ（最悪ケース寄り）
        let data: Vec<u32> = (0..size as u32).rev().collect(); // 逆順

        group.bench_with_input(
            BenchmarkId::new("std_sort", size),
            &data,
            |b, data| {
                b.iter_batched(
                    || data.clone(),
                    |mut v| { v.sort(); black_box(v) },
                    criterion::BatchSize::SmallInput,
                )
            },
        );

        group.bench_with_input(
            BenchmarkId::new("std_sort_unstable", size),
            &data,
            |b, data| {
                b.iter_batched(
                    || data.clone(),
                    |mut v| { v.sort_unstable(); black_box(v) },
                    criterion::BatchSize::SmallInput,
                )
            },
        );

        group.bench_with_input(
            BenchmarkId::new("insertion_sort", size),
            &data,
            |b, data| {
                b.iter_batched(
                    || data.clone(),
                    |mut v| { insertion_sort(&mut v); black_box(v) },
                    criterion::BatchSize::SmallInput,
                )
            },
        );
    }

    group.finish();
}

criterion_group!(benches, bench_sorting);
criterion_main!(benches);
```

実行結果の例：

```
sorting/std_sort/10         time:   [45.234 ns 45.301 ns 45.371 ns]
sorting/std_sort_unstable/10 time:  [38.102 ns 38.178 ns 38.261 ns]
sorting/insertion_sort/10   time:   [22.543 ns 22.601 ns 22.664 ns]

sorting/std_sort/1000       time:   [12.345 µs 12.367 µs 12.391 µs]
sorting/std_sort_unstable/1000 time: [9.876 µs 9.901 µs 9.928 µs]
sorting/insertion_sort/1000 time:   [1.2345 ms 1.2378 ms 1.2413 ms]
```

小さいサイズ（n=10）では `insertion_sort` が速いが、大きいサイズ（n=1000）では `std::sort_unstable` が圧倒的に速い、という典型的な結果が得られます。

---

## まとめ

| 項目 | ポイント |
|------|----------|
| 必ず `--release` ビルド | デバッグビルドは最適化なしで全く意味が違う |
| `black_box` を使う | コンパイラの不正な最適化を防ぐ |
| `BenchmarkGroup` で比較 | 複数実装を公平に比較できる |
| `iter_batched` を使う | セットアップコストを計測外にできる |
| p 値を確認する | p < 0.05 で統計的に有意 |
| HTML レポートを活用 | バイオリンプロットで分布を確認 |

次節では実際の CPU プロファイルを取得して、ホットスポットを特定する方法を学びます。

→ [02 - perf & flamegraph によるプロファイリング](./02-perf-flamegraph.md)
