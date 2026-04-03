# 02. Rayon — データ並列

`rayon` はRustのデータ並列ライブラリです。既存のイテレータコードを `.par_iter()` に変更するだけでワークスティーリングによるマルチスレッド並列化が実現します。

## なぜ Rayon か

```
通常のイテレータ:
data.iter().map(f).collect()
     ↓ (シリアル)
[f(1), f(2), f(3), f(4), f(5), f(6), f(7), f(8)]

rayon の並列イテレータ:
data.par_iter().map(f).collect()
     ↓ (ワークスティーリング)
スレッド1: [f(1), f(2)]
スレッド2: [f(3), f(4)]
スレッド3: [f(5), f(6)]
スレッド4: [f(7), f(8)]
     ↓
結果を結合
```

**ワークスティーリング**: 暇なスレッドが他のスレッドのキューからタスクを「盗む」ことで負荷を均等化するアルゴリズムです。

## セットアップ

```toml
[dependencies]
rayon = "1.10"
```

```rust
use rayon::prelude::*;
```

## `par_iter` と `par_iter_mut`

### 基本的な並列マップ

```rust
use rayon::prelude::*;

fn main() {
    let data: Vec<f64> = (0..1_000_000).map(|x| x as f64).collect();

    // シングルスレッド
    let result_single: Vec<f64> = data.iter().map(|x| x.sqrt()).collect();

    // マルチスレッド (par_iter に変えるだけ)
    let result_parallel: Vec<f64> = data.par_iter().map(|x| x.sqrt()).collect();

    assert_eq!(result_single, result_parallel);
    println!("要素数: {}", result_parallel.len());
}
```

### `par_iter_mut` による可変変換

```rust
use rayon::prelude::*;

fn normalize_in_place(data: &mut Vec<f64>) {
    let max = data.par_iter().cloned().reduce_with(f64::max).unwrap_or(1.0);
    // 各要素を最大値で割って正規化
    data.par_iter_mut().for_each(|x| *x /= max);
}

fn main() {
    let mut data: Vec<f64> = (1..=1_000_000).map(|x| x as f64).collect();
    normalize_in_place(&mut data);
    println!("先頭5要素: {:?}", &data[..5]);
    println!("最大値: {}", data.par_iter().cloned().reduce_with(f64::max).unwrap());
    // → 最大値: 1.0
}
```

### `into_par_iter` による所有権の移動

```rust
use rayon::prelude::*;

fn main() {
    let strings = vec!["hello", "world", "foo", "bar"];

    // 所有権ごと並列処理
    let upper: Vec<String> = strings
        .into_par_iter()
        .map(|s| s.to_uppercase())
        .collect();

    println!("{:?}", upper); // ["HELLO", "WORLD", "FOO", "BAR"]
}
```

## 並列フィルタとリデュース

```rust
use rayon::prelude::*;

fn main() {
    let data: Vec<i64> = (0..10_000_000).collect();

    // 素数判定を並列で実行
    let primes: Vec<i64> = data
        .par_iter()
        .filter(|&&n| n >= 2 && is_prime(n))
        .copied()
        .collect();

    println!("1千万以下の素数の個数: {}", primes.len());

    // 並列リデュース
    let sum: i64 = data.par_iter().sum();
    let product: f64 = (1..=20_i64)
        .into_par_iter()
        .map(|x| x as f64)
        .product();

    println!("合計: {}, 20の階乗: {}", sum, product);
}

fn is_prime(n: i64) -> bool {
    if n < 2 { return false; }
    if n == 2 { return true; }
    if n % 2 == 0 { return false; }
    let mut i = 3;
    while i * i <= n {
        if n % i == 0 { return false; }
        i += 2;
    }
    true
}
```

## `rayon::join` — タスク並列

`join` は2つのクロージャを並列に実行します。再帰的分割統治アルゴリズムに適しています。

```rust
use rayon;

fn parallel_sum(data: &[i64]) -> i64 {
    if data.len() <= 1024 {
        // 小さければシリアルに処理（並列化のオーバーヘッドを避ける）
        return data.iter().sum();
    }

    let mid = data.len() / 2;
    let (left, right) = data.split_at(mid);

    let (left_sum, right_sum) = rayon::join(
        || parallel_sum(left),
        || parallel_sum(right),
    );

    left_sum + right_sum
}

fn main() {
    let data: Vec<i64> = (0..10_000_000).collect();
    let result = parallel_sum(&data);
    println!("並列合計: {}", result);
    println!("検証:     {}", data.iter().sum::<i64>());
}
```

### マージソートの並列化

