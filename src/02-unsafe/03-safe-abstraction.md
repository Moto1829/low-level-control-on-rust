# 03: 安全な抽象化 (Safe Abstraction)

## 安全な抽象化とは

`unsafe` の真の目的は「unsafe をゼロにすること」ではなく、**unsafe なコアを安全なインターフェースで包むこと**です。
標準ライブラリの `Vec<T>`、`Box<T>`、`Mutex<T>` はすべてこのパターンで実装されています。

```
外側から見た世界
┌─────────────────────────────────────────────┐
│   安全な公開 API                             │
│   fn push(&mut self, val: T)                │
│   fn get(&self, i: usize) -> Option<&T>     │
│   fn split_at_mut(&mut self, ..) -> (..,..) │
└────────────────────┬────────────────────────┘
                     │ 内部実装（非公開）
┌────────────────────▼────────────────────────┐
│   unsafe コア                               │
│   *mut T、生メモリアロケーション             │
│   ポインタ演算、from_raw_parts              │
└─────────────────────────────────────────────┘
```

設計の鉄則:

> **外部から呼ばれるすべての安全な関数は、どんな入力に対しても UB を起こしてはならない。**

---

## Invariant の文書化

安全な抽象化の核心は **invariant（不変条件）** の文書化です。
invariant とは「この型のどのインスタンスも、常に満たしていなければならない条件」です。

```rust
/// 循環バッファ
///
/// # Invariants
///
/// 1. `head < capacity`
/// 2. `tail < capacity`
/// 3. `len <= capacity`
/// 4. `ptr` は `capacity` 個の `T` を保持できるアラインされたメモリを指す
/// 5. インデックス `[head, head+len) mod capacity` の範囲の要素は初期化済み
pub struct RingBuffer<T> {
    ptr: std::ptr::NonNull<T>,
    capacity: usize,
    head: usize,  // 読み出し位置
    len: usize,   // 現在の要素数
}
```

invariant を満たす状態を維持するために、`unsafe` な操作は常に invariant を回復してから制御を返す必要があります。

---

## `NonNull<T>` の活用

`*mut T` の代わりに `NonNull<T>` を使うと、**null でないことが型で保証**され、
`Option<NonNull<T>>` を `*mut T` のサイズで表現できます（ニッチ最適化）。

```rust
use std::ptr::NonNull;

struct MyBox<T> {
    ptr: NonNull<T>,
}

impl<T> MyBox<T> {
    fn new(val: T) -> Self {
        let layout = std::alloc::Layout::new::<T>();
        let raw_ptr = unsafe {
            // SAFETY: layout のサイズは 0 でない（T: Sized）
            std::alloc::alloc(layout) as *mut T
        };

        // alloc が null を返した場合（OOM）はハンドル
        let ptr = NonNull::new(raw_ptr).unwrap_or_else(|| {
            std::alloc::handle_alloc_error(layout)
        });

        // SAFETY: ptr は有効なアラインされたメモリ。まだ初期化されていないので write を使う
        unsafe { ptr.as_ptr().write(val) };

        MyBox { ptr }
    }

    fn get(&self) -> &T {
        // SAFETY: ptr は有効な T を指し、&self が生きている間は有効
        unsafe { self.ptr.as_ref() }
    }

    fn get_mut(&mut self) -> &mut T {
        // SAFETY: &mut self により排他アクセス保証
        unsafe { self.ptr.as_mut() }
    }
}

impl<T> Drop for MyBox<T> {
    fn drop(&mut self) {
        unsafe {
            // SAFETY: ptr は有効な T を指す。drop 後は二度と使われない
            std::ptr::drop_in_place(self.ptr.as_ptr());
            std::alloc::dealloc(
                self.ptr.as_ptr() as *mut u8,
                std::alloc::Layout::new::<T>(),
            );
        }
    }
}

fn main() {
    let mut b = MyBox::new(String::from("hello"));
    println!("{}", b.get());        // hello
    b.get_mut().push_str(", world");
    println!("{}", b.get());        // hello, world
    // Drop が自動で呼ばれる
}
```

### `NonNull<T>` と `*mut T` の比較

```
*mut T                 NonNull<T>
┌──────────────┐      ┌──────────────────────────────┐
│ null の可能性│      │ null でないことが保証         │
│ size = usize │      │ size = usize                  │
│              │      │ Option<NonNull<T>> もサイズ同 │
│              │      │ （ニッチ最適化が効く）         │
└──────────────┘      └──────────────────────────────┘
```

