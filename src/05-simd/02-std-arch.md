# 2. std::arch intrinsics

## `std::arch` とは

`std::arch` はRust標準ライブラリが提供するCPUアーキテクチャ固有のintrinsicsモジュールです。
Cの `<immintrin.h>` に相当し、SIMD命令を **ほぼゼロオーバーヘッドで** 呼び出せます。

```rust
// x86_64のAVX2 intrinsicsをインポート
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

// 32ビットx86用
#[cfg(target_arch = "x86")]
use std::arch::x86::*;
```

> **重要**: intrinsicsはすべて `unsafe` 関数です。
> 対応していないCPUで呼び出すと未定義動作（UB）になります。
> 必ず `is_x86_feature_detected!` またはコンパイル時機能チェックと組み合わせて使います。

---

## `#[target_feature(enable = "...")]` アトリビュート

intrinsicsを含む関数には `#[target_feature]` を付けます。
これにより**その関数のコンパイル時のみ**指定した命令セットを有効化します。

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

/// AVX2命令を使う関数には必ずこのアトリビュートを付ける
#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2")]
unsafe fn my_avx2_function(data: &[f32]) -> f32 {
    // ここではAVX2命令が使える
    // ...
    0.0
}

/// 複数の命令セットを同時に有効化することも可能
#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2,fma")]
unsafe fn my_avx2_fma_function(a: &[f32], b: &[f32], c: &[f32]) -> Vec<f32> {
    // AVX2とFMA両方の命令が使える
    todo!()
}
```

### `target_feature` なしでintrinsicsを呼ぶとどうなるか

```rust
// これはコンパイルエラーになる（target_featureがない）
unsafe fn bad_function() {
    let v = _mm256_setzero_ps(); // エラー: AVX2が有効でない
}
```

---

## 基本的なAVX2 intrinsics

### データのロードとストア

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2")]
unsafe fn load_store_example() {
    let data: [f32; 8] = [1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0];
    let mut output = [0.0f32; 8];

    // 非アライメントロード（uはunaligned）
    let vec = _mm256_loadu_ps(data.as_ptr());

    // ゼロで初期化されたYMMレジスタを作成
    let zeros = _mm256_setzero_ps();

    // すべてのレーンを同じ値で埋める（broadcastとも呼ぶ）
    let twos = _mm256_set1_ps(2.0);

    // 個別の値でYMMレジスタを初期化（注意: 引数は逆順）
    // _mm256_set_ps(lane7, lane6, lane5, lane4, lane3, lane2, lane1, lane0)
    let manual = _mm256_set_ps(8.0, 7.0, 6.0, 5.0, 4.0, 3.0, 2.0, 1.0);

    // ストア（非アライメント）
    _mm256_storeu_ps(output.as_mut_ptr(), vec);

    println!("output: {:?}", output);  // [1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0]
}
```

### f32ベクトルの四則演算

```rust
#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2")]
unsafe fn arithmetic_example() {
    let a = [1.0f32, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0];
    let b = [2.0f32, 2.0, 2.0, 2.0, 2.0, 2.0, 2.0, 2.0];
    let mut result = [0.0f32; 8];

    let va = _mm256_loadu_ps(a.as_ptr());
    let vb = _mm256_loadu_ps(b.as_ptr());

    // 加算: a + b
    let add = _mm256_add_ps(va, vb);
    _mm256_storeu_ps(result.as_mut_ptr(), add);
    println!("add: {:?}", result);  // [3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0, 10.0]

    // 減算: a - b
    let sub = _mm256_sub_ps(va, vb);
    _mm256_storeu_ps(result.as_mut_ptr(), sub);
    println!("sub: {:?}", result);  // [-1.0, 0.0, 1.0, 2.0, 3.0, 4.0, 5.0, 6.0]

    // 乗算: a * b
    let mul = _mm256_mul_ps(va, vb);
    _mm256_storeu_ps(result.as_mut_ptr(), mul);
    println!("mul: {:?}", result);  // [2.0, 4.0, 6.0, 8.0, 10.0, 12.0, 14.0, 16.0]

    // 除算: a / b
    let div = _mm256_div_ps(va, vb);
    _mm256_storeu_ps(result.as_mut_ptr(), div);
    println!("div: {:?}", result);  // [0.5, 1.0, 1.5, 2.0, 2.5, 3.0, 3.5, 4.0]

    // 平方根: sqrt(a)
    let sqrt = _mm256_sqrt_ps(va);
    _mm256_storeu_ps(result.as_mut_ptr(), sqrt);
    println!("sqrt: {:?}", result);  // [1.0, 1.41, 1.73, 2.0, 2.23, 2.44, 2.64, 2.82]
}
```

