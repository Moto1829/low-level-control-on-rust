# 1. SIMDの基礎

## スカラー演算とSIMD演算の違い

### スカラー演算（従来の逐次処理）

通常のCPU演算は「1命令で1データ」を処理します。
例えばf32の配列8要素を加算する場合：

```
命令1: result[0] = a[0] + b[0]
命令2: result[1] = a[1] + b[1]
命令3: result[2] = a[2] + b[2]
命令4: result[3] = a[3] + b[3]
命令5: result[4] = a[4] + b[4]
命令6: result[5] = a[5] + b[5]
命令7: result[6] = a[6] + b[6]
命令8: result[7] = a[7] + b[7]
```

8回の加算命令が必要です。

### SIMD演算（並列処理）

SIMDでは、256ビット幅のレジスタに8つのf32を格納し、**1命令で8要素を同時加算**します：

```
      YMM0: [a0 | a1 | a2 | a3 | a4 | a5 | a6 | a7]   ← 256ビット
      YMM1: [b0 | b1 | b2 | b3 | b4 | b5 | b6 | b7]   ← 256ビット
               ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓
命令1: VADDPS
      YMM2: [r0 | r1 | r2 | r3 | r4 | r5 | r6 | r7]   ← 1命令で完了
```

---

## SIMDの基本概念

### レジスタの幅（Width）

SIMDレジスタは以下の幅を持ちます：

| レジスタ名 | 幅 | 命令セット |
|-----------|-----|----------|
| XMM0〜XMM15 | 128ビット | SSE/SSE2〜SSE4 |
| YMM0〜YMM15 | 256ビット | AVX/AVX2 |
| ZMM0〜ZMM31 | 512ビット | AVX-512 |

```
XMM (128-bit): [  32  |  32  |  32  |  32  ]  ← f32×4, f64×2, i32×4 など
YMM (256-bit): [  32  |  32  |  32  |  32  |  32  |  32  |  32  |  32  ]  ← f32×8
ZMM (512-bit): [ f32×16 または f64×8 または i32×16 など ]
```

### レーン（Lane）

レジスタ内の各要素スロットを**レーン**と呼びます。
256ビットのYMMレジスタにf32を格納すると、256 ÷ 32 = **8レーン**になります。

```
YMM0 の f32×8 レーン:
 レーン0  レーン1  レーン2  レーン3  レーン4  レーン5  レーン6  レーン7
[  3.14 |  1.41 |  2.71 |  1.73 |  0.57 |  1.61 |  2.23 |  3.00 ]
 bit0-31  32-63  64-95  96-127 128-159 160-191 192-223 224-255
```

### SIMD型の命名規則

`std::arch` の型名は以下のパターンに従います：

```
__m256i
  │  │└── i = integer（整数）
  │  └─── 256 = ビット幅
  └─────── m = SIMD型のプレフィックス

__m128  → 128ビット float (f32×4)
__m128d → 128ビット double (f64×2)
__m128i → 128ビット integer
__m256  → 256ビット float (f32×8)
__m256d → 256ビット double (f64×4)
__m256i → 256ビット integer
__m512  → 512ビット float (f32×16)
```

### 関数名の命名規則

intrinsicsの関数名も体系的な命名規則があります：

```
_mm256_add_ps
   │    │   └── ps = packed single (f32のパックド演算)
   │    └─────── add = 演算の種類
   └──────────── mm256 = 256ビット幅

サフィックスの意味:
  ps  = packed single-precision float (f32)
  pd  = packed double-precision float (f64)
  epi8/16/32/64 = packed integer (符号あり)
  epu8/16/32/64 = packed integer (符号なし)
  ss  = scalar single (1要素のf32)
  sd  = scalar double (1要素のf64)
```

---

## スカラーvsSIMDの実装比較

```rust
/// スカラー版: 要素ごとに加算
pub fn add_arrays_scalar(a: &[f32], b: &[f32], result: &mut [f32]) {
    for i in 0..a.len() {
        result[i] = a[i] + b[i];
    }
}

/// コンパイラが自動ベクトル化しやすいイテレータ版
pub fn add_arrays_iter(a: &[f32], b: &[f32], result: &mut [f32]) {
    result
        .iter_mut()
        .zip(a.iter().zip(b.iter()))
        .for_each(|(r, (x, y))| *r = x + y);
}
```

---

## CPUのSIMD対応確認

### `cpuid` 命令とは

`cpuid` はCPUの機能情報を問い合わせる特殊命令です。
Rustでは `std::arch::x86_64::__cpuid` で直接呼び出せますが、
通常は安全なマクロを使います。

### `is_x86_feature_detected!` マクロ

Rustの標準ライブラリが提供するランタイム機能検出マクロです。
**コンパイル時ではなく実行時**にCPUの対応状況を確認します。

