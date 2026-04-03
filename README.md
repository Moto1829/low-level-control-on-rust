# Rustによる低レベル制御 — High Performance Rust

Rustを用いて**さらに高速なプログラム**を実現するための実践的学習コンテンツです。
低レベル制御・OSシステムコール・unsafeコード・並列化・CPU命令など、パフォーマンスに直結するトピックを詳細に解説します。

## このコンテンツで学べること

| カテゴリ | 内容 |
|---------|------|
| メモリ管理 | スタック/ヒープ、アロケータ、メモリレイアウト最適化 |
| unsafeコード | 生ポインタ操作、FFI、unsafeの安全な使い方 |
| OSシステムコール | libc、直接syscall、ファイル/プロセス/ソケット制御 |
| 並列化 | スレッド、Rayon、チャネル、ロックフリーデータ構造 |
| CPU命令・SIMD | std::arch、AVX2/SSE、自動ベクトル化 |
| インラインアセンブリ | `asm!`マクロ、レジスタ操作、最適化 |
| 高性能データ構造 | キャッシュ効率・ロックフリー・アリーナ・データ指向設計 |
| プロファイリング | perf、flamegraph、ベンチマーク手法 |

## 対象読者

- Rustの基本文法（所有権・借用・ライフタイム）を理解している方
- パフォーマンスチューニングに興味がある方
- OS・アーキテクチャレベルでRustの動作を理解したい方

## コンテンツ構成

```
src/
├── SUMMARY.md                    # mdBook目次
├── introduction.md               # はじめに
│
├── 01-memory/                    # 第1章: メモリ管理
│   ├── README.md
│   ├── 01-stack-heap.md          # スタックとヒープ
│   ├── 02-memory-layout.md       # メモリレイアウトと構造体の最適化
│   ├── 03-allocator.md           # グローバルアロケータとカスタムアロケータ
│   └── 04-box-raw-parts.md       # Box/Vec の内部構造とunsafe操作
│
├── 02-unsafe/                    # 第2章: unsafeコード
│   ├── README.md
│   ├── 01-raw-pointers.md        # 生ポインタ (*const T / *mut T)
│   ├── 02-unsafe-functions.md    # unsafe関数とunsafeブロック
│   ├── 03-safe-abstraction.md    # unsafeを安全にラップする設計
│   └── 04-ffi.md                 # FFI — CライブラリをRustから呼ぶ
│
├── 03-syscall/                   # 第3章: OSシステムコール
│   ├── README.md
│   ├── 01-libc.md                # libcクレートの使い方
│   ├── 02-direct-syscall.md      # syscallマクロで直接カーネルを呼ぶ
│   ├── 03-file-io.md             # ファイルI/O の低レベル制御
│   ├── 04-process.md             # プロセス生成・シグナル処理
│   └── 05-socket.md              # ソケットプログラミング
│
├── 04-concurrency/               # 第4章: 並列化
│   ├── README.md
│   ├── 01-threads.md             # std::thread とスレッド管理
│   ├── 02-rayon.md               # Rayonによるデータ並列
│   ├── 03-atomics.md             # Atomics とメモリオーダリング
│   ├── 04-lock-free.md           # ロックフリーデータ構造
│   └── 05-async-perf.md          # 非同期処理のパフォーマンス
│
├── 05-simd/                      # 第5章: SIMD・CPU命令
│   ├── README.md
│   ├── 01-simd-basics.md         # SIMDとは何か
│   ├── 02-std-arch.md            # std::arch — 組み込みSIMD命令
│   ├── 03-auto-vectorization.md  # 自動ベクトル化を促す書き方
│   └── 04-portable-simd.md       # portable_simd (nightly)
│
├── 06-asm/                       # 第6章: インラインアセンブリ
│   ├── README.md
│   ├── 01-asm-macro.md           # asm!マクロの基本構文
│   ├── 02-registers.md           # レジスタの指定と制約
│   └── 03-asm-optimization.md    # アセンブリによる局所最適化
│
├── 07-data-structures/           # 第7章: 高性能データ構造
│   ├── README.md
│   ├── 01-cache-friendly.md      # データ構造とキャッシュ効率
│   ├── 02-vec-variants.md        # Vec・SmallVec・配列の使い分け
│   ├── 03-ring-buffer.md         # リングバッファ（循環キュー）
│   ├── 04-arena-allocator.md     # アリーナアロケータとスラブアロケータ
│   ├── 05-lock-free-queue.md     # ロックフリーキューとスタック
│   ├── 06-bloom-filter.md        # Bloom Filter と確率的データ構造
│   └── 07-data-oriented-design.md # データ指向設計 (DoD) — SoA vs AoS
│
├── 08-profiling/                 # 第8章: プロファイリングと最適化
│   ├── README.md
│   ├── 01-criterion.md           # Criterionによるベンチマーク
│   ├── 02-perf-flamegraph.md     # perf + flamegraph
│   ├── 03-compiler-hints.md      # #[inline]・likely/unlikely・cold
│   └── 04-cache-optimization.md  # キャッシュ効率の最適化
│
└── appendix/                     # 付録
    ├── tools.md                  # 使用ツール一覧
    └── references.md             # 参考文献・リソース
```

## 使い方

### ローカルでmdBookを読む

```bash
# mdBookのインストール
cargo install mdbook

# ローカルサーバーで確認
mdbook serve

# ブラウザで http://localhost:3000 を開く
```

### GitHub Pagesで公開

このリポジトリはGitHub Actionsで自動的にGitHub Pagesへデプロイされます。

## サンプルコードの実行

各章のサンプルコードは `examples/` ディレクトリにあります。

```bash
# 例: SIMDサンプルの実行
cargo run --example simd_sum

# ベンチマークの実行
cargo bench
```

## ライセンス

MIT License
