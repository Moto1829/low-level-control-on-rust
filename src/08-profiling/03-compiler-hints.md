# コンパイラヒントによる最適化

Rustには、コンパイラの最適化を誘導するための属性（attribute）やヒントが用意されている。
これらを適切に使うことで、`unsafe`なインラインアセンブリを書くことなく、
コンパイラに対して最適化の判断材料を与えられる。

---

## `#[inline]` / `#[inline(always)]` / `#[inline(never)]` の使い分け

### `#[inline]`

コンパイラに「インライン化を検討してほしい」というヒントを与える。
コンパイラは状況に応じてインライン化するかどうかを判断する。

```rust
/// 小さな関数にはデフォルトで #[inline] を付けることが多い
#[inline]
pub fn add(a: f32, b: f32) -> f32 {
    a + b
}

/// 別クレートから呼ばれる関数をインライン化するには必須
/// （クレート境界を越えるにはLTOか #[inline] が必要）
#[inline]
pub fn clamp(value: f32, min: f32, max: f32) -> f32 {
    value.max(min).min(max)
}
```

### `#[inline(always)]`

コンパイラに「必ずインライン化する」よう強制する。
ループ内部の小さなヘルパー関数など、確実にインライン化したい場合に使う。

```rust
/// ホットパスにある小さな演算：必ずインライン化する
#[inline(always)]
fn fast_sigmoid(x: f32) -> f32 {
    1.0 / (1.0 + (-x).exp())
}

/// ループ内から呼ばれる変換関数
#[inline(always)]
fn rgb_to_luminance(r: u8, g: u8, b: u8) -> u8 {
    ((77u32 * r as u32 + 150 * g as u32 + 29 * b as u32) >> 8) as u8
}

pub fn apply_sigmoid(data: &mut [f32]) {
    for x in data.iter_mut() {
        *x = fast_sigmoid(*x); // インライン化されてループが最適化される
    }
}
```

> **注意**: `#[inline(always)]` を多用するとコードサイズが増大し、
> 命令キャッシュのミスが増えて逆に遅くなることがある。
> ベンチマークで効果を確認してから使うこと。

### `#[inline(never)]`

コンパイラに「インライン化しない」よう強制する。
主にデバッグ・プロファイリング・コードサイズ最適化で使う。

```rust
/// プロファイラがこの関数を個別に計測できるよう、インライン化を禁止
#[inline(never)]
pub fn process_chunk(data: &[f32]) -> f32 {
    data.iter().sum()
}

/// エラーハンドリング関数：コールドパスなのでコードサイズ節約のためインライン化しない
#[inline(never)]
#[cold]
fn handle_error(msg: &str) -> ! {
    panic!("致命的なエラー: {}", msg);
}

/// コードサイズを重視する場合（組み込み向けなど）
#[inline(never)]
pub fn large_function(data: &[u8]) -> u32 {
    // 大きな処理：インライン化するとコードサイズが膨れ上がる
    data.iter().fold(0u32, |acc, &x| {
        acc.wrapping_add(x as u32).wrapping_mul(1664525).wrapping_add(1013904223)
    })
}
```

### インライン化の判断基準

```
関数のサイズ・呼び出し頻度・コードサイズの要件によって使い分ける:

小さくてホットパスにある         → #[inline(always)]
小さくてクレート境界を越える     → #[inline]
大きくてコールドパスにある       → #[inline(never)] または デフォルト
プロファイリングしたい           → #[inline(never)]
組み込みでコードサイズを節約     → #[inline(never)]
```

---

## `#[cold]` でコールドパスをマーク

`#[cold]` はその関数が「めったに呼ばれない」ことをコンパイラに伝える。
コンパイラは `#[cold]` な関数の呼び出し元において、
コールドパスを分岐予測のデフォルトではないほうに配置し、
ホットパスのコードレイアウトを改善する。

```rust
/// パニック処理はめったに呼ばれない → #[cold]
#[cold]
#[inline(never)]
fn panic_out_of_bounds(index: usize, len: usize) -> ! {
    panic!("インデックス {} が範囲外です（長さ {}）", index, len);
}

/// 独自の境界チェック（コールドパスを分離してホットパスを最適化）
#[inline(always)]
pub fn safe_get(data: &[f32], index: usize) -> f32 {
    if index >= data.len() {
        panic_out_of_bounds(index, data.len()); // コールドパス
    }
    data[index] // ホットパス
}

/// エラー処理関数全般に適用できる
#[cold]
fn allocation_failed(size: usize) -> ! {
    panic!("メモリ割り当て失敗: {} バイト", size);
}

/// Result を返す関数でのコールドパス
#[cold]
fn build_error_message(code: u32) -> String {
    format!("エラーコード {}: {}", code, describe_error(code))
}

fn describe_error(code: u32) -> &'static str {
    match code {
        1 => "入力が無効です",
        2 => "リソースが不足しています",
        _ => "不明なエラー",
    }
}

pub fn process(code: u32) -> Result<(), String> {
    if code == 0 {
        Ok(()) // ホットパス: 成功時
    } else {
        Err(build_error_message(code)) // コールドパス: エラー時
    }
}
```

