# 01: キャッシュフレンドリーなデータ構造

## キャッシュラインの仕組み

### キャッシュラインとは

CPUがメモリからデータを読み取るとき、要求した1バイトだけでなく、周囲のデータをまとめて**キャッシュライン**という単位で転送します。現代の x86_64 CPU では、キャッシュラインのサイズは **64バイト** です。

```
メモリ上のアドレス (64バイト境界で区切られる)
┌────────────────────────────────────────────────────────────────┐
│ 0x00  0x01  0x02  ...  0x3E  0x3F │  <- キャッシュライン #0   │
│ 0x40  0x41  0x42  ...  0x7E  0x7F │  <- キャッシュライン #1   │
│ 0x80  0x81  0x82  ...  0xBE  0xBF │  <- キャッシュライン #2   │
└────────────────────────────────────────────────────────────────┘

アドレス 0x05 の 1 バイトを読むと...
  -> キャッシュライン #0 (0x00 〜 0x3F) が丸ごとキャッシュに入る
  -> 0x01〜0x3F のデータも無料でキャッシュされる！
```

### キャッシュラインサイズの確認

```rust
use std::arch::x86_64;

fn main() {
    // コンパイル時定数として定義するのが一般的
    const CACHE_LINE_SIZE: usize = 64;

    println!("キャッシュラインサイズ: {} バイト", CACHE_LINE_SIZE);
    println!("i32 は {} 個入る", CACHE_LINE_SIZE / std::mem::size_of::<i32>());
    println!("i64 は {} 個入る", CACHE_LINE_SIZE / std::mem::size_of::<i64>());
    println!("f64 は {} 個入る", CACHE_LINE_SIZE / std::mem::size_of::<f64>());
}
```

出力:
```
キャッシュラインサイズ: 64 バイト
i32 は 16 個入る
i64 は 8 個入る
f64 は 8 個入る
```

---

## キャッシュミスのコスト測定

### キャッシュミスの種類

| 種類 | 説明 | 対策 |
|------|------|------|
| コールドミス | 初回アクセス（強制的に発生） | プリフェッチ |
| キャパシティミス | データがキャッシュに収まらない | データサイズ削減 |
| コンフリクトミス | 同じキャッシュセットに集中 | データ配置の調整 |

### 測定実験: 配列 vs ランダムアクセス

```rust
use std::time::Instant;

const SIZE: usize = 64 * 1024 * 1024; // 64MB

fn sequential_access(data: &[u64]) -> u64 {
    let mut sum = 0u64;
    for &x in data {
        sum = sum.wrapping_add(x);
    }
    sum
}

fn random_access(data: &[u64], indices: &[usize]) -> u64 {
    let mut sum = 0u64;
    for &i in indices {
        sum = sum.wrapping_add(data[i]);
    }
    sum
}

fn main() {
    let data: Vec<u64> = (0..SIZE).map(|i| i as u64).collect();

    // ランダムインデックスを生成
    use std::collections::hash_map::DefaultHasher;
    use std::hash::{Hash, Hasher};
    let indices: Vec<usize> = (0..SIZE)
        .map(|i| {
            let mut h = DefaultHasher::new();
            i.hash(&mut h);
            (h.finish() as usize) % SIZE
        })
        .collect();

    // シーケンシャルアクセス
    let start = Instant::now();
    let sum1 = sequential_access(&data);
    let seq_time = start.elapsed();

    // ランダムアクセス
    let start = Instant::now();
    let sum2 = random_access(&data, &indices);
    let rand_time = start.elapsed();

    println!("シーケンシャルアクセス: {:?} (sum={})", seq_time, sum1);
    println!("ランダムアクセス:       {:?} (sum={})", rand_time, sum2);
    println!("速度比: {:.1}x", rand_time.as_nanos() as f64 / seq_time.as_nanos() as f64);
}
```

典型的な結果:
```
シーケンシャルアクセス: 45ms
ランダムアクセス:       1200ms
速度比: 26.7x
```

---

## 連結リスト vs 配列の比較ベンチマーク

### メモリレイアウトの違い

```
Vec<i32> (配列):
┌────┬────┬────┬────┬────┬────┬────┬────┐
│  1 │  2 │  3 │  4 │  5 │  6 │  7 │  8 │  <- 連続したメモリ
└────┴────┴────┴────┴────┴────┴────┴────┘
 ↑ 1回のキャッシュライン読み込みで複数要素を取得可能

LinkedList<i32>:
┌───────────┐     ┌───────────┐     ┌───────────┐
│ val: 1    │     │ val: 2    │     │ val: 3    │
│ next: ────┼────>│ next: ────┼────>│ next: nil │
└───────────┘     └───────────┘     └───────────┘
  0x7f3a1000         0x7f3b2340        0x7f3a9870
  ↑ 各ノードがヒープ上の別の場所に存在 -> キャッシュミス多発
```

### ベンチマーク実装

