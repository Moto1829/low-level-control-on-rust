# B-Tree と Radix Tree の実装

データベースのインデックスやネットワークルーティングなど、**大量データの高速検索**が求められる場面では、汎用のハッシュマップや線形リストでは性能が不十分になります。本章では、そのような要件に応える2つの木構造データ構造 — **B-Tree** と **Radix Tree（Trie）** — をRustで実装し、その内部動作を詳しく解説します。

---

## B-Tree

### B-Tree とは

B-Tree は**平衡多分木（balanced multi-way tree）**です。二分探索木（BST）の「各ノードが子を2つまでしか持てない」という制約を取り除き、**1ノードが複数のキーと複数の子ノードを持てる**ように拡張した構造です。

```
二分探索木（BST）の例:          B-Tree（次数 t=2）の例:
         50                         [20 | 50]
        /  \                       /    |    \
       20   70              [10|15] [30|40] [60|70]
      /  \   \
     10  30  60
```

B-Tree が強力な理由は**ディスクI/Oの最小化**にあります。HDDやSSDからの読み取りは1ブロック（通常4KB〜16KB）単位で行われます。ノードにキーを詰め込むことで、1回のI/Oで多くのキーを読み込め、木の高さを低く保てます。

### B-Tree の特性（次数 t）

次数 `t`（最小次数、minimum degree）を定めると、以下の制約が生まれます。

```
次数 t = 3 の B-Tree の制約:

┌──────────────────────────────────────────────────────────┐
│ 各ノード（ルートを除く）                                 │
│   最小キー数:  t - 1 = 2                                │
│   最大キー数: 2t - 1 = 5                                │
│   最小子ノード数:  t     = 3                             │
│   最大子ノード数: 2t     = 6                             │
├──────────────────────────────────────────────────────────┤
│ ルートノード                                             │
│   最小キー数:  1（木が空でない場合）                    │
│   最大キー数: 2t - 1 = 5                                │
└──────────────────────────────────────────────────────────┘

ノード構造の例（t=3、最大5キー・6子）:
┌────────────────────────────────────────────────────┐
│ keys:     [ 10 | 20 | 30 | 40 | 50 ]              │
│ children: [c0 | c1 | c2 | c3 | c4 | c5]           │
│                                                    │
│ c0 < 10 < c1 < 20 < c2 < 30 < c3 < 40 < c4 < 50 < c5 │
└────────────────────────────────────────────────────┘
```

### 用途

| 用途 | 代表例 |
|------|--------|
| データベースインデックス | PostgreSQL, MySQL InnoDB, SQLite |
| ファイルシステム | NTFS, HFS+, Btrfs, ext4 |
| Key-Value ストア | LevelDB, RocksDB（LSM-Tree ベースだが B-Tree も採用例あり）|
| Rustの標準ライブラリ | `std::collections::BTreeMap` / `BTreeSet` |

---

### B-Tree の挿入と分割（ASCIIアート）

挿入操作で最も重要なのが**ノード分割（split）**です。ノードが満杯（`2t-1`個のキー）になったとき、中央のキーを親に押し上げ、ノードを2つに分割します。

```
t=2（最大3キー）の B-Tree への挿入の例:

初期状態（空）:
  [空]

10 を挿入:
  [10]

20 を挿入:
  [10 | 20]

30 を挿入（ノード満杯 → ルート分割）:
  挿入前: [10 | 20 | 30]  ← 満杯（2t-1 = 3）
  中央値 20 を新ルートに昇格
         [20]
        /    \
      [10]  [30]

40 を挿入（右の子 [30] へ）:
         [20]
        /    \
      [10]  [30 | 40]

50 を挿入（右の子 [30|40] が満杯 → 分割）:
  挿入前: 右子 [30 | 40 | 50]  ← 満杯
  中央値 40 を親 [20] に押し上げ
         [20 | 40]
        /    |    \
      [10]  [30]  [50]

35 を挿入（[30] へ）:
         [20 | 40]
        /    |    \
      [10] [30|35] [50]
```

---

### Rust による B-Tree の実装

#### 基本的なノード構造