---

## 実例: スプリットバッファ実装

標準ライブラリの `slice::split_at_mut` の動作原理を自分で実装します。
これは「1つの可変スライスを重複しない2つの可変スライスに分割する」操作です。

### なぜこれが unsafe を必要とするか

```rust
fn split_at_mut_naive(slice: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let left = &mut slice[..mid];
    // let right = &mut slice[mid..]; // エラー! slice はすでに借用済み
    // コンパイラは2つの部分が重複しないと証明できない
    todo!()
}
```

### unsafe を使った正しい実装

```rust
/// スライスを mid で分割し、2つの可変スライスを返す
///
/// # Panics
///
/// `mid > slice.len()` の場合パニック
pub fn split_at_mut<T>(slice: &mut [T], mid: usize) -> (&mut [T], &mut [T]) {
    let len = slice.len();
    let ptr = slice.as_mut_ptr();

    assert!(mid <= len, "mid ({}) > len ({})", mid, len);

    // SAFETY:
    // - ptr は有効な T の配列を指す（スライスから取得）
    // - left:  [ptr, ptr+mid) — mid <= len なので範囲内
    // - right: [ptr+mid, ptr+len) — mid <= len なので範囲内
    // - left と right は重複しない（left は [0,mid)、right は [mid,len)）
    // - 元の slice は消費されるため、left/right と同時に使われることはない
    unsafe {
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}

fn main() {
    let mut v = vec![1, 2, 3, 4, 5, 6];
    let (left, right) = split_at_mut(&mut v, 3);

    // left と right を同時に可変参照できる!
    left[0] = 10;
    right[0] = 40;

    println!("{:?}", left);  // [10, 2, 3]
    println!("{:?}", right); // [40, 5, 6]

    drop(left);
    drop(right);
    println!("{:?}", v); // [10, 2, 3, 40, 5, 6]
}
```

---

## 実例: 簡易 Vec の実装

`Vec<T>` のコアロジックを自分で実装することで、安全な抽象化の全体像を把握します。

```rust
use std::alloc::{self, Layout};
use std::ptr::NonNull;

pub struct MyVec<T> {
    ptr: NonNull<T>,
    capacity: usize,
    len: usize,
}

// SAFETY: T が Send なら MyVec<T> もスレッド間で送れる
unsafe impl<T: Send> Send for MyVec<T> {}
// SAFETY: T が Sync なら MyVec<T> の参照を複数スレッドで共有できる
unsafe impl<T: Sync> Sync for MyVec<T> {}

impl<T> MyVec<T> {
    pub fn new() -> Self {
        MyVec {
            // SAFETY: capacity=0 なので実際にアクセスしない。NonNull::dangling は
            //         null でないが有効なアドレスとは限らない（アラインされたダングリング）
            ptr: NonNull::dangling(),
            capacity: 0,
            len: 0,
        }
    }

    pub fn push(&mut self, val: T) {
        if self.len == self.capacity {
            self.grow();
        }

        // SAFETY:
        // - grow() 後、self.len < self.capacity が保証される
        // - ptr.add(self.len) は有効なアロケーション内
        // - この位置はまだ初期化されていないので write を使う
        unsafe {
            self.ptr.as_ptr().add(self.len).write(val);
        }
        self.len += 1;
    }

    pub fn pop(&mut self) -> Option<T> {
        if self.len == 0 {
            return None;
        }
        self.len -= 1;
        // SAFETY:
        // - self.len はデクリメント後も >= 0 (usize)
        // - ptr.add(self.len) は初期化済みの要素を指す
        // - read により所有権を取り出す（その後 drop されても二重解放にならないよう len を減らした）
        Some(unsafe { self.ptr.as_ptr().add(self.len).read() })
    }

    pub fn get(&self, index: usize) -> Option<&T> {
        if index >= self.len {
            return None;
        }
        // SAFETY: index < self.len なので初期化済みの要素を指す
        Some(unsafe { &*self.ptr.as_ptr().add(index) })
    }

    pub fn len(&self) -> usize { self.len }
    pub fn is_empty(&self) -> bool { self.len == 0 }

    fn grow(&mut self) {
        let new_capacity = if self.capacity == 0 { 1 } else { self.capacity * 2 };

        let new_layout = Layout::array::<T>(new_capacity).expect("layout overflow");

        let new_ptr = if self.capacity == 0 {
            // SAFETY: new_layout はサイズ > 0 で適切にアラインされている
            unsafe { alloc::alloc(new_layout) }
        } else {
            let old_layout = Layout::array::<T>(self.capacity).unwrap();
            // SAFETY:
            // - self.ptr は old_layout で確保されたメモリを指す
            // - new_layout.size() > 0
            unsafe {
                alloc::realloc(self.ptr.as_ptr() as *mut u8, old_layout, new_layout.size())
            }
        };

        self.ptr = NonNull::new(new_ptr as *mut T)
            .unwrap_or_else(|| alloc::handle_alloc_error(new_layout));
        self.capacity = new_capacity;
    }
}

impl<T> Drop for MyVec<T> {
    fn drop(&mut self) {
        if self.capacity == 0 {
            return; // ダングリングポインタなのでアクセスしない
        }

        // 各要素のデストラクタを呼ぶ
        // SAFETY: [0, self.len) の要素はすべて初期化済み
        unsafe {
            for i in 0..self.len {
                std::ptr::drop_in_place(self.ptr.as_ptr().add(i));
            }
            alloc::dealloc(
                self.ptr.as_ptr() as *mut u8,
                Layout::array::<T>(self.capacity).unwrap(),
            );
        }
    }
}

fn main() {
    let mut v: MyVec<String> = MyVec::new();
    v.push(String::from("hello"));
    v.push(String::from("world"));
    v.push(String::from("!"));

    println!("len = {}", v.len()); // 3
    println!("{:?}", v.get(0));    // Some("hello")
    println!("{:?}", v.pop());     // Some("!")
    println!("len = {}", v.len()); // 2
}
// Drop が自動で呼ばれ、各 String のメモリも正しく解放される
```

