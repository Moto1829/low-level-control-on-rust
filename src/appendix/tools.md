# 付録A: 使用ツール一覧

本コンテンツで使用するツールとクレートの一覧です。
カテゴリ別に整理し、インストール方法とバージョン確認コマンドを記載します。

---

## 目次

- [ベンチマーク](#ベンチマーク)
- [プロファイリング](#プロファイリング)
- [SIMD・CPU命令](#simdcpu命令)
- [並列化](#並列化)
- [メモリ管理・アロケータ](#メモリ管理アロケータ)
- [FFI・バインディング生成](#ffiバインディング生成)
- [アセンブリ・低レベル制御](#アセンブリ低レベル制御)
- [システムコール](#システムコール)
- [データ構造](#データ構造)
- [開発支援ツール](#開発支援ツール)

---

## ベンチマーク

### criterion

統計的に正確なマイクロベンチマークフレームワーク。ウォームアップ・外れ値検出・HTMLレポート生成に対応します。

| 項目 | 内容 |
|------|------|
| 章 | 第8章（プロファイリングと最適化） |
| crates.io | [criterion](https://crates.io/crates/criterion) |
| ドキュメント | [docs.rs/criterion](https://docs.rs/criterion) |

**Cargo.toml への追記:**

```toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "my_benchmark"
harness = false
```

**バージョン確認:**

```bash
cargo tree | grep criterion
```

---

## プロファイリング

### perf（Linux）

Linux カーネルに付属するパフォーマンス解析ツール。CPU サイクル・キャッシュミス・分岐予測ミスなどのハードウェアイベントを計測します。

| 項目 | 内容 |
|------|------|
| 章 | 第8章（プロファイリングと最適化） |
| 対応 OS | Linux のみ |

**インストール:**

```bash
# Ubuntu / Debian
sudo apt install linux-perf

# Fedora / RHEL
sudo dnf install perf
```

**バージョン確認:**

```bash
perf --version
```

---

### flamegraph

`perf` や `dtrace` のプロファイルデータからフレームグラフ（炎のような SVG）を生成する Cargo サブコマンドです。

| 項目 | 内容 |
|------|------|
| 章 | 第8章（プロファイリングと最適化） |
| crates.io | [flamegraph](https://crates.io/crates/flamegraph) |

**インストール:**

```bash
cargo install flamegraph
```

**バージョン確認:**

```bash
cargo flamegraph --version
```

**使用例:**

```bash
cargo flamegraph --bin my_binary
```

---

### cargo-asm

コンパイル済みバイナリから特定の関数のアセンブリ出力を表示するサブコマンドです。コンパイラの最適化結果を手軽に確認できます。

| 項目 | 内容 |
|------|------|
| 章 | 第6章（インラインアセンブリ）、第5章（SIMD） |
| crates.io | [cargo-show-asm](https://crates.io/crates/cargo-show-asm) |

**インストール:**

```bash
cargo install cargo-show-asm
```

**使用例:**

```bash
cargo asm --release my_crate::my_function
```

---

### Godbolt Compiler Explorer

ブラウザ上でコードをコンパイルしてアセンブリを確認できるオンラインツールです。インストール不要。

| 項目 | 内容 |
|------|------|
| 章 | 第5章（SIMD）、第6章（インラインアセンブリ） |
| URL | [godbolt.org](https://godbolt.org/) |

---

## SIMD・CPU命令

### std::arch（標準ライブラリ）

Rust 標準ライブラリに含まれる CPU intrinsics（組み込み関数）モジュールです。追加インストール不要で使用できます。

| 項目 | 内容 |
|------|------|
| 章 | 第5章（SIMD・CPU命令） |
| ドキュメント | [doc.rust-lang.org/std/arch](https://doc.rust-lang.org/std/arch/) |
| 対応アーキテクチャ | x86, x86_64, aarch64, arm, wasm32 |

**主なモジュール:**

```rust
use std::arch::x86_64::*;   // x86-64 intrinsics (AVX, SSE 等)
use std::arch::aarch64::*;  // ARM NEON intrinsics
```

**ランタイム機能検出:**

```rust
if is_x86_feature_detected!("avx2") {
    // AVX2 対応 CPU の場合のみ実行
}
```

---

### std::simd / portable-simd（nightly）

アーキテクチャ非依存の安全な SIMD 抽象化レイヤーです。nightly Rust が必要です。

| 項目 | 内容 |
|------|------|
| 章 | 第5章（SIMD・CPU命令） §4 |
| ツールチェーン | nightly 必須 |
| GitHub | [rust-lang/portable-simd](https://github.com/rust-lang/portable-simd) |

**有効化方法（`main.rs` または `lib.rs` 冒頭）:**

```rust
#![feature(portable_simd)]
use std::simd::*;
```

**nightly ツールチェーンの切り替え:**

```bash
rustup override set nightly
# または rust-toolchain.toml で指定
```

**rust-toolchain.toml（推奨）:**

```toml
[toolchain]
channel = "nightly"
```

---

### Intel Intrinsics Guide

Intel 公式の intrinsics リファレンス。ブラウザで検索・フィルタリングが可能なオンラインドキュメントです。

| 項目 | 内容 |
|------|------|
| URL | [intel.com/content/www/us/en/docs/intrinsics-guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html) |

---

## 並列化

### rayon

データ並列処理のためのクレートです。`iter()` を `par_iter()` に変えるだけでマルチスレッド並列化できます。

| 項目 | 内容 |
|------|------|
| 章 | 第4章（並列化） §2 |
| crates.io | [rayon](https://crates.io/crates/rayon) |
| ドキュメント | [docs.rs/rayon](https://docs.rs/rayon) |

**Cargo.toml への追記:**

```toml
[dependencies]
rayon = "1.10"
```

**バージョン確認:**

```bash
cargo tree | grep rayon
```

---

### crossbeam

ロックフリーデータ構造・スレッド間通信チャネル・スコープ付きスレッドなど、高度な並列プリミティブを提供します。

| 項目 | 内容 |
|------|------|
| 章 | 第4章（並列化） §4、第7章（データ構造） §5 |
| crates.io | [crossbeam](https://crates.io/crates/crossbeam) |
| ドキュメント | [docs.rs/crossbeam](https://docs.rs/crossbeam) |

**Cargo.toml への追記:**

```toml
[dependencies]
crossbeam = "0.8"
```

---

### tokio

非同期ランタイム。`async/await` の実行基盤として広く使われます。

| 項目 | 内容 |
|------|------|
| 章 | 第4章（並列化） §5 |
| crates.io | [tokio](https://crates.io/crates/tokio) |
| ドキュメント | [docs.rs/tokio](https://docs.rs/tokio) |

**Cargo.toml への追記:**

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

---

### dashmap

スレッドセーフな並列ハッシュマップです。`RwLock<HashMap>` の代替として使用できます。

| 項目 | 内容 |
|------|------|
| 章 | 第4章（並列化） |
| crates.io | [dashmap](https://crates.io/crates/dashmap) |

**Cargo.toml への追記:**

```toml
[dependencies]
dashmap = "6.0"
```

---

### atomic_float

標準ライブラリにない `AtomicF32` / `AtomicF64` を提供する軽量クレートです。

| 項目 | 内容 |
|------|------|
| 章 | 第4章（並列化） §3 |
| crates.io | [atomic_float](https://crates.io/crates/atomic_float) |

**Cargo.toml への追記:**

```toml
[dependencies]
atomic_float = "1.0"
```

---

## メモリ管理・アロケータ

### memoffset

`offset_of!` マクロを提供します。構造体フィールドのバイトオフセットをコンパイル時に取得できます。

| 項目 | 内容 |
|------|------|
| 章 | 第1章（メモリ管理） §2 |
| crates.io | [memoffset](https://crates.io/crates/memoffset) |
| ドキュメント | [docs.rs/memoffset](https://docs.rs/memoffset) |

**Cargo.toml への追記:**

```toml
[dependencies]
memoffset = "0.9"
```

---

### jemallocator / tikv-jemallocator

Facebook 製の高性能メモリアロケータ jemalloc を Rust から使用するためのクレートです。

| 項目 | 内容 |
|------|------|
| 章 | 第1章（メモリ管理） §3 |
| crates.io | [tikv-jemallocator](https://crates.io/crates/tikv-jemallocator) |

**Cargo.toml への追記:**

```toml
[dependencies]
tikv-jemallocator = "0.5"
```

**使用例:**

```rust
#[global_allocator]
static GLOBAL: tikv_jemallocator::Jemalloc = tikv_jemallocator::Jemalloc;
```

---

### mimalloc

Microsoft 製の高速メモリアロケータ。スレッドローカルキャッシュによる高速割り当てが特徴です。

| 項目 | 内容 |
|------|------|
| 章 | 第1章（メモリ管理） §3 |
| crates.io | [mimalloc](https://crates.io/crates/mimalloc) |

**Cargo.toml への追記:**

```toml
[dependencies]
mimalloc = { version = "0.1", default-features = false }
```

---

### bumpalo

アリーナアロケータ（バンプアロケータ）の実装です。大量の短命なオブジェクトを高速に確保できます。

| 項目 | 内容 |
|------|------|
| 章 | 第7章（データ構造） §4 |
| crates.io | [bumpalo](https://crates.io/crates/bumpalo) |
| ドキュメント | [docs.rs/bumpalo](https://docs.rs/bumpalo) |

**Cargo.toml への追記:**

```toml
[dependencies]
bumpalo = "3"
```

---

### smallvec

スタック上に小さな配列を持ち、容量超過時のみヒープへフォールバックする `SmallVec<[T; N]>` を提供します。

| 項目 | 内容 |
|------|------|
| 章 | 第7章（データ構造） §2 |
| crates.io | [smallvec](https://crates.io/crates/smallvec) |

**Cargo.toml への追記:**

```toml
[dependencies]
smallvec = "1"
```

---

### arrayvec

ヒープを一切使わない固定長ベクタ `ArrayVec<T, N>` を提供します。

| 項目 | 内容 |
|------|------|
| 章 | 第7章（データ構造） §2 |
| crates.io | [arrayvec](https://crates.io/crates/arrayvec) |

**Cargo.toml への追記:**

```toml
[dependencies]
arrayvec = "0.7"
```

---

## FFI・バインディング生成

### bindgen

C/C++ ヘッダファイルから Rust の FFI バインディングを自動生成するツールです。

| 項目 | 内容 |
|------|------|
| 章 | 第2章（unsafe コード） §4 |
| crates.io | [bindgen](https://crates.io/crates/bindgen) |
| ドキュメント | [rust-lang.github.io/rust-bindgen](https://rust-lang.github.io/rust-bindgen/) |

**インストール（CLI ツールとして）:**

```bash
cargo install bindgen-cli
```

**Cargo.toml への追記（ビルドスクリプト用）:**

```toml
[build-dependencies]
bindgen = "0.69"
```

**バージョン確認:**

```bash
bindgen --version
```

---

### libc

Linux・macOS・Windows の C 標準ライブラリや POSIX API の型定義・関数宣言を提供します。

| 項目 | 内容 |
|------|------|
| 章 | 第3章（OS システムコール） §1 |
| crates.io | [libc](https://crates.io/crates/libc) |
| ドキュメント | [docs.rs/libc](https://docs.rs/libc) |

**Cargo.toml への追記:**

```toml
[dependencies]
libc = "0.2"
```

---

### io-uring（Linux 限定）

Linux の非同期 I/O インターフェース `io_uring` を Rust から使用するためのクレートです。

| 項目 | 内容 |
|------|------|
| 章 | 第3章（OS システムコール） §3 |
| crates.io | [io-uring](https://crates.io/crates/io-uring) |
| 対応 OS | Linux カーネル 5.1 以降 |

**Cargo.toml への追記（Linux 限定）:**

```toml
[target.'cfg(target_os = "linux")'.dependencies]
io-uring = "0.6"
```

---

## アセンブリ・低レベル制御

### std::arch::asm!（標準ライブラリ）

Rust 1.59（stable）から使用可能なインラインアセンブリマクロです。追加インストール不要です。

| 項目 | 内容 |
|------|------|
| 章 | 第6章（インラインアセンブリ） |
| 安定版 | Rust 1.59 以降 |
| ドキュメント | [doc.rust-lang.org/reference/inline-assembly](https://doc.rust-lang.org/reference/inline-assembly.html) |

**使用例:**

```rust
use std::arch::asm;

unsafe {
    asm!("nop");
}
```

---

## データ構造

### bevy（ECS の参照実装）

データ指向設計（DoD）・ECS（Entity Component System）の実装例として参照します。ゲームエンジンですが、ECS の設計思想を学ぶ教材として扱います。

| 項目 | 内容 |
|------|------|
| 章 | 第7章（データ構造） §7 |
| crates.io | [bevy](https://crates.io/crates/bevy) |
| 公式サイト | [bevyengine.org](https://bevyengine.org/) |

---

## 開発支援ツール

### mdBook

このコンテンツのビルドに使用しているドキュメントフレームワークです。

| 項目 | 内容 |
|------|------|
| crates.io | [mdbook](https://crates.io/crates/mdbook) |
| ドキュメント | [rust-lang.github.io/mdBook](https://rust-lang.github.io/mdBook/) |

**インストール:**

```bash
cargo install mdbook
```

**バージョン確認:**

```bash
mdbook --version
```

**ビルド・プレビュー:**

```bash
mdbook build   # docs/ ディレクトリに出力
mdbook serve   # ローカルサーバーで確認（http://localhost:3000）
```

---

### rustup

Rust ツールチェーンのインストール・管理ツールです。

**インストール:**

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

**よく使うコマンド:**

```bash
rustup update                        # ツールチェーンを最新版に更新
rustup toolchain install nightly     # nightly をインストール
rustup override set nightly          # カレントディレクトリを nightly に設定
rustup target add x86_64-unknown-linux-gnu  # クロスコンパイルターゲット追加
rustup component add rust-src        # ソースコード（IDE 補完に必要）
rustup component add llvm-tools-preview  # LLVM ツール群
```

**バージョン確認:**

```bash
rustup --version
rustc --version
cargo --version
```

---

## Cargo.toml まとめ

本コンテンツのすべての章で使用するクレートをまとめた `Cargo.toml` の例です。

```toml
[package]
name = "low-level-control-on-rust"
version = "0.1.0"
edition = "2021"

[dependencies]
# メモリ管理（第1章）
memoffset = "0.9"
tikv-jemallocator = "0.5"
mimalloc = { version = "0.1", default-features = false }

# システムコール（第3章）
libc = "0.2"

# 並列化（第4章）
rayon = "1.10"
crossbeam = "0.8"
dashmap = "6.0"
tokio = { version = "1", features = ["full"] }
atomic_float = "1.0"

# データ構造（第7章）
smallvec = "1"
arrayvec = "0.7"
bumpalo = "3"

# Linux 限定
[target.'cfg(target_os = "linux")'.dependencies]
io-uring = "0.6"

[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[build-dependencies]
bindgen = "0.69"
```

---

## バージョン情報の確認

```bash
# インストール済みの全クレートのバージョンツリーを表示
cargo tree

# 特定クレートのバージョンのみ確認
cargo tree | grep criterion
cargo tree | grep rayon

# 最新バージョンを確認（cargo-outdated が必要）
cargo install cargo-outdated
cargo outdated

# 脆弱性チェック（cargo-audit が必要）
cargo install cargo-audit
cargo audit
```
