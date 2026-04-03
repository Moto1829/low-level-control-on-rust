# portable_simd (nightly)

`std::simd` は、特定のCPUアーキテクチャに依存しない形でSIMD演算を記述できるRustのnightly専用APIである。
`std::arch` の手動intrinsicsと異なり、1つのコードで複数のアーキテクチャ（x86_64・ARM NEON・WASM SIMDなど）に対応できる。

---

## `#![feature(portable_simd)]` の有効化

`std::simd` はnightly限定の機能であるため、クレートルートで明示的に有効化する必要がある。

```bash
# nightly ツールチェインをインストール
rustup install nightly

# nightly で実行
cargo +nightly run
cargo +nightly test
```

`src/main.rs` または `src/lib.rs` の先頭に記述する:

```rust
#![feature(portable_simd)]

use std::simd::prelude::*;
```

`rust-toolchain.toml` でプロジェクト全体をnightly固定にする方法:

```toml
[toolchain]
channel = "nightly"
```

---

## `std::simd` モジュールの概要

`std::simd` が提供する主な型と構造:

| 型 | 説明 |
|----|------|
| `Simd<T, N>` | T型のN要素SIMDベクター |
| `Mask<T, N>` | 比較演算の結果を表すマスク型 |
| `SimdFloat` | 浮動小数点演算トレイト |
| `SimdInt` | 整数演算トレイト |
| `SimdOrd` | 順序比較トレイト |
| `SimdPartialEq` | 等値比較トレイト |

型エイリアスが定義されており、よく使うものは以下のとおり:

```rust
#![feature(portable_simd)]
use std::simd::prelude::*;

// 型エイリアスの例
type F32x4  = Simd<f32, 4>;   // 128bit: SSE / NEON
type F32x8  = Simd<f32, 8>;   // 256bit: AVX2
type F32x16 = Simd<f32, 16>;  // 512bit: AVX-512
type I32x8  = Simd<i32, 8>;   // 256bit 整数
type U8x32  = Simd<u8, 32>;   // 256bit バイト列

fn main() {
    let a = F32x8::splat(1.0); // 全要素を 1.0 で初期化
    let b = F32x8::splat(2.0);
    let c = a + b;             // 要素ごとの加算
    println!("{:?}", c);       // [3.0, 3.0, 3.0, 3.0, 3.0, 3.0, 3.0, 3.0]
}
```

---

## `Simd<f32, 8>` の作り方・演算

### ベクターの生成

```rust
#![feature(portable_simd)]
use std::simd::prelude::*;

fn simd_creation_examples() {
    // 1. 全要素を同じ値で初期化
    let zeros = Simd::<f32, 8>::splat(0.0);
    let ones  = Simd::<f32, 8>::splat(1.0);

    // 2. 配列から生成
    let arr = [1.0f32, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0];
    let v = Simd::<f32, 8>::from_array(arr);

    // 3. スライスから読み込み（境界チェックあり）
    let data: Vec<f32> = (1..=8).map(|x| x as f32).collect();
    let v2 = Simd::<f32, 8>::from_slice(&data);

    // 4. 個々の要素を指定
    let v3 = Simd::<f32, 8>::from([0.0, 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0]);

    // 要素の取り出し
    let val = v[0]; // インデックスアクセス
    let arr_out = v.to_array(); // 配列に変換

    println!("zeros: {:?}", zeros);
    println!("v: {:?}", v);
    println!("v[0]: {}", val);
}
```

### 基本演算

