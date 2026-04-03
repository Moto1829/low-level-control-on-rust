# Box/Vec の内部構造と unsafe 操作

`Box<T>` や `Vec<T>` は Rust の安全な抽象ですが、内部では生ポインタとヒープアロケーションを直接操作しています。
unsafe API を通じてその内部構造に触れることで、低レベルのメモリ操作・FFI・独自コンテナの実装が可能になります。

---

## 1. `Box<T>` の内部構造

`Box<T>` の本質は「ヒープ上に確保された `T` の所有権を持つ生ポインタ」です。
標準ライブラリの内部では `Unique<T>`（`NonNull<T>` の薄いラッパー）を通じて表現されています。

```text
Stack                Heap
┌─────────────┐     ┌───────────┐
│ Box<T>      │     │     T     │
│ ptr: NonNull├────>│  (値本体) │
└─────────────┘     └───────────┘
```

### `NonNull<T>` と null チェック

```rust
use std::ptr::NonNull;

fn main() {
    // Box を生成してポインタを覗く
    let b: Box<i32> = Box::new(42);

    // Box の内部ポインタを NonNull として取得（安定 API はない、参照経由で確認）
    let ptr: *const i32 = &*b as *const i32;
    println!("ヒープアドレス: {:p}", ptr);

    // NonNull は「ヌルでないことが保証された生ポインタ」
    let nn: NonNull<i32> = NonNull::new(ptr as *mut i32).expect("null だった");
    println!("NonNull の値: {}", unsafe { *nn.as_ptr() });
}
```

> **補足**: `NonNull<T>` はヌルチェックをコンパイル時に排除できるため、
> `Option<Box<T>>` は `Option<NonNull<T>>` と同じサイズになります（ヌルポインタ最適化）。

---

## 2. `Box::into_raw` / `Box::from_raw` と所有権の移転

### `Box::into_raw` — 所有権を生ポインタへ移す

`Box::into_raw` を呼ぶと `Box` は**デストラクタなしで消滅**し、生ポインタだけが残ります。
以降のメモリ管理は呼び出し元の責任です。

```rust
fn main() {
    let b: Box<String> = Box::new(String::from("hello"));

    // Box の所有権を手放し、生ポインタを取得する
    let raw: *mut String = Box::into_raw(b);
    // ここから b は使えない（ムーブ済み）

    // 生ポインタ越しに値を操作する（unsafe）
    unsafe {
        (*raw).push_str(", world");
        println!("{}", *raw);
    }

    // 必ず Box::from_raw で所有権を取り戻してから解放する
    let _b2 = unsafe { Box::from_raw(raw) };
    // _b2 がスコープを抜けると自動でヒープ解放
}
```

### `Box::from_raw` — 生ポインタから所有権を復元する

```rust
use std::alloc::{alloc, Layout};

fn main() {
    // 手動でヒープにメモリを確保して Box に変換する
    let layout = Layout::new::<u64>();
    let ptr = unsafe { alloc(layout) as *mut u64 };
    assert!(!ptr.is_null());

    unsafe {
        // 初期化してから Box に渡す
        ptr.write(1234567890u64);
        let b: Box<u64> = Box::from_raw(ptr);
        println!("Box の値: {}", b);
        // b がドロップされると dealloc も呼ばれる
    }
}
```

### よくある誤り: 二重解放

```rust
fn double_free_example() {
    let b = Box::new(99i32);
    let raw = Box::into_raw(b);

    unsafe {
        let _b1 = Box::from_raw(raw); // OK: 一度目の解放
        // let _b2 = Box::from_raw(raw); // UB! 同じアドレスを二度解放
    }
}
```

---

## 3. `Vec<T>` の内部構造

`Vec<T>` は 3 つのフィールドで構成されています。

```text
Stack
┌──────────────────────┐
│ Vec<T>               │
│  ptr: NonNull<T> ────┼──> ヒープ上のバッファ [T; cap]
│  len: usize          │    (len 個は初期化済み)
│  cap: usize          │
└──────────────────────┘
```

| フィールド | 意味 |
|------------|------|
| `ptr` | ヒープバッファの先頭ポインタ（`NonNull<T>`） |
| `len` | 現在の要素数（初期化済み要素の個数） |
| `cap` | 確保済みの要素キャパシティ |