```rust
/// B-Tree のノード
#[derive(Debug)]
pub struct BTreeNode<K, V> {
    /// このノードが保持するキーと値のペア（ソート済み）
    pub keys: Vec<K>,
    pub values: Vec<V>,
    /// 子ノードへのポインタ（葉ノードは空）
    pub children: Vec<Box<BTreeNode<K, V>>>,
    /// 葉ノードかどうか
    pub is_leaf: bool,
}

impl<K: Ord + Clone, V: Clone> BTreeNode<K, V> {
    /// 新しい葉ノードを作成
    pub fn new_leaf() -> Self {
        BTreeNode {
            keys: Vec::new(),
            values: Vec::new(),
            children: Vec::new(),
            is_leaf: true,
        }
    }

    /// 新しい内部ノードを作成
    pub fn new_internal() -> Self {
        BTreeNode {
            keys: Vec::new(),
            values: Vec::new(),
            children: Vec::new(),
            is_leaf: false,
        }
    }

    /// ノードが満杯かどうか（次数 t に対して 2t-1 個のキー）
    pub fn is_full(&self, t: usize) -> bool {
        self.keys.len() == 2 * t - 1
    }
}
```

#### B-Tree 全体の構造

```rust
/// B-Tree 本体
pub struct BTree<K, V> {
    root: Option<Box<BTreeNode<K, V>>>,
    /// 最小次数 t（各ノードの子数は t〜2t）
    t: usize,
}

impl<K: Ord + Clone + std::fmt::Debug, V: Clone + std::fmt::Debug> BTree<K, V> {
    /// 次数 t の B-Tree を作成（t >= 2 を推奨）
    pub fn new(t: usize) -> Self {
        assert!(t >= 2, "次数 t は 2 以上でなければなりません");
        BTree { root: None, t }
    }

    /// キーを検索してその値への参照を返す
    pub fn search(&self, key: &K) -> Option<&V> {
        match &self.root {
            None => None,
            Some(root) => Self::search_node(root, key),
        }
    }

    fn search_node<'a>(node: &'a BTreeNode<K, V>, key: &K) -> Option<&'a V> {
        // ノード内で key の位置を二分探索
        let pos = node.keys.partition_point(|k| k < key);

        if pos < node.keys.len() && node.keys[pos] == *key {
            // キーがこのノードに存在
            return Some(&node.values[pos]);
        }

        if node.is_leaf {
            // 葉ノードまで来てしまった → 存在しない
            None
        } else {
            // 子ノードを再帰的に検索
            Self::search_node(&node.children[pos], key)
        }
    }
}
```

#### `split_child`（子ノードの分割）

```rust
impl<K: Ord + Clone + std::fmt::Debug, V: Clone + std::fmt::Debug> BTree<K, V> {
    /// 満杯の子ノードを分割する
    /// parent: 分割対象の子を持つ親ノード
    /// child_index: 分割する子ノードのインデックス
    fn split_child(parent: &mut BTreeNode<K, V>, child_index: usize, t: usize) {
        let mid = t - 1; // 中央インデックス

        // 分割対象の子ノードから右半分を取り出して新ノードを作成
        let right_child = {
            let left_child = &mut parent.children[child_index];
            let mut right = if left_child.is_leaf {
                BTreeNode::new_leaf()
            } else {
                BTreeNode::new_internal()
            };

            // 右半分のキーと値を移動
            right.keys = left_child.keys.split_off(mid + 1);
            right.values = left_child.values.split_off(mid + 1);

            // 内部ノードなら子も分割
            if !left_child.is_leaf {
                right.children = left_child.children.split_off(mid + 1);
            }

            right
        };

        // 中央のキーと値を親ノードに挿入
        let mid_key = parent.children[child_index].keys.remove(mid);
        let mid_val = parent.children[child_index].values.remove(mid);
        parent.keys.insert(child_index, mid_key);
        parent.values.insert(child_index, mid_val);

        // 右の新ノードを親の子として挿入
        parent.children.insert(child_index + 1, Box::new(right_child));
    }
}
```

#### `insert`（挿入操作）

