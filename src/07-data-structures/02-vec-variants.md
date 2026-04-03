# 02: Vec の変形と代替

## `Vec<T>` の内部構造

### ptr / len / cap の三要素

`Vec<T>` は内部的に3つのフィールドで管理されています。

```
Vec<T> のメモリレイアウト:

スタック:                    ヒープ:
┌──────────────┐            ┌────┬────┬────┬────┬────┬────┐
│ ptr: *mut T  ├───────────>│ 0  │ 1  │ 2  │ 3  │(未│(未 │
│ len: usize   │            └────┴────┴────┴────┴────┴────┘
│ cap: usize   │             使用済み領域   │ 確保済み未使用│
└──────────────┘             <-- len=4 --> <-- cap=6 -->
   24バイト (64bit)
```

```rust
use std::mem;

fn inspect_vec<T>(v: &Vec<T>) {
    println!("len:  {}", v.len());
    println!("cap:  {}", v.capacity());
    println!("ptr:  {:p}", v.as_ptr());
    println!("size of Vec<T> on stack: {} bytes", mem::size_of::<Vec<T>>());
}

fn main() {
    let mut v: Vec<i32> = Vec::new();
    println!("--- 空のVec ---");
    inspect_vec(&v);
    // ptr は dangling pointer (実装依存) だが len=cap=0 なので安全

    v.push(1);
    println!("\n--- push(1) 後 ---");
    inspect_vec(&v);

    // with_capacity で再確保を避ける
    let mut v2: Vec<i32> = Vec::with_capacity(1000);
    println!("\n--- with_capacity(1000) ---");
    inspect_vec(&v2);
    for i in 0..1000 {
        v2.push(i);
    }
    println!("--- 1000要素push後 (再確保なし) ---");
    inspect_vec(&v2);
}
```

### 成長戦略と再確保コスト

`Vec<T>` は容量が不足すると**倍々で拡張**します（実装依存、おおよそ2倍）。

```rust
fn show_growth_pattern() {
    let mut v: Vec<i32> = Vec::new();
    let mut last_cap = 0;

    println!("{:>6} {:>6} {:>10}", "len", "cap", "realloc?");
    println!("{}", "-".repeat(26));

    for i in 0..65 {
        v.push(i);
        let realloc = v.capacity() != last_cap;
        if realloc {
            println!("{:>6} {:>6} {:>10}", v.len(), v.capacity(), "YES ← 再確保");
            last_cap = v.capacity();
        }
    }
}

fn main() {
    show_growth_pattern();
}
```

出力:
```
   len    cap   realloc?
--------------------------
     1      4        YES ← 再確保
     5      8        YES ← 再確保
     9     16        YES ← 再確保
    17     32        YES ← 再確保
    33     64        YES ← 再確保
    65    128        YES ← 再確保
```

> 要素数が予測できる場合は `Vec::with_capacity(n)` で事前確保すること。

---

## `SmallVec`: スタック上に小さな要素を保持

### 概要

