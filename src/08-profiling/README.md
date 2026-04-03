# 第8章: プロファイリングと最適化

## 「計測なき最適化は推測である」

> *"Premature optimization is the root of all evil."*
> — Donald Knuth

Knuth のこの言葉はしばしば「最適化するな」と誤解されますが、文脈を読むと「計測せずに最適化するな」という意味です。
Rust はゼロコスト抽象化を謳いますが、それは「自動的に速い」ではなく「余分なオーバーヘッドを払わない」という意味です。本当の最適化は**計測 → 特定 → 改善 → 再計測**というサイクルで行います。

```
計測 ──→ ボトルネック特定 ──→ コード改善 ──→ 再計測
  ↑                                              │
  └──────────────── 効果確認 ──────────────────┘
```

---

## 本章の学習目標

本章を通じて、以下のスキルを習得します。

| スキル | 対応節 |
|--------|--------|
| マイクロベンチマークの正確な計測 | [01 - criterion](./01-criterion.md) |
| CPU プロファイルとフレームグラフ生成 | [02 - perf & flamegraph](./02-perf-flamegraph.md) |
| コンパイラへのヒントでコード生成を制御 | [03 - compiler hints](./03-compiler-hints.md) |
| キャッシュ効率を意識したデータ構造設計 | [04 - cache optimization](./04-cache-optimization.md) |

---

## 最適化の大原則

### 1. まず正しく、次に速く

最適化の前に、コードが**正しく動作する**ことを確認してください。
最適化によってバグが混入しやすいため、テストカバレッジが重要です。

### 2. プロファイルデータを信用する

人間の直感は外れることが多いです。「遅そうなところ」ではなく、**実際に遅いところ**を直してください。

```
よくある思い込み                実際のボトルネック
─────────────────               ─────────────────
アルゴリズムが遅い    →        キャッシュミスが原因
ループが多い          →        メモリアロケーションが原因
分岐が多い            →        I/O 待ちが原因
```

### 3. 最適化のレベル

最適化にはいくつかのレベルがあり、上位ほど効果が大きいです。

```
レベル1: アルゴリズム・データ構造の選択      ← 最も効果大
レベル2: メモリレイアウトとキャッシュ効率
レベル3: 並列化・非同期処理
レベル4: コンパイラヒントと SIMD
レベル5: インラインアセンブリ               ← 最後の手段
```

### 4. 回帰しないようにする

最適化の効果を計測したら、CI/CD パイプラインにベンチマークを組み込み、**性能回帰を検知**できるようにしてください。

---

## 章の構成

### [01 - criterion によるベンチマーク](./01-criterion.md)

`criterion` クレートを使ったマイクロベンチマークの書き方を学びます。統計的に有意な計測を行い、HTML レポートを生成します。

**キーワード**: `Criterion`, `BenchmarkGroup`, `black_box`, `throughput`, 統計的有意性

---

### [02 - perf & flamegraph によるプロファイリング](./02-perf-flamegraph.md)

Linux の `perf` コマンドと `cargo-flamegraph` を使って CPU プロファイルを取得し、ホットスポットを特定します。macOS での代替手段（Instruments）も紹介します。

**キーワード**: `perf record`, `perf report`, フレームグラフ, ホットスポット, `Instruments`

---

### [03 - コンパイラヒント](./03-compiler-hints.md)

`#[inline]`, `#[cold]`, `#[target_feature]` などのアトリビュートで Rust コンパイラの出力を制御します。LTO（Link-Time Optimization）と PGO（Profile-Guided Optimization）の設定も扱います。

**キーワード**: `#[inline]`, `#[cold]`, `likely/unlikely`, `#[target_feature]`, LTO, PGO

---

### [04 - キャッシュ最適化](./04-cache-optimization.md)

CPU キャッシュ階層を理解し、キャッシュフレンドリーなデータ構造とアクセスパターンを設計します。`cachegrind` でキャッシュミスを計測します。

**キーワード**: L1/L2/L3 キャッシュ, ストライドアクセス, プリフェッチ, SoA vs AoS, メモリプール

---

## 前提知識

- Rust の所有権・借用の基本理解
- `Cargo.toml` の依存関係追加方法
- 基本的なコマンドライン操作

第5章（SIMD）・第6章（インラインアセンブリ）を読んでいると、より深く理解できます。

---

## 参考リソース

- [The Rust Performance Book](https://nnethercote.github.io/perf-book/)
- [criterion.rs ドキュメント](https://bheisler.github.io/criterion.rs/book/)
- [Flamegraph on GitHub](https://github.com/brendangregg/FlameGraph)
- [cargo-flamegraph](https://github.com/flamegraph-rs/flamegraph)