```rust
fn main() {
    // 基本的な使い方
    if is_x86_feature_detected!("avx2") {
        println!("このCPUはAVX2に対応しています");
    } else {
        println!("AVX2は未対応です");
    }

    // 複数の機能を確認
    let features = [
        "sse", "sse2", "sse3", "ssse3",
        "sse4.1", "sse4.2",
        "avx", "avx2",
        "avx512f",
        "fma",
        "bmi1", "bmi2",
        "popcnt",
    ];

    println!("\n=== CPU機能一覧 ===");
    for feature in &features {
        let supported = match *feature {
            "sse"     => is_x86_feature_detected!("sse"),
            "sse2"    => is_x86_feature_detected!("sse2"),
            "sse3"    => is_x86_feature_detected!("sse3"),
            "ssse3"   => is_x86_feature_detected!("ssse3"),
            "sse4.1"  => is_x86_feature_detected!("sse4.1"),
            "sse4.2"  => is_x86_feature_detected!("sse4.2"),
            "avx"     => is_x86_feature_detected!("avx"),
            "avx2"    => is_x86_feature_detected!("avx2"),
            "avx512f" => is_x86_feature_detected!("avx512f"),
            "fma"     => is_x86_feature_detected!("fma"),
            "bmi1"    => is_x86_feature_detected!("bmi1"),
            "bmi2"    => is_x86_feature_detected!("bmi2"),
            "popcnt"  => is_x86_feature_detected!("popcnt"),
            _         => false,
        };
        println!("  {:<12}: {}", feature, if supported { "対応" } else { "未対応" });
    }
}
```

### 実行結果の例（Haswell世代のIntel Core i7）

```
=== CPU機能一覧 ===
  sse         : 対応
  sse2        : 対応
  sse3        : 対応
  ssse3       : 対応
  sse4.1      : 対応
  sse4.2      : 対応
  avx         : 対応
  avx2        : 対応
  avx512f     : 未対応
  fma         : 対応
  bmi1        : 対応
  bmi2        : 対応
  popcnt      : 対応
```

### ディスパッチパターン：機能に応じて実装を切り替える

```rust
pub fn compute_sum(data: &[f32]) -> f32 {
    // ランタイムでCPU機能を確認し最適な実装を選択
    #[cfg(target_arch = "x86_64")]
    {
        if is_x86_feature_detected!("avx2") {
            // AVX2対応CPUなら高速なSIMD実装を使う
            // SAFETY: AVX2対応を確認済み
            return unsafe { sum_avx2(data) };
        }
        if is_x86_feature_detected!("sse4.1") {
            return unsafe { sum_sse41(data) };
        }
    }
    // フォールバック: 通常のスカラー演算
    sum_scalar(data)
}

fn sum_scalar(data: &[f32]) -> f32 {
    data.iter().sum()
}

// 実装は次節で詳しく説明します
#[cfg(target_arch = "x86_64")]
unsafe fn sum_avx2(data: &[f32]) -> f32 {
    // AVX2 intrinsicsを使った実装（02-std-arch.md で詳解）
    todo!()
}

#[cfg(target_arch = "x86_64")]
unsafe fn sum_sse41(data: &[f32]) -> f32 {
    todo!()
}
```

---

## コンパイル時機能検出：`#[cfg(target_feature = "...")]`

ランタイム検出に対して、コンパイル時に機能を確認する方法もあります：

```rust
fn main() {
    // コンパイル時にAVX2が有効な場合のみコンパイルされる
    #[cfg(target_feature = "avx2")]
    println!("AVX2はコンパイル時に有効化されています");

    #[cfg(not(target_feature = "avx2"))]
    println!("AVX2はコンパイル時に無効です");
}
```

コンパイル時有効化は `-C target-feature=+avx2` フラグや
`#[target_feature(enable = "avx2")]` アトリビュートで制御します。

| 検出方法 | タイミング | 柔軟性 | 用途 |
|---------|-----------|-------|------|
| `is_x86_feature_detected!` | 実行時 | 高（配布バイナリに最適） | ランタイムディスパッチ |
| `#[cfg(target_feature)]` | コンパイル時 | 低（特定CPU向けビルド） | 専用最適化ビルド |
| `#[target_feature(enable)]` | コンパイル時（関数単位） | 中（関数ごとに制御） | intrinsics関数に必須 |

---

## アライメントの重要性

SIMDのロード・ストア命令はメモリアライメントに敏感です。

```rust
use std::alloc::{alloc, dealloc, Layout};

/// 32バイトアライメント保証付きのf32バッファを確保
fn alloc_aligned_f32(count: usize) -> (*mut f32, Layout) {
    let layout = Layout::from_size_align(
        count * std::mem::size_of::<f32>(),
        32,  // AVX2は256ビット = 32バイトアライメントが最適
    ).unwrap();
    let ptr = unsafe { alloc(layout) as *mut f32 };
    (ptr, layout)
}
```

実際には `Vec` を使い、アライメント済みのスライスを渡す方法が一般的です。
AVX2では非アライメントロード命令（`_mm256_loadu_ps`）も使えるため、
現代のコードでは厳密なアライメントは必須ではありませんが、
アライメントが揃っている方がパフォーマンスは向上します。

---

## まとめ

| 概念 | 内容 |
|------|------|
| SIMD | Single Instruction, Multiple Data：1命令で複数データを並列処理 |
| レーン | SIMDレジスタ内の各要素スロット |
| 幅 | XMM=128bit, YMM=256bit, ZMM=512bit |
| 命名規則 | `_mm256_add_ps` → 256bit幅・加算・f32パックド |
| ランタイム検出 | `is_x86_feature_detected!("avx2")` |
| コンパイル時検出 | `#[cfg(target_feature = "avx2")]` |

次節では実際に `std::arch` の intrinsics を使ってAVX2命令を呼び出します。

---

[次へ: 2. std::arch intrinsics](./02-std-arch.md)
