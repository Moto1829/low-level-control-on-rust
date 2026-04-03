# 02. メモリレイアウトとパディング

## 概要

Rustの構造体は、フィールドの型が持つアライメント制約を満たすために
コンパイラが自動的に**パディング**（詰め物バイト）を挿入します。
フィールドの宣言順を変えるだけでサイズが変わることがあり、
パフォーマンスやFFI互換性に直接影響します。
この節では、パディングの仕組みと、それを制御する `#[repr]` 属性を学びます。

---

## パディングとはなにか

### アライメント制約のおさらい

型 `T` のアライメントが `N` バイトであるとは、
**T のアドレスは必ず N の倍数でなければならない** という制約です。

| 型 | アライメント |
|-----|-------------|
| u8  | 1 バイト |
| u16 | 2 バイト |
| u32 | 4 バイト |
| u64 | 8 バイト |

### パディングが挿入される理由

構造体の各フィールドがアライメント制約を満たすため、
コンパイラは前のフィールドの後ろに**使われないバイト（パディング）**を挿入します。

```rust
use std::mem;

struct BadLayout {
    a: u8,   // 1 byte
    b: u32,  // 4 bytes (アライメント 4)
    c: u8,   // 1 byte
}

fn main() {
    println!("size = {}", mem::size_of::<BadLayout>());   // 12 !!
    println!("align = {}", mem::align_of::<BadLayout>()); // 4
}
```

なぜ 6 バイトではなく **12 バイト**になるのかを図で見てみましょう。

```
オフセット  0   1   2   3   4   5   6   7   8   9  10  11
           ┌───┬───────────┬───┬───┬───────────┬───┐
           │ a │  padding  │       b       │ c │pad│
           │u8 │ 3 bytes   │    u32        │u8 │ 3 │
           └───┴───────────┴───────────────┴───┘───┘
```

- `a` (u8) はオフセット 0 に置かれる
- `b` (u32, align=4) はオフセット 4 から始める必要がある → 3 バイトのパディングが入る
- `c` (u8) はオフセット 8 に置かれる
- 構造体全体のサイズは `align(4)` の倍数に切り上げられる → 12 バイト

---

## フィールド順を最適化する

**大きいフィールドを先に並べる**と自然にアライメントが揃い、パディングが減ります。

```rust
use std::mem;

struct BadLayout {
    a: u8,   // 1 byte
    b: u32,  // 4 bytes
    c: u8,   // 1 byte
}
// size = 12

struct GoodLayout {
    b: u32,  // 4 bytes  ← 大きいものから
    a: u8,   // 1 byte
    c: u8,   // 1 byte
}
// size = 8?

fn main() {
    println!("BadLayout:  size = {}", mem::size_of::<BadLayout>());  // 12
    println!("GoodLayout: size = {}", mem::size_of::<GoodLayout>()); // 8
}
```

`GoodLayout` の内部レイアウト:

```
オフセット  0   1   2   3   4   5   6   7
           ┌───────────────┬───┬───┬───┐
           │       b       │ a │ c │pad│
           │     u32       │u8 │u8 │ 2 │
           └───────────────┴───┴───┴───┘
```

パディングが 2 バイトに減り、12 → 8 バイトになりました。

### より複雑な例

```rust
use std::mem;

#[allow(dead_code)]
struct Suboptimal {
    flag: bool,   // 1 byte, align 1
    value: f64,   // 8 bytes, align 8
    count: u32,   // 4 bytes, align 4
    tag: u8,      // 1 byte, align 1
}

#[allow(dead_code)]
struct Optimal {
    value: f64,   // 8 bytes, align 8  ← 最大アライメントを先頭に
    count: u32,   // 4 bytes, align 4
    flag: bool,   // 1 byte
    tag: u8,      // 1 byte
                  // 末尾に 2 byte パディング（構造体全体は align 8 の倍数に）
}

fn main() {
    println!("Suboptimal: {} bytes", mem::size_of::<Suboptimal>()); // 24
    println!("Optimal:    {} bytes", mem::size_of::<Optimal>());    // 16
}
```

`Suboptimal` のレイアウト:
```
オフセット:  0    1─7    8─15   16─19   20   21─23
            flag  pad    value   count   tag   pad
```
合計 = 24 バイト