---

## 安全な抽象化のパターン集

### パターン1: 前提条件を `assert!` で事前チェック

```rust
pub fn read_u32_le(buf: &[u8], offset: usize) -> u32 {
    // 安全: パニックするが UB は起こさない
    assert!(offset + 4 <= buf.len(), "buffer too short");

    // SAFETY: assert! により offset + 4 <= buf.len() 保証
    unsafe {
        let ptr = buf.as_ptr().add(offset) as *const u32;
        // u32 は1バイトアラインでも読める（unaligned read）
        ptr.read_unaligned()
    }
}

fn main() {
    let buf = [0x78u8, 0x56, 0x34, 0x12, 0xFF];
    println!("0x{:08X}", read_u32_le(&buf, 0)); // 0x12345678
}
```

### パターン2: builder パターンで invariant を構築

```rust
struct FixedBuffer {
    data: Box<[u8]>,
    filled: usize,
}

struct FixedBufferBuilder {
    capacity: usize,
}

impl FixedBufferBuilder {
    pub fn with_capacity(n: usize) -> Self {
        FixedBufferBuilder { capacity: n }
    }

    pub fn build(self) -> FixedBuffer {
        FixedBuffer {
            data: vec![0u8; self.capacity].into_boxed_slice(),
            filled: 0,
        }
    }
}

impl FixedBuffer {
    pub fn write(&mut self, bytes: &[u8]) -> usize {
        let available = self.data.len() - self.filled;
        let to_write = bytes.len().min(available);
        // safe: スライスの範囲チェックは Rust が自動で行う
        self.data[self.filled..self.filled + to_write]
            .copy_from_slice(&bytes[..to_write]);
        self.filled += to_write;
        to_write
    }

    /// 書き込み済みデータを返す（unsafe 不要）
    pub fn as_filled(&self) -> &[u8] {
        &self.data[..self.filled]
    }
}

fn main() {
    let mut buf = FixedBufferBuilder::with_capacity(16).build();
    buf.write(b"hello");
    buf.write(b", world");
    println!("{}", std::str::from_utf8(buf.as_filled()).unwrap());
    // hello, world
}
```

### パターン3: `unsafe` のカプセル化スコープを型で明示