```rust
use rayon;

fn parallel_merge_sort(data: &mut Vec<i32>) {
    let len = data.len();
    if len <= 1 {
        return;
    }

    let mid = len / 2;
    let mut left = data[..mid].to_vec();
    let mut right = data[mid..].to_vec();

    // 2つのサブ問題を並列に解く
    rayon::join(
        || parallel_merge_sort(&mut left),
        || parallel_merge_sort(&mut right),
    );

    // マージ
    let mut i = 0;
    let mut j = 0;
    let mut k = 0;
    while i < left.len() && j < right.len() {
        if left[i] <= right[j] {
            data[k] = left[i];
            i += 1;
        } else {
            data[k] = right[j];
            j += 1;
        }
        k += 1;
    }
    data[k..k + (left.len() - i)].copy_from_slice(&left[i..]);
    data[k + (left.len() - i)..].copy_from_slice(&right[j..]);
}

fn main() {
    let mut data: Vec<i32> = (0..100_000).rev().collect();
    parallel_merge_sort(&mut data);
    assert!(data.windows(2).all(|w| w[0] <= w[1]));
    println!("ソート完了: 先頭5要素 = {:?}", &data[..5]);
}
```

## `rayon::scope` — 複数タスクの並列化

`scope` は複数の並列タスクを生成できます。スコープ内のすべてのタスクが完了してから次に進みます。

```rust
use rayon;
use std::sync::Mutex;

fn main() {
    let mut results = vec![0i64; 4];

    rayon::scope(|s| {
        // スコープ内で複数タスクを spawn
        for (i, result) in results.iter_mut().enumerate() {
            s.spawn(move |_| {
                *result = heavy_computation(i as i64);
                println!("タスク {} 完了: {}", i, *result);
            });
        }
        // スコープを抜けると全タスクの完了を待つ
    });

    println!("全結果: {:?}", results);
}

fn heavy_computation(n: i64) -> i64 {
    // 重い計算のシミュレーション
    (0..n * 1_000_000).fold(0, |acc, x| acc ^ x)
}
```

### `scope` を使った並列パイプライン

```rust
use rayon;
use std::sync::mpsc;

fn parallel_pipeline(input: Vec<i32>) -> Vec<i32> {
    let (tx1, rx1) = mpsc::channel();
    let (tx2, rx2) = mpsc::channel();

    rayon::scope(|s| {
        // ステージ1: フィルタ
        s.spawn(|_| {
            input.into_par_iter()
                .filter(|&x| x % 2 == 0)
                .for_each(|x| tx1.send(x).unwrap());
            drop(tx1);
        });

        // ステージ2: 変換
        s.spawn(|_| {
            for x in rx1 {
                tx2.send(x * x).unwrap();
            }
            drop(tx2);
        });
    });

    rx2.iter().collect()
}

use rayon::prelude::*;

fn main() {
    let input: Vec<i32> = (0..20).collect();
    let output = parallel_pipeline(input);
    println!("偶数の二乗: {:?}", output);
    // → [0, 4, 16, 36, 64, 100, 144, 196, 256, 324]
}
```

## カスタムスレッドプール

デフォルトではグローバルスレッドプールを使いますが、`ThreadPoolBuilder` で独立したプールを作れます。

```rust
use rayon::{ThreadPool, ThreadPoolBuilder};
use rayon::prelude::*;

fn main() {
    // CPUコア数の半分だけ使うプール
    let pool = ThreadPoolBuilder::new()
        .num_threads(4)
        .thread_name(|i| format!("my-worker-{}", i))
        .build()
        .unwrap();

    // このプールのコンテキストで実行
    let result: Vec<i32> = pool.install(|| {
        (0..1000).into_par_iter().map(|x| x * x).collect()
    });

    println!("要素数: {}", result.len());

    // 複数の独立したプールを使い分けることも可能
    let io_pool = ThreadPoolBuilder::new()
        .num_threads(16) // I/Oバウンドなら多めに
        .build()
        .unwrap();

    let cpu_pool = ThreadPoolBuilder::new()
        .num_threads(num_cpus()) // CPUバウンドはコア数に合わせる
        .build()
        .unwrap();

    println!("I/Oプール設定完了");
}

fn num_cpus() -> usize {
    std::thread::available_parallelism()
        .map(|n| n.get())
        .unwrap_or(4)
}
```

## 並列ソートとマップリデュース

### 並列ソート

```rust
use rayon::prelude::*;
use std::time::Instant;

fn main() {
    let n = 10_000_000;
    let mut data: Vec<i32> = (0..n).rev().collect();

    // シングルスレッドソート
    let mut data_single = data.clone();
    let t = Instant::now();
    data_single.sort();
    println!("シングルスレッドソート: {:.2}ms", t.elapsed().as_secs_f64() * 1000.0);

    // 並列ソート (rayon)
    let t = Instant::now();
    data.par_sort();
    println!("並列ソート:             {:.2}ms", t.elapsed().as_secs_f64() * 1000.0);

    // 不安定な並列ソート (より高速)
    let mut data2: Vec<i32> = (0..n).rev().collect();
    let t = Instant::now();
    data2.par_sort_unstable();
    println!("並列不安定ソート:       {:.2}ms", t.elapsed().as_secs_f64() * 1000.0);

    // カスタム比較関数
    let mut words = vec!["banana", "apple", "cherry", "date"];
    words.par_sort_by_key(|s| s.len());
    println!("長さでソート: {:?}", words);
}
```

**実行結果例** (8コア):

```
シングルスレッドソート: 1842.33ms
並列ソート:              312.18ms
並列不安定ソート:         198.44ms
```

### マップリデュース

