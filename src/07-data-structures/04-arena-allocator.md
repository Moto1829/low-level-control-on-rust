# アリーナアロケータとスラブアロケータ

通常のメモリアロケータ（`malloc`/`free`）は汎用的ですが、小さなオブジェクトを頻繁に確保・解放するとフラグメンテーションや同期コストが問題になります。
**アリーナアロケータ**と**スラブアロケータ**はこれを回避するための専用アロケータです。

---

## バンプアロケータの仕組みと実装

バンプアロケータ（Bump Allocator / Arena Allocator）は最も単純なアリーナ実装です。
大きなメモリブロックを事前に確保し、ポインタを一方向に「バンプ（ずらす）」するだけで確保を行います。

```
メモリブロック（capacity = 64 bytes）

アドレス: [0         16        32        48        64]
          +----------+----------+----------+----------+
          | Object A | Object B | Object C |  空き    |
          +----------+----------+----------+----------+
                                            ^
                                          bump
                                       （次の確保位置）

- 確保: bump を object のサイズ分だけ進める → O(1)
- 解放: 個別には不可。アリーナごと一括解放 → O(1)
- フラグメンテーション: 発生しない
```

```rust
use std::alloc::{alloc, dealloc, Layout};
use std::cell::Cell;
use std::ptr::NonNull;

/// シンプルなバンプアロケータ
pub struct BumpAllocator {
    start: NonNull<u8>,
    end: *const u8,
    bump: Cell<*mut u8>, // 現在の確保位置
    capacity: usize,
}

impl BumpAllocator {
    pub fn new(capacity: usize) -> Self {
        let layout = Layout::from_size_align(capacity, 16).unwrap();
        // SAFETY: layout のサイズは 0 より大きい
        let ptr = unsafe { alloc(layout) };
        let start = NonNull::new(ptr).expect("メモリ確保に失敗");
        let end = unsafe { start.as_ptr().add(capacity) };
        Self {
            start,
            end,
            bump: Cell::new(start.as_ptr()),
            capacity,
        }
    }

    /// アライメント済みのメモリを確保する
    pub fn alloc_bytes(&self, size: usize, align: usize) -> Option<*mut u8> {
        let bump = self.bump.get();
        // アライメント調整
        let aligned = bump as usize;
        let aligned = (aligned + align - 1) & !(align - 1);
        let new_bump = aligned + size;
        if new_bump > self.end as usize {
            return None; // 容量不足
        }
        self.bump.set(new_bump as *mut u8);
        Some(aligned as *mut u8)
    }

    /// 型付きオブジェクトを確保して初期化する
    pub fn alloc<T>(&self, val: T) -> Option<&T> {
        let ptr = self.alloc_bytes(
            std::mem::size_of::<T>(),
            std::mem::align_of::<T>(),
        )? as *mut T;
        // SAFETY: ptr は正しくアライメントされ有効な領域
        unsafe {
            ptr.write(val);
            Some(&*ptr)
        }
    }

    /// リセット: 全オブジェクトを一括解放（Drop は呼ばれない）
    pub fn reset(&self) {
        self.bump.set(self.start.as_ptr());
    }

    /// 使用済みバイト数
    pub fn used(&self) -> usize {
        self.bump.get() as usize - self.start.as_ptr() as usize
    }
}

impl Drop for BumpAllocator {
    fn drop(&mut self) {
        let layout = Layout::from_size_align(self.capacity, 16).unwrap();
        // SAFETY: start は new() で alloc したポインタ
        unsafe { dealloc(self.start.as_ptr(), layout) };
    }
}

fn main() {
    let arena = BumpAllocator::new(1024);

    let a = arena.alloc(42u32).unwrap();
    let b = arena.alloc(3.14f64).unwrap();
    let c = arena.alloc("hello").unwrap();

    println!("a={}, b={}, c={}", a, b, c);
    println!("使用量: {} bytes", arena.used());

    arena.reset(); // 全てを一括解放（Drop は呼ばれない）
    println!("リセット後の使用量: {} bytes", arena.used());
}
```

---

## `bumpalo` クレートの使い方

`bumpalo` は安全な Rust で使えるバンプアロケータライブラリです。
ライフタイムを利用してアリーナ内の参照が安全にアクセスできることを保証します。

```toml
# Cargo.toml
[dependencies]
bumpalo = { version = "3", features = ["collections"] }
```