```rust
impl<K: Ord + Clone + std::fmt::Debug, V: Clone + std::fmt::Debug> BTree<K, V> {
    pub fn insert(&mut self, key: K, value: V) {
        let t = self.t;

        if self.root.is_none() {
            // 空の木 → 葉ノードをルートとして作成
            let mut root = BTreeNode::new_leaf();
            root.keys.push(key);
            root.values.push(value);
            self.root = Some(Box::new(root));
            return;
        }

        // ルートが満杯の場合は先に分割（top-down アプローチ）
        let root_full = self.root.as_ref().unwrap().is_full(t);
        if root_full {
            let old_root = self.root.take().unwrap();
            let mut new_root = BTreeNode::new_internal();
            new_root.children.push(old_root);
            Self::split_child(&mut new_root, 0, t);
            Self::insert_non_full(&mut new_root, key, value, t);
            self.root = Some(Box::new(new_root));
        } else {
            Self::insert_non_full(self.root.as_mut().unwrap(), key, value, t);
        }
    }

    /// 満杯でないノードへの挿入（再帰的）
    fn insert_non_full(node: &mut BTreeNode<K, V>, key: K, value: V, t: usize) {
        let pos = node.keys.partition_point(|k| k < &key);

        if node.is_leaf {
            // 葉ノード: 正しい位置に挿入
            node.keys.insert(pos, key);
            node.values.insert(pos, value);
        } else {
            // 内部ノード: 適切な子へ再帰
            let child_full = node.children[pos].is_full(t);
            if child_full {
                // 子が満杯なら分割してから挿入
                Self::split_child(node, pos, t);
                // 分割後、どちらの子に挿入するか再判定
                if key > node.keys[pos] {
                    Self::insert_non_full(&mut node.children[pos + 1], key, value, t);
                } else {
                    Self::insert_non_full(&mut node.children[pos], key, value, t);
                }
            } else {
                Self::insert_non_full(&mut node.children[pos], key, value, t);
            }
        }
    }
}
```

#### テストコード

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_btree_insert_and_search() {
        let mut tree: BTree<i32, &str> = BTree::new(2);

        // 挿入
        tree.insert(10, "ten");
        tree.insert(20, "twenty");
        tree.insert(5, "five");
        tree.insert(15, "fifteen");
        tree.insert(25, "twenty-five");

        // 検索
        assert_eq!(tree.search(&10), Some(&"ten"));
        assert_eq!(tree.search(&5), Some(&"five"));
        assert_eq!(tree.search(&25), Some(&"twenty-five"));
        assert_eq!(tree.search(&99), None);
    }

    #[test]
    fn test_btree_split_on_insert() {
        let mut tree: BTree<i32, i32> = BTree::new(2); // t=2 → 最大3キー

        // ルート分割を引き起こす
        for i in [10, 20, 30, 40, 50, 35] {
            tree.insert(i, i * 10);
        }

        // 全キーが検索できることを確認
        assert_eq!(tree.search(&10), Some(&100));
        assert_eq!(tree.search(&35), Some(&350));
        assert_eq!(tree.search(&50), Some(&500));
    }

    #[test]
    fn test_btree_large_insertion() {
        let mut tree: BTree<i32, i32> = BTree::new(3);

        // 100個のキーを挿入
        for i in 0..100 {
            tree.insert(i, i * i);
        }

        // 全検索
        for i in 0..100 {
            assert_eq!(tree.search(&i), Some(&(i * i)));
        }
        assert_eq!(tree.search(&100), None);
    }
}
```

---

### `std::collections::BTreeMap` の内部実装

Rust の標準ライブラリの `BTreeMap` は、上記のシンプルな実装よりもずっと洗練されています。

```rust
use std::collections::BTreeMap;