```rust
use std::collections::LinkedList;
use std::time::Instant;

const N: usize = 1_000_000;

fn bench_vec_sum() -> (u64, std::time::Duration) {
    let vec: Vec<u64> = (0..N as u64).collect();

    let start = Instant::now();
    let sum: u64 = vec.iter().sum();
    (sum, start.elapsed())
}

fn bench_linked_list_sum() -> (u64, std::time::Duration) {
    let mut list: LinkedList<u64> = LinkedList::new();
    for i in 0..N as u64 {
        list.push_back(i);
    }

    let start = Instant::now();
    let sum: u64 = list.iter().sum();
    (sum, start.elapsed())
}

fn bench_vec_insert_front() -> std::time::Duration {
    let mut vec: Vec<u64> = (0..N as u64).collect();
    let start = Instant::now();
    for i in 0..100u64 {
        vec.insert(0, i); // O(n) だが実際の性能は？
    }
    start.elapsed()
}

fn bench_linked_list_insert_front() -> std::time::Duration {
    let mut list: LinkedList<u64> = (0..N as u64).collect();
    let start = Instant::now();
    for i in 0..100u64 {
        list.push_front(i); // O(1)
    }
    start.elapsed()
}

fn main() {
    println!("=== 要素の合計計算 (N={}) ===", N);
    let (sum1, t1) = bench_vec_sum();
    println!("Vec:        {:?} (sum={})", t1, sum1);
    let (sum2, t2) = bench_linked_list_sum();
    println!("LinkedList: {:?} (sum={})", t2, sum2);
    println!("Vec は LinkedList の {:.1}x 高速", t2.as_nanos() as f64 / t1.as_nanos() as f64);

    println!("\n=== 先頭への挿入 (100回) ===");
    let t3 = bench_vec_insert_front();
    println!("Vec (insert(0, _)):   {:?}", t3);
    let t4 = bench_linked_list_insert_front();
    println!("LinkedList (push_front): {:?}", t4);
}
```

典型的な結果:
```
=== 要素の合計計算 (N=1000000) ===
Vec:        800µs  (sum=499999500000)
LinkedList: 12ms   (sum=499999500000)
Vec は LinkedList の 15.0x 高速

=== 先頭への挿入 (100回) ===
Vec (insert(0, _)):      850µs
LinkedList (push_front): 2µs
```

> **注意:** 先頭挿入は LinkedList の方が圧倒的に速い（O(1) vs O(n)）。しかし、**イテレーション**では Vec が大幅に速い。用途に応じた選択が重要。

---

## False Sharing とその回避

### False Sharing とは

複数のCPUコアが異なる変数を読み書きしているにもかかわらず、それらが**同じキャッシュライン**に存在する場合、不必要なキャッシュの無効化が発生します。これを **False Sharing（偽共有）** と呼びます。

```
キャッシュライン (64バイト):
┌──────────────────────────────────────────────────────────────┐
│ counter_a (8B) │ counter_b (8B) │ padding (48B)              │
└──────────────────────────────────────────────────────────────┘
      ↑                 ↑
   Core 0 が書く    Core 1 が書く

Core 0 が counter_a を更新すると、キャッシュライン全体を「所有」しようとする
-> Core 1 のキャッシュが無効化される
-> Core 1 は counter_b を読み直す必要がある
-> 互いに干渉し合う！（実際のデータ依存はないのに）
```

### False Sharing の実証

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use std::thread;
use std::time::Instant;

const ITERATIONS: u64 = 10_000_000;

// False Sharing が発生する構造体
struct SharedCounters {
    a: AtomicU64, // 8バイト
    b: AtomicU64, // 8バイト <- a と同じキャッシュラインに入る可能性
}

// False Sharing を回避した構造体
#[repr(C)]
struct PaddedCounters {
    a: AtomicU64,
    _pad_a: [u8; 56], // 64 - 8 = 56 バイトのパディング
    b: AtomicU64,
    _pad_b: [u8; 56],
}

fn bench_false_sharing() -> std::time::Duration {
    let counters = Arc::new(SharedCounters {
        a: AtomicU64::new(0),
        b: AtomicU64::new(0),
    });

    let c = Arc::clone(&counters);
    let t1 = thread::spawn(move || {
        for _ in 0..ITERATIONS {
            c.a.fetch_add(1, Ordering::Relaxed);
        }
    });

    let c = Arc::clone(&counters);
    let start = Instant::now();
    let t2 = thread::spawn(move || {
        for _ in 0..ITERATIONS {
            c.b.fetch_add(1, Ordering::Relaxed);
        }
    });

    let start = Instant::now();
    t1.join().unwrap();
    t2.join().unwrap();
    start.elapsed()
}

