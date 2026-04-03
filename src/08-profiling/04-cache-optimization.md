# キャッシュ効率の最適化

現代の CPU は演算器が非常に高速である一方、メインメモリへのアクセスは相対的に非常に遅いです。
プログラムの性能がメモリアクセスで律速されている場合（メモリバウンド）、
アルゴリズムを改善するよりもキャッシュ効率を改善する方が大きな効果を生むことがあります。

---

## キャッシュ階層とアクセスレイテンシ

```
CPU と各メモリ階層のアクセスレイテンシ（参考値、実測値はハードウェアにより異なる）:

  ┌─────────────────────────────────────────────────────────────────┐
  │  CPU Core                                                       │
  │  ┌──────────┐                                                   │
  │  │ Register │  ~1 サイクル   (約 0.3 ns)  容量: 数十〜数百 byte │
  │  └────┬─────┘                                                   │
  │       │                                                         │
  │  ┌────▼─────┐                                                   │
  │  │ L1 Cache │  ~4 サイクル   (約 1.2 ns)  容量: 32〜64 KB       │
  │  └────┬─────┘  キャッシュライン: 64 bytes                       │
  │       │                                                         │
  │  ┌────▼─────┐                                                   │
  │  │ L2 Cache │  ~12 サイクル  (約 4 ns)    容量: 256 KB〜1 MB    │
  │  └────┬─────┘                                                   │
  │       │                                                         │
  │  ┌────▼─────┐                                                   │
  │  │ L3 Cache │  ~40 サイクル  (約 13 ns)   容量: 8〜32 MB        │
  │  └────┬─────┘  （コア間で共有）                                  │
  └───────┼─────────────────────────────────────────────────────────┘
          │
  ┌───────▼──────┐
  │  Main Memory │  ~100〜200 サイクル (約 70 ns)  容量: GB オーダー │
  └──────────────┘

L1 ヒット vs DRAM アクセス: 約 50〜100 倍の差！
```

### キャッシュラインの仕組み

```
メモリは 64 bytes 単位（キャッシュライン）でキャッシュに転送される。

int array[16]; // 64 bytes (16 × 4 bytes)

array[0] にアクセス → キャッシュラインに array[0..15] が全て載る
array[1] にアクセス → L1 ヒット！（すでにキャッシュに載っている）
array[16] にアクセス → 新しいキャッシュライン → L1 ミス

→ 連続アクセスは高速、ランダムアクセスは低速
```

---

## ストライドアクセスのコスト（行優先 vs 列優先）

2 次元配列のアクセスパターンが性能に大きく影響します。

```rust
const N: usize = 1024;

/// 行優先アクセス（Row-Major Order）— 連続メモリ、高速
fn row_major_sum(matrix: &[[f32; N]; N]) -> f32 {
    let mut sum = 0.0f32;
    for row in matrix {
        for &val in row {      // 連続メモリアクセス ← L1 ヒット続く
            sum += val;
        }
    }
    sum
}

/// 列優先アクセス（Column-Major Order）— ストライドアクセス、低速
fn col_major_sum(matrix: &[[f32; N]; N]) -> f32 {
    let mut sum = 0.0f32;
    for col in 0..N {
        for row in 0..N {
            sum += matrix[row][col]; // ストライド N×4 = 4096 bytes ← キャッシュミス頻発
        }
    }
    sum
}

fn main() {
    use std::time::Instant;

    // スタックに置くと overflow するのでヒープに確保
    let matrix: Box<[[f32; N]; N]> = {
        let mut m = vec![[0.0f32; N]; N];
        for (i, row) in m.iter_mut().enumerate() {
            for (j, val) in row.iter_mut().enumerate() {
                *val = (i + j) as f32;
            }
        }
        // Box<[[f32; N]; N]> への変換
        let boxed: Box<[[f32; N]; N]> = unsafe {
            let ptr = Box::into_raw(m.into_boxed_slice()) as *mut [[f32; N]; N];
            Box::from_raw(ptr)
        };
        boxed
    };

    let t1 = Instant::now();
    let s1 = row_major_sum(&matrix);
    println!("行優先: {:.3?} (sum={:.0})", t1.elapsed(), s1);

    let t2 = Instant::now();
    let s2 = col_major_sum(&matrix);
    println!("列優先: {:.3?} (sum={:.0})", t2.elapsed(), s2);
}
```

