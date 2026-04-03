# 自動ベクトル化を促す書き方

コンパイラ（LLVM）は、適切な条件が揃ったループを自動的にSIMD命令に変換する。
この章では、自動ベクトル化（auto-vectorization）が発動する条件を理解し、
コードの書き方を工夫することでコンパイラの最適化を最大限に引き出す手法を解説する。

---

## 自動ベクトル化とは

手動で `std::arch` のintrinsicsを書かなくても、LLVMのループベクトライザーが
スカラーループをSIMD命令に変換することがある。
例えば以下の単純な加算ループは、`-C target-cpu=native` 付きでコンパイルすると
AVX2の `vpaddd` 命令を使ったベクトル化ループに変換される。

```rust
pub fn add_arrays(a: &[f32], b: &[f32], out: &mut [f32]) {
    for i in 0..a.len() {
        out[i] = a[i] + b[i];
    }
}
```

ベクトル化されると、1命令で8要素（AVX2のf32の場合）を同時処理できる。

---

## ベクトル化の条件

LLVMのループベクトライザーが動作するには、以下の条件が重要になる。

### 1. ループの反復間に依存関係がない

各イテレーションが独立して計算できる必要がある。
前のイテレーションの結果を次のイテレーションで使うループは、依存関係があるためベクトル化が難しい。

```rust
// ベクトル化されやすい（各要素が独立）
pub fn scale(data: &mut [f32], factor: f32) {
    for x in data.iter_mut() {
        *x *= factor;
    }
}

// ベクトル化されにくい（前の要素に依存）
pub fn prefix_sum(data: &mut [f32]) {
    for i in 1..data.len() {
        data[i] += data[i - 1]; // data[i] は data[i-1] に依存
    }
}
```

### 2. エイリアシングがない

2つのポインタが同じメモリを指していないことをコンパイラが証明できる必要がある。
Rustでは借用チェッカーにより、可変参照と不変参照の同時保持が禁止されるため、
C/C++よりもエイリアシングフリーの証明がしやすい。

```rust
// Rustの借用規則により、aとoutが同一でないことが保証される
pub fn copy_scaled(a: &[f32], out: &mut [f32], factor: f32) {
    for (src, dst) in a.iter().zip(out.iter_mut()) {
        *dst = *src * factor;
    }
}
```

### 3. ループ回数が事前にわかる（または推定できる）

コンパイラがベクトル幅の倍数でループを展開できる場合、ベクトル化しやすい。

---

## `assert!` によるスライス長の事前保証

Rustのスライスアクセスはデフォルトで境界チェックを行う。
この境界チェックがループの各イテレーションに挿入されると、コンパイラの最適化を妨げる。

`assert!` でスライス長を事前に保証すると、コンパイラは境界チェックを除去できる。

```rust
// 境界チェックあり（ベクトル化が阻害されることがある）
pub fn dot_product_naive(a: &[f32], b: &[f32]) -> f32 {
    let mut sum = 0.0f32;
    for i in 0..a.len().min(b.len()) {
        sum += a[i] * b[i]; // 各アクセスに境界チェック
    }
    sum
}

// assert! で事前保証（境界チェックが除去される）
pub fn dot_product_asserted(a: &[f32], b: &[f32]) -> f32 {
    assert_eq!(a.len(), b.len(), "スライスの長さが一致しません");
    let n = a.len();
    let mut sum = 0.0f32;
    for i in 0..n {
        // コンパイラは i < n を知っているので境界チェックを省略できる
        sum += a[i] * b[i];
    }
    sum
}
```

### イテレータを使う方法

イテレータ（`iter()`・`zip()`）を使うと、境界チェックが構造的に除去されることが多い。

```rust
pub fn dot_product_iter(a: &[f32], b: &[f32]) -> f32 {
    a.iter().zip(b.iter()).map(|(x, y)| x * y).sum()
}
```

これはコンパイラにとってベクトル化しやすいパターンの一つである。

---