```rust
#![feature(portable_simd)]
use std::simd::prelude::*;

fn arithmetic_examples() {
    let a = Simd::<f32, 8>::from_array([1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0]);
    let b = Simd::<f32, 8>::from_array([8.0, 7.0, 6.0, 5.0, 4.0, 3.0, 2.0, 1.0]);

    let add = a + b;      // 要素ごとの加算 → [9.0; 8]
    let sub = a - b;      // 要素ごとの減算
    let mul = a * b;      // 要素ごとの乗算 → [8.0, 14.0, 18.0, 20.0, ...]
    let div = a / b;      // 要素ごとの除算
    let neg = -a;         // 符号反転

    // FMA（積和演算）: a * b + c
    let c = Simd::<f32, 8>::splat(1.0);
    // FMA は mul_add メソッドで実現（SimdFloat トレイト）
    let fma = a * b + c;  // コンパイラが FMA 命令に変換することがある

    println!("add: {:?}", add);
    println!("mul: {:?}", mul);
}
```

---

## `SimdFloat`・`SimdOrd` 等のトレイト

### SimdFloat トレイト

浮動小数点数特有の演算を提供する。

```rust
#![feature(portable_simd)]
use std::simd::prelude::*;
use std::simd::StdFloat; // sqrt, ceil, floor, round

fn simd_float_examples() {
    let v = Simd::<f32, 8>::from_array([1.0, 4.0, 9.0, 16.0, 25.0, 36.0, 49.0, 64.0]);

    let sq = v.sqrt();   // 要素ごとの平方根
    let fl = v.floor();  // 切り捨て
    let ce = v.ceil();   // 切り上げ
    let ro = v.round();  // 四捨五入
    let ab = v.abs();    // 絶対値

    println!("sqrt: {:?}", sq); // [1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0]
}
```

### SimdOrd と比較演算

```rust
#![feature(portable_simd)]
use std::simd::prelude::*;

fn simd_comparison_examples() {
    let a = Simd::<f32, 8>::from_array([1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0]);
    let b = Simd::<f32, 8>::splat(4.0);

    // 比較演算（Mask を返す）
    let mask_gt = a.simd_gt(b);   // a > b の各要素
    let mask_lt = a.simd_lt(b);   // a < b の各要素
    let mask_eq = a.simd_eq(b);   // a == b の各要素

    println!("a > 4.0: {:?}", mask_gt);
    // Mask([false, false, false, false, true, true, true, true])

    // マスクを使った条件付き選択
    let result = mask_gt.select(a, b);
    // a > 4.0 の部分は a、それ以外は b を選択
    println!("select: {:?}", result);
    // [4.0, 4.0, 4.0, 4.0, 5.0, 6.0, 7.0, 8.0]

    // 整数型での SimdOrd（min/max）
    let ia = Simd::<i32, 8>::from_array([1, 5, 3, 7, 2, 8, 4, 6]);
    let ib = Simd::<i32, 8>::splat(4);
    let min_v = ia.simd_min(ib);
    let max_v = ia.simd_max(ib);
    println!("min: {:?}", min_v); // [1, 4, 3, 4, 2, 4, 4, 4]
    println!("max: {:?}", max_v); // [4, 5, 4, 7, 4, 8, 4, 6]
}
```

### リダクション演算

```rust
#![feature(portable_simd)]
use std::simd::prelude::*;
use std::simd::num::SimdFloat;

fn reduction_examples() {
    let v = Simd::<f32, 8>::from_array([1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0]);

    let sum = v.reduce_sum();   // 全要素の合計
    let prod = v.reduce_product(); // 全要素の積
    let max = v.reduce_max();   // 最大値
    let min = v.reduce_min();   // 最小値

    println!("sum: {}", sum);   // 36.0
    println!("max: {}", max);   // 8.0
}
```

---

## `std::arch` との使い分け

| 観点 | `std::simd` (portable_simd) | `std::arch` (手動intrinsics) |
|------|----------------------------|------------------------------|
| 移植性 | 高い（x86・ARM・WASMなど）| 低い（特定アーキテクチャ固定） |
| 最大性能 | コンパイラ依存（概ね良好）| 完全制御（最高性能が出やすい） |
| コードの読みやすさ | 高い | 低い（intrinsics名が難解） |
| 安全性 | `unsafe` 不要 | `unsafe` ブロックが必要 |
| 利用可能時期 | nightly限定（2024年時点） | stable |
| FMA・特殊命令 | 制限あり | 全命令が使える |