```rust
use rayon::prelude::*;
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct LogEntry {
    path: String,
    status: u16,
    duration_ms: u64,
}

fn analyze_logs(logs: &[LogEntry]) -> HashMap<String, (usize, f64)> {
    // 並列にエンドポイントごとの集計を行う
    logs.par_iter()
        // Map: 各ログを (パス, (件数, 合計時間)) のペアに変換
        .map(|entry| {
            let mut map = HashMap::new();
            map.insert(entry.path.clone(), (1usize, entry.duration_ms as f64));
            map
        })
        // Reduce: 部分的な結果を結合
        .reduce(HashMap::new, |mut acc, map| {
            for (key, (count, duration)) in map {
                let entry = acc.entry(key).or_insert((0, 0.0));
                entry.0 += count;
                entry.1 += duration;
            }
            acc
        })
}

fn main() {
    let logs: Vec<LogEntry> = vec![
        LogEntry { path: "/api/users".to_string(), status: 200, duration_ms: 45 },
        LogEntry { path: "/api/posts".to_string(), status: 200, duration_ms: 120 },
        LogEntry { path: "/api/users".to_string(), status: 404, duration_ms: 5 },
        LogEntry { path: "/api/posts".to_string(), status: 200, duration_ms: 80 },
        LogEntry { path: "/api/users".to_string(), status: 200, duration_ms: 30 },
    ];

    let stats = analyze_logs(&logs);

    for (path, (count, total_ms)) in &stats {
        println!("{}: {}件, 平均 {:.1}ms", path, count, total_ms / *count as f64);
    }
    // /api/users: 3件, 平均 26.7ms
    // /api/posts: 2件, 平均 100.0ms
}
```

### `fold` と `reduce` の使い分け

```rust
use rayon::prelude::*;

fn main() {
    let data: Vec<i64> = (1..=1_000_000).collect();

    // reduce: 初期値なし、型を保持
    let sum = data.par_iter()
        .copied()
        .reduce(|| 0, |a, b| a + b);
    println!("sum: {}", sum);

    // fold: スレッドごとに中間値を作り、最後に結合
    // 大きなコレクションのカウントに適している
    let (evens, odds) = data.par_iter().fold(
        || (0usize, 0usize),
        |(e, o), &x| if x % 2 == 0 { (e + 1, o) } else { (e, o + 1) },
    ).reduce(
        || (0, 0),
        |(e1, o1), (e2, o2)| (e1 + e2, o1 + o2),
    );
    println!("偶数: {}, 奇数: {}", evens, odds);
}
```

## パフォーマンス比較

```rust
use rayon::prelude::*;
use std::time::Instant;

fn benchmark() {
    let n = 50_000_000usize;
    let data: Vec<f64> = (0..n).map(|x| x as f64).collect();

    // ベンチマーク関数
    let bench = |label: &str, f: &dyn Fn() -> f64| {
        let start = Instant::now();
        let result = f();
        let elapsed = start.elapsed();
        println!("{}: {:.6} ({:.2}ms)", label, result, elapsed.as_secs_f64() * 1000.0);
    };

    // シングルスレッド
    bench("シングル (sqrt sum)", &|| {
        data.iter().map(|x| x.sqrt()).sum()
    });

    // rayon par_iter
    bench("rayon   (sqrt sum)", &|| {
        data.par_iter().map(|x| x.sqrt()).sum()
    });
}

fn main() {
    benchmark();
}
```

**実行結果例** (M2 Pro, 10コア):

```
シングル (sqrt sum): 7071067.811865 (487.23ms)
rayon   (sqrt sum): 7071067.811865  (58.91ms)
スピードアップ: 約 8.3倍
```

## rayon を使う際の注意点

### パニック安全性

```rust
use rayon::prelude::*;

fn main() {
    let data = vec![1, 2, 0, 4, 5];

    // 並列処理中にパニックが起きると他のスレッドも停止し、
    // collect() / reduce() 等でパニックが再スローされる
    let result = std::panic::catch_unwind(|| {
        data.par_iter().for_each(|&x| {
            if x == 0 {
                panic!("ゼロ除算!");
            }
        });
    });

    println!("パニックをキャッチ: {}", result.is_err()); // true
}
```

### 副作用に注意

```rust
use rayon::prelude::*;
use std::sync::atomic::{AtomicUsize, Ordering};

fn main() {
    let counter = AtomicUsize::new(0);

    // rayon ではイテレータの順序は保証されない
    // 副作用がある場合はアトミック操作か Mutex を使う
    (0..1000).into_par_iter().for_each(|_| {
        counter.fetch_add(1, Ordering::Relaxed);
    });

    println!("カウント: {}", counter.load(Ordering::SeqCst)); // 1000
}
```

## まとめ

- `.par_iter()` で既存コードを最小限の変更で並列化できる
- `rayon::join` で再帰的分割統治アルゴリズムを並列化できる
- `rayon::scope` で動的な複数タスクを並列実行できる
- `ThreadPoolBuilder` でスレッド数やスレッド名をカスタマイズできる
- `par_sort_unstable` はデフォルトのシングルスレッドソートより大幅に高速
- 副作用を持つ処理にはアトミック操作や `Mutex` を使う