fn btreemap_demo() {
    let mut map: BTreeMap<String, u64> = BTreeMap::new();

    // 挿入はO(log n)
    map.insert("apple".to_string(), 100);
    map.insert("banana".to_string(), 200);
    map.insert("cherry".to_string(), 150);
    map.insert("date".to_string(), 300);

    // キーはソート順で反復される（BTreeMapの最大の特徴）
    for (k, v) in &map {
        println!("{}: {}", k, v);
        // apple: 100
        // banana: 200
        // cherry: 150
        // date: 300
    }

    // 範囲検索 O(log n + k)  ← HashMap では不可能
    use std::ops::Bound::*;
    for (k, v) in map.range("b".to_string()..="c".to_string()) {
        println!("範囲内: {}: {}", k, v);
        // 範囲内: banana: 200
        // 範囲内: cherry: 150
    }

    // 最小・最大の取得 O(log n)
    println!("最小: {:?}", map.iter().next());
    println!("最大: {:?}", map.iter().next_back());
}
```

> **実装の詳細**: `std::collections::BTreeMap` は Rust 独自の **node-based B-Tree** で、次数が固定（B = 6 程度）のコンパクトな実装です。各ノードはスタック上の配列（`[MaybeUninit<K>; CAPACITY]` 等）として管理され、ヒープアロケーションを最小限に抑えています。

### B-Tree vs BTreeMap の比較

| 項目 | 自作 BTree | `std::BTreeMap` |
|------|------------|-----------------|
| 次数 | 可変（引数で指定） | 固定（コンパイル時定数） |
| メモリ管理 | `Box<BTreeNode>` で都度ヒープ確保 | ノード内配列でコンパクト |
| 削除 | 複雑（未実装） | 完全実装済み |
| イテレータ | 未対応 | `Iterator` トレイト実装済み |
| ユースケース | 学習・カスタム実装 | プロダクション利用 |

---

### B-Tree と B+Tree の違い

```
B-Tree（値をすべてのノードに格納）:

                  [20(val) | 50(val)]
                 /          |         \
    [10(val)|15(val)]  [30(val)|40(val)]  [60(val)|70(val)]

    └─ 内部ノードにも値を持つ
    └─ 葉ノード間のリンクなし

B+Tree（値は葉ノードのみ、内部ノードはキーのみ）:

                    [  20  |  50  ]         ← 内部ノード（値なし、キーのみ）
                   /        |        \
          [ 20 | 50 ]  [ 20 | 50 ]  [ 20 | 50 ]
         /       |    \
[10|15]→[20|30]→[40|50]→[60|70]   ← 葉ノード（値あり、リンクで連結）
 ↑ 葉ノードが連結リストで繋がっている

B+Tree の利点:
  1. 範囲スキャンが高速（葉リストを辿るだけ）
  2. 内部ノードがコンパクト → 同じページにより多くのキーを格納
  3. データベースのインデックスはほぼ全てB+Tree
```

```rust
/// B+Tree の葉ノード（B-Tree との主な違い）
pub struct BPlusLeafNode<K, V> {
    pub keys: Vec<K>,
    pub values: Vec<V>,
    /// 次の葉ノードへのポインタ（範囲スキャン用）
    pub next: Option<Box<BPlusLeafNode<K, V>>>,
}

/// B+Tree での範囲スキャンのイメージ
fn range_scan_demo() {
    // std::collections::BTreeMap は B+Tree に近い動作をする
    let mut map = std::collections::BTreeMap::new();
    for i in 0..=100i32 {
        map.insert(i, i * i);
    }

    // 50〜60 の範囲スキャン: 葉リストを順にたどるだけで O(k)
    let results: Vec<_> = map.range(50..=60).collect();
    println!("範囲スキャン結果: {:?}", results);
}
```

---

## Radix Tree（Trie）

### Radix Tree とは

**Trie（トライ）** は文字列をキーとする木構造です。共通のプレフィックス（前置部分）を持つキーは同じパスを共有します。**Radix Tree**（またはコンパクトTrie, Patricia Trie）は、子が1つしかないノードをマージして圧縮したTrieです。

```
通常の Trie（文字単位で分岐）:
"ant", "any", "ant", "apple" を格納

       (root)
         |
         a
         |
         n ─── p
        / \     \
       t   y     p
       *   *      \
                   l
                    \
                     e
                     *

Radix Tree（共通プレフィックスを圧縮）:
       (root)
         |
        "a"
        / \
      "n"  "pple"*
      / \
    "t"* "y"*