---

## `std::hint::likely` / `unlikely`（nightly）と安定版の代替

分岐予測ヒントをコンパイラに与える機能。
nightly では `std::hint::likely` / `std::hint::unlikely` が提供される（不安定機能）。

### nightly での使い方

```rust
// nightly 限定（2024年時点では unstable）
#![feature(likely_unlikely)]

use std::hint::{likely, unlikely};

pub fn process_data(data: &[u32]) -> u64 {
    let mut sum = 0u64;
    for &x in data {
        if likely(x < 1000) {
            // ほとんどのケースはここを通る（ホットパス）
            sum += x as u64;
        } else {
            // まれなケース（コールドパス）
            sum += (x as u64).pow(2);
        }
    }
    sum
}
```

### 安定版での代替（`#[cold]` による間接的な指示）

```rust
// stable で使えるアプローチ

/// unlikely な分岐を #[cold] 関数に分離する
#[cold]
#[inline(never)]
fn handle_large_value(x: u32) -> u64 {
    (x as u64).pow(2)
}

pub fn process_data_stable(data: &[u32]) -> u64 {
    let mut sum = 0u64;
    for &x in data {
        if x < 1000 {
            sum += x as u64;                 // 通常パス
        } else {
            sum += handle_large_value(x);    // コールドパス（別関数に分離）
        }
    }
    sum
}

/// intrinsics::unlikely の代替マクロ（unsafe）
///
/// コンパイラ固有の機能を使う方法（安定版でも動作するが、保証なし）
macro_rules! likely {
    ($expr:expr) => {
        // LLVM の expect intrinsic を使う（stable では直接使えないため unsafe）
        // 実際には std::hint::black_box の逆の目的で使う
        $expr
    };
}

macro_rules! unlikely {
    ($expr:expr) => {
        $expr
    };
}
```

---

## `#[target_feature(enable = "avx2")]` によるCPU特化コード

特定のCPU機能（AVX2・SSE4.2など）を持つ場合のみ有効になるコードを記述できる。
実行時にCPU機能を検出して切り替えることで、古いCPUとの互換性を保ちつつ
新しいCPUでは高速なコードを実行できる。

