# Bloom Filter と確率的データ構造

**確率的データ構造**は、正確な答えの代わりに少量のメモリで近似的な答えを返すデータ構造です。
データ量が膨大な場合（数十億件のレコード、数 TB のデータ）に特に有効で、
偽陽性（False Positive）率と引き換えに劇的なメモリ削減を実現します。

---

## Bloom Filter の原理

Bloom Filter はある要素が「集合に含まれていない」ことを確実に判定できます（偽陰性ゼロ）。
ただし「含まれている」という判定には誤りが含まれることがあります（偽陽性）。

```
ビット配列（m = 18 ビット）と k = 3 個のハッシュ関数

初期状態:
 bit: 0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17
      [0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0][0]

"cat" を追加:
  hash1("cat") % 18 = 1
  hash2("cat") % 18 = 5
  hash3("cat") % 18 = 13

 bit: 0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17
      [0][1][0][0][0][1][0][0][0][0][0][0][0][1][0][0][0][0]
          ^           ^                       ^

"dog" を追加:
  hash1("dog") % 18 = 4
  hash2("dog") % 18 = 9
  hash3("dog") % 18 = 14

 bit: 0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17
      [0][1][0][0][1][1][0][0][0][1][0][0][0][1][1][0][0][0]

"cat" を検索:
  bit[1]=1, bit[5]=1, bit[13]=1 → 全て 1 → 「おそらく存在する」 ✓

"fish" を検索:
  hash1("fish") % 18 = 3
  hash2("fish") % 18 = 5  ← 既存の 1（cat が立てた）
  hash3("fish") % 18 = 11
  bit[3]=0 → 1 つでも 0 なら「確実に存在しない」 ✗
```

### 偽陽性率の計算式

```
パラメータ:
  n: 挿入する要素数
  m: ビット配列のサイズ（ビット）
  k: ハッシュ関数の数

偽陽性率 p の近似式:
  p ≈ (1 - e^(-k*n/m))^k

最適なハッシュ関数の数:
  k_opt = (m/n) * ln(2) ≈ 0.693 * (m/n)

最適なビット配列サイズ（目標偽陽性率 p から）:
  m = -n * ln(p) / (ln(2))^2 ≈ -1.4427 * n * log2(p)

例: n=1,000,000, p=0.01 (1%) の場合
  m ≈ 9,585,058 ビット ≈ 1.14 MB
  k ≈ 7 個のハッシュ関数
  （同じデータをハッシュセットで保持すると約 32 MB 必要）
```

---

## Rust での実装例（`BitVec` 使用）

```toml
[dependencies]
bit-vec = "0.8"
```

```rust
use bit_vec::BitVec;
use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};

pub struct BloomFilter {
    bits: BitVec,
    m: usize, // ビット配列のサイズ
    k: usize, // ハッシュ関数の数
}

impl BloomFilter {
    /// 期待する要素数 `n` と偽陽性率 `fp_rate` からパラメータを計算して生成
    pub fn new(n: usize, fp_rate: f64) -> Self {
        let m = Self::optimal_m(n, fp_rate);
        let k = Self::optimal_k(m, n);
        Self {
            bits: BitVec::from_elem(m, false),
            m,
            k,
        }
    }

    /// 最適なビット配列サイズ
    fn optimal_m(n: usize, fp_rate: f64) -> usize {
        let ln2_sq = std::f64::consts::LN_2 * std::f64::consts::LN_2;
        (-(n as f64 * fp_rate.ln()) / ln2_sq).ceil() as usize
    }

    /// 最適なハッシュ関数の数
    fn optimal_k(m: usize, n: usize) -> usize {
        ((m as f64 / n as f64) * std::f64::consts::LN_2).round().max(1.0) as usize
    }

    /// ダブルハッシング: h1 + i*h2 で k 個のインデックスを生成
    fn hash_indices<T: Hash>(&self, item: &T) -> Vec<usize> {
        let mut h1 = DefaultHasher::new();
        item.hash(&mut h1);
        let h1 = h1.finish();

        // 2 つ目のハッシュ: 先頭バイトを XOR してシード変更
        let mut h2 = DefaultHasher::new();
        (h1 ^ 0x9e3779b97f4a7c15).hash(&mut h2); // フィボナッチハッシング定数
        let h2 = h2.finish();

        (0..self.k)
            .map(|i| {
                (h1.wrapping_add((i as u64).wrapping_mul(h2)) as usize) % self.m
            })
            .collect()
    }

    /// 要素を追加
    pub fn insert<T: Hash>(&mut self, item: &T) {
        for idx in self.hash_indices(item) {
            self.bits.set(idx, true);
        }
    }

    /// 要素が含まれているか確認
    /// true: おそらく含まれている（偽陽性の可能性あり）
    /// false: 確実に含まれていない
    pub fn contains<T: Hash>(&self, item: &T) -> bool {
        self.hash_indices(item).iter().all(|&idx| self.bits[idx])
    }

    /// メモリ使用量（バイト）
    pub fn memory_bytes(&self) -> usize {
        (self.m + 7) / 8
    }

    /// 現在の推定偽陽性率
    pub fn estimated_fp_rate(&self, n_inserted: usize) -> f64 {
        let exponent = -(self.k as f64 * n_inserted as f64) / self.m as f64;
        (1.0 - exponent.exp()).powi(self.k as i32)
    }
}

fn main() {
    let mut bf = BloomFilter::new(1_000_000, 0.01);
    println!("m = {} bits ({:.1} KB)", bf.m, bf.memory_bytes() as f64 / 1024.0);
    println!("k = {} ハッシュ関数", bf.k);

    // 要素を追加
    let words = ["apple", "banana", "cherry", "date", "elderberry"];
    for word in &words {
        bf.insert(word);
    }

    // 検索テスト
    println!("\n検索結果:");
    for word in &["apple", "banana", "fig", "grape", "cherry"] {
        println!("  '{}' → {}", word,
            if bf.contains(word) { "おそらく存在" } else { "確実に不在" });
    }

    // 大量挿入後の偽陽性率を計測
    let mut bf2 = BloomFilter::new(100_000, 0.01);
    for i in 0..100_000u64 {
        bf2.insert(&i);
    }

    let mut fp_count = 0;
    let test_count = 10_000;
    for i in 100_000u64..100_000 + test_count {
        if bf2.contains(&i) {
            fp_count += 1; // 実際には存在しないが陽性と判定
        }
    }
    println!("\n実測偽陽性率: {:.2}%", fp_count as f64 / test_count as f64 * 100.0);
    println!("推定偽陽性率: {:.2}%", bf2.estimated_fp_rate(100_000) * 100.0);
}
```