```
典型的な結果（N=1024, 参考値）:
  行優先:   ~0.8 ms   （L1/L2 ヒット率高）
  列優先:   ~6.5 ms   （L2/L3 ミス多発）
  → 約 8 倍の差
```

### ストライドアクセスのパターン図

```
N=8 の行列（行優先ストレージ）:

メモリ上の配置:
[0,0][0,1][0,2][0,3][0,4][0,5][0,6][0,7] [1,0][1,1]...
 ←── 行 0 (32 bytes) ──→               ←── 行 1...

行優先アクセス [row][0..N]:
  アクセス順: →→→→→→→→  →→→→→→→→  →→→
  キャッシュライン利用率: 100%

列優先アクセス [0..N][col]:
  アクセス順: ↓ (N×4 byte ジャンプ)
  col=0: [0,0], [1,0], [2,0]... ← 各アクセスで別キャッシュライン
  キャッシュライン利用率: 12.5% (8要素中1要素しか使わない)
```

---

## `PREFETCHNTA` による手動プリフェッチ

CPU はストリームアクセスを自動的にプリフェッチしますが、複雑なアクセスパターンでは手動プリフェッチが有効です。

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::_mm_prefetch;
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::_MM_HINT_NTA;

/// PREFETCHNTA: Non-Temporal hint
/// キャッシュ階層を汚染しない（一度だけ使うデータに最適）
fn process_with_prefetch(data: &[f32]) -> f32 {
    let mut sum = 0.0f32;
    let prefetch_distance = 16; // 16 要素先（64 bytes = 1 キャッシュライン）をプリフェッチ

    for i in 0..data.len() {
        // 未来のキャッシュラインをプリフェッチ
        let prefetch_idx = i + prefetch_distance;
        if prefetch_idx < data.len() {
            #[cfg(target_arch = "x86_64")]
            unsafe {
                _mm_prefetch(
                    data.as_ptr().add(prefetch_idx) as *const i8,
                    _MM_HINT_NTA,
                );
            }
        }
        sum += data[i];
    }
    sum
}

/// 比較: プリフェッチなし
fn process_without_prefetch(data: &[f32]) -> f32 {
    data.iter().sum()
}

fn main() {
    let data: Vec<f32> = (0..10_000_000).map(|i| i as f32).collect();
    use std::time::Instant;

    let t1 = Instant::now();
    let s1 = process_without_prefetch(&data);
    println!("プリフェッチなし: {:?} (sum={:.0})", t1.elapsed(), s1);

    let t2 = Instant::now();
    let s2 = process_with_prefetch(&data);
    println!("プリフェッチあり: {:?} (sum={:.0})", t2.elapsed(), s2);
}
```

### プリフェッチヒントの種類

```
_MM_HINT_T0:  全キャッシュレベルにプリフェッチ（再利用が多いデータ）
_MM_HINT_T1:  L2 以上にプリフェッチ
_MM_HINT_T2:  L3 以上にプリフェッチ
_MM_HINT_NTA: Non-Temporal（一度だけ使うデータ、キャッシュを汚染しない）
```

---

## `cachegrind` / `valgrind` によるキャッシュミス計測

```bash
# コンパイル（デバッグ情報付き、最適化あり）
cargo build --release

# cachegrind でキャッシュミスを計測
valgrind --tool=cachegrind --cache-sim=yes \
    ./target/release/my_program

# 結果の確認
cg_annotate cachegrind.out.<pid>
```

```
cachegrind の出力例:

I refs:        512,345,678    (命令フェッチ数)
I1  misses:          1,234    (L1 命令キャッシュミス)
LLi misses:            567    (LL 命令キャッシュミス)

D refs:        234,567,890    (データ参照数)
  D1  misses:    5,678,901   ← L1 データキャッシュミス（多い場合は要注意）
  LLd misses:    1,234,567   ← LL データキャッシュミス（DRAM アクセス）

D1  miss rate:       2.4%
LLd miss rate:       0.5%
```

### `callgrind` で関数ごとのキャッシュミスを特定

```bash
valgrind --tool=callgrind --cache-sim=yes \
    ./target/release/my_program

# GUI での可視化
kcachegrind callgrind.out.<pid>
```

---

## Linux の `perf stat` で L1/L2/L3 ミスを計測

```bash
# 基本的なキャッシュイベントの計測
perf stat -e \
    cache-references,cache-misses,\
    L1-dcache-loads,L1-dcache-load-misses,\
    L1-dcache-stores,\
    LLC-loads,LLC-load-misses \
    ./target/release/my_program