`Optimal` のレイアウト:
```
オフセット:  0─7    8─11   12    13   14─15
            value   count  flag   tag   pad
```
合計 = 16 バイト

---

## `#[repr(C)]` — C 互換レイアウト

### Rust のデフォルトレイアウトの問題

Rust コンパイラはデフォルトでフィールドを**自由に並び替えてよい**とされています
（`#[repr(Rust)]`）。実際に並び替えが起きるかどうかはコンパイラ次第ですが、
**ABI 境界（FFI）を越える構造体では保証が必要**です。

```rust
use std::mem;

// Rust デフォルト: コンパイラが自由に最適化できる
struct DefaultRepr {
    a: u8,
    b: u32,
    c: u16,
}

// C 互換レイアウト: フィールドの宣言順が保証される
#[repr(C)]
struct CRepr {
    a: u8,
    b: u32,
    c: u16,
}

fn main() {
    // どちらも同じサイズになることが多いが、
    // DefaultRepr はコンパイラが並び替えを行うこともある
    println!("DefaultRepr: {} bytes", mem::size_of::<DefaultRepr>());
    println!("CRepr:       {} bytes", mem::size_of::<CRepr>());
}
```

### `#[repr(C)]` の使いどころ

```rust
// C の構造体と対応させる場合 (FFI)
#[repr(C)]
pub struct CPoint {
    pub x: f64,
    pub y: f64,
}

// extern "C" 関数で受け渡す
extern "C" {
    fn c_distance(a: *const CPoint, b: *const CPoint) -> f64;
}

// Rust 側の実装例 (実際には C 側が提供する関数を呼ぶ)
fn distance(a: &CPoint, b: &CPoint) -> f64 {
    let dx = a.x - b.x;
    let dy = a.y - b.y;
    (dx * dx + dy * dy).sqrt()
}

fn main() {
    let p1 = CPoint { x: 0.0, y: 0.0 };
    let p2 = CPoint { x: 3.0, y: 4.0 };
    println!("distance = {}", distance(&p1, &p2)); // 5
}
```

### `#[repr(C)]` ではパディングの位置が固定される

```rust
use std::mem;

#[repr(C)]
struct Padded {
    a: u8,   // offset 0
    // 3 bytes padding ← ここに必ず入る
    b: u32,  // offset 4
    c: u8,   // offset 8
    // 3 bytes padding ← 末尾にも入る
}

fn main() {
    println!("size  = {}", mem::size_of::<Padded>());   // 12
    println!("align = {}", mem::align_of::<Padded>());  // 4
}
```

---

## `#[repr(packed)]` — パディングなし

### 概要

`#[repr(packed)]` はパディングを一切挿入せず、フィールドを詰め込みます。

```rust
use std::mem;

#[repr(packed)]
struct PackedStruct {
    a: u8,   // offset 0
    b: u32,  // offset 1 (アライメント違反！)
    c: u8,   // offset 5
}

fn main() {
    println!("size = {}", mem::size_of::<PackedStruct>()); // 6 (1+4+1)
}
```

### 危険性: アライメント違反

パックされた構造体のフィールドは**アライメントされていないアドレス**に置かれる可能性があります。
そのため、フィールドへの参照を取ることは `unsafe` です。

```rust
#[repr(packed)]
struct Packed {
    a: u8,
    b: u32,
}

fn main() {
    let p = Packed { a: 1, b: 0xDEADBEEF };

    // コンパイルエラー: packed 構造体のフィールドへの参照は unsafe
    // let ref_b = &p.b; // ← これはエラー

    // 安全な読み取り方: コピーを取る
    let b_copy = p.b; // u32 はコピー型なので値渡しは安全
    println!("b = {:#x}", b_copy);

    // または read_unaligned を使う
    let b_ptr = std::ptr::addr_of!(p.b);
    let b_val = unsafe { b_ptr.read_unaligned() };
    println!("b (unaligned read) = {:#x}", b_val);
}
```

### `#[repr(packed)]` の主な用途

1. ネットワークパケットや通信プロトコルのヘッダー構造体
2. バイナリファイルフォーマットの解析（FAT ファイルシステムなど）
3. マイコンなどメモリが極端に限られた環境