※ * = 終端（そこまでが1つの完全なキー）
```

### 圧縮の効果

```
例: IPアドレスのルーティングテーブル

192.168.0.0/24  → next hop: gw1
192.168.1.0/24  → next hop: gw2
192.168.0.0/16  → next hop: gw3 (より広いサブネット)
10.0.0.0/8      → next hop: gw4

Radix Tree での表現（ビット単位）:

                  (root)
                /        \
        "1100_0000"      "0000_1010"   (192.xxx vs 10.xxx)
        (192.xxx)          → gw4
           /
    "1010_1000"
    (168.xxx)
          → gw3 (192.168.0.0/16)
         /       \
   "0000_0000" "0000_0001"
   → gw1        → gw2
```

### 用途

| 用途 | 詳細 |
|------|------|
| **IPルーティングテーブル** | Linux カーネルの `fib_trie`、BGPルータ |
| **HTTPルーター** | `actix-web`, `axum` のルートマッチング |
| **オートコンプリート** | 検索エンジンのサジェスト機能 |
| **DNSキャッシュ** | ドメイン名の逆引きツリー |
| **プログラミング言語のシンボルテーブル** | 変数名・関数名の管理 |

---

### Rust による基本的な Trie の実装

#### TrieNode 構造体

```rust
use std::collections::HashMap;

/// Trie のノード（HashMap ベース）
#[derive(Debug, Default)]
pub struct TrieNode {
    /// 子ノードのマップ（文字 → ノード）
    children: HashMap<char, TrieNode>,
    /// このノードで終端するキーの値（Noneなら終端でない）
    value: Option<String>,
    /// 終端フラグ
    is_end: bool,
}

/// Trie 本体
#[derive(Debug, Default)]
pub struct Trie {
    root: TrieNode,
}
```

#### `insert` 操作

```rust
impl Trie {
    pub fn new() -> Self {
        Trie {
            root: TrieNode::default(),
        }
    }

    /// キーと値を Trie に挿入する O(m)  m = キーの長さ
    pub fn insert(&mut self, key: &str, value: &str) {
        let mut current = &mut self.root;

        for ch in key.chars() {
            // 子ノードが存在しなければ作成
            current = current
                .children
                .entry(ch)
                .or_insert_with(TrieNode::default);
        }

        // 末尾ノードに値をセット
        current.is_end = true;
        current.value = Some(value.to_string());
    }

    /// キーを検索して値を返す O(m)
    pub fn search(&self, key: &str) -> Option<&str> {
        let mut current = &self.root;

        for ch in key.chars() {
            match current.children.get(&ch) {
                Some(node) => current = node,
                None => return None,
            }
        }

        if current.is_end {
            current.value.as_deref()
        } else {
            None
        }
    }

    /// 前方一致検索: prefix で始まるキーが存在すれば true O(m)
    pub fn starts_with(&self, prefix: &str) -> bool {
        let mut current = &self.root;

        for ch in prefix.chars() {
            match current.children.get(&ch) {
                Some(node) => current = node,
                None => return false,
            }
        }

        true // prefix まで辿れた = 前方一致するキーが存在
    }

    /// prefix で始まる全てのキーと値を収集する
    pub fn collect_with_prefix(&self, prefix: &str) -> Vec<(String, String)> {
        let mut current = &self.root;

        // prefix の末尾ノードまで移動
        for ch in prefix.chars() {
            match current.children.get(&ch) {
                Some(node) => current = node,
                None => return vec![],
            }
        }

        // そこ以降を DFS で収集
        let mut results = Vec::new();
        Self::dfs_collect(current, prefix.to_string(), &mut results);
        results
    }

    fn dfs_collect(node: &TrieNode, current_key: String, results: &mut Vec<(String, String)>) {
        if node.is_end {
            if let Some(val) = &node.value {
                results.push((current_key.clone(), val.clone()));
            }
        }
        for (ch, child) in &node.children {
            let mut next_key = current_key.clone();
            next_key.push(*ch);
            Self::dfs_collect(child, next_key, results);
        }
    }
}
```

#### テストコード

```rust
#[cfg(test)]
mod trie_tests {
    use super::*;