```rust
use std::arch::x86_64::*;

/// AVX2 を使った高速なドット積
///
/// # Safety
/// この関数を呼び出す前に、CPU が AVX2 をサポートしているか確認すること。
#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2")]
pub unsafe fn dot_product_avx2(a: &[f32], b: &[f32]) -> f32 {
    assert_eq!(a.len(), b.len());
    let n = a.len();
    let mut sum = _mm256_setzero_ps();

    let chunks = n / 8;
    for i in 0..chunks {
        let va = _mm256_loadu_ps(a.as_ptr().add(i * 8));
        let vb = _mm256_loadu_ps(b.as_ptr().add(i * 8));
        // FMA: sum += va * vb
        sum = _mm256_fmadd_ps(va, vb, sum);
    }

    // 水平加算で8要素を合計
    let sum128_hi = _mm256_extractf128_ps(sum, 1);
    let sum128_lo = _mm256_castps256_ps128(sum);
    let sum128 = _mm_add_ps(sum128_hi, sum128_lo);
    let sum64 = _mm_add_ps(sum128, _mm_movehl_ps(sum128, sum128));
    let sum32 = _mm_add_ss(sum64, _mm_shuffle_ps(sum64, sum64, 1));
    let simd_part = _mm_cvtss_f32(sum32);

    // 残り要素をスカラーで処理
    let remainder: f32 = a[chunks * 8..]
        .iter()
        .zip(b[chunks * 8..].iter())
        .map(|(x, y)| x * y)
        .sum();

    simd_part + remainder
}

/// SSE4.2 を使った文字列比較（PCMPESTRI 命令）
#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "sse4.2")]
pub unsafe fn count_char_sse42(haystack: &[u8], needle: u8) -> usize {
    // sse4.2 の文字列命令を使った高速検索（簡易版）
    haystack.iter().filter(|&&c| c == needle).count()
}

/// 実行時に CPU 機能を検出して最適な実装を選択
pub fn dot_product_dispatch(a: &[f32], b: &[f32]) -> f32 {
    #[cfg(target_arch = "x86_64")]
    {
        if is_x86_feature_detected!("avx2") && is_x86_feature_detected!("fma") {
            return unsafe { dot_product_avx2(a, b) };
        }
    }
    // フォールバック: スカラー実装
    a.iter().zip(b.iter()).map(|(x, y)| x * y).sum()
}

/// multiversion マクロパターン（手動実装）
///
/// 実行時ディスパッチのオーバーヘッドを最小化するために
/// 関数ポインタをキャッシュする。
use std::sync::OnceLock;

static DOT_PRODUCT_FN: OnceLock<fn(&[f32], &[f32]) -> f32> = OnceLock::new();

pub fn dot_product_cached(a: &[f32], b: &[f32]) -> f32 {
    let f = DOT_PRODUCT_FN.get_or_init(|| {
        #[cfg(target_arch = "x86_64")]
        if is_x86_feature_detected!("avx2") {
            return |a: &[f32], b: &[f32]| -> f32 {
                unsafe { dot_product_avx2(a, b) }
            };
        }
        |a: &[f32], b: &[f32]| -> f32 {
            a.iter().zip(b.iter()).map(|(x, y)| x * y).sum()
        }
    });
    f(a, b)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_dot_product_dispatch() {
        let a = vec![1.0f32, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0];
        let b = vec![8.0f32, 7.0, 6.0, 5.0, 4.0, 3.0, 2.0, 1.0];
        let expected: f32 = a.iter().zip(b.iter()).map(|(x, y)| x * y).sum();
        let result = dot_product_dispatch(&a, &b);
        assert!((result - expected).abs() < 1e-4,
            "dispatch結果が不正: {} vs {}", result, expected);
    }
}
```

---

## LTO（Link-Time Optimization）の設定

LTO はコンパイル時ではなくリンク時に最適化を行う。
クレート境界を越えたインライン化・定数伝播・デッドコード除去が可能になる。

### `Cargo.toml` の設定

```toml
[profile.release]
# LTO の設定
# false  : LTO なし（デフォルト）
# "thin" : thin LTO（高速、ほどほどの最適化）
# true   : fat LTO（遅いが最大の最適化）
lto = "thin"

# コードジェネレーションユニット数（1にするとLTOの効果が高まる）
# デフォルトは16（並列コンパイルのため）
codegen-units = 1

# 最適化レベル（0-3 または "s"（サイズ）・"z"（最小サイズ））
opt-level = 3

# シンボルのストリップ（バイナリサイズ削減）
strip = "symbols"

# パニック時の動作（"abort" にするとunwindコードが不要になる）
panic = "abort"
```

### 設定の比較

| 設定 | ビルド時間 | 実行速度 | バイナリサイズ | 用途 |
|------|-----------|----------|--------------|------|
| `lto = false` | 最速 | 基準 | 大 | 開発時 |
| `lto = "thin"` | 中程度 | +5〜15% | 中 | 本番リリース |
| `lto = true` | 遅い | +10〜30% | 小 | 最高性能が必要な場合 |

### プロファイル別の設定例

```toml
[profile.release]
lto = "thin"
codegen-units = 1
opt-level = 3
panic = "abort"

# 最大性能プロファイル（ビルド時間を犠牲にして最大最適化）
[profile.perf]
inherits = "release"
lto = true
codegen-units = 1

# サイズ優先プロファイル（組み込み向け）
[profile.size]
inherits = "release"
opt-level = "z"
lto = true
codegen-units = 1
strip = "symbols"
panic = "abort"
```

```bash
# 最大性能プロファイルでビルド
cargo build --profile perf

# サイズ優先プロファイルでビルド
cargo build --profile size

# thin LTO のリリースビルド
cargo build --release
```

---

## PGO（Profile-Guided Optimization）の適用手順

PGO はプログラムを実際に実行して収集したプロファイルデータを使って、
コンパイラが最適化の判断を改善する手法。
ホットパスの推定精度が上がり、インライン化・分岐予測・コードレイアウトが改善される。

### PGO の手順（rustcのビルトインPGO）