```rust
/// IPv4 ヘッダー（簡略版）の例
#[repr(C, packed)]
struct Ipv4Header {
    version_ihl: u8,       // バージョン(4bit) + IHL(4bit)
    dscp_ecn: u8,
    total_length: u16,
    identification: u16,
    flags_fragment: u16,
    ttl: u8,
    protocol: u8,
    checksum: u16,
    src_addr: [u8; 4],
    dst_addr: [u8; 4],
}

fn main() {
    use std::mem;
    println!("IPv4 ヘッダーサイズ: {} bytes", mem::size_of::<Ipv4Header>()); // 20
}
```

---

## `#[repr(align(N))]` — アライメントを強制する

逆にアライメントを**大きく**したい場合は `#[repr(align(N))]` を使います。
SIMD 操作やキャッシュライン最適化に使われます。

```rust
use std::mem;

// デフォルト: align = 1
struct Normal {
    data: [u8; 16],
}

// 16 バイト境界に強制アライン (SIMD 用途など)
#[repr(align(16))]
struct Aligned16 {
    data: [u8; 16],
}

// キャッシュライン (64 bytes) にアライン
#[repr(align(64))]
struct CacheAligned {
    value: u64,
    _pad: [u8; 56], // キャッシュラインをフルに使う
}

fn main() {
    println!("Normal    align = {}", mem::align_of::<Normal>());       // 1
    println!("Aligned16 align = {}", mem::align_of::<Aligned16>());    // 16
    println!("CacheAligned align = {}", mem::align_of::<CacheAligned>()); // 64

    // 実際のアドレスがアライメントを満たしているか確認
    let a = Aligned16 { data: [0; 16] };
    let addr = &a as *const Aligned16 as usize;
    assert_eq!(addr % 16, 0, "アドレスが 16 バイト境界にない");
    println!("Aligned16 @ {:#x}: アライメント OK", addr);
}
```

---

## `memoffset` クレートでオフセットを確認する

### セットアップ

`Cargo.toml` に追加します:

```toml
[dependencies]
memoffset = "0.9"
```

### `offset_of!` マクロ

フィールドが構造体の先頭から何バイト目にあるかを確認できます。

```rust
use memoffset::offset_of;
use std::mem;

#[repr(C)]
struct Data {
    id: u32,       // offset 0
    flag: bool,    // offset 4
    // 3 bytes padding
    value: f64,    // offset 8
    count: u16,    // offset 16
    // 6 bytes padding
}

fn main() {
    println!("size  = {}", mem::size_of::<Data>()); // 24

    println!("id    @ offset {}", offset_of!(Data, id));    // 0
    println!("flag  @ offset {}", offset_of!(Data, flag));  // 4
    println!("value @ offset {}", offset_of!(Data, value)); // 8
    println!("count @ offset {}", offset_of!(Data, count)); // 16
}
```

### ネスト構造体のオフセット

```rust
use memoffset::offset_of;

#[repr(C)]
struct Inner {
    x: u32,
    y: u32,
}

#[repr(C)]
struct Outer {
    a: u8,
    // 3 bytes padding
    inner: Inner,
    b: u16,
}

fn main() {
    println!("inner @ offset {}", offset_of!(Outer, inner));   // 4
    println!("inner.x @ offset {}", offset_of!(Outer, inner.x)); // 4
    println!("inner.y @ offset {}", offset_of!(Outer, inner.y)); // 8
    println!("b @ offset {}", offset_of!(Outer, b));           // 12
}
```

---

## ゼロサイズ型 (ZST)

サイズが 0 の型は Rust で有効であり、
型システムでマーカー情報を持つために広く使われます。

```rust
use std::mem;

struct Unit;               // ZST
struct PhantomWrapper<T>(std::marker::PhantomData<T>); // ZST

fn main() {
    println!("Unit のサイズ: {}", mem::size_of::<Unit>());          // 0
    println!("PhantomWrapper のサイズ: {}", mem::size_of::<PhantomWrapper<u64>>()); // 0

    // ZST はアドレスを持てるが、複数の ZST が同じアドレスを持つこともある
    let a = Unit;
    let b = Unit;
    println!("a @ {:p}", &a);
    println!("b @ {:p}", &b); // a と同じアドレスになることがある

    // ZST の Vec はヒープ確保をしない
    let v: Vec<Unit> = Vec::with_capacity(1_000_000);
    println!("Vec<Unit> の capacity = {}, ヒープ確保 = 0 bytes", v.capacity());
}
```

