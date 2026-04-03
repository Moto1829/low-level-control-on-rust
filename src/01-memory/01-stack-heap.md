# 01. スタックとヒープ

## 概要

Rustプログラムが動作するとき、メモリは大きく**スタック**と**ヒープ**の2つの領域に分かれます。
どちらに値が置かれるかによって、パフォーマンス特性・ライフタイム・アクセス方法がまったく異なります。
この節では両者の違いを具体的なコードと図で確認し、
`std::mem::size_of` / `align_of` を使って実際のサイズを調べる方法を学びます。

---

## スタック (Stack)

### 仕組み

スタックは**LIFO (Last In, First Out)** 構造のメモリ領域です。
関数が呼び出されるたびに**スタックフレーム**が積まれ、
関数が返るとフレームがまるごと解放されます。

```
高アドレス
┌────────────────────┐
│  main のフレーム   │  ← プログラム開始時に作成
│    x: i32 = 5      │
├────────────────────┤
│  foo のフレーム    │  ← foo() 呼び出し時に積まれる
│    a: i32 = 10     │
│    b: i32 = 20     │
├────────────────────┤  ← スタックポインタ (SP) が指す位置
│      (空き)        │
└────────────────────┘
低アドレス
```

### 特徴

| 項目 | 内容 |
|------|------|
| 確保/解放 | 自動（コンパイラが命令を生成） |
| 速度 | 非常に速い（SP のインクリメント/デクリメントのみ） |
| サイズ制限 | 通常 8 MB 前後（OS ごとに異なる） |
| ライフタイム | スコープと一致 |
| 使用する型 | コンパイル時にサイズが決まる型 |

### スタックへの配置を確認する

```rust
fn main() {
    // i32, f64, bool はコンパイル時にサイズが確定するためスタックに置かれる
    let x: i32 = 42;
    let y: f64 = 3.14;
    let flag: bool = true;

    // 変数のアドレスを表示して実際の配置を確認する
    println!("x のアドレス: {:p}", &x);
    println!("y のアドレス: {:p}", &y);
    println!("flag のアドレス: {:p}", &flag);

    // 出力例 (スタックは高アドレスから伸びるため数値が近い):
    // x のアドレス: 0x7ffd3b2c1a04
    // y のアドレス: 0x7ffd3b2c1a08
    // flag のアドレス: 0x7ffd3b2c1a10
}
```

> **ポイント**: 3 つの変数のアドレスが隣接していることがわかります。
> スタックフレーム内では変数が連続して並ぶため、キャッシュ効率が高くなります。

---

## ヒープ (Heap)

### 仕組み

ヒープはOSから動的に確保されるメモリ領域です。
Rustでは `Box<T>` / `Vec<T>` / `String` などのスマートポインタを通じて使われます。
確保と解放はアロケータ（後述）が管理します。

```
高アドレス
┌────────────────────────────────────────────┐
│            スタック                         │
│  box_val: Box<i32>  →  ptr = 0x55a...      │
│  vec: Vec<i32>      →  ptr, len=3, cap=4   │
└────────────────────────────────────────────┘
                │
                │  ポインタで参照
                ▼
┌────────────────────────────────────────────┐
│            ヒープ                           │
│  [0x55a...] = 100   ← Box が指す i32       │
│  [0x55b...] = 1, 2, 3, (未使用)  ← Vec バッファ│
└────────────────────────────────────────────┘
低アドレス
```

### 特徴

| 項目 | 内容 |
|------|------|
| 確保/解放 | 明示的（`Box::new`, `drop` など） |
| 速度 | スタックより遅い（アロケータの呼び出しが必要） |
| サイズ制限 | 実質的に物理メモリの上限まで |
| ライフタイム | 所有者が `drop` されるまで |
| 使用する型 | サイズが動的に変わる型 / 大きすぎてスタックに乗らない型 |

### ヒープへの配置を確認する

```rust
fn main() {
    // Box::new でヒープに i32 を確保する
    let box_val = Box::new(100_i32);

    // box_val 自体（ポインタ）はスタック上にある
    println!("box_val (ポインタ) のアドレス: {:p}", &box_val);
    // box_val が指す先（ヒープ上のデータ）のアドレス
    println!("ヒープ上のデータのアドレス:    {:p}", box_val.as_ref());

    // Vec もヒープにバッファを確保する
    let v = vec![1_i32, 2, 3];
    println!("v (管理構造体) のアドレス: {:p}", &v);
    println!("v のバッファのアドレス:    {:p}", v.as_ptr());
}
```