### 完全なベクトル加算の実装

```rust
use std::time::Instant;

#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

/// スカラー版（ベースライン）
pub fn add_arrays_scalar(a: &[f32], b: &[f32]) -> Vec<f32> {
    a.iter().zip(b.iter()).map(|(x, y)| x + y).collect()
}

/// AVX2 SIMD版
#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2")]
pub unsafe fn add_arrays_avx2(a: &[f32], b: &[f32]) -> Vec<f32> {
    assert_eq!(a.len(), b.len());
    let len = a.len();
    let mut result = vec![0.0f32; len];

    // 8要素ずつ処理できるチャンク数
    let chunks = len / 8;

    for i in 0..chunks {
        let offset = i * 8;
        // ポインタ演算でオフセット位置からロード
        let va = _mm256_loadu_ps(a.as_ptr().add(offset));
        let vb = _mm256_loadu_ps(b.as_ptr().add(offset));
        let vc = _mm256_add_ps(va, vb);
        _mm256_storeu_ps(result.as_mut_ptr().add(offset), vc);
    }

    // 末尾の端数（8で割り切れない部分）をスカラーで処理
    for i in (chunks * 8)..len {
        result[i] = a[i] + b[i];
    }

    result
}

/// 安全なラッパー（ランタイムディスパッチ付き）
pub fn add_arrays(a: &[f32], b: &[f32]) -> Vec<f32> {
    #[cfg(target_arch = "x86_64")]
    if is_x86_feature_detected!("avx2") {
        return unsafe { add_arrays_avx2(a, b) };
    }
    add_arrays_scalar(a, b)
}

fn benchmark_add() {
    const SIZE: usize = 1_000_000;
    let a: Vec<f32> = (0..SIZE).map(|x| x as f32).collect();
    let b: Vec<f32> = (0..SIZE).map(|x| x as f32 * 2.0).collect();

    // スカラー版の計測
    let start = Instant::now();
    for _ in 0..100 {
        let _ = add_arrays_scalar(&a, &b);
    }
    let scalar_time = start.elapsed();

    // SIMD版の計測
    let start = Instant::now();
    for _ in 0..100 {
        let _ = add_arrays(&a, &b);
    }
    let simd_time = start.elapsed();

    println!("スカラー: {:?}", scalar_time);
    println!("SIMD:     {:?}", simd_time);
    println!("速度比:   {:.2}x", scalar_time.as_secs_f64() / simd_time.as_secs_f64());
}
```

---

## FMA（Fused Multiply-Add）

FMAは `a * b + c` を1命令で実行する特殊命令です。
通常の乗算＋加算（2命令）より：
- **高速**（レイテンシ削減）
- **高精度**（中間結果の丸めが1回で済む）

```
通常: temp = a * b   (丸め発生)
      result = temp + c  (丸め発生) → 2回の丸め

FMA:  result = a * b + c  (最後に1回だけ丸め) → より高精度
```

### FMA intrinsicsの種類

| 関数名 | 計算式 |
|--------|--------|
| `_mm256_fmadd_ps` | `a * b + c` |
| `_mm256_fmsub_ps` | `a * b - c` |
| `_mm256_fnmadd_ps` | `-(a * b) + c` |
| `_mm256_fnmsub_ps` | `-(a * b) - c` |
| `_mm256_fmaddsub_ps` | 偶数レーン: `a*b - c`, 奇数レーン: `a*b + c` |

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

/// FMAを使ったドット積（内積）の計算
/// result = sum(a[i] * b[i])
#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2,fma")]
pub unsafe fn dot_product_fma(a: &[f32], b: &[f32]) -> f32 {
    assert_eq!(a.len(), b.len());
    let len = a.len();
    let chunks = len / 8;

    // アキュムレータを0で初期化
    let mut acc = _mm256_setzero_ps();

    for i in 0..chunks {
        let offset = i * 8;
        let va = _mm256_loadu_ps(a.as_ptr().add(offset));
        let vb = _mm256_loadu_ps(b.as_ptr().add(offset));

        // acc = va * vb + acc  （FMAで乗算と累積加算を1命令で）
        acc = _mm256_fmadd_ps(va, vb, acc);
    }

    // 8レーンの合計を求める（水平加算）
    let sum = horizontal_sum_avx(acc);

    // 端数のスカラー処理
    let tail: f32 = a[(chunks * 8)..].iter()
        .zip(b[(chunks * 8)..].iter())
        .map(|(x, y)| x * y)
        .sum();

    sum + tail
}