---

## 実践: 構造体パディング診断ツール

フィールドのオフセットとサイズを一覧表示する汎用関数を書いてみましょう。

```rust
/// フィールドのレイアウト情報を表示する (デモ用に手動でオフセットを渡す)
fn print_layout(name: &str, total_size: usize, fields: &[(&str, usize, usize)]) {
    println!("=== {} (size = {}) ===", name, total_size);
    println!("{:<12} {:>8}  {:>6}  {:>8}", "field", "offset", "size", "end");
    println!("{}", "-".repeat(40));

    let mut prev_end = 0;
    for (fname, offset, size) in fields {
        if *offset > prev_end {
            println!(
                "{:<12} {:>8}  {:>6}  {:>8}  ← padding {} bytes",
                "(pad)", prev_end, offset - prev_end, offset,
                offset - prev_end
            );
        }
        println!("{:<12} {:>8}  {:>6}  {:>8}", fname, offset, size, offset + size);
        prev_end = offset + size;
    }
    if prev_end < total_size {
        println!(
            "{:<12} {:>8}  {:>6}  {:>8}  ← trailing padding",
            "(pad)", prev_end, total_size - prev_end, total_size
        );
    }
    println!();
}

#[repr(C)]
struct Example {
    a: u8,
    b: u32,
    c: u8,
    d: u16,
}

fn main() {
    use std::mem;
    use memoffset::offset_of;

    print_layout(
        "Example",
        mem::size_of::<Example>(),
        &[
            ("a: u8",  offset_of!(Example, a), mem::size_of::<u8>()),
            ("b: u32", offset_of!(Example, b), mem::size_of::<u32>()),
            ("c: u8",  offset_of!(Example, c), mem::size_of::<u8>()),
            ("d: u16", offset_of!(Example, d), mem::size_of::<u16>()),
        ],
    );
}
```

出力例:
```
=== Example (size = 12) ===
field          offset    size       end
----------------------------------------
a: u8               0       1         1
(pad)               1       3         4  ← padding 3 bytes
b: u32              4       4         8
c: u8               8       1         9
(pad)               9       1        10  ← padding 1 bytes
d: u16             10       2        12
```

---

## `#[repr]` 属性一覧まとめ

| 属性 | 効果 |
|------|------|
| `#[repr(Rust)]` | デフォルト。コンパイラが自由に最適化 |
| `#[repr(C)]` | C 互換。宣言順のレイアウト保証 |
| `#[repr(packed)]` | パディングなし。アライメント違反に注意 |
| `#[repr(packed(N))]` | N バイト境界にパック |
| `#[repr(align(N))]` | アライメントを最低 N バイトに設定 |
| `#[repr(transparent)]` | 単一フィールドの構造体と同じレイアウト |

### `#[repr(transparent)]` の使いどころ

```rust
use std::mem;

// newtype パターン: i32 と完全に同じレイアウトを保証する
#[repr(transparent)]
struct Meters(i32);

#[repr(transparent)]
struct Celsius(f64);

fn main() {
    // レイアウトが保証されているため、ポインタキャストが安全
    assert_eq!(mem::size_of::<Meters>(), mem::size_of::<i32>());
    assert_eq!(mem::align_of::<Meters>(), mem::align_of::<i32>());

    let m = Meters(100);
    // Meters* ↔ i32* の変換が安全
    let ptr: *const i32 = &m as *const Meters as *const i32;
    println!("value = {}", unsafe { *ptr }); // 100
}
```

---

## まとめ

- パディングはアライメント制約を満たすために自動挿入される
- **大きいフィールドを先に置く**とパディングを削減できる
- `#[repr(C)]` は FFI 用途でフィールド順を保証する
- `#[repr(packed)]` はパディングを排除するが、アライメント違反に注意
- `#[repr(align(N))]` でアライメントを拡大できる（SIMD・キャッシュ最適化）
- `memoffset::offset_of!` でフィールドのバイトオフセットを確認できる

---

## 次の節

[03. アロケータ](./03-allocator.md) では、
ヒープメモリの確保・解放を担う `GlobalAlloc` トレイトと
カスタムアロケータの実装方法を学びます。