```rust
fn main() {
    let mut v: Vec<i32> = Vec::with_capacity(8);
    v.extend([1, 2, 3, 4]);

    println!("len={}, cap={}", v.len(), v.capacity());

    // ptr: 生ポインタとして参照
    let ptr = v.as_ptr();
    println!("先頭アドレス: {:p}", ptr);

    unsafe {
        // ptr + offset で各要素に直接アクセス
        for i in 0..v.len() {
            print!("{} ", *ptr.add(i));
        }
        println!();
    }
}
```

---

## 4. `Vec::into_raw_parts` / `Vec::from_raw_parts`

### `Vec::into_raw_parts` — 3 つのフィールドを取り出す

```rust
fn main() {
    let v: Vec<u8> = vec![10, 20, 30, 40, 50];

    // Vec を分解して生ポインタ・len・cap を取得
    let (ptr, len, cap) = v.into_raw_parts();
    // v は無効。以降、このメモリの管理は手動

    println!("ptr={:p}, len={}, cap={}", ptr, len, cap);

    unsafe {
        // ポインタ演算で各要素にアクセス
        for i in 0..len {
            print!("{} ", *ptr.add(i));
        }
        println!();

        // 必ず同じ ptr/len/cap で Vec を復元して解放
        let _v2 = Vec::from_raw_parts(ptr, len, cap);
    }
}
```

### `Vec::from_raw_parts` — 生ポインタから Vec を構築する

```rust
use std::alloc::{alloc, Layout};

fn main() {
    let cap: usize = 4;
    let layout = Layout::array::<f64>(cap).unwrap();

    unsafe {
        let ptr = alloc(layout) as *mut f64;
        assert!(!ptr.is_null());

        // 手動で初期化
        for i in 0..cap {
            ptr.add(i).write((i as f64) * 1.5);
        }

        // Vec として所有権を受け取る
        let v = Vec::from_raw_parts(ptr, cap, cap);
        println!("{:?}", v);
        // v がドロップされるとバッファも解放される
    }
}
```

> **前提条件**: `from_raw_parts` に渡すポインタは
> - グローバルアロケータで確保されたものであること
> - `T` のアラインメントを満たしていること
> - `len` 個の要素がすべて初期化済みであること
> - `cap` は確保サイズ（要素数）と一致すること

---

## 5. メモリリークと二重解放を避けるパターン

### パターン 1: `into_raw` 後は必ず `from_raw` で回収する

```rust
/// ヒープ確保した値を一時的に生ポインタとして渡し、確実に回収する
fn round_trip_ownership() {
    let boxed = Box::new(vec![1u32, 2, 3]);
    let raw = Box::into_raw(boxed);

    // --- ここで生ポインタを FFI 等に渡す ---

    // 戻ってきたら必ず Box に戻す
    let _recovered = unsafe { Box::from_raw(raw) };
    // _recovered がドロップされると解放される
}

fn main() {
    round_trip_ownership();
    println!("リークなし");
}
```

### パターン 2: `ManuallyDrop` でデストラクタを抑制する（次節で詳述）

### パターン 3: `scopeguard` 等で RAII ガードを作る

```rust
/// raw ポインタを保持し、スコープ終了時に自動解放するガード
struct RawGuard<T> {
    ptr: *mut T,
}

impl<T> Drop for RawGuard<T> {
    fn drop(&mut self) {
        if !self.ptr.is_null() {
            // 所有権を Box に戻して解放
            unsafe { drop(Box::from_raw(self.ptr)); }
        }
    }
}

fn main() {
    let b = Box::new(String::from("guard me"));
    let ptr = Box::into_raw(b);

    let _guard = RawGuard { ptr };
    // 何らかの処理...
    println!("スコープ終了時に自動解放される");
    // _guard がドロップ → Box::from_raw → String と heap メモリが解放
}
```

---

## 6. `ManuallyDrop<T>` によるデストラクタ抑制

`ManuallyDrop<T>` はコンパイラに「このデストラクタを自動で呼ばないでほしい」と伝えるラッパーです。
FFI や特殊な所有権転送パターンで使います。

```rust
use std::mem::ManuallyDrop;

fn main() {
    // 通常は Vec がスコープを抜けると dealloc される
    // ManuallyDrop で包むとデストラクタが呼ばれない
    let v: ManuallyDrop<Vec<u8>> = ManuallyDrop::new(vec![1, 2, 3, 4, 5]);

    // 中身への参照は通常通り得られる
    println!("len={}", v.len());

    // into_inner でラッパーを外す（所有権を取り戻す）
    let inner: Vec<u8> = ManuallyDrop::into_inner(v);
    println!("inner: {:?}", inner);
    // inner は通常の Vec なので drop 時に解放される
}
```