出力例:
```
box_val (ポインタ) のアドレス: 0x7ffd3b2c1a00   ← スタック
ヒープ上のデータのアドレス:    0x55a12b3c4d50   ← ヒープ
v (管理構造体) のアドレス: 0x7ffd3b2c1a10        ← スタック
v のバッファのアドレス:    0x55a12b3c4e80        ← ヒープ
```

---

## `size_of` と `align_of` で型を調べる

### size_of

`std::mem::size_of::<T>()` はコンパイル時に型 `T` のバイトサイズを返します。

```rust
use std::mem;

fn main() {
    // プリミティブ型
    println!("bool:  {} byte",  mem::size_of::<bool>());   // 1
    println!("u8:    {} byte",  mem::size_of::<u8>());     // 1
    println!("u16:   {} bytes", mem::size_of::<u16>());    // 2
    println!("u32:   {} bytes", mem::size_of::<u32>());    // 4
    println!("u64:   {} bytes", mem::size_of::<u64>());    // 8
    println!("u128:  {} bytes", mem::size_of::<u128>());   // 16
    println!("usize: {} bytes", mem::size_of::<usize>());  // 4 or 8 (アーキテクチャ依存)
    println!("f32:   {} bytes", mem::size_of::<f32>());    // 4
    println!("f64:   {} bytes", mem::size_of::<f64>());    // 8
    println!("char:  {} bytes", mem::size_of::<char>());   // 4 (Unicode スカラー値)

    // ポインタ / 参照
    println!("&u8:   {} bytes", mem::size_of::<&u8>());    // 8 (64 bit 環境)
    println!("Box<u8>: {} bytes", mem::size_of::<Box<u8>>()); // 8

    // スライス参照はファットポインタ (ptr + len)
    println!("&[u8]: {} bytes", mem::size_of::<&[u8]>());  // 16

    // Option<&T> は null 最適化により参照と同サイズ
    println!("Option<&u8>: {} bytes", mem::size_of::<Option<&u8>>()); // 8
}
```

### align_of

アライメントとは「メモリのどのアドレスに置けるか」の制約です。
`align_of::<T>()` は T が配置されるべきアドレスの倍数を返します。

```rust
use std::mem;

fn main() {
    println!("u8  のアライメント: {} byte",  mem::align_of::<u8>());   // 1
    println!("u16 のアライメント: {} bytes", mem::align_of::<u16>());  // 2
    println!("u32 のアライメント: {} bytes", mem::align_of::<u32>());  // 4
    println!("u64 のアライメント: {} bytes", mem::align_of::<u64>());  // 8
    println!("f64 のアライメント: {} bytes", mem::align_of::<f64>());  // 8

    // 構造体のアライメントは最大フィールドのアライメントに揃う
    #[repr(C)]
    struct Foo {
        a: u8,   // align 1
        b: u32,  // align 4
    }
    println!("Foo のアライメント: {} bytes", mem::align_of::<Foo>()); // 4
}
```

### なぜアライメントが重要か

多くのCPUは整列されていないアドレスへのアクセスを禁止しているか、
著しいパフォーマンス低下を招きます。

```
アドレス: 0  1  2  3  4  5  6  7
          ─────────────────────────
u32 (正): [      u32      ]       ← アドレス 4 の倍数 → OK
u32 (誤):    [      u32      ]    ← アドレス 1 → アライメント違反
```

```rust
use std::mem;

fn main() {
    // 値のアドレスが実際にアライメントを満たしているか確認
    let x: u64 = 0;
    let addr = &x as *const u64 as usize;
    let align = mem::align_of::<u64>();
    assert_eq!(addr % align, 0, "アドレスがアライメントを満たしていない！");
    println!("u64 @ {:#x} はアライメント {} を満たしている", addr, align);
}
```

### size_of_val で動的サイズを調べる

スライスや `dyn Trait` のような動的サイズ型 (DST) は `size_of_val` を使います。

```rust
use std::mem;

fn main() {
    let arr = [1_u32, 2, 3, 4, 5];
    let slice: &[u32] = &arr[1..4]; // 要素数 3

    // size_of::<[u32]>() はコンパイルエラー (サイズ不明)
    // size_of_val は実行時にスライスの長さを考慮してサイズを計算
    println!("スライスのバイトサイズ: {}", mem::size_of_val(slice)); // 12 (3 * 4)
}
```

---