```rust
use bumpalo::Bump;
use bumpalo::collections::Vec as BumpVec;

fn main() {
    // アリーナを生成（複数回の内部チャンクを自動管理）
    let arena = Bump::new();

    // 基本的な確保
    let x: &mut i32 = arena.alloc(42);
    *x += 1;
    println!("x = {}", x); // 43

    // アリーナ内に Vec を確保
    let mut v: BumpVec<u32> = BumpVec::new_in(&arena);
    for i in 0..100 {
        v.push(i * i);
    }
    println!("v[10] = {}", v[10]); // 100

    // 文字列のコピーをアリーナに確保
    let s: &str = arena.alloc_str("hello, arena!");
    println!("{}", s);

    // スライスの確保
    let slice: &[i32] = arena.alloc_slice_copy(&[1, 2, 3, 4, 5]);
    println!("{:?}", slice);

    // イテレータからスライスを確保
    let squares: &[u64] = arena.alloc_slice_fill_iter((0u64..10).map(|i| i * i));
    println!("{:?}", squares);

    println!("アリーナ使用量: {} bytes", arena.allocated_bytes());
} // arena がここで drop → 全メモリが一括解放
```

---

## アリーナのリセット（一括解放）の利点

```
通常のアロケータ（malloc/free）の問題:
  確保 → 確保 → 確保 → 解放 → 確保 → 解放 → 解放
  各 free() で:
    - フラグメンテーションの管理
    - フリーリストの更新
    - マルチスレッド時のロック取得
  → 多数の小オブジェクトで著しく遅い

アリーナアロケータ:
  確保 → 確保 → 確保 → ... → [reset() で全解放]
  各確保: ポインタを足すだけ (1〜2 命令)
  解放:  アリーナ全体を一括解放 (1 命令)
  → フラグメンテーション ゼロ、ロック不要
```

```rust
use bumpalo::Bump;
use std::time::Instant;

fn bench_arena(n: usize) {
    let arena = Bump::new();
    let start = Instant::now();
    for _ in 0..n {
        let _: &[u8; 64] = arena.alloc([0u8; 64]);
    }
    // arena drop で一括解放
    println!("Arena: {:?} ({} 確保)", start.elapsed(), n);
}

fn bench_box(n: usize) {
    let start = Instant::now();
    for _ in 0..n {
        let _: Box<[u8; 64]> = Box::new([0u8; 64]); // 個別に確保・解放
    }
    println!("Box:   {:?} ({} 確保)", start.elapsed(), n);
}

fn main() {
    bench_arena(1_000_000);
    bench_box(1_000_000);
}
```

```
典型的な結果（参考値）:
  Arena: ~8 ms
  Box:   ~180 ms
```

---

## スラブアロケータの概念（固定サイズオブジェクトプール）

スラブアロケータ（Slab Allocator）は同一サイズのオブジェクトを効率的に管理します。
Linux カーネルのメモリ管理でも使われている手法です。

```
スラブの構造:

スラブ（= 固定サイズのページ群）
+--------+--------+--------+--------+--------+
| obj[0] | obj[1] | obj[2] | obj[3] | obj[4] |
+--------+--------+--------+--------+--------+
   使用中   空き     使用中   空き     空き

フリーリスト（単方向リスト）:
  obj[1] → obj[3] → obj[4] → NULL

確保: フリーリストの先頭を外す → O(1)
解放: フリーリストの先頭に追加 → O(1)
```

---

## オブジェクトプールの実装例