**判断基準**:
- 新規開発でnightyが使える → まず `std::simd` を試す
- AVX-512の特定命令など、最大性能が必要 → `std::arch`
- ライブラリとして広く配布する → stable版のため `std::arch` か自動ベクトル化

---

## 実例1: 配列の合計

```rust
#![feature(portable_simd)]
use std::simd::prelude::*;
use std::simd::num::SimdFloat;

/// portable_simd を使った f32 配列の合計
pub fn sum_f32_simd(data: &[f32]) -> f32 {
    const LANES: usize = 8;
    let (head, chunks, tail) = data.as_simd::<LANES>();
    // as_simd() はスライスを3つに分割する:
    //   head:   SIMD アライメントが合うまでのスカラー先頭部分
    //   chunks: SIMD ベクターのスライス
    //   tail:   端数のスカラー末尾部分

    // チャンク部分をSIMDで処理
    let mut acc = Simd::<f32, LANES>::splat(0.0);
    for chunk in chunks {
        acc += chunk;
    }

    // スカラー部分とSIMD部分を合計
    let simd_sum = acc.reduce_sum();
    let scalar_sum: f32 = head.iter().chain(tail.iter()).sum();

    simd_sum + scalar_sum
}

/// 比較用: 通常のスカラー実装
pub fn sum_f32_scalar(data: &[f32]) -> f32 {
    data.iter().sum()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_sum_correctness() {
        let data: Vec<f32> = (1..=100).map(|x| x as f32).collect();
        let expected = 5050.0f32;

        let simd_result = sum_f32_simd(&data);
        let scalar_result = sum_f32_scalar(&data);

        assert!((simd_result - expected).abs() < 1e-3,
            "SIMD合計が不正: {} (期待値: {})", simd_result, expected);
        assert!((scalar_result - expected).abs() < 1e-3,
            "スカラー合計が不正: {}", scalar_result);
    }

    #[test]
    fn test_sum_empty() {
        let data: Vec<f32> = vec![];
        assert_eq!(sum_f32_simd(&data), 0.0);
    }

    #[test]
    fn test_sum_non_multiple_of_8() {
        // SIMD幅(8)の倍数でない長さでも正しく動作するか
        let data: Vec<f32> = (1..=13).map(|x| x as f32).collect();
        let expected: f32 = (1..=13).map(|x| x as f32).sum();
        let result = sum_f32_simd(&data);
        assert!((result - expected).abs() < 1e-3,
            "端数処理が不正: {} (期待値: {})", result, expected);
    }
}
```

---

## 実例2: ドット積（dot product）