## 型ごとのメモリ配置まとめ

```rust
use std::mem;

fn type_info<T: ?Sized>(label: &str, val: &T) {
    println!(
        "{:<20} size={:>3}, align={:>2}, addr={:p}",
        label,
        mem::size_of_val(val),
        mem::align_of_val(val),
        val as *const T as *const u8,
    );
}

fn main() {
    let a: i32 = 1;
    let b: f64 = 2.0;
    let c: bool = true;
    let d: char = 'A';
    let e: Box<i32> = Box::new(99);
    let f: Vec<u8> = vec![1, 2, 3];

    type_info("i32", &a);
    type_info("f64", &b);
    type_info("bool", &c);
    type_info("char", &d);
    type_info("Box<i32>", &e);
    type_info("Vec<u8> (管理構造体)", &f);
    type_info("Vec<u8> のバッファ", f.as_slice());
}
```

---

## スタックオーバーフロー

### 原因

スタックサイズには上限（デフォルト: Linux/macOS 8 MB、Windows 1 MB）があります。
以下の場合にオーバーフローが起きます。

1. 無限再帰（または深すぎる再帰）
2. スタック上に巨大な配列を確保する

### 例1: 無限再帰

```rust
fn infinite() -> u64 {
    infinite() + 1 // 基底ケースがないため永遠に再帰する
}

fn main() {
    // スレッドがスタックオーバーフローでクラッシュする
    // thread 'main' has overflowed its stack
    // fatal runtime error: stack overflow
    println!("{}", infinite());
}
```

### 例2: 巨大なスタック配列

```rust
fn allocate_large_array() {
    // スタック上に 10 MB の配列を確保しようとする → オーバーフロー
    let _huge: [u8; 10 * 1024 * 1024] = [0; 10 * 1024 * 1024];
    println!("確保できた");
}

fn main() {
    allocate_large_array();
}
```

### 対策: ヒープに逃がす

```rust
fn main() {
    // Box や Vec を使ってヒープに確保すればオーバーフローしない
    let huge: Box<[u8; 10 * 1024 * 1024]> = Box::new([0; 10 * 1024 * 1024]);
    println!("ヒープに {} bytes 確保した", huge.len());

    // または Vec を使う
    let large_vec: Vec<u8> = vec![0; 10 * 1024 * 1024];
    println!("Vec に {} bytes 確保した", large_vec.len());
}
```

### 対策: スレッドのスタックサイズを増やす

```rust
use std::thread;

fn deep_recursion(n: u64) -> u64 {
    if n == 0 { return 0; }
    deep_recursion(n - 1) + 1
}

fn main() {
    // スタックサイズを 32 MB に設定して新スレッドを起動する
    let handler = thread::Builder::new()
        .stack_size(32 * 1024 * 1024)
        .spawn(|| {
            println!("{}", deep_recursion(100_000));
        })
        .expect("スレッドの起動に失敗");

    handler.join().expect("スレッドがパニックした");
}
```

### 対策: 再帰をイテレータに変換する（スタック消費ゼロ）

```rust
/// 末尾再帰をループに変換して深い再帰を回避する例
fn sum_recursive(n: u64) -> u64 {
    if n == 0 { return 0; }
    n + sum_recursive(n - 1) // 末尾呼び出しではないため最適化されにくい
}

fn sum_iterative(n: u64) -> u64 {
    (1..=n).sum() // イテレータ版: スタック消費なし
}

fn main() {
    println!("再帰: {}", sum_recursive(1000));
    println!("反復: {}", sum_iterative(1_000_000)); // 再帰版より遥かに深くても安全
}
```

---

## まとめ

| | スタック | ヒープ |
|---|----------|--------|
| 管理方法 | コンパイラが自動生成 | アロケータが動的に管理 |
| 速度 | 非常に速い | 相対的に遅い |
| サイズ制限 | 数 MB | 物理メモリ上限まで |
| ライフタイム | スコープ内 | 所有者が drop されるまで |
| 適する用途 | 小さな固定サイズの値 | 大きな / 動的サイズの値 |

- `size_of::<T>()` でコンパイル時に型サイズを確認できる
- `align_of::<T>()` でアライメントを確認できる
- 大きなデータは `Box` / `Vec` でヒープに確保してスタックオーバーフローを防ぐ

---

## 次の節

[02. メモリレイアウトとパディング](./02-memory-layout.md) では、
構造体のフィールド順がどのようにメモリレイアウトに影響するかを学びます。