    #[test]
    fn test_trie_insert_and_search() {
        let mut trie = Trie::new();

        trie.insert("apple", "りんご");
        trie.insert("app", "アプリ");
        trie.insert("application", "アプリケーション");
        trie.insert("banana", "バナナ");

        assert_eq!(trie.search("apple"), Some("りんご"));
        assert_eq!(trie.search("app"), Some("アプリ"));
        assert_eq!(trie.search("ap"), None); // 途中までは None
        assert_eq!(trie.search("banana"), Some("バナナ"));
        assert_eq!(trie.search("grape"), None);
    }

    #[test]
    fn test_trie_starts_with() {
        let mut trie = Trie::new();
        trie.insert("rust", "Rustプログラミング言語");
        trie.insert("ruby", "Rubyプログラミング言語");
        trie.insert("python", "Pythonプログラミング言語");

        assert!(trie.starts_with("ru"));   // rust, ruby
        assert!(trie.starts_with("rust")); // rust
        assert!(trie.starts_with("py"));   // python
        assert!(!trie.starts_with("java"));
        assert!(!trie.starts_with("rusts")); // "rusts" というキーは存在しない
    }

    #[test]
    fn test_trie_collect_with_prefix() {
        let mut trie = Trie::new();
        trie.insert("rust", "lang");
        trie.insert("ruby", "lang");
        trie.insert("python", "lang");
        trie.insert("rust-lang", "org");

        let mut results = trie.collect_with_prefix("ru");
        results.sort(); // HashMap の順序は非決定的なのでソート
        assert_eq!(results.len(), 3); // rust, ruby, rust-lang
        assert!(results.iter().any(|(k, _)| k == "rust"));
        assert!(results.iter().any(|(k, _)| k == "ruby"));
        assert!(results.iter().any(|(k, _)| k == "rust-lang"));
    }
}
```

---

### 圧縮 Radix Tree（Patricia Trie）の実装

通常の Trie は子が1つしかない「一本道」のノードが多くなり、メモリ効率が悪くなります。Patricia Trie はそのような連続するノードをマージして、**エッジにサブ文字列**を持たせます。

```
通常の Trie（"interview", "interact", "internal" を格納）:

 i─n─t─e─r─v─i─e─w*
              |
              a─c─t*
              |
              n─a─l*

Patricia Trie（圧縮後）:

 "inter"
 /    |
"view"*  "a"
         / \
       "ct"* "nal"*
```

```rust
/// Patricia Trie のノード
#[derive(Debug)]
pub struct PatriciaNode {
    /// このエッジが表すラベル（圧縮された文字列）
    label: String,
    /// 子ノードのリスト
    children: Vec<PatriciaNode>,
    /// 終端かどうか
    is_end: bool,
}

impl PatriciaNode {
    fn new(label: &str, is_end: bool) -> Self {
        PatriciaNode {
            label: label.to_string(),
            children: Vec::new(),
            is_end,
        }
    }

    /// 共通プレフィックスの長さを返す
    fn common_prefix_len(a: &str, b: &str) -> usize {
        a.chars()
            .zip(b.chars())
            .take_while(|(ca, cb)| ca == cb)
            .count()
    }
}

/// Patricia Trie 本体（簡略版）
pub struct PatriciaTrie {
    root: PatriciaNode,
}

impl PatriciaTrie {
    pub fn new() -> Self {
        PatriciaTrie {
            root: PatriciaNode::new("", false),
        }
    }

    pub fn insert(&mut self, key: &str) {
        Self::insert_node(&mut self.root, key);
    }