## `-C target-cpu=native` フラグの効果

デフォルトのRustコンパイルでは、幅広いCPUとの互換性を保つために汎用的な命令セットが使われる。
`-C target-cpu=native` を指定すると、コンパイルを実行しているCPUの全機能（AVX2・AVX-512など）が使用可能になる。

```bash
# 開発・ベンチマーク時に使用
RUSTFLAGS="-C target-cpu=native" cargo build --release

# ベンチマーク実行
RUSTFLAGS="-C target-cpu=native" cargo bench
```

`Cargo.toml` に恒久的に設定する場合（`.cargo/config.toml`）:

```toml
[build]
rustflags = ["-C", "target-cpu=native"]
```

> **注意**: `target-cpu=native` でビルドしたバイナリは、他のCPUでは動作しないことがある。
> 配布用バイナリには使用しないこと。

---

## Compiler Explorerでアセンブリ出力を確認する手順

[Compiler Explorer（godbolt.org）](https://godbolt.org) を使うと、Rustコードが生成するアセンブリをブラウザで確認できる。

### 手順

1. [https://godbolt.org](https://godbolt.org) にアクセスする
2. 言語として **Rust** を選択する
3. コンパイラを選択する（例: `rustc 1.78.0`）
4. コンパイラオプションに最適化フラグを入力する:

```
-C opt-level=3 -C target-cpu=native
```

5. 左側ペインにRustコードを入力する
6. 右側ペインにアセンブリが表示される

### ベクトル化を確認する例

```rust
pub fn sum_f32(data: &[f32]) -> f32 {
    data.iter().sum()
}
```

コンパイラオプション: `-C opt-level=3 -C target-cpu=haswell`

出力アセンブリに `vaddps`（AVX加算）や `vperm2f128`（AVX並び替え）が含まれていれば、ベクトル化成功を示す。

### ローカルでアセンブリを確認する方法

```bash
# アセンブリファイルを出力
RUSTFLAGS="-C target-cpu=native" cargo rustc --release -- --emit=asm

# 出力先（例）
cat target/release/deps/<crate_name>-<hash>.s | grep -A 20 "sum_f32"
```

---

## ベクトル化を阻害するパターンと回避方法

### パターン1: 関数呼び出しをループ内に含める

コンパイラがインライン化できない外部関数はベクトル化を妨げる。

```rust
extern "C" {
    fn external_fn(x: f32) -> f32;
}

// ベクトル化されない（external_fn の内部が不明）
pub fn apply_external(data: &mut [f32]) {
    for x in data.iter_mut() {
        unsafe { *x = external_fn(*x); }
    }
}

// ベクトル化される（インライン化可能）
#[inline(always)]
fn my_fn(x: f32) -> f32 {
    x * x + 1.0
}

pub fn apply_inline(data: &mut [f32]) {
    for x in data.iter_mut() {
        *x = my_fn(*x);
    }
}
```

### パターン2: 条件分岐をループ内に含める

条件分岐があると、コンパイラは一般に両方の分岐をベクトル化して後で選択する（predication）か、
あるいはベクトル化を断念する。

```rust
// ベクトル化が困難なパターン
pub fn conditional_op(data: &mut [f32], threshold: f32) {
    for x in data.iter_mut() {
        if *x > threshold {
            *x = x.sqrt(); // sqrt はベクトル化できるが条件付きアクセスが問題
        }
    }
}

// ベクトル化しやすいパターン（分岐をなくす）
pub fn branchless_op(data: &mut [f32], threshold: f32) {
    for x in data.iter_mut() {
        // 条件式を算術演算に変換
        let above = (*x > threshold) as i32 as f32;
        *x = above * x.sqrt() + (1.0 - above) * *x;
    }
}
```

### パターン3: 異なるサイズ型の混在

```rust
// u8 から f32 への変換がループ内にあると複雑になる
pub fn convert_and_sum(data: &[u8]) -> f32 {
    data.iter().map(|&x| x as f32).sum() // コンパイラが自動ベクトル化を試みる
}

// 明示的な変換バッファを用意すると最適化しやすい
pub fn convert_and_sum_buffered(data: &[u8]) -> f32 {
    // チャンクごとに変換して処理
    data.chunks(8)
        .map(|chunk| {
            chunk.iter().map(|&x| x as f32).sum::<f32>()
        })
        .sum()
}
```

### パターン4: 不整列なメモリアクセス

```rust
// ストライドアクセスはベクトル化が難しい
pub fn sum_stride(data: &[f32], stride: usize) -> f32 {
    (0..data.len()).step_by(stride).map(|i| data[i]).sum()
}

// 連続アクセスはベクトル化されやすい
pub fn sum_contiguous(data: &[f32]) -> f32 {
    data.iter().sum()
}
```

---

## 実践例: 自動ベクトル化を意識したRGBグレースケール変換

```rust
/// RGB画像をグレースケールに変換する
/// ベクトル化を促すために:
///   - ループを3つの独立したスライス走査に分離
///   - assert! で長さを事前保証
///   - イテレータパターンを使用
pub fn rgb_to_grayscale(
    r: &[u8],
    g: &[u8],
    b: &[u8],
    out: &mut [u8],
) {
    let n = r.len();
    assert_eq!(n, g.len(), "r と g の長さが異なります");
    assert_eq!(n, b.len(), "r と b の長さが異なります");
    assert_eq!(n, out.len(), "r と out の長さが異なります");

    for i in 0..n {
        // BT.601 輝度係数（整数演算でベクトル化を促す）
        // Y = 0.299*R + 0.587*G + 0.114*B
        // 固定小数点: (77*R + 150*G + 29*B) >> 8
        let luma = (77u32 * r[i] as u32
            + 150u32 * g[i] as u32
            + 29u32 * b[i] as u32)
            >> 8;
        out[i] = luma as u8;
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_grayscale_white() {
        let r = vec![255u8; 64];
        let g = vec![255u8; 64];
        let b = vec![255u8; 64];
        let mut out = vec![0u8; 64];
        rgb_to_grayscale(&r, &g, &b, &mut out);
        // 白(255,255,255)は輝度255に近いはず
        for &v in &out {
            assert!(v >= 254, "白のグレースケール値が不正: {}", v);
        }
    }

    #[test]
    fn test_grayscale_black() {
        let r = vec![0u8; 64];
        let g = vec![0u8; 64];
        let b = vec![0u8; 64];
        let mut out = vec![255u8; 64];
        rgb_to_grayscale(&r, &g, &b, &mut out);
        for &v in &out {
            assert_eq!(v, 0, "黒のグレースケール値が不正: {}", v);
        }
    }
}
```

ビルドとアセンブリ確認:

```bash
RUSTFLAGS="-C target-cpu=native" cargo build --release

# アセンブリを確認して vpmullw や vpaddw が使われているか確認
RUSTFLAGS="-C target-cpu=native" cargo rustc --release -- --emit=asm
grep -n "vp\|ymm\|zmm" target/release/deps/*.s | head -30
```

---

## まとめ

| 手法 | 効果 |
|------|------|
| `assert!` で長さを事前保証 | 境界チェックを除去し、ループをシンプルにする |
| イテレータ（`zip`・`iter_mut`）を使う | 安全なエイリアシングフリーを保証し、ベクトル化を促す |
| ループ内の条件分岐を算術に変換 | 条件分岐による最適化阻害を回避する |
| `-C target-cpu=native` | CPUの最新SIMD命令セットを使用可能にする |
| Compiler Explorerで確認 | 実際に生成されるアセンブリを目視確認する |

自動ベクトル化は「ヒントを与えればコンパイラが判断する」手法であり、
`std::arch` の手動intrinsicsよりも移植性が高い。まずは自動ベクトル化を試み、
それで不十分な場合に手動SIMD（次章）へ進むことを推奨する。