---

## 偽陽性率とメモリ使用量のトレードオフ計算

```
n = 1,000,000 要素の場合のトレードオフ:

 偽陽性率 | ビット/要素 |  メモリ  | ハッシュ数
---------|-----------|---------|----------
  50.0%  |    2.1 b  |  256 KB |     1
  10.0%  |    4.8 b  |  595 KB |     3
   1.0%  |    9.6 b  | 1.14 MB |     7
   0.1%  |   14.4 b  | 1.72 MB |    10
  0.01%  |   19.2 b  | 2.29 MB |    13
  0.001% |   24.0 b  | 2.86 MB |    17

比較: HashSet<u64> での 1,000,000 要素 ≈ 32 MB

→ 偽陽性率 1% で約 28 倍のメモリ削減
```

```rust
/// トレードオフ表を出力するユーティリティ
fn print_tradeoff_table(n: usize) {
    println!("n = {} 要素のトレードオフ:", n);
    println!("{:>10} | {:>10} | {:>10} | {:>8}", "偽陽性率", "bits/要素", "メモリ(KB)", "k");
    println!("{}", "-".repeat(48));

    for &fp_rate in &[0.5, 0.1, 0.01, 0.001, 0.0001] {
        let ln2_sq = std::f64::consts::LN_2 * std::f64::consts::LN_2;
        let m = (-(n as f64 * fp_rate.ln()) / ln2_sq).ceil() as usize;
        let k = ((m as f64 / n as f64) * std::f64::consts::LN_2).round().max(1.0) as usize;
        let bits_per_elem = m as f64 / n as f64;
        let memory_kb = ((m + 7) / 8) as f64 / 1024.0;
        println!("{:>10.1}% | {:>10.1} | {:>10.1} | {:>8}",
            fp_rate * 100.0, bits_per_elem, memory_kb, k);
    }
}
```

---

## `bloomfilter` クレートの使い方

```toml
[dependencies]
bloomfilter = "1"
```

```rust
use bloomfilter::Bloom;

fn main() {
    // 1,000,000 要素を 1% 偽陽性率で管理
    let mut bloom = Bloom::new_for_fp_rate(1_000_000, 0.01);

    // URL の重複チェック（クローラーの例）
    let urls = [
        "https://example.com/page1",
        "https://example.com/page2",
        "https://example.com/page3",
    ];

    for url in &urls {
        if bloom.check(url) {
            println!("スキップ（訪問済みの可能性）: {}", url);
        } else {
            println!("クロール: {}", url);
            bloom.set(url);
        }
    }

    // 再チェック
    println!("\n再チェック:");
    println!("page1 訪問済み: {}", bloom.check("https://example.com/page1")); // true
    println!("page99 訪問済み: {}", bloom.check("https://example.com/page99")); // false
}
```

---

## Count-Min Sketch の概要（頻度推定）

Count-Min Sketch は Bloom Filter を拡張し、要素の「頻度」を近似的に推定します。