```

```
出力例:

 Performance counter stats for './target/release/my_program':

     1,234,567,890      cache-references
        12,345,678      cache-misses              #    1.00 % of all cache refs
     2,345,678,901      L1-dcache-loads
        56,789,012      L1-dcache-load-misses     #    2.42 % of all L1-dcache accesses
       890,123,456      L1-dcache-stores
        23,456,789      LLC-loads
         1,234,567      LLC-load-misses           #    5.26 % of all LLC cache hits

       2.345678901 seconds time elapsed

目安:
  L1 miss rate:  < 5%  → 良好
  LLC miss rate: < 1%  → 良好（これが高いと DRAM アクセスが多い）
```

```bash
# より詳細な計測（サンプリングでホットスポット特定）
perf record -e LLC-load-misses ./target/release/my_program
perf report
```

```bash
# リアルタイム表示
perf top -e LLC-load-misses
```

---

## メモリプールパターンによるキャッシュ局所性の改善

小さなオブジェクトを `Box` で個別にヒープ確保すると、メモリが断片化してキャッシュ効率が悪化します。

```rust
use std::time::Instant;

// 悪い例: 各ノードが別々のヒープ領域に確保される
struct NodeBoxed {
    value: f64,
    next: Option<Box<NodeBoxed>>,
}

fn build_linked_list_boxed(n: usize) -> Box<NodeBoxed> {
    let mut head = Box::new(NodeBoxed { value: 0.0, next: None });
    for i in 1..n {
        head = Box::new(NodeBoxed { value: i as f64, next: Some(head) });
    }
    head
}

fn sum_boxed(head: &NodeBoxed) -> f64 {
    let mut sum = 0.0;
    let mut curr = head;
    loop {
        sum += curr.value;
        match &curr.next {
            Some(next) => curr = next,
            None => break,
        }
    }
    sum
}

// 良い例: 全ノードを Vec（連続メモリ）に格納
struct NodePooled {
    value: f64,
    next: Option<usize>, // インデックスで参照
}

struct LinkedListPool {
    nodes: Vec<NodePooled>,
    head: Option<usize>,
}

impl LinkedListPool {
    fn build(n: usize) -> Self {
        let mut nodes = Vec::with_capacity(n);
        for i in 0..n {
            nodes.push(NodePooled {
                value: i as f64,
                next: if i + 1 < n { Some(i + 1) } else { None },
            });
        }
        Self { nodes, head: Some(0) }
    }

    fn sum(&self) -> f64 {
        let mut sum = 0.0;
        let mut curr = self.head;
        while let Some(idx) = curr {
            sum += self.nodes[idx].value;
            curr = self.nodes[idx].next;
        }
        sum
    }
}

fn main() {
    let n = 100_000;

    let list = build_linked_list_boxed(n);
    let t1 = Instant::now();
    for _ in 0..100 { std::hint::black_box(sum_boxed(&list)); }
    println!("Box linked list:  {:?}", t1.elapsed());

    let pool = LinkedListPool::build(n);
    let t2 = Instant::now();
    for _ in 0..100 { std::hint::black_box(pool.sum()); }
    println!("Pooled list (Vec): {:?}", t2.elapsed());
}
```

```
典型的な結果（100,000 要素 × 100 回, 参考値）:
  Box linked list:    ~320 ms  （ポインタ追跡 → ランダムメモリアクセス）
  Pooled list (Vec):  ~15 ms   （連続メモリ → L1/L2 ヒット率高）
  → 約 21 倍の差
```

---

## `#[repr(align(64))]` によるキャッシュライン境界合わせ

### False Sharing の問題と解決

複数のスレッドが同じキャッシュラインに存在する異なる変数を書き込む際、
一方の書き込みで他方のキャッシュが無効化される「False Sharing」が発生します。

```
False Sharing の例:

キャッシュライン (64 bytes):
[counter_a: u64][counter_b: u64][..............padding..............]
 Thread 1 が書く  Thread 2 が書く

Thread 1 が counter_a を更新
  → このキャッシュラインが「変更あり」とマーク
  → Thread 2 のキャッシュも無効化 ← False Sharing!
  → Thread 2 はキャッシュから再読み込み → スロー
```

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::thread;

// 悪い例: False Sharing が発生
struct CountersBad {
    a: AtomicU64, // 両方が 1 つのキャッシュラインに収まる
    b: AtomicU64,
}