/// AVXレジスタの8レーンを水平加算して1つのf32に縮約
#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2")]
unsafe fn horizontal_sum_avx(v: __m256) -> f32 {
    // 上位128ビットと下位128ビットを加算
    // [a0,a1,a2,a3,a4,a5,a6,a7] → [a0+a4, a1+a5, a2+a6, a3+a7]
    let hi = _mm256_extractf128_ps(v, 1);  // 上位128bit取り出し
    let lo = _mm256_castps256_ps128(v);     // 下位128bit取り出し
    let sum128 = _mm_add_ps(hi, lo);        // 要素ごとに加算

    // [s0, s1, s2, s3] → [s0+s2, s1+s3, s0+s2, s1+s3]
    let shuf = _mm_movehdup_ps(sum128);
    let sum64 = _mm_add_ps(sum128, shuf);

    // [s01, s23, ...] → s0123
    let shuf2 = _mm_movehl_ps(sum64, sum64);
    let sum32 = _mm_add_ss(sum64, shuf2);

    // スカラー値として取り出す
    _mm_cvtss_f32(sum32)
}
```

### 水平加算の図解

```
初期:   [a0 | a1 | a2 | a3 | a4 | a5 | a6 | a7]  (256bit)

Step1: extractf128でhi/loに分割
  lo: [a0 | a1 | a2 | a3]
  hi: [a4 | a5 | a6 | a7]

Step2: add_ps で要素ごとに加算
  sum128: [a0+a4 | a1+a5 | a2+a6 | a3+a7]

Step3: movehdup_ps
  shuf:   [a1+a5 | a1+a5 | a3+a7 | a3+a7]  (偶数レーンを奇数レーンにコピー)

Step4: add_ps
  sum64:  [a0+a1+a4+a5 | a1+a5+a1+a5 | ...]

Step5: movehl_ps + add_ss
  最終:   a0+a1+a2+a3+a4+a5+a6+a7
```

---

## 実践例：ポリアップリケーション（スカラー vs SIMD 速度比較）

```rust
use std::time::Instant;

#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

/// 配列の各要素にax^2 + bx + c を適用（二次多項式評価）
pub fn poly_eval_scalar(x: &[f32], a: f32, b: f32, c: f32) -> Vec<f32> {
    x.iter().map(|&xi| a * xi * xi + b * xi + c).collect()
}

#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2,fma")]
pub unsafe fn poly_eval_avx2_fma(x: &[f32], a: f32, b: f32, c: f32) -> Vec<f32> {
    let len = x.len();
    let mut result = vec![0.0f32; len];
    let chunks = len / 8;

    let va = _mm256_set1_ps(a);
    let vb = _mm256_set1_ps(b);
    let vc = _mm256_set1_ps(c);

    for i in 0..chunks {
        let offset = i * 8;
        let vx = _mm256_loadu_ps(x.as_ptr().add(offset));

        // ax^2 + bx + c をHorner法で計算: (ax + b)x + c
        // Step1: t1 = a * x + b   (FMA: a*x + b)
        let t1 = _mm256_fmadd_ps(va, vx, vb);
        // Step2: t2 = t1 * x + c  (FMA: (ax+b)*x + c = ax^2 + bx + c)
        let t2 = _mm256_fmadd_ps(t1, vx, vc);

        _mm256_storeu_ps(result.as_mut_ptr().add(offset), t2);
    }

    // 端数処理
    for i in (chunks * 8)..len {
        let xi = x[i];
        result[i] = a * xi * xi + b * xi + c;
    }

    result
}