```rust
use std::mem::MaybeUninit;

/// 未初期化の配列を安全に初期化するユーティリティ
fn init_array<T, F, const N: usize>(init: F) -> [T; N]
where
    F: Fn(usize) -> T,
{
    // MaybeUninit: 未初期化でもコンパイラが許可する特殊型
    let mut arr: [MaybeUninit<T>; N] = unsafe {
        // SAFETY: MaybeUninit の配列は未初期化が許可される
        MaybeUninit::uninit().assume_init()
    };

    for (i, slot) in arr.iter_mut().enumerate() {
        slot.write(init(i));
    }

    // SAFETY: すべてのスロットが write() で初期化された
    unsafe { std::mem::transmute_copy(&arr) }
    // Note: transmute_copy は transmute のサイズ不一致を回避
}

fn main() {
    let squares: [u32; 8] = init_array(|i| (i * i) as u32);
    println!("{:?}", squares); // [0, 1, 4, 9, 16, 25, 36, 49]

    let greetings: [String; 3] = init_array(|i| format!("item_{}", i));
    println!("{:?}", greetings); // ["item_0", "item_1", "item_2"]
}
```

---

## invariant 違反のデバッグ

`debug_assert!` を使うと、デバッグビルド時だけ invariant チェックを有効にできます。

```rust
pub struct SortedVec<T: Ord>(Vec<T>);

impl<T: Ord> SortedVec<T> {
    pub fn new() -> Self { SortedVec(Vec::new()) }

    pub fn insert(&mut self, val: T) {
        let pos = self.0.partition_point(|x| x < &val);
        self.0.insert(pos, val);
        // invariant チェック（デバッグビルドのみ）
        debug_assert!(self.0.windows(2).all(|w| w[0] <= w[1]),
            "SortedVec invariant violated after insert");
    }

    /// # Safety
    /// 呼び出し側は val がソート順を壊さないことを保証すること
    pub unsafe fn push_unchecked(&mut self, val: T) {
        self.0.push(val);
        // debug_assert で事後条件を検証（デバッグ時のみコスト発生）
        debug_assert!(self.0.windows(2).all(|w| w[0] <= w[1]),
            "SortedVec invariant violated: push_unchecked の契約違反");
    }

    pub fn as_slice(&self) -> &[T] { &self.0 }
}

fn main() {
    let mut sv = SortedVec::new();
    sv.insert(3);
    sv.insert(1);
    sv.insert(4);
    sv.insert(1);
    println!("{:?}", sv.as_slice()); // [1, 1, 3, 4]

    // 安全な push_unchecked の使用例
    unsafe {
        // SAFETY: 5 は現在の最大値 4 より大きいのでソート順を壊さない
        sv.push_unchecked(5);
    }
    println!("{:?}", sv.as_slice()); // [1, 1, 3, 4, 5]
}
```

---

## チェックリスト: 安全な抽象化の設計

```
□ 型の invariant をドキュメントとして明記したか？
□ unsafe ブロックごとに // SAFETY: コメントを書いたか？
□ 安全な公開 API は invariant を破壊しないか？
□ Drop 実装は二重解放・リークを起こさないか？
□ Send/Sync の手動実装が必要な場合、その根拠はあるか？
□ debug_assert! で invariant の事後チェックを入れたか？
□ Miri (unsafe コード検証ツール) でテストしたか？
```

---

## Miri によるテスト

[Miri](https://github.com/rust-lang/miri) は unsafe コードの未定義動作を検出する Rust のインタープリタです。

```bash
# Miri のインストール
rustup component add miri

# テスト実行
cargo miri test

# 単一バイナリの実行
cargo miri run
```

Miri が検出できるエラー:

- 境界外アクセス
- ダングリングポインタのデリファレンス
- 未初期化メモリの読み取り
- アラインメント違反
- データ競合

---

## まとめ

安全な抽象化は「unsafe をゼロにする」のではなく、「unsafe のリスクを局所化・管理する」技術です。

```
unsafe な核心
    ↓ NonNull<T> で null 安全を型で保証
    ↓ invariant を文書化
    ↓ // SAFETY: コメントで根拠を明示
    ↓ debug_assert! で実行時検証
    ↓ 安全な公開 API でラップ
    ↓ Miri でテスト
外から見れば完全に安全なライブラリ
```

---

## 演習

1. `NonNull<T>` を使った `Stack<T>` (LIFO) を実装してください。`push`、`pop`、`peek` を安全な API として公開してください。

2. 以下の invariant を持つ `CircularBuffer<T>` を実装してください。
   - `head < capacity`
   - `len <= capacity`
   - `[head, head+len) mod capacity` の範囲が初期化済み

3. Miri を使って自分の実装をテストし、検出された問題を修正してください。