// 良い例: キャッシュラインをパディングで分離
#[repr(align(64))] // キャッシュライン境界（64 bytes）にアライン
struct CacheAligned<T>(T);

struct CountersGood {
    a: CacheAligned<AtomicU64>, // 別々のキャッシュラインに配置
    b: CacheAligned<AtomicU64>,
}

fn bench_false_sharing() {
    let counters = std::sync::Arc::new(CountersBad {
        a: AtomicU64::new(0),
        b: AtomicU64::new(0),
    });

    let t = std::time::Instant::now();
    let c1 = std::sync::Arc::clone(&counters);
    let h1 = thread::spawn(move || {
        for _ in 0..10_000_000 {
            c1.a.fetch_add(1, Ordering::Relaxed);
        }
    });
    let c2 = std::sync::Arc::clone(&counters);
    let h2 = thread::spawn(move || {
        for _ in 0..10_000_000 {
            c2.b.fetch_add(1, Ordering::Relaxed);
        }
    });
    h1.join().unwrap(); h2.join().unwrap();
    println!("False Sharing あり: {:?}", t.elapsed());
}

fn bench_no_false_sharing() {
    let counters = std::sync::Arc::new(CountersGood {
        a: CacheAligned(AtomicU64::new(0)),
        b: CacheAligned(AtomicU64::new(0)),
    });

    let t = std::time::Instant::now();
    let c1 = std::sync::Arc::clone(&counters);
    let h1 = thread::spawn(move || {
        for _ in 0..10_000_000 {
            c1.a.0.fetch_add(1, Ordering::Relaxed);
        }
    });
    let c2 = std::sync::Arc::clone(&counters);
    let h2 = thread::spawn(move || {
        for _ in 0..10_000_000 {
            c2.b.0.fetch_add(1, Ordering::Relaxed);
        }
    });
    h1.join().unwrap(); h2.join().unwrap();
    println!("False Sharing なし: {:?}", t.elapsed());
}

fn main() {
    bench_false_sharing();
    bench_no_false_sharing();
}
```

```
典型的な結果（10,000,000 回の atomic add × 2 スレッド, 参考値）:
  False Sharing あり:  ~180 ms （コア間でキャッシュラインが ping-pong）
  False Sharing なし:  ~45 ms  （各スレッドが独立したキャッシュラインを使用）
  → 約 4 倍の差
```

### `repr(align)` のその他の用途

```rust
/// SIMD 命令用に 32 バイトアライメントを要求
#[repr(align(32))]
struct SimdBuffer([f32; 8]);

/// DMA 転送用に 4096 バイト（ページ）境界アライン
#[repr(align(4096))]
struct DmaBuffer([u8; 4096]);

/// キャッシュライン境界アラインの汎用ラッパー
#[repr(align(64))]
pub struct CacheLinePadded<T> {
    pub inner: T,
    // Rust がパディングを自動追加して 64 バイト境界に収める
}

impl<T> CacheLinePadded<T> {
    pub fn new(inner: T) -> Self {
        Self { inner }
    }
}
```

---

## 最適化チェックリスト

```
キャッシュ効率最適化のチェックリスト:

データレイアウト:
  □ 2次元配列は行優先でアクセスしているか？
  □ 頻繁にアクセスするフィールドは構造体の先頭にあるか？
  □ ホット/コールドデータは分離されているか？
  □ SoA レイアウトを検討したか？
  □ False Sharing のあるカウンタは #[repr(align(64))] で分離したか？

アクセスパターン:
  □ ランダムアクセスを連続アクセスに変換できるか？
  □ 木構造の代わりに配列+インデックスで表現できるか？
  □ ポインタ追跡チェーンを短くできるか？

計測:
  □ perf stat でキャッシュミス率を確認したか？
  □ valgrind --tool=cachegrind で関数別ミスを確認したか？
  □ 最適化前後でベンチマークを取ったか？
```

---

## まとめ

| テクニック | 効果 | 適用場面 |
|---|---|---|
| 行優先アクセス | L1/L2 ミス削減 | 2D 配列の走査 |
| SoA レイアウト | キャッシュ密度向上 | 同種データの一括処理 |
| メモリプール | 断片化の解消 | 多数の小オブジェクト |
| ホット/コールド分離 | ホットデータの密度向上 | アクセス頻度が偏る場合 |
| キャッシュライン境界合わせ | False Sharing 防止 | マルチスレッドカウンタ |
| 手動プリフェッチ | レイテンシ隠蔽 | 予測困難なアクセスパターン |