### `ManuallyDrop` + `into_raw_parts` の組み合わせ

```rust
use std::mem::ManuallyDrop;

/// Vec のバッファを「盗む」（メモリを移転する）
fn steal_buffer(v: Vec<u8>) -> (*mut u8, usize, usize) {
    // ManuallyDrop で包んでデストラクタを止める
    let mut md = ManuallyDrop::new(v);

    // 内部フィールドを直接取り出す
    let ptr = md.as_mut_ptr();
    let len = md.len();
    let cap = md.capacity();

    // 呼び出し元がメモリ管理の責任を持つ
    (ptr, len, cap)
}

fn main() {
    let v = vec![10u8, 20, 30, 40];
    let (ptr, len, cap) = steal_buffer(v);

    unsafe {
        // ポインタを操作
        for i in 0..len {
            print!("{} ", *ptr.add(i));
        }
        println!();

        // 使い終わったら Vec に戻して解放
        let _restored = Vec::from_raw_parts(ptr, len, cap);
    }
}
```

### C ライブラリへのバッファ受け渡し例

```rust
use std::mem::ManuallyDrop;

// 仮の C 関数シグネチャ（実際は extern "C" ブロック内）
// void c_process(uint8_t* buf, size_t len);
extern "C" {
    // fn c_process(buf: *mut u8, len: usize);
}

fn send_to_c(data: Vec<u8>) {
    let mut md = ManuallyDrop::new(data);
    let ptr = md.as_mut_ptr();
    let len = md.len();

    // C 関数にポインタを渡す（C 側はこのバッファを解放しない約束）
    unsafe {
        // c_process(ptr, len);
        println!("C に渡すポインタ: {:p}, len={}", ptr, len);
    }

    // 処理後に Rust 側で解放
    unsafe {
        Vec::from_raw_parts(ptr, len, md.capacity());
    }
}

fn main() {
    let data: Vec<u8> = b"Hello from Rust".to_vec();
    send_to_c(data);
}
```

---

## 7. 実践例: カスタムスタックの実装

`Box::into_raw` / `Box::from_raw` を使ってリンクリストベースのスタックを実装します。

```rust
use std::ptr;

struct Node<T> {
    value: T,
    next: *mut Node<T>,
}

pub struct Stack<T> {
    head: *mut Node<T>,
    len: usize,
}

impl<T> Stack<T> {
    pub fn new() -> Self {
        Stack { head: ptr::null_mut(), len: 0 }
    }

    pub fn push(&mut self, value: T) {
        let node = Box::new(Node {
            value,
            next: self.head,
        });
        self.head = Box::into_raw(node);
        self.len += 1;
    }

    pub fn pop(&mut self) -> Option<T> {
        if self.head.is_null() {
            return None;
        }
        unsafe {
            // 先頭ノードの所有権を Box に戻す
            let node = Box::from_raw(self.head);
            self.head = node.next;
            self.len -= 1;
            Some(node.value) // node がドロップされ Node のメモリは解放される
        }
    }

    pub fn peek(&self) -> Option<&T> {
        unsafe { self.head.as_ref().map(|n| &n.value) }
    }

    pub fn len(&self) -> usize {
        self.len
    }
}

impl<T> Drop for Stack<T> {
    fn drop(&mut self) {
        // 残っているすべてのノードを解放
        while self.pop().is_some() {}
    }
}

fn main() {
    let mut s: Stack<i32> = Stack::new();
    s.push(1);
    s.push(2);
    s.push(3);

    println!("len={}", s.len());           // 3
    println!("peek={:?}", s.peek());       // Some(3)
    println!("pop={:?}", s.pop());         // Some(3)
    println!("pop={:?}", s.pop());         // Some(2)
    println!("pop={:?}", s.pop());         // Some(1)
    println!("pop={:?}", s.pop());         // None
}
```

---

## まとめ

| 操作 | 用途 |
|------|------|
| `Box::into_raw` / `from_raw` | Box の所有権を生ポインタと相互変換 |
| `Vec::into_raw_parts` / `from_raw_parts` | Vec のバッファを手動管理 |
| `ManuallyDrop<T>` | デストラクタを意図的に止める |
| `NonNull<T>` | ヌルでないポインタの型安全な表現 |

これらの API は `unsafe` であり、誤用するとメモリリーク・二重解放・未定義動作を引き起こします。
使用する際は「誰がいつ解放するか」を明示的に設計し、必ずテストと Miri による検証を行ってください。

```bash
# Miri で未定義動作を検出する
cargo +nightly miri test
```
