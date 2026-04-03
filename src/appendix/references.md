# 付録B: 参考文献・リソース

本コンテンツの各章で参照・引用しているドキュメント、書籍、ブログ記事、動画、論文の一覧です。
カテゴリ別に整理しています。

---

## 目次

- [公式ドキュメント](#公式ドキュメント)
- [書籍](#書籍)
- [ブログ・記事](#ブログ記事)
- [動画・カンファレンス](#動画カンファレンス)
- [論文・仕様書](#論文仕様書)
- [オンラインツール](#オンラインツール)
- [章別リソース早見表](#章別リソース早見表)

---

## 公式ドキュメント

### Rust 言語・標準ライブラリ

| タイトル | URL | 説明 |
|---------|-----|------|
| The Rust Programming Language（通称 "The Book"） | [doc.rust-lang.org/book](https://doc.rust-lang.org/book/) | Rust 公式入門書。所有権・借用・ライフタイムの基礎 |
| The Rust Reference | [doc.rust-lang.org/reference](https://doc.rust-lang.org/reference/) | 言語仕様の公式リファレンス。型レイアウト・インラインアセンブリ構文の権威ある情報源 |
| **The Rustonomicon**（通称 "Nomicon"） | [doc.rust-lang.org/nomicon](https://doc.rust-lang.org/nomicon/) | **unsafe Rust の公式ガイド**。生ポインタ・FFI・未定義動作・安全な抽象化のすべてを網羅 |
| Rust Standard Library ドキュメント | [doc.rust-lang.org/std](https://doc.rust-lang.org/std/) | `std::mem`・`std::alloc`・`std::arch`・`std::sync` 等の API リファレンス |
| Rust By Example | [doc.rust-lang.org/rust-by-example](https://doc.rust-lang.org/rust-by-example/) | コード例中心の学習リソース。インラインアセンブリの例も含む |
| The Rust Performance Book | [nnethercote.github.io/perf-book](https://nnethercote.github.io/perf-book/) | Rust プログラムのパフォーマンス改善テクニック集 |
| Rust Async Book | [rust-lang.github.io/async-book](https://rust-lang.github.io/async-book/) | 非同期プログラミングの公式ガイド |

---

### unsafe・メモリモデル

| タイトル | URL | 説明 |
|---------|-----|------|
| Unsafe Code Guidelines Reference | [rust-lang.github.io/unsafe-code-guidelines](https://rust-lang.github.io/unsafe-code-guidelines/) | unsafe コードの正しい書き方に関する公式ガイドライン。未定義動作の定義を詳述 |
| Miri（未定義動作検出器） | [github.com/rust-lang/miri](https://github.com/rust-lang/miri) | unsafe コードの未定義動作をランタイム検出する Rust インタープリタ |
| Stacked Borrows | [plv.mpi-sws.org/rustbelt/stacked-borrows](https://plv.mpi-sws.org/rustbelt/stacked-borrows/) | Rust のメモリモデルに関する学術的な形式化。Miri の基礎理論 |

---

### SIMD・CPU アーキテクチャ

| タイトル | URL | 説明 |
|---------|-----|------|
| Rust std::arch ドキュメント | [doc.rust-lang.org/std/arch](https://doc.rust-lang.org/std/arch/) | x86_64・aarch64 等の CPU intrinsics の公式 API リファレンス |
| Rust Portable SIMD Project | [github.com/rust-lang/portable-simd](https://github.com/rust-lang/portable-simd) | nightly の `std::simd`（portable_simd）の開発リポジトリ・ドキュメント |
| Intel Intrinsics Guide | [intel.com/.../intrinsics-guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html) | Intel SIMD intrinsics の公式リファレンス。SSE/AVX/AVX-512 全命令を網羅 |
| ARM Intrinsics | [developer.arm.com/architectures/instruction-sets/intrinsics](https://developer.arm.com/architectures/instruction-sets/intrinsics) | ARM NEON 等の intrinsics リファレンス |

---

### OS・システムプログラミング

| タイトル | URL | 説明 |
|---------|-----|------|
| Linux man-pages | [man7.org/linux/man-pages](https://man7.org/linux/man-pages/index.html) | Linux システムコールの公式マニュアル |
| Linux Kernel Documentation | [kernel.org/doc/html/latest](https://www.kernel.org/doc/html/latest/) | `io_uring` 等のカーネル機能の公式ドキュメント |
| POSIX 仕様 | [pubs.opengroup.org/onlinepubs/9699919799](https://pubs.opengroup.org/onlinepubs/9699919799/) | POSIX API（`open`, `mmap`, `socket` 等）の公式仕様 |

---

### インラインアセンブリ

| タイトル | URL | 説明 |
|---------|-----|------|
| Rust Reference: Inline Assembly | [doc.rust-lang.org/reference/inline-assembly](https://doc.rust-lang.org/reference/inline-assembly.html) | `asm!` マクロの公式言語仕様。オペランド・制約・options の全構文を定義 |
| Intel 64 and IA-32 Software Developer Manuals | [intel.com/.../intel-sdm](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) | x86-64 命令セットの完全なリファレンス（全4巻） |
| System V AMD64 ABI | [refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf](https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf) | x86-64 Linux の呼び出し規約（ABI）の公式仕様 |

---

## 書籍

### Rust 関連

| タイトル | 著者 | 説明 | 関連章 |
|---------|------|------|--------|
| **Programming Rust** (2nd ed.) | Jim Blandy, Jason Orendorff, Leonora Tindall | Rust の型システム・所有権・unsafe を深く掘り下げた実践書 | 全章 |
| **Rust for Rustaceans** | Jon Gjengset | 中〜上級者向け。型システム・unsafe・非同期・マクロの内部構造 | 全章 |
| **Systems Programming with Rust** | — | OS・ドライバ・組み込みなど低レベルプログラミングへの応用 | 第3・6章 |

---

### システムプログラミング・CPU アーキテクチャ

| タイトル | 著者 | 説明 | 関連章 |
|---------|------|------|--------|
| **Computer Systems: A Programmer's Perspective** (CS:APP, 3rd ed.) | Bryant, O'Hallaron | スタック・ヒープ・キャッシュ・アセンブリ・システムコールの基礎 | 第1・3・6・8章 |
| **Computer Architecture: A Quantitative Approach** (6th ed.) | Hennessy, Patterson | CPU パイプライン・キャッシュ・SIMD の定量的分析 | 第5・7・8章 |
| **The Art of Writing Efficient Programs** | Fedor Pikus | プロファイリング・キャッシュ効率・並列化・SIMD の実践的解説 | 第4・5・7・8章 |
| **Data-Oriented Design** | Richard Fabian | DoD・SoA/AoS・ECS の考え方を体系的に解説した無料書籍 | 第7章 |

---

### 並列・並行プログラミング

| タイトル | 著者 | 説明 | 関連章 |
|---------|------|------|--------|
| **The Art of Multiprocessor Programming** (2nd ed.) | Herlihy, Shavit | ロックフリーアルゴリズム・メモリモデルの理論的バイブル | 第4章 |
| **Is Parallel Programming Hard, And, If So, What Can You Do About It?** | Paul McKenney | Linux カーネルのメモリモデル・RCU アルゴリズムを詳述。無料 PDF あり | 第4章 |

---

## ブログ・記事

### パフォーマンス・最適化

| タイトル | 著者・媒体 | 説明 | 関連章 |
|---------|----------|------|--------|
| [What Every Programmer Should Know About Memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf) | Ulrich Drepper | CPU キャッシュの仕組みとメモリアクセスパターンの最適化を詳説した名論文 | 第7・8章 |
| [Rust Performance Pitfalls](https://llogiq.github.io/2017/06/01/perf-pitfalls.html) | llogiq | Rust コードの一般的なパフォーマンス落とし穴 | 第8章 |
| [How to Write Fast Rust Code](https://deterministic.space/high-performance-rust.html) | Pascal Hertleif | Rust のパフォーマンス最適化テクニック集 | 第8章 |
| [The Unofficial Rust Performance Guide](https://www.brendangregg.com/) | Brendan Gregg | `perf` や `flamegraph` を使ったプロファイリング手法 | 第8章 |

---

### unsafe・メモリ管理

| タイトル | 著者・媒体 | 説明 | 関連章 |
|---------|----------|------|--------|
| [Learn Rust With Entirely Too Many Linked Lists](https://rust-unofficial.github.io/too-many-lists/) | Aria Beingessner | 連結リストの実装を通じて unsafe・生ポインタ・借用を深く学ぶ | 第2章 |
| [Rust Allocator Designs](https://os.phil-opp.com/allocator-designs/) | Philipp Oppermann | アリーナ・バディ・スラブアロケータの設計と Rust 実装 | 第1章 |
| [Writing an OS in Rust](https://os.phil-opp.com/) | Philipp Oppermann | Rust でゼロから OS を書くシリーズ。unsafe・ページング・割り込みを実践 | 第1・2・6章 |

---

### SIMD・CPU 命令

| タイトル | 著者・媒体 | 説明 | 関連章 |
|---------|----------|------|--------|
| [Fast SIMD String Processing in Rust](https://www.nickwilcox.com/blog/arm_vs_x86_simd/) | Nick Wilcox | x86 と ARM の SIMD 実装比較 | 第5章 |
| [Auto-vectorization in Rust](https://www.nickwilcox.com/blog/autovec/) | Nick Wilcox | コンパイラの自動ベクトル化を促すコードパターン | 第5章 |
| [SIMD for faster computing](https://pythonspeed.com/articles/simd-in-rust/) | Itamar Turner-Trauring | Rust での SIMD 入門 | 第5章 |

---

### 並列・並行プログラミング

| タイトル | 著者・媒体 | 説明 | 関連章 |
|---------|----------|------|--------|
| [Fearless Concurrency](https://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html) | Aaron Turon (Rust Blog) | Rust の並行性モデルの設計思想を解説した公式ブログ | 第4章 |
| [Crossbeam: Support for Parallelism and Low-Level Concurrency in Rust](https://smallcultfollowing.com/babysteps/blog/2015/12/18/rayon-data-parallelism-in-rust/) | Niko Matsakis | Rayon の設計思想と並列イテレータの仕組み | 第4章 |
| [Lock-free Rust: Crossbeam in 2019](https://stjepang.github.io/2019/01/29/lock-free-rust-crossbeam-in-2019.html) | Stjepan Glavina | crossbeam のロックフリーデータ構造の解説 | 第4・7章 |

---

### アセンブリ

| タイトル | 著者・媒体 | 説明 | 関連章 |
|---------|----------|------|--------|
| [Rust Inline Assembly Stabilization](https://blog.rust-lang.org/2021/05/17/Stabilizing-Inline-Assembly.html) | Rust Blog | `asm!` マクロ安定化の背景と設計の解説 | 第6章 |
| [A Guide to Porting C and C++ code to Rust](https://locka99.gitbooks.io/a-guide-to-porting-c-to-rust/content/) | — | C から Rust への移植時の FFI・unsafe の扱い方 | 第2章 |

---

## 動画・カンファレンス

### RustConf / RustFest

| タイトル | 発表者 | 年 | 説明 | 関連章 |
|---------|--------|-----|------|--------|
| [Performance Matters](https://www.youtube.com/watch?v=r-TLq427agc) | Emery Berger | 2019 | プロファイリングの落とし穴と統計的に正しい計測手法 | 第8章 |
| [Rust and SIMD](https://www.youtube.com/watch?v=4Hju37nKPOk) | — | — | Rust での SIMD プログラミング入門 | 第5章 |
| [Writing Unsafe Rust](https://www.youtube.com/watch?v=eTBCE5J2uYA) | Jon Gjengset | — | unsafe Rust の正しい書き方とメモリモデル | 第2章 |

---

### YouTube チャンネル

| チャンネル | 説明 | 関連章 |
|-----------|------|--------|
| [Jon Gjengset](https://www.youtube.com/c/JonGjengset) | 上級 Rust トピックのライブコーディング。unsafe・非同期・標準ライブラリの内部実装 | 全章 |
| [fasterthanlime](https://www.youtube.com/c/fasterthanlime) | Rust の深い仕組みを分かりやすく解説。メモリ・FFI など | 第1・2章 |

---

## 論文・仕様書

| タイトル | 著者 | 説明 | 関連章 |
|---------|------|------|--------|
| [RustBelt: Securing the Foundations of the Rust Programming Language](https://plv.mpi-sws.org/rustbelt/popl18/) | Jung et al. (POPL 2018) | Rust の型安全性を形式的に証明した学術論文 | 第2章 |
| [Stacked Borrows: An Aliasing Model for Rust](https://plv.mpi-sws.org/rustbelt/stacked-borrows/) | Ralf Jung et al. | Rust の借用規則のメモリモデルとしての形式化 | 第2章 |
| [Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms](https://www.cs.rochester.edu/~scott/papers/1996_PODC_queues.pdf) | Michael & Scott (1996) | Michael-Scott ロックフリーキューの原著論文 | 第7章 |
| [A Pragmatic Implementation of Non-Blocking Linked Lists](https://www.cl.cam.ac.uk/research/srg/netos/papers/2001-caslists.pdf) | Harris (2001) | ロックフリー連結リストの実装技法 | 第7章 |
| [io_uring: The Definitive Guide](https://unixism.net/loti/) | Lord of the io_uring | Linux の io_uring インターフェースの包括的解説 | 第3章 |
| [Intel 64 and IA-32 Architectures Optimization Reference Manual](https://www.intel.com/content/www/us/en/develop/documentation/cpp-compiler-developer-guide-and-reference/top.html) | Intel | CPU の最適化ガイドライン。レイテンシ・スループット・マイクロアーキテクチャ詳細 | 第5・6・8章 |
| [Agner Fog's Optimization Manuals](https://www.agner.org/optimize/) | Agner Fog | x86 CPU の命令レイテンシ・マイクロアーキテクチャ解析の決定版 | 第5・6・8章 |

---

## オンラインツール

| ツール | URL | 説明 | 関連章 |
|--------|-----|------|--------|
| Godbolt Compiler Explorer | [godbolt.org](https://godbolt.org/) | コンパイラの出力アセンブリをブラウザ上でリアルタイム確認 | 第5・6章 |
| Rust Playground | [play.rust-lang.org](https://play.rust-lang.org/) | ブラウザ上で Rust コードを実行。stable/beta/nightly を切り替え可能 | 全章 |
| crates.io | [crates.io](https://crates.io/) | Rust クレートの公式レジストリ | 全章 |
| docs.rs | [docs.rs](https://docs.rs/) | crates.io 公開クレートのドキュメントを自動生成・ホスト | 全章 |
| Lib.rs | [lib.rs](https://lib.rs/) | crates.io の非公式検索・カテゴリ別インデックス | 全章 |

---

## 章別リソース早見表

各章で特に重要なリソースを章ごとにまとめます。

### 第1章: メモリ管理

- [The Rust Reference: Type Layout](https://doc.rust-lang.org/reference/type-layout.html) — 型のメモリレイアウト仕様
- [The Rustonomicon: Data Representation in Rust](https://doc.rust-lang.org/nomicon/repr-rust.html) — `repr` の詳細
- [Rust std::alloc](https://doc.rust-lang.org/std/alloc/index.html) — アロケータ API
- [Writing an OS in Rust: Allocator Designs](https://os.phil-opp.com/allocator-designs/) — バンプ・スラブアロケータ実装

### 第2章: unsafe コード

- [The Rustonomicon](https://doc.rust-lang.org/nomicon/) — unsafe Rust の公式ガイド（**必読**）
- [Rust Reference: Unsafety](https://doc.rust-lang.org/reference/unsafety.html) — unsafe の言語仕様
- [Unsafe Code Guidelines Reference](https://rust-lang.github.io/unsafe-code-guidelines/) — 実践的なガイドライン
- [Learn Rust With Entirely Too Many Linked Lists](https://rust-unofficial.github.io/too-many-lists/) — 実践的な unsafe 演習
- [rust-bindgen ドキュメント](https://rust-lang.github.io/rust-bindgen/) — FFI バインディング生成ツール

### 第3章: OS システムコール

- [Linux man-pages](https://man7.org/linux/man-pages/index.html) — システムコールの仕様
- [io_uring: The Definitive Guide](https://unixism.net/loti/) — io_uring の解説
- [libc docs.rs](https://docs.rs/libc) — libc クレート API

### 第4章: 並列化

- [Fearless Concurrency（Rust Blog）](https://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html) — Rust の並行性設計思想
- [Rayon docs.rs](https://docs.rs/rayon) — データ並列クレート
- [crossbeam docs.rs](https://docs.rs/crossbeam) — ロックフリークレート
- [The Art of Multiprocessor Programming](https://www.elsevier.com/books/the-art-of-multiprocessor-programming/herlihy/978-0-12-415950-1) — ロックフリーアルゴリズムの理論

### 第5章: SIMD・CPU命令

- [Intel Intrinsics Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html) — SIMD 命令リファレンス（**必須ブックマーク**）
- [Rust std::arch](https://doc.rust-lang.org/std/arch/) — Rust SIMD API
- [Rust portable-simd](https://github.com/rust-lang/portable-simd) — portable SIMD
- [Godbolt](https://godbolt.org/) — 自動ベクトル化の確認
- [Agner Fog's Optimization Manuals](https://www.agner.org/optimize/) — CPU 命令のレイテンシ・スループット

### 第6章: インラインアセンブリ

- [Rust Reference: Inline Assembly](https://doc.rust-lang.org/reference/inline-assembly.html) — `asm!` マクロ仕様（**必読**）
- [Intel SDM](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) — x86-64 命令セット完全リファレンス
- [System V AMD64 ABI](https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf) — 呼び出し規約
- [Godbolt](https://godbolt.org/) — アセンブリ出力確認

### 第7章: 高性能データ構造

- [What Every Programmer Should Know About Memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf) — キャッシュ最適化の名論文（**必読**）
- [Data-Oriented Design](https://www.dataorienteddesign.com/dodbook/) — DoD・SoA/AoS の理論
- [The Rust Performance Book](https://nnethercote.github.io/perf-book/) — Rust パフォーマンス改善
- [crossbeam docs.rs](https://docs.rs/crossbeam) — ロックフリーキュー
- [bumpalo docs.rs](https://docs.rs/bumpalo) — アリーナアロケータ

### 第8章: プロファイリングと最適化

- [The Rust Performance Book](https://nnethercote.github.io/perf-book/) — Rust パフォーマンス改善の実践ガイド
- [criterion docs.rs](https://docs.rs/criterion) — ベンチマークフレームワーク
- [Brendan Gregg's Blog](https://www.brendangregg.com/) — `perf` / flamegraph の権威
- [Agner Fog's Optimization Manuals](https://www.agner.org/optimize/) — CPU 最適化の詳細資料
- [Intel Optimization Reference Manual](https://www.intel.com/content/www/us/en/develop/documentation/cpp-compiler-developer-guide-and-reference/top.html) — Intel CPU の最適化ガイド