    fn insert_node(node: &mut PatriciaNode, remaining: &str) {
        for child in &mut node.children {
            let common = PatriciaNode::common_prefix_len(&child.label, remaining);
            if common == 0 {
                continue;
            }

            if common == child.label.len() && common == remaining.len() {
                // 完全一致 → 終端マーク
                child.is_end = true;
                return;
            } else if common == child.label.len() {
                // child.label が remaining の前置 → 残りを再帰
                Self::insert_node(child, &remaining[common..]);
                return;
            } else {
                // 途中で分岐 → ノードを分割
                let old_label = child.label[common..].to_string();
                let new_label = remaining[common..].to_string();
                let old_is_end = child.is_end;
                let old_children = std::mem::take(&mut child.children);

                // 既存ノードを短縮
                child.label = child.label[..common].to_string();
                child.is_end = new_label.is_empty();

                // 旧ラベルの残り部分を子に
                let mut old_child = PatriciaNode::new(&old_label, old_is_end);
                old_child.children = old_children;
                child.children.push(old_child);

                // 新しいキーの残り部分を子に
                if !new_label.is_empty() {
                    child.children.push(PatriciaNode::new(&new_label, true));
                }
                return;
            }
        }

        // マッチする子がない → 新しい子を追加
        node.children.push(PatriciaNode::new(remaining, true));
    }

    pub fn search(&self, key: &str) -> bool {
        Self::search_node(&self.root, key)
    }

    fn search_node(node: &PatriciaNode, remaining: &str) -> bool {
        if remaining.is_empty() {
            return node.is_end;
        }
        for child in &node.children {
            let common = PatriciaNode::common_prefix_len(&child.label, remaining);
            if common == child.label.len() {
                return Self::search_node(child, &remaining[common..]);
            }
        }
        false
    }
}

#[test]
fn test_patricia_trie() {
    let mut trie = PatriciaTrie::new();
    trie.insert("interview");
    trie.insert("interact");
    trie.insert("internal");
    trie.insert("int");

    assert!(trie.search("interview"));
    assert!(trie.search("interact"));
    assert!(trie.search("internal"));
    assert!(trie.search("int"));
    assert!(!trie.search("inter"));   // "inter" は挿入していない
    assert!(!trie.search("interop")); // 存在しないキー
}
```

---

### クレートの紹介

#### `radix_trie` クレート

プロダクション品質の Radix Trie 実装です。

```toml
# Cargo.toml
[dependencies]
radix_trie = "0.2"
```

```rust
use radix_trie::Trie;

fn radix_trie_demo() {
    let mut trie: Trie<String, u32> = Trie::new();

    trie.insert("apple".to_string(), 1);
    trie.insert("application".to_string(), 2);
    trie.insert("apply".to_string(), 3);
    trie.insert("banana".to_string(), 4);

    // 前方一致で最長一致を取得
    if let Some(val) = trie.get("apple") {
        println!("apple → {}", val); // apple → 1
    }

    // サブトライの取得（前方一致検索）
    if let Some(subtrie) = trie.get_ancestor("app") {
        println!("'app' で始まるエントリ数: {}", subtrie.len());
    }
}
```

#### `patricia_tree` クレート

Patricia Tree の効率的な実装で、特にビット単位のIPアドレス管理に強みがあります。

```toml
# Cargo.toml
[dependencies]
patricia_tree = "0.6"
```

```rust
use patricia_tree::PatriciaMap;

fn patricia_tree_demo() {
    let mut map = PatriciaMap::new();

    // HTTPルーターの例
    map.insert("/api/users", "UsersHandler");
    map.insert("/api/users/:id", "UserDetailHandler");
    map.insert("/api/posts", "PostsHandler");
    map.insert("/health", "HealthCheckHandler");

    // 完全一致検索
    if let Some(&handler) = map.get("/api/users") {
        println!("ハンドラ: {}", handler);
    }

    // 共通プレフィックスを持つエントリを列挙
    for (key, val) in map.iter_prefix(b"/api") {
        println!("{}: {}", String::from_utf8_lossy(key), val);
    }
}
```

---

### `BTreeMap` vs `HashMap` vs Radix Tree の使い分け

```
特性比較表:

┌─────────────────────┬─────────────┬─────────────┬─────────────────┐
│ 操作                │  BTreeMap   │   HashMap   │   Radix Tree    │
├─────────────────────┼─────────────┼─────────────┼─────────────────┤
│ 挿入                │  O(log n)   │  O(1) amort │  O(m) ※m=長さ  │
│ 検索                │  O(log n)   │  O(1) amort │  O(m)           │
│ 削除                │  O(log n)   │  O(1) amort │  O(m)           │
│ 範囲検索            │  O(log n+k) │  不可       │  O(log n+k)     │
│ 前方一致検索        │  O(m log n) │  不可       │  O(m)           │
│ 最小・最大          │  O(log n)   │  不可       │  O(h) ※h=高さ  │
│ イテレーション順序  │  ソート済み │  ランダム   │  辞書順         │
│ メモリ効率          │  中         │  高         │ 共通prefix圧縮  │
├─────────────────────┼─────────────┼─────────────┼─────────────────┤
│ キーの型            │ Ord を実装  │ Hash+Eq実装 │  主に文字列     │
│ キャッシュ効率      │  高（局所性）│ 低（散乱）  │  中             │
└─────────────────────┴─────────────┴─────────────┴─────────────────┘
```

```rust
/// どのデータ構造を選ぶべきか？ 判断フローチャート
///
///  キーが文字列系（URL, ドメイン名, 単語）?
///  ├─ Yes: 前方一致検索や共通プレフィックス操作が必要?
///  │        ├─ Yes → Radix Tree / Trie
///  │        └─ No  → HashMap（単純な文字列 → 値のマップ）
///  └─ No: キーの順序（範囲検索・最小最大）が必要?
///           ├─ Yes → BTreeMap
///           └─ No  → HashMap（最速の単純 K-V）

// 使い分けの具体例
fn choose_data_structure() {
    use std::collections::{BTreeMap, HashMap};

    // ケース1: IPアドレスのルーティングテーブル → Radix Tree
    //   理由: ビット前方一致（最長一致）が必要
    let mut routing: HashMap<&str, &str> = HashMap::new(); // 簡易代替
    routing.insert("192.168.0.0/24", "gw1");

    // ケース2: 時系列データのインデックス → BTreeMap
    //   理由: タイムスタンプ範囲での検索が必要
    let mut time_series: BTreeMap<u64, f64> = BTreeMap::new();
    time_series.insert(1_700_000_000, 98.6);
    time_series.insert(1_700_000_060, 98.7);
    // 特定時間範囲の検索
    let _range: Vec<_> = time_series.range(1_700_000_000..1_700_000_120).collect();

    // ケース3: セッションID → ユーザー情報 → HashMap
    //   理由: ランダムなキー、順序不要、O(1) が欲しい
    let mut sessions: HashMap<String, u64> = HashMap::new();
    sessions.insert("abc123xyz".to_string(), 42);

    // ケース4: オートコンプリート → Trie
    //   理由: 前方一致で候補を高速列挙したい
    let mut autocomplete = Trie::new();
    autocomplete.insert("rust", "lang");
    autocomplete.insert("rustfmt", "tool");
    autocomplete.insert("rust-analyzer", "lsp");
    let suggestions = autocomplete.collect_with_prefix("rust");
    println!("候補: {:?}", suggestions);
}
```

---

## まとめ

```
B-Tree の要点:
  ・平衡多分木。次数 t でキー数と子数を制御
  ・ディスクI/Oを最小化（DBインデックス・FSに最適）
  ・std::BTreeMap は範囲検索・ソート順イテレーションが強み
  ・B+Tree は葉ノードをリンクし、範囲スキャンをさらに高速化

Radix Tree（Trie）の要点:
  ・文字列キーの共通プレフィックスを共有するツリー
  ・前方一致検索が O(m)（文字列長のみに依存）
  ・Patricia Trie で1子ノードを圧縮 → メモリ削減
  ・IPルーティング・HTTPルーター・オートコンプリートに最適

使い分けの指針:
  ・順序・範囲が必要 → BTreeMap
  ・単純 K-V でとにかく速く → HashMap
  ・文字列の前方一致・プレフィックス操作 → Radix Tree / Trie
```

### 参考リンク

- [Rust std::collections::BTreeMap ドキュメント](https://doc.rust-lang.org/std/collections/struct.BTreeMap.html)
- [radix_trie クレート](https://crates.io/crates/radix_trie)
- [patricia_tree クレート](https://crates.io/crates/patricia_tree)
- [B-Tree の可視化（University of San Francisco）](https://www.cs.usfca.edu/~galles/visualization/BTree.html)