[`smallvec`](https://docs.rs/smallvec) クレートの `SmallVec<[T; N]>` は、要素数が `N` 以内の場合はヒープを確保せず、スタックに直接データを保持します。

```
SmallVec<[i32; 4]> の動作:

要素数 <= 4 の場合 (スタックのみ):
┌────────────────────────────────────┐
│ Inline([1, 2, 3, 4], len=4)        │  <- ヒープ確保なし！
└────────────────────────────────────┘

要素数 > 4 の場合 (ヒープへ移行):
┌────────────────────────────────────┐     ┌──────────────────┐
│ Heap(ptr, len=6, cap=8)            ├────>│ 1  2  3  4  5  6 │
└────────────────────────────────────┘     └──────────────────┘
  通常の Vec と同じ動作
```

### `Cargo.toml` の設定

```toml
[dependencies]
smallvec = { version = "1.11", features = ["union"] }
```

### 基本的な使い方

```rust
use smallvec::{SmallVec, smallvec};

fn main() {
    // スタックに最大4要素を保持
    let mut sv: SmallVec<[i32; 4]> = SmallVec::new();

    sv.push(1);
    sv.push(2);
    sv.push(3);
    sv.push(4);
    println!("spilled: {}", sv.spilled()); // false - スタック上

    sv.push(5); // ここでヒープへ移行
    println!("spilled: {}", sv.spilled()); // true - ヒープへ

    // マクロでの初期化
    let sv2: SmallVec<[&str; 3]> = smallvec!["hello", "world", "!"];
    println!("{:?}", sv2);

    // Vec と同様に使える
    for x in &sv {
        print!("{} ", x);
    }
    println!();
}
```

### ベンチマーク: 少量要素での Vec vs SmallVec

```rust
use smallvec::{SmallVec, smallvec};
use std::time::Instant;

const ITERATIONS: usize = 10_000_000;

fn bench_vec_small() -> std::time::Duration {
    let start = Instant::now();
    for _ in 0..ITERATIONS {
        let mut v: Vec<i32> = Vec::new();
        v.push(1);
        v.push(2);
        v.push(3);
        // 毎回ヒープ確保・解放が発生
        std::hint::black_box(v.iter().sum::<i32>());
    }
    start.elapsed()
}

fn bench_smallvec_small() -> std::time::Duration {
    let start = Instant::now();
    for _ in 0..ITERATIONS {
        let mut sv: SmallVec<[i32; 4]> = SmallVec::new();
        sv.push(1);
        sv.push(2);
        sv.push(3);
        // ヒープ確保なし！
        std::hint::black_box(sv.iter().sum::<i32>());
    }
    start.elapsed()
}

fn main() {
    println!("Vec (少量要素):     {:?}", bench_vec_small());
    println!("SmallVec (少量要素): {:?}", bench_smallvec_small());
}
```

典型的な結果:
```
Vec (少量要素):      850ms
SmallVec (少量要素): 120ms
速度比: 7.1x 高速
```

---

## `ArrayVec`: 固定容量・ヒープ確保ゼロ

### 概要

[`arrayvec`](https://docs.rs/arrayvec) の `ArrayVec<T, N>` は、最大容量 `N` が**コンパイル時に固定**された配列ベースのベクターです。ヒープ確保が一切なく、スタックのみで動作します。

```
ArrayVec<i32, 8> のメモリレイアウト:

スタック:
┌────┬────┬────┬────┬────┬────┬────┬────┬─────┐
│  1 │  2 │  3 │(未)│(未)│(未)│(未)│(未)│len=3│
└────┴────┴────┴────┴────┴────┴────┴────┴─────┘
 <-- 8 * sizeof(i32) = 32バイト --> + usize

NOTE: Vec と違い ptr フィールドが不要。常にスタック内に存在。
```

### `Cargo.toml` の設定

```toml
[dependencies]
arrayvec = "0.7"
```

### 基本的な使い方

```rust
use arrayvec::ArrayVec;

fn main() {
    // 最大8要素のArrayVec
    let mut av: ArrayVec<i32, 8> = ArrayVec::new();

    av.push(10);
    av.push(20);
    av.push(30);

    println!("len: {}, capacity: {}", av.len(), av.capacity());

    // 容量を超えるとパニックではなく Err を返す
    for i in 0..10 {
        match av.try_push(i) {
            Ok(()) => println!("pushed {}", i),
            Err(e) => println!("容量オーバー: {}", e),
        }
    }

    // 配列からの変換
    let av2: ArrayVec<i32, 4> = [1, 2, 3, 4].into_iter().collect();
    println!("{:?}", av2);

    // 固定長配列への変換（ちょうど満杯の場合）
    if let Ok(arr) = av2.into_inner() {
        println!("固定長配列: {:?}", arr);
    }
}
```

### 組み込みシステムでの活用

`ArrayVec` はヒープがない環境（`no_std`）でも使えます。

```rust
#![no_std]
use arrayvec::ArrayVec;

/// 割り込みハンドラで使用するイベントキュー
/// ヒープなし・アロケータなしで動作
static mut EVENT_QUEUE: ArrayVec<u8, 64> = ArrayVec::new_const();

fn handle_interrupt(event: u8) {
    unsafe {
        // 組み込みでは安全性を手動で管理
        let _ = EVENT_QUEUE.try_push(event);
    }
}
```

---

## 固定長配列との使い分け

### `[T; N]` 固定長配列

最も単純で、スタック上に全データが連続して存在します。

```rust
fn main() {
    // 固定長配列: サイズはコンパイル時に決定
    let arr: [i32; 8] = [0; 8];
    println!("size: {} bytes", std::mem::size_of_val(&arr)); // 32バイト

    // 欠点: サイズが固定。使用済み要素数を別途管理する必要がある
    let mut buf = [0i32; 8];
    let mut len = 0usize;
    buf[len] = 42; len += 1;
    buf[len] = 43; len += 1;
    // len より前が有効なデータ
}
```

### 使い分けのガイドライン

```
                          要素数が予測できる？
                          /              \
                        YES              NO
                        /                \
              小さい（<32要素程度）？      → Vec<T>
              /              \
            YES              NO
            /                 \
  ヒープ禁止？               スタック優先したい？
  /        \                 /              \
YES        NO              YES              NO
 ↓          ↓               ↓                ↓
[T;N]    ArrayVec<T,N>   SmallVec<[T;N]>   Vec<T>
固定長配列  ヒープ確保なし  少量はスタック    標準的
```

### 具体的な比較表

| 型 | ヒープ確保 | 動的サイズ変更 | `no_std` | 推奨用途 |
|----|-----------|-------------|---------|---------|
| `[T; N]` | なし | 不可 | 可 | サイズが完全に固定 |
| `ArrayVec<T, N>` | なし | 可（N以内） | 可 | 最大サイズが決まっている |
| `SmallVec<[T; N]>` | N超過時のみ | 可 | 不可 | 大抵はNより小さい |
| `Vec<T>` | 常に | 可 | 不可 | 一般用途 |

### パフォーマンス比較

```rust
use arrayvec::ArrayVec;
use smallvec::SmallVec;
use std::time::Instant;

const ITER: usize = 5_000_000;

macro_rules! bench {
    ($name:expr, $block:block) => {{
        let start = Instant::now();
        for _ in 0..ITER $block
        println!("{:>20}: {:?}", $name, start.elapsed());
    }};
}

fn main() {
    println!("=== 4要素の push + sum ===");

    bench!("[i32; 4] (手動)", {
        let mut arr = [0i32; 4];
        let mut len = 0;
        for i in 0..4i32 { arr[len] = i; len += 1; }
        std::hint::black_box(arr[..len].iter().sum::<i32>());
    });

    bench!("ArrayVec<i32, 4>", {
        let mut av = ArrayVec::<i32, 4>::new();
        for i in 0..4i32 { av.push(i); }
        std::hint::black_box(av.iter().sum::<i32>());
    });

    bench!("SmallVec<[i32; 4]>", {
        let mut sv = SmallVec::<[i32; 4]>::new();
        for i in 0..4i32 { sv.push(i); }
        std::hint::black_box(sv.iter().sum::<i32>());
    });

    bench!("Vec<i32>", {
        let mut v = Vec::<i32>::new();
        for i in 0..4i32 { v.push(i); }
        std::hint::black_box(v.iter().sum::<i32>());
    });
}
```

典型的な結果:
```
=== 4要素の push + sum ===
     [i32; 4] (手動): 28ms
    ArrayVec<i32, 4>: 35ms
  SmallVec<[i32; 4]>: 42ms
            Vec<i32>: 210ms
```

---

## 実践例: AST ノードの子要素リスト

多くのASTノードは子要素が少ない（0〜4個）ため、`SmallVec` が効果的です。

```rust
use smallvec::SmallVec;

#[derive(Debug)]
enum Expr {
    Literal(i64),
    BinOp {
        op: char,
        // 通常は2つの子ノード。3項演算子でも3つ。
        // SmallVec<[Box<Expr>; 2]> でほぼヒープ確保なし
        children: SmallVec<[Box<Expr>; 2]>,
    },
    Call {
        func: String,
        // 引数は0〜数個が多い
        args: SmallVec<[Box<Expr>; 4]>,
    },
}

impl Expr {
    fn eval(&self) -> i64 {
        match self {
            Expr::Literal(n) => *n,
            Expr::BinOp { op, children } => {
                let a = children[0].eval();
                let b = children[1].eval();
                match op {
                    '+' => a + b,
                    '-' => a - b,
                    '*' => a * b,
                    '/' => a / b,
                    _ => panic!("unknown op"),
                }
            }
            Expr::Call { func, args } => {
                let vals: Vec<i64> = args.iter().map(|a| a.eval()).collect();
                match func.as_str() {
                    "sum" => vals.iter().sum(),
                    "max" => *vals.iter().max().unwrap_or(&0),
                    _ => 0,
                }
            }
        }
    }
}

fn main() {
    // (1 + 2) * (3 + 4) = 21
    use smallvec::smallvec;
    let expr = Expr::BinOp {
        op: '*',
        children: smallvec![
            Box::new(Expr::BinOp {
                op: '+',
                children: smallvec![
                    Box::new(Expr::Literal(1)),
                    Box::new(Expr::Literal(2)),
                ],
            }),
            Box::new(Expr::BinOp {
                op: '+',
                children: smallvec![
                    Box::new(Expr::Literal(3)),
                    Box::new(Expr::Literal(4)),
                ],
            }),
        ],
    };

    println!("結果: {}", expr.eval()); // 21
}
```

---

## まとめ

- `Vec<T>`: 汎用。サイズが大きい・予測不能な場合に使う
- `SmallVec<[T; N]>`: 「大抵は N 個以内」のケースに最適
- `ArrayVec<T, N>`: 最大容量が決まっていてヒープを使いたくない場合
- `[T; N]`: 完全に固定サイズで最速が必要な場合

[-> 03: リングバッファ](./03-ring-buffer.md)