```rust
#![feature(portable_simd)]
use std::simd::prelude::*;
use std::simd::num::SimdFloat;

/// portable_simd を使った f32 ドット積
///
/// ドット積は機械学習・物理シミュレーション・音響処理など
/// 幅広い分野で使われる基本演算。
pub fn dot_product_simd(a: &[f32], b: &[f32]) -> f32 {
    assert_eq!(a.len(), b.len(), "ベクターの長さが一致しません");

    const LANES: usize = 8;

    let (a_head, a_chunks, a_tail) = a.as_simd::<LANES>();
    let (b_head, b_chunks, b_tail) = b.as_simd::<LANES>();

    // チャンク部分: SIMD 積和
    let mut acc = Simd::<f32, LANES>::splat(0.0);
    for (ca, cb) in a_chunks.iter().zip(b_chunks.iter()) {
        acc += ca * cb;
    }

    let simd_part = acc.reduce_sum();

    // スカラー部分（head と tail）
    let scalar_part: f32 = a_head.iter().zip(b_head.iter())
        .chain(a_tail.iter().zip(b_tail.iter()))
        .map(|(x, y)| x * y)
        .sum();

    simd_part + scalar_part
}

/// FMA（積和融合）を明示的に使ったバージョン
/// FMA は浮動小数点誤差を低減し、性能も向上する。
pub fn dot_product_fma(a: &[f32], b: &[f32]) -> f32 {
    assert_eq!(a.len(), b.len(), "ベクターの長さが一致しません");

    const LANES: usize = 8;
    let mut acc = Simd::<f32, LANES>::splat(0.0);
    let zero = Simd::<f32, LANES>::splat(0.0);

    let chunks_a = a.chunks_exact(LANES);
    let remainder_a = chunks_a.remainder();
    let chunks_b = b.chunks_exact(LANES);
    let remainder_b = chunks_b.remainder();

    for (ca, cb) in chunks_a.zip(chunks_b) {
        let va = Simd::<f32, LANES>::from_slice(ca);
        let vb = Simd::<f32, LANES>::from_slice(cb);
        acc += va * vb; // コンパイラが FMA に変換する場合がある
    }

    let simd_sum = acc.reduce_sum();
    let scalar_sum: f32 = remainder_a.iter().zip(remainder_b.iter())
        .map(|(x, y)| x * y)
        .sum();

    simd_sum + scalar_sum
}

/// 比較用スカラー実装
pub fn dot_product_scalar(a: &[f32], b: &[f32]) -> f32 {
    assert_eq!(a.len(), b.len());
    a.iter().zip(b.iter()).map(|(x, y)| x * y).sum()
}

#[cfg(test)]
mod tests {
    use super::*;

    fn approx_eq(a: f32, b: f32, tol: f32) -> bool {
        (a - b).abs() < tol
    }

    #[test]
    fn test_dot_product_basic() {
        let a = vec![1.0f32, 2.0, 3.0, 4.0];
        let b = vec![4.0f32, 3.0, 2.0, 1.0];
        // 1*4 + 2*3 + 3*2 + 4*1 = 4 + 6 + 6 + 4 = 20
        let expected = 20.0f32;

        assert!(approx_eq(dot_product_simd(&a, &b), expected, 1e-4));
        assert!(approx_eq(dot_product_fma(&a, &b), expected, 1e-4));
        assert!(approx_eq(dot_product_scalar(&a, &b), expected, 1e-4));
    }

    #[test]
    fn test_dot_product_large() {
        let n = 1000;
        let a: Vec<f32> = (0..n).map(|i| i as f32).collect();
        let b: Vec<f32> = (0..n).map(|_| 1.0f32).collect();

        let simd_result = dot_product_simd(&a, &b);
        let scalar_result = dot_product_scalar(&a, &b);

        assert!(approx_eq(simd_result, scalar_result, 1e-2),
            "SIMD={}, scalar={}", simd_result, scalar_result);
    }

    #[test]
    fn test_dot_product_unit_vectors() {
        // 直交するベクトルの内積は 0
        let a = vec![1.0f32, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0];
        let b = vec![0.0f32, 1.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0];
        assert!(approx_eq(dot_product_simd(&a, &b), 0.0, 1e-6));
    }
}
```

---

## ビルドと実行

```bash
# nightly で実行
cargo +nightly run

# nightly でテスト
cargo +nightly test

# nightly でベンチマーク（criterion使用時）
cargo +nightly bench

# アセンブリを確認（ベクトル命令が使われているか）
RUSTFLAGS="-C target-cpu=native" cargo +nightly rustc --release -- --emit=asm
grep -n "vaddps\|vmulps\|vfmadd" target/release/deps/*.s
```

---

## まとめ

- `std::simd` はnightlyで利用可能な移植性の高いSIMD APIである
- `Simd<T, N>` は型安全で、`unsafe` なしにSIMD演算を記述できる
- `as_simd()` を使うとスライスをhead・chunks・tailに分割してSIMD処理できる
- `SimdFloat`・`SimdOrd` などのトレイトで豊富な演算が利用できる
- 最大性能が必要な場合や特定命令を使いたい場合は `std::arch` を選択する
- stableが必要な本番ライブラリでは自動ベクトル化か `std::arch` を検討する
