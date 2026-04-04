# Summary

[はじめに](introduction.md)

---

# 第1章: メモリ管理

- [概要](01-memory/README.md)
  - [スタックとヒープ](01-memory/01-stack-heap.md)
  - [メモリレイアウトと構造体の最適化](01-memory/02-memory-layout.md)
  - [グローバルアロケータとカスタムアロケータ](01-memory/03-allocator.md)
  - [Box/Vec の内部構造とunsafe操作](01-memory/04-box-raw-parts.md)
  - [mmap・共有メモリ・スタック計測](01-memory/05-mmap-shared-memory.md)

# 第2章: unsafeコード

- [概要](02-unsafe/README.md)
  - [生ポインタ (*const T / *mut T)](02-unsafe/01-raw-pointers.md)
  - [unsafe関数とunsafeブロック](02-unsafe/02-unsafe-functions.md)
  - [unsafeを安全にラップする設計](02-unsafe/03-safe-abstraction.md)
  - [FFI — CライブラリをRustから呼ぶ](02-unsafe/04-ffi.md)
  - [Pin・自己参照構造体・PhantomData](02-unsafe/05-pin-phantom.md)

# 第3章: OSシステムコール

- [概要](03-syscall/README.md)
  - [libcクレートの使い方](03-syscall/01-libc.md)
  - [syscallマクロで直接カーネルを呼ぶ](03-syscall/02-direct-syscall.md)
  - [ファイルI/O の低レベル制御](03-syscall/03-file-io.md)
  - [プロセス生成・シグナル処理](03-syscall/04-process.md)
  - [ソケットプログラミング](03-syscall/05-socket.md)

# 第4章: 並列化

- [概要](04-concurrency/README.md)
  - [std::thread とスレッド管理](04-concurrency/01-threads.md)
  - [Rayonによるデータ並列](04-concurrency/02-rayon.md)
  - [Atomics とメモリオーダリング](04-concurrency/03-atomics.md)
  - [ロックフリーデータ構造](04-concurrency/04-lock-free.md)
  - [非同期処理のパフォーマンス](04-concurrency/05-async-perf.md)
  - [Work-Stealing と NUMA 対応](04-concurrency/06-work-stealing-numa.md)

# 第5章: SIMD・CPU命令

- [概要](05-simd/README.md)
  - [SIMDとは何か](05-simd/01-simd-basics.md)
  - [std::arch — 組み込みSIMD命令](05-simd/02-std-arch.md)
  - [自動ベクトル化を促す書き方](05-simd/03-auto-vectorization.md)
  - [portable_simd (nightly)](05-simd/04-portable-simd.md)
  - [SIMD実践 — 行列積・base64・文字列処理](05-simd/05-simd-advanced.md)

# 第6章: インラインアセンブリ

- [概要](06-asm/README.md)
  - [asm!マクロの基本構文](06-asm/01-asm-macro.md)
  - [レジスタの指定と制約](06-asm/02-registers.md)
  - [アセンブリによる局所最適化](06-asm/03-asm-optimization.md)

# 第7章: 高性能データ構造

- [概要](07-data-structures/README.md)
  - [データ構造とキャッシュ効率](07-data-structures/01-cache-friendly.md)
  - [Vec・SmallVec・配列の使い分け](07-data-structures/02-vec-variants.md)
  - [リングバッファ（循環キュー）](07-data-structures/03-ring-buffer.md)
  - [アリーナアロケータとスラブアロケータ](07-data-structures/04-arena-allocator.md)
  - [ロックフリーキューとスタック](07-data-structures/05-lock-free-queue.md)
  - [Bloom Filter と確率的データ構造](07-data-structures/06-bloom-filter.md)
  - [データ指向設計 (DoD) — SoA vs AoS](07-data-structures/07-data-oriented-design.md)
  - [B-Tree と Radix Tree の実装](07-data-structures/08-btree-radix.md)

# 第8章: プロファイリングと最適化

- [概要](08-profiling/README.md)
  - [Criterionによるベンチマーク](08-profiling/01-criterion.md)
  - [perf + flamegraph](08-profiling/02-perf-flamegraph.md)
  - [コンパイラヒント (#[inline]・likely/unlikely)](08-profiling/03-compiler-hints.md)
  - [キャッシュ効率の最適化](08-profiling/04-cache-optimization.md)

# 第9章: Rustでのランタイム実装

- [概要](09-runtime/README.md)
  - [スレッドプールの自作](09-runtime/01-thread-pool.md)
  - [タスクスケジューラの設計と実装](09-runtime/02-task-scheduler.md)
  - [Futureとwaker — 非同期ランタイムの仕組み](09-runtime/03-future-waker.md)
  - [グリーンスレッド（コルーチン）の実装](09-runtime/04-green-threads.md)
  - [ミニ非同期ランタイムを作る](09-runtime/05-mini-async-runtime.md)

---

# 付録

- [使用ツール一覧](appendix/tools.md)
- [参考文献・リソース](appendix/references.md)