```rust
use std::mem::MaybeUninit;

/// 固定サイズ T のオブジェクトプール
pub struct ObjectPool<T, const N: usize> {
    storage: Box<[MaybeUninit<T>; N]>,
    free_list: Vec<usize>, // 空きスロットのインデックスリスト
}

impl<T, const N: usize> ObjectPool<T, N> {
    pub fn new() -> Self {
        Self {
            // SAFETY: MaybeUninit は未初期化でよい
            storage: Box::new(unsafe { MaybeUninit::uninit().assume_init() }),
            free_list: (0..N).collect(),
        }
    }

    /// オブジェクトを確保してインデックスを返す
    pub fn alloc(&mut self, val: T) -> Option<usize> {
        let idx = self.free_list.pop()?;
        // SAFETY: idx は有効なスロット
        unsafe { self.storage[idx].write(val) };
        Some(idx)
    }

    /// インデックスで参照を取得
    pub fn get(&self, idx: usize) -> Option<&T> {
        if self.free_list.contains(&idx) {
            return None; // 未確保スロット
        }
        // SAFETY: idx は初期化済み
        unsafe { Some(self.storage[idx].assume_init_ref()) }
    }

    /// オブジェクトを解放
    pub fn dealloc(&mut self, idx: usize) {
        assert!(!self.free_list.contains(&idx), "二重解放");
        // SAFETY: idx は初期化済み
        unsafe { self.storage[idx].assume_init_drop() };
        self.free_list.push(idx);
    }
}

impl<T, const N: usize> Drop for ObjectPool<T, N> {
    fn drop(&mut self) {
        // 確保中のオブジェクトを全て解放
        let all_indices: Vec<usize> = (0..N)
            .filter(|i| !self.free_list.contains(i))
            .collect();
        for idx in all_indices {
            // SAFETY: 確保中スロットは初期化済み
            unsafe { self.storage[idx].assume_init_drop() };
        }
    }
}

#[derive(Debug)]
struct Particle {
    x: f32,
    y: f32,
    vx: f32,
    vy: f32,
    lifetime: f32,
}

fn main() {
    let mut pool: ObjectPool<Particle, 1000> = ObjectPool::new();

    // パーティクルを確保
    let p1 = pool.alloc(Particle { x: 0.0, y: 0.0, vx: 1.0, vy: 2.0, lifetime: 5.0 }).unwrap();
    let p2 = pool.alloc(Particle { x: 10.0, y: 5.0, vx: -1.0, vy: 0.5, lifetime: 3.0 }).unwrap();

    println!("p1: {:?}", pool.get(p1));
    println!("p2: {:?}", pool.get(p2));

    // p1 を解放（スロットを再利用可能に）
    pool.dealloc(p1);
    println!("p1 解放後: {:?}", pool.get(p1)); // None

    // 同じスロットを再利用
    let p3 = pool.alloc(Particle { x: 99.0, y: 0.0, vx: 0.0, vy: 0.0, lifetime: 1.0 }).unwrap();
    assert_eq!(p1, p3); // 同じインデックスが再利用される
    println!("p3 (p1 と同インデックス): {:?}", pool.get(p3));
}
```

---

## ゲーム開発・GC 実装・パーサーでの活用例

### ゲーム開発: フレームアリーナ

```rust
use bumpalo::Bump;

struct GameState {
    frame_arena: Bump,
}

impl GameState {
    fn update(&mut self) {
        // フレーム開始時にアリーナをリセット
        self.frame_arena.reset();

        // このフレームだけ生きる一時データを確保
        let visible_entities: &mut Vec<u32> =
            self.frame_arena.alloc(Vec::new());

        // ... エンティティの可視判定 ...
        for id in 0..100u32 {
            if id % 3 == 0 {
                visible_entities.push(id);
            }
        }

        // フレーム終了後、次の reset() で自動的に解放される
        println!("可視エンティティ数: {}", visible_entities.len());
    }
}
```

### パーサー: AST ノードのアリーナ確保

```rust
use bumpalo::Bump;

#[derive(Debug)]
enum Expr<'a> {
    Num(f64),
    Add(&'a Expr<'a>, &'a Expr<'a>),
    Mul(&'a Expr<'a>, &'a Expr<'a>),
}

fn build_ast<'a>(arena: &'a Bump) -> &'a Expr<'a> {
    // (1 + 2) * 3
    let one   = arena.alloc(Expr::Num(1.0));
    let two   = arena.alloc(Expr::Num(2.0));
    let three = arena.alloc(Expr::Num(3.0));
    let add   = arena.alloc(Expr::Add(one, two));
    arena.alloc(Expr::Mul(add, three))
}

fn eval(expr: &Expr) -> f64 {
    match expr {
        Expr::Num(n)     => *n,
        Expr::Add(l, r)  => eval(l) + eval(r),
        Expr::Mul(l, r)  => eval(l) * eval(r),
    }
}

fn main() {
    let arena = Bump::new();
    let ast = build_ast(&arena);
    println!("結果: {}", eval(ast)); // 9
} // arena drop → AST 全体が一括解放
```

### GC 実装: 世代別コレクション

```
世代別 GC でのアリーナ利用:

Young Generation:
  +--[Arena 0]--+  ← 新規オブジェクト確保
  | obj | obj   |
  | obj | ...   |
  +-------------+
       ↓ GC 発生
  生き残ったオブジェクトを Old Generation へコピー
  Arena 0 をまるごとリセット → フラグメンテーションゼロ

Old Generation:
  +--[Arena 1]--+
  | 長寿命 obj  |
  +-------------+
```

---

## まとめ

| アロケータ | 確保コスト | 解放コスト | フラグメンテーション | 用途 |
|---|---|---|---|---|
| malloc/free | O(log N) | O(log N) | あり | 汎用 |
| バンプ（アリーナ） | O(1) | O(1) 一括 | なし | 短命オブジェクト群 |
| スラブ（プール） | O(1) | O(1) | なし | 同サイズオブジェクト |
| bumpalo | O(1) | O(1) 一括 | なし | 安全な Rust での利用 |
