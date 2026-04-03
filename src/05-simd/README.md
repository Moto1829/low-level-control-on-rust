# 第5章: SIMD・CPU命令

## 概要

SIMD（Single Instruction, Multiple Data）は、1つの命令で複数のデータを同時に処理するCPUの並列演算機能です。
画像処理・音声処理・機械学習・科学技術計算など、大量の数値データを扱う場面で劇的な高速化をもたらします。

本章では、RustからSIMD命令を直接制御する方法を学びます。
低レベルな `std::arch` intrinsics から始め、コンパイラの自動ベクトル化の活用、さらにnightly Rustで使える安全な抽象化 `std::simd`（portable_simd）まで体系的に習得します。

---

## なぜSIMDを学ぶのか

通常のスカラー演算は1命令で1データを処理します。
一方SIMDは、例えば256ビット幅のレジスタに8つのf32値を詰め込み、1命令で8つ同時に加算できます。

```
スカラー: a[0]+b[0], a[1]+b[1], ..., a[7]+b[7]  ← 8命令
SIMD:    [a0,a1,a2,a3,a4,a5,a6,a7] + [b0,b1,b2,b3,b4,b5,b6,b7]  ← 1命令
```

理論上8倍の速度向上が得られます（実際はメモリ帯域・レイテンシ等の影響を受けます）。

---

## SIMD命令セットの種類

現代のx86-64 CPUは世代を重ねるごとにSIMD命令セットを拡張してきました。

| 命令セット | 幅 | 整数 | 浮動小数点 | 導入世代 |
|-----------|-----|------|-----------|---------|
| **MMX** | 64-bit | ○ | ✗ | Pentium MMX (1997) |
| **SSE** | 128-bit | ✗ | ○ (f32×4) | Pentium III (1999) |
| **SSE2** | 128-bit | ○ | ○ (f64×2) | Pentium 4 (2001) |
| **SSE4.1/4.2** | 128-bit | ○ | ○ | Core 2 (2007) |
| **AVX** | 256-bit | ✗ | ○ (f32×8, f64×4) | Sandy Bridge (2011) |
| **AVX2** | 256-bit | ○ | ○ | Haswell (2013) |
| **AVX-512** | 512-bit | ○ | ○ (f32×16) | Skylake-X (2017) |
| **NEON** | 128-bit | ○ | ○ | ARM Cortex-A (ARM用) |

> **注意**: AMDの消費者向けCPUはAVX-512を2022年以降のRyzen 7000シリーズから対応。
> Apple Silicon（M1/M2/M3）はx86 SIMDではなくARM NEONを使います。

### 命令セットの包含関係

```
AVX-512
  └── AVX2
        └── AVX
              └── SSE4.2
                    └── SSE4.1
                          └── SSE3
                                └── SSE2
                                      └── SSE
```

新しい命令セットは旧命令セットを含みます。
ただし実行時に対象CPUが対応しているかを確認してから使う必要があります。

---

## ARM / Apple Silicon について

本章のコードサンプルはx86-64を主軸に説明しますが、RustのPortable SIMDは
ARMを含むクロスプラットフォーム対応です。第4節でそちらも扱います。

ARMのintrinsicsは `std::arch::aarch64` 以下に定義されています（`neon` 等）。

---

## 学習目標

本章を修了すると、以下のことができるようになります。

- SIMDレジスタの構造（レーン・幅・型）を理解する
- `is_x86_feature_detected!` でランタイム機能検出を行う
- `std::arch` の intrinsics を使い、AVX2命令を直接呼び出す
- FMA（Fused Multiply-Add）を使った高精度・高速演算を実装する
- コンパイラの自動ベクトル化を促す最適なイテレータの書き方を理解する
- `-C target-cpu=native` とGodboltでアセンブリ出力を確認する
- nightly の `std::simd`（Portable SIMD）で安全かつポータブルなSIMDコードを書く

---

## 各節の内容

| 節 | タイトル | 主なトピック |
|----|----------|------------|
| [1. SIMDの基礎](./01-simd-basics.md) | スカラー vs SIMD、レーン・幅・型、`cpuid`、`is_x86_feature_detected!` |
| [2. std::arch intrinsics](./02-std-arch.md) | `#[target_feature]`、`_mm256_*` 関数群、FMA、水平加算 |
| [3. 自動ベクトル化](./03-auto-vectorization.md) | コンパイラ条件、イテレータ最適化、`-C target-cpu=native`、アセンブリ確認 |
| [4. Portable SIMD](./04-portable-simd.md) | `std::simd`、`Simd<f32, 8>`、`SimdFloat`、クロスプラットフォーム抽象化 |

---

## 前提知識

- Rustの基本的な文法（所有権・借用・ライフタイム）
- 第1章「メモリ管理」の内容（アライメント等）
- 浮動小数点数の基本的な知識

---

## 参考資料

- [Intel Intrinsics Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html)
- [Rust std::arch ドキュメント](https://doc.rust-lang.org/std/arch/)
- [Godbolt Compiler Explorer](https://godbolt.org/)
- [Rust Portable SIMD Project](https://github.com/rust-lang/portable-simd)