```
構造: d 行 × w 列のカウンタ行列

     ハッシュ1  →  列 3
     ハッシュ2  →  列 7
     ハッシュ3  →  列 1

      列:  0  1  2  3  4  5  6  7
行 1:    [ 0  0  0  3  0  0  0  0 ]
行 2:    [ 0  0  0  0  0  0  0  2 ]
行 3:    [ 0  5  0  0  0  0  0  0 ]

"apple" の頻度推定 = min(3, 2, 5) = 2
（実際より多く見積もることがあるが、少なく見積もることはない）
```

```rust
use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};

pub struct CountMinSketch {
    table: Vec<Vec<u64>>,
    depth: usize, // ハッシュ関数の数（行数）
    width: usize, // カウンタの数（列数）
}

impl CountMinSketch {
    pub fn new(width: usize, depth: usize) -> Self {
        Self {
            table: vec![vec![0u64; width]; depth],
            depth,
            width,
        }
    }

    fn hash<T: Hash>(&self, item: &T, seed: u64) -> usize {
        let mut h = DefaultHasher::new();
        seed.hash(&mut h);
        item.hash(&mut h);
        h.finish() as usize % self.width
    }

    pub fn add<T: Hash>(&mut self, item: &T, count: u64) {
        for i in 0..self.depth {
            let col = self.hash(item, i as u64);
            self.table[i][col] = self.table[i][col].saturating_add(count);
        }
    }

    pub fn estimate<T: Hash>(&self, item: &T) -> u64 {
        (0..self.depth)
            .map(|i| {
                let col = self.hash(item, i as u64);
                self.table[i][col]
            })
            .min()
            .unwrap_or(0)
    }
}

fn main() {
    let mut cms = CountMinSketch::new(1024, 5);

    // ログエントリの頻度を記録
    let events = ["login", "logout", "login", "purchase", "login", "login", "logout"];
    for event in &events {
        cms.add(event, 1);
    }

    println!("login 推定頻度: {}", cms.estimate(&"login"));   // ~4
    println!("logout 推定頻度: {}", cms.estimate(&"logout")); // ~2
    println!("error 推定頻度: {}", cms.estimate(&"error"));   // ~0
}
```

---

## HyperLogLog の概要（カーディナリティ推定）

HyperLogLog は「ユニーク要素数（カーディナリティ）」を少量のメモリで推定します。

```
原理（簡略版）:

1. 各要素をハッシュ → ビット列
2. ビット列の先頭から連続するゼロの数を記録
   例: hash("apple") = 0001_0101... → 先頭ゼロ数 = 3
3. 最大の先頭ゼロ数 M を記録
4. カーディナリティ推定 ≈ 2^M

精度向上: レジスタを 2^b 個に分割して平均を取る

メモリ効率:
  10億ユニーク要素 → 約 1.5 KB（誤差 ±2%）
  HashSet での正確な計算 → 約 8 GB
```

---

## ユースケース

### キャッシュ（Redis の Bloom Filter）

```
CDN のキャッシュ決定:
  リクエスト → Bloom Filter チェック
    ├─ 不在(確実) → バックエンドへ転送（キャッシュ不要と判断）
    └─ 存在(推定) → キャッシュから取得を試みる

→ キャッシュ汚染（1 回だけアクセスされたデータがキャッシュを占有する問題）を防止
```

### 重複 URL 検出（Web クローラー）

```rust
use bloomfilter::Bloom;

struct Crawler {
    visited: Bloom<String>,
    queue: std::collections::VecDeque<String>,
}

impl Crawler {
    fn new() -> Self {
        Self {
            visited: Bloom::new_for_fp_rate(100_000_000, 0.001), // 1億 URL, 0.1% FP
            queue: Default::default(),
        }
    }

    fn enqueue(&mut self, url: String) {
        if !self.visited.check(&url) {
            self.visited.set(&url);
            self.queue.push_back(url);
        }
    }
}
```

### DB インデックス（BigTable / Cassandra）

```
LSM ツリーの読み取り最適化:
  SSTables (ディスク):
    [SSTable 1] [SSTable 2] [SSTable 3] ...

  各 SSTable に Bloom Filter を持たせる
    キー "user_123" を検索:
      Filter 1: 不在 → スキップ（ディスクアクセスなし）
      Filter 2: 不在 → スキップ
      Filter 3: 存在の可能性 → 実際に SSTable を読む

→ 存在しないキーのディスクアクセスを大幅に削減
```

---

## まとめ

| データ構造 | メモリ | 操作 | 誤り | 用途 |
|---|---|---|---|---|
| Bloom Filter | 極小 | add/contains | 偽陽性 | 集合メンバーシップ |
| Count-Min Sketch | 小 | add/estimate | 過大評価 | 頻度推定 |
| HyperLogLog | 極小 | add/count | ±2% | カーディナリティ推定 |
| HashSet | 大 | add/contains | なし | 正確な集合演算 |