pub fn benchmark_poly_eval() {
    const SIZE: usize = 4_000_000;
    let x: Vec<f32> = (0..SIZE).map(|i| i as f32 / 1000.0).collect();

    let iterations = 50;

    // スカラー版
    let start = Instant::now();
    for _ in 0..iterations {
        let _ = poly_eval_scalar(&x, 2.5, -1.3, 0.7);
    }
    let scalar_elapsed = start.elapsed();

    // SIMD版（対応CPUのみ）
    #[cfg(target_arch = "x86_64")]
    let simd_elapsed = {
        if is_x86_feature_detected!("avx2") && is_x86_feature_detected!("fma") {
            let start = Instant::now();
            for _ in 0..iterations {
                let _ = unsafe { poly_eval_avx2_fma(&x, 2.5, -1.3, 0.7) };
            }
            Some(start.elapsed())
        } else {
            None
        }
    };

    println!("=== ポリ評価ベンチマーク ({} 要素 × {} 回) ===", SIZE, iterations);
    println!("スカラー: {:>10.2}ms", scalar_elapsed.as_secs_f64() * 1000.0);
    #[cfg(target_arch = "x86_64")]
    if let Some(elapsed) = simd_elapsed {
        println!("AVX2+FMA: {:>10.2}ms", elapsed.as_secs_f64() * 1000.0);
        println!("速度向上: {:.2}x", scalar_elapsed.as_secs_f64() / elapsed.as_secs_f64());
    }
}
```

---

## 整数SIMD演算

浮動小数点以外に整数SIMDも重要です。
AVX2では256ビット幅の整数演算が使えます。

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

/// i32配列の要素ごと乗算（AVX2）
#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2")]
pub unsafe fn multiply_i32_avx2(a: &[i32], b: &[i32]) -> Vec<i32> {
    assert_eq!(a.len(), b.len());
    let len = a.len();
    let mut result = vec![0i32; len];
    let chunks = len / 8;  // __m256i は i32×8

    for i in 0..chunks {
        let offset = i * 8;
        // i32のロード（非アライメント）
        let va = _mm256_loadu_si256(a.as_ptr().add(offset) as *const __m256i);
        let vb = _mm256_loadu_si256(b.as_ptr().add(offset) as *const __m256i);

        // i32×8 の乗算（低32ビットのみ保持）
        let vc = _mm256_mullo_epi32(va, vb);

        _mm256_storeu_si256(result.as_mut_ptr().add(offset) as *mut __m256i, vc);
    }

    // 端数
    for i in (chunks * 8)..len {
        result[i] = a[i] * b[i];
    }

    result
}

/// バイトカウント: 値が閾値以上の要素数を数える
#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2")]
pub unsafe fn count_above_threshold_avx2(data: &[i16], threshold: i16) -> usize {
    let len = data.len();
    let chunks = len / 16;  // __m256i は i16×16
    let mut count = 0usize;

    let vthresh = _mm256_set1_epi16(threshold);

    for i in 0..chunks {
        let offset = i * 16;
        let vd = _mm256_loadu_si256(data.as_ptr().add(offset) as *const __m256i);

        // 閾値より大きいレーンのビットマスク（0xFFFF or 0x0000）
        let cmp = _mm256_cmpgt_epi16(vd, vthresh);

        // movemask: 各レーンの符号ビットを1ビットに圧縮
        // 注意: epi16では movemask_epi8で2ビット/レーンとなるため /2 する
        let mask = _mm256_movemask_epi8(cmp) as u32;
        count += (mask.count_ones() / 2) as usize;
    }

    // 端数処理
    for i in (chunks * 16)..len {
        if data[i] > threshold {
            count += 1;
        }
    }

    count
}
```

---

## AVX-512 入門（対応CPUのみ）

AVX-512はZMMレジスタ（512ビット）を使い、f32×16 を1命令で処理できます。

```rust
// AVX-512はコンパイル時フラグが必要: RUSTFLAGS="-C target-feature=+avx512f"
#[cfg(all(target_arch = "x86_64", target_feature = "avx512f"))]
use std::arch::x86_64::*;

#[cfg(all(target_arch = "x86_64", target_feature = "avx512f"))]
#[target_feature(enable = "avx512f")]
pub unsafe fn add_arrays_avx512(a: &[f32], b: &[f32]) -> Vec<f32> {
    let len = a.len();
    let mut result = vec![0.0f32; len];
    let chunks = len / 16;  // ZMMは f32×16

    for i in 0..chunks {
        let offset = i * 16;
        let va = _mm512_loadu_ps(a.as_ptr().add(offset));
        let vb = _mm512_loadu_ps(b.as_ptr().add(offset));
        let vc = _mm512_add_ps(va, vb);
        _mm512_storeu_ps(result.as_mut_ptr().add(offset), vc);
    }

    for i in (chunks * 16)..len {
        result[i] = a[i] + b[i];
    }

    result
}
```

---

## まとめ

| intrinsic | 役割 |
|-----------|------|
| `_mm256_loadu_ps` | 非アライメントf32×8ロード |
| `_mm256_storeu_ps` | 非アライメントf32×8ストア |
| `_mm256_set1_ps` | 全レーンを同じ値で初期化（broadcast） |
| `_mm256_add_ps` | f32×8加算 |
| `_mm256_mul_ps` | f32×8乗算 |
| `_mm256_fmadd_ps` | FMA: `a*b+c` （要FMAフラグ） |
| `_mm256_extractf128_ps` | 上位/下位128ビット抽出 |
| `_mm256_mullo_epi32` | i32×8乗算（低32ビット） |
| `_mm256_cmpgt_epi16` | i16×16比較 |

- intrinsics関数はすべて `unsafe`
- `#[target_feature(enable = "avx2")]` を必ず付ける
- 端数処理（8で割り切れない部分）をスカラーで補完する
- FMAは精度と速度の両面で有利

---

[前へ: 1. SIMDの基礎](./01-simd-basics.md) | [次へ: 3. 自動ベクトル化](./03-auto-vectorization.md)