fn bench_no_false_sharing() -> std::time::Duration {
    let counters = Arc::new(PaddedCounters {
        a: AtomicU64::new(0),
        _pad_a: [0u8; 56],
        b: AtomicU64::new(0),
        _pad_b: [0u8; 56],
    });

    let c = Arc::clone(&counters);
    let t1 = thread::spawn(move || {
        for _ in 0..ITERATIONS {
            c.a.fetch_add(1, Ordering::Relaxed);
        }
    });

    let c = Arc::clone(&counters);
    let start = Instant::now();
    let t2 = thread::spawn(move || {
        for _ in 0..ITERATIONS {
            c.b.fetch_add(1, Ordering::Relaxed);
        }
    });

    let start = Instant::now();
    t1.join().unwrap();
    t2.join().unwrap();
    start.elapsed()
}

fn main() {
    println!("False Sharing あり: {:?}", bench_false_sharing());
    println!("False Sharing なし: {:?}", bench_no_false_sharing());
}
```

### `#[repr(align(64))]` を使った解決法

手動パディングよりも `#[repr(align(N))]` を使う方がエレガントです。

```rust
use std::sync::atomic::{AtomicU64, Ordering};

/// キャッシュライン境界にアラインされたカウンター
/// False Sharing を完全に防ぐ
#[repr(align(64))]
struct CacheLineAligned<T> {
    value: T,
}

impl<T> CacheLineAligned<T> {
    fn new(value: T) -> Self {
        Self { value }
    }
}

// コンパイル時アサート: サイズが 64 バイト以上であることを確認
const _: () = assert!(
    std::mem::align_of::<CacheLineAligned<AtomicU64>>() == 64
);

fn main() {
    let counter_a = CacheLineAligned::new(AtomicU64::new(0));
    let counter_b = CacheLineAligned::new(AtomicU64::new(0));

    println!("counter_a のアドレス: {:p}", &counter_a.value);
    println!("counter_b のアドレス: {:p}", &counter_b.value);
    println!("アライメント: {}", std::mem::align_of::<CacheLineAligned<AtomicU64>>());

    // これらは必ず異なるキャッシュラインに入る
    counter_a.value.fetch_add(1, Ordering::Relaxed);
    counter_b.value.fetch_add(1, Ordering::Relaxed);
}
```

### `crossbeam` の `CachePadded` を使う

実用的には `crossbeam` の `CachePadded<T>` を使うのが最も簡単です。

```toml
# Cargo.toml
[dependencies]
crossbeam = "0.8"
```

```rust
use crossbeam::utils::CachePadded;
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use std::thread;

fn main() {
    // CachePadded でラップするだけで False Sharing を防げる
    let a = Arc::new(CachePadded::new(AtomicU64::new(0)));
    let b = Arc::new(CachePadded::new(AtomicU64::new(0)));

    println!("CachePadded<AtomicU64> のサイズ: {} バイト",
        std::mem::size_of::<CachePadded<AtomicU64>>());
    // -> 64 バイト（キャッシュライン1本分）

    let a_clone = Arc::clone(&a);
    let b_clone = Arc::clone(&b);

    let t1 = thread::spawn(move || {
        for _ in 0..10_000_000 {
            a_clone.fetch_add(1, Ordering::Relaxed);
        }
    });

    let t2 = thread::spawn(move || {
        for _ in 0..10_000_000 {
            b_clone.fetch_add(1, Ordering::Relaxed);
        }
    });

    t1.join().unwrap();
    t2.join().unwrap();

    println!("a = {}", a.load(Ordering::Relaxed));
    println!("b = {}", b.load(Ordering::Relaxed));
}
```

---

## プリフェッチの活用

CPUに次のメモリアクセスを事前に教えることで、レイテンシを隠蔽できます。

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::_mm_prefetch;
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::_MM_HINT_T0;

fn process_with_prefetch(data: &[u64]) -> u64 {
    let mut sum = 0u64;
    let prefetch_distance = 16; // 16要素先をプリフェッチ

    for i in 0..data.len() {
        // 将来アクセスするアドレスをプリフェッチ
        #[cfg(target_arch = "x86_64")]
        if i + prefetch_distance < data.len() {
            unsafe {
                _mm_prefetch(
                    data.as_ptr().add(i + prefetch_distance) as *const i8,
                    _MM_HINT_T0, // L1キャッシュにプリフェッチ
                );
            }
        }
        sum = sum.wrapping_add(data[i]);
    }
    sum
}
```

---

## まとめ

| テクニック | 効果 | 適用場面 |
|-----------|------|---------|
| 連続メモリ使用 | キャッシュ効率最大化 | イテレーションが多い処理 |
| `#[repr(align(64))]` | False Sharing 防止 | マルチスレッドの共有データ |
| `CachePadded<T>` | False Sharing 防止（簡単） | crossbeam を使う場合 |
| データサイズの最小化 | キャッシュ収容数増加 | 大量のオブジェクト処理 |
| プリフェッチ | レイテンシ隠蔽 | 予測可能なアクセスパターン |

次章では、`Vec<T>` の変形版として `SmallVec` や `ArrayVec` を学びます。

[-> 02: Vec の変形と代替](./02-vec-variants.md)