```bash
# ステップ 1: プロファイル収集用にビルド
# LLVM のインストゥルメンテーションを有効化
mkdir -p /tmp/pgo-data

RUSTFLAGS="-Cprofile-generate=/tmp/pgo-data" \
    cargo build --release

# ステップ 2: 代表的なワークロードを実行してプロファイルを収集
# （実際の使用パターンに近い入力で実行することが重要）
./target/release/my_program --benchmark
./target/release/my_program --run-typical-workload

# ステップ 3: プロファイルデータをマージ
# llvm-profdata は rustup でインストールしたツールチェインに含まれる
llvm-profdata merge -output=/tmp/pgo-data/merged.profdata /tmp/pgo-data

# ステップ 4: マージしたプロファイルを使ってビルド
RUSTFLAGS="-Cprofile-use=/tmp/pgo-data/merged.profdata -Cllvm-args=-pgo-warn-missing-function" \
    cargo build --release
```

### `cargo-pgo` クレートを使った簡略化

```bash
# cargo-pgo をインストール
cargo install cargo-pgo

# プロファイル収集ビルド
cargo pgo build

# プロファイルを収集しながら実行
cargo pgo run -- --your-args

# PGO 最適化ビルド
cargo pgo optimize build
```

### PGO の効果が大きいケース

```rust
/// 1. 仮想ディスパッチ（dyn Trait）
///    実際の実装が分かればインライン化できる
trait Processor {
    fn process(&self, data: &[f32]) -> f32;
}

struct FastProcessor;
struct SlowProcessor;

impl Processor for FastProcessor {
    fn process(&self, data: &[f32]) -> f32 {
        data.iter().sum()
    }
}

impl Processor for SlowProcessor {
    fn process(&self, data: &[f32]) -> f32 {
        data.iter().map(|&x| x * x).sum::<f32>().sqrt()
    }
}

/// PGO により、FastProcessor が 99% 呼ばれると分かれば
/// その呼び出しをインライン化できる
pub fn run_processor(proc: &dyn Processor, data: &[f32]) -> f32 {
    proc.process(data)
}

/// 2. 条件分岐
///    どちらのパスが頻繁に取られるかをプロファイルで判断
pub fn categorize(value: f32) -> &'static str {
    if value > 0.0 {
        "正"    // 90% のケース
    } else if value < 0.0 {
        "負"    // 9% のケース
    } else {
        "ゼロ"  // 1% のケース
    }
}
```

### PGO とLTOの組み合わせ

```toml
[profile.release]
lto = "thin"        # LTO と PGO を組み合わせると最大効果
codegen-units = 1
opt-level = 3
```

```bash
# LTO + PGO の組み合わせ
RUSTFLAGS="-Cprofile-use=/tmp/pgo-data/merged.profdata" \
    cargo build --release
# Cargo.toml の lto = "thin" と組み合わさって動作する
```

---

## 効果測定：コンパイラヒントの前後比較

ベンチマークで各最適化の効果を測定するコード例:

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn bench_inline(c: &mut Criterion) {
    let data: Vec<f32> = (0..10000).map(|x| x as f32).collect();

    c.bench_function("with_inline_always", |b| {
        b.iter(|| {
            apply_sigmoid(black_box(&mut data.clone()))
        })
    });
}

#[inline(always)]
fn fast_sigmoid_inlined(x: f32) -> f32 {
    1.0 / (1.0 + (-x).exp())
}

fn apply_sigmoid(data: &mut [f32]) {
    for x in data.iter_mut() {
        *x = fast_sigmoid_inlined(*x);
    }
}

criterion_group!(benches, bench_inline);
criterion_main!(benches);
```

```bash
# ベンチマーク実行
cargo bench

# LTO あり・なしの比較
cargo bench                                  # lto = false
CARGO_PROFILE_RELEASE_LTO=thin cargo bench  # thin LTO
```

---

## まとめ

| ヒント | 効果 | 使いどころ |
|--------|------|-----------|
| `#[inline]` | クレート境界を越えたインライン化を許可 | 小さなユーティリティ関数 |
| `#[inline(always)]` | 強制インライン化 | ループ内の小さな計算 |
| `#[inline(never)]` | インライン化を禁止 | プロファイリング・コードサイズ節約 |
| `#[cold]` | コールドパスのレイアウト最適化 | エラー処理・パニック関数 |
| `likely`/`unlikely` | 分岐予測ヒント | nightly限定・頻度が偏った分岐 |
| `#[target_feature]` | CPU特化命令の有効化 | SIMD・暗号化など |
| `lto = "thin"` | クレート間の最適化 | 本番リリースビルド |
| PGO | プロファイルに基づく最適化 | 本番ワークロードが明確な場合 |

これらのヒントは「コンパイラへのお願い」であり、最終的な最適化判断はコンパイラが行う。
必ずベンチマークで効果を測定し、逆効果にならないか確認すること。
