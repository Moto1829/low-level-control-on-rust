# 02: unsafe 関数・unsafe トレイト

## unsafe fn とは

`unsafe fn` は「この関数を正しく呼び出すために、呼び出し側が守らなければならない条件（**契約・invariant**）が存在する」ことを型システムで表明する仕組みです。

```rust
//  unsafe fn の定義
//  ↓
unsafe fn dangerous(ptr: *mut u8, len: usize) {
    // ここでは unsafe ブロックなしで unsafe 操作ができる
    for i in 0..len {
        *ptr.add(i) = 0;
    }
}

fn main() {
    let mut buf = vec![1u8, 2, 3, 4];
    // 呼び出し側は unsafe ブロックで「契約を守る」と宣言する
    unsafe {
        // SAFETY: ptr は buf の先頭、len は buf.len() と等しいので範囲外アクセスなし
        dangerous(buf.as_mut_ptr(), buf.len());
    }
    println!("{:?}", buf); // [0, 0, 0, 0]
}
```

```
      呼び出し側                    関数定義側
┌─────────────────────┐      ┌─────────────────────────────┐
│  unsafe { ... }     │ ───> │  unsafe fn foo(...)         │
│  // SAFETY: ...     │      │  // 契約: ... を満たすこと   │
│  foo(...);          │      │  { 内部は自由に unsafe 操作 } │
└─────────────────────┘      └─────────────────────────────┘
         ↑
  unsafe ブロックは
  「私が契約を守ります」
  という宣言
```

---

## unsafe fn の実践例

### 標準ライブラリの unsafe fn

```rust
fn main() {
    let v = vec![10, 20, 30, 40, 50];

    // get_unchecked: 境界チェックなしのインデックスアクセス
    // 契約: index < v.len() であること
    let val = unsafe {
        // SAFETY: 2 < 5 (v.len()) なので範囲内
        v.get_unchecked(2)
    };
    println!("{}", val); // 30

    // str::from_utf8_unchecked: UTF-8 チェックなしの変換
    // 契約: バイト列が有効な UTF-8 であること
    let bytes: &[u8] = b"hello world";
    let s = unsafe {
        // SAFETY: ASCII のみなので有効な UTF-8
        std::str::from_utf8_unchecked(bytes)
    };
    println!("{}", s);
}
```

### 独自の unsafe fn を定義する

```rust
/// メモリ上の連続した T の配列から &[T] を作る
///
/// # Safety
///
/// - `ptr` は null であってはならない
/// - `ptr` は `len * size_of::<T>()` バイトの有効なメモリを指していなければならない
/// - `T` のアラインメント要件を満たしていること
/// - 返したスライスの生存期間中、データは変更されないこと（不変スライスの場合）
unsafe fn make_slice<'a, T>(ptr: *const T, len: usize) -> &'a [T] {
    std::slice::from_raw_parts(ptr, len)
}

fn main() {
    let data: [i32; 4] = [1, 2, 3, 4];
    let slice = unsafe {
        // SAFETY: data は有効な配列でアラインメントも正しい
        make_slice(data.as_ptr(), data.len())
    };
    println!("{:?}", slice); // [1, 2, 3, 4]
}
```

---

## unsafe ブロックのスコープを最小化する

`unsafe` ブロックを広く取ると、どのコードが「本当に unsafe な操作」なのかが不明瞭になります。
スコープは**常に最小限**にするべきです。

### 悪い例: unsafe が広すぎる

```rust
fn process_data(data: &mut Vec<u8>, value: u8) {
    unsafe {
        // これは unsafe 不要
        let len = data.len();
        let capacity = data.capacity();
        println!("len={}, cap={}", len, capacity);

        // これは unsafe 必要
        let ptr = data.as_mut_ptr();
        for i in 0..len {
            *ptr.add(i) = value;
        }

        // これも unsafe 不要
        println!("done");
    }
}
```

### 良い例: unsafe を最小限に

```rust
fn process_data(data: &mut Vec<u8>, value: u8) {
    let len = data.len();          // safe
    let capacity = data.capacity(); // safe
    println!("len={}, cap={}", len, capacity);

    let ptr = data.as_mut_ptr();   // safe（ポインタ取得だけ）
    // SAFETY: i < len なので ptr.add(i) は Vec の有効な範囲内
    unsafe {
        for i in 0..len {
            ptr.add(i).write(value);
        }
    }

    println!("done"); // safe
}

fn main() {
    let mut v = vec![1, 2, 3, 4, 5];
    process_data(&mut v, 99);
    println!("{:?}", v); // [99, 99, 99, 99, 99]
}
```

---

## unsafe trait

`unsafe trait` は「このトレイトを正しく実装するために、実装側が守らなければならない不変条件がある」ことを表明します。

代表例が `Send` と `Sync` です。

```rust
// Send: 型の所有権をスレッド間で転送できる
// Sync: 型の共有参照を複数スレッドで同時に使える
//
// これらは「コンパイラが自動実装」または「unsafe impl で明示的に実装」

// 自動実装の例（通常は自動）
struct MySafeStruct {
    data: Vec<i32>,
}
// Vec<i32> が Send + Sync なので MySafeStruct も自動で Send + Sync

// 自動実装されない例
use std::cell::UnsafeCell;

struct MyWrapper<T>(UnsafeCell<T>);

// UnsafeCell は Sync でないため MyWrapper も自動では Sync にならない
// 内部で適切な同期を保証するなら手動で実装できる
unsafe impl<T: Send> Sync for MyWrapper<T> {
    // 実装側が「複数スレッドから同時アクセスしても安全」を保証する
}
```

### 独自の unsafe trait を定義する

```rust
/// このトレイトを実装する型は、バイト列として安全にシリアライズ可能でなければならない
/// （パディングバイトが存在しない、ポインタを含まないなど）
///
/// # Safety
///
/// 実装型は以下を満たさなければならない:
/// - `repr(C)` または `repr(transparent)` である
/// - パディングバイトを含まない
/// - ポインタや参照を含まない
unsafe trait PlainOldData: Sized {
    fn as_bytes(&self) -> &[u8] {
        // SAFETY: PlainOldData の契約により、self は生バイトとして安全に読める
        unsafe {
            std::slice::from_raw_parts(
                self as *const Self as *const u8,
                std::mem::size_of::<Self>(),
            )
        }
    }
}

#[repr(C)]
struct Pixel {
    r: u8,
    g: u8,
    b: u8,
    a: u8,
}

// Pixel はパディングなし・ポインタなし・repr(C) なので契約を満たす
unsafe impl PlainOldData for Pixel {}

fn main() {
    let p = Pixel { r: 255, g: 128, b: 0, a: 255 };
    let bytes = p.as_bytes();
    println!("{:?}", bytes); // [255, 128, 0, 255]
}
```

---

## unsafe ブロックのテクニック集

### テクニック1: 結果だけを unsafe から取り出す

```rust
fn get_element(slice: &[i32], index: usize) -> i32 {
    assert!(index < slice.len(), "index out of bounds");

    // 境界チェック済みなので unsafe が安全
    // SAFETY: assert! により index < slice.len() が保証済み
    unsafe { *slice.get_unchecked(index) }
}

fn main() {
    let v = vec![10, 20, 30];
    println!("{}", get_element(&v, 1)); // 20
}
```

### テクニック2: ヘルパー関数に抽出する

```rust
// unsafe な操作をカプセル化した小さなヘルパー
#[inline]
unsafe fn write_u32_le(ptr: *mut u8, val: u32) {
    // SAFETY: 呼び出し側が ptr + 4 バイトの有効性を保証する
    ptr.write(val as u8);
    ptr.add(1).write((val >> 8) as u8);
    ptr.add(2).write((val >> 16) as u8);
    ptr.add(3).write((val >> 24) as u8);
}

fn encode_u32_le(buf: &mut [u8], offset: usize, val: u32) {
    assert!(offset + 4 <= buf.len(), "buffer too small");
    // SAFETY: assert! により offset + 4 <= buf.len() 保証済み
    unsafe {
        write_u32_le(buf.as_mut_ptr().add(offset), val);
    }
}

fn main() {
    let mut buf = vec![0u8; 8];
    encode_u32_le(&mut buf, 0, 0xDEADBEEF);
    encode_u32_le(&mut buf, 4, 0x12345678);
    println!("{:02x?}", buf);
    // [ef, be, ad, de, 78, 56, 34, 12]  (リトルエンディアン)
}
```

### テクニック3: 型状態パターンで unsafe の前提を型で保証

```rust
use std::marker::PhantomData;

// 未検証状態
struct Unvalidated;
// 検証済み状態
struct Validated;

struct Buffer<State> {
    data: Vec<u8>,
    _state: PhantomData<State>,
}

impl Buffer<Unvalidated> {
    fn new(data: Vec<u8>) -> Self {
        Buffer { data, _state: PhantomData }
    }

    fn validate(self) -> Option<Buffer<Validated>> {
        // UTF-8 検証など
        if self.data.is_ascii() {
            Some(Buffer { data: self.data, _state: PhantomData })
        } else {
            None
        }
    }
}

impl Buffer<Validated> {
    // Validated 状態のときだけ呼べる（unsafe 不要！）
    fn as_str(&self) -> &str {
        // SAFETY: validate() で ASCII (= 有効な UTF-8) を確認済み
        unsafe { std::str::from_utf8_unchecked(&self.data) }
    }
}

fn main() {
    let buf = Buffer::new(b"hello".to_vec());
    if let Some(valid) = buf.validate() {
        println!("{}", valid.as_str()); // hello
    }
}
```

---

## can_call_unsafe の判断フロー

```
unsafe fn を呼ぶ前に自問する:

1. 関数のドキュメントに書かれた Safety 条件を読んだか？
         │
         ▼
2. 各条件がコードの文脈で満たされているか確認できるか？
         │ Yes
         ▼
3. その根拠を // SAFETY: コメントで書き下したか？
         │ Yes
         ▼
4. unsafe { ... } の中に unsafe 操作だけが入っているか？
         │ Yes
         ▼
    ✅ 呼んでよい
```

---

## まとめ

| 概念 | 説明 |
|------|------|
| `unsafe fn` | 契約を守らないと UB になる関数。呼び出し側が unsafe ブロックで責任を取る |
| `unsafe trait` | 実装側に安全性の不変条件を課すトレイト。`unsafe impl` で実装する |
| `unsafe {}` ブロック | 「この範囲内の unsafe 操作の安全性を私が保証する」という宣言 |
| `// SAFETY:` コメント | なぜ unsafe が安全かを文書化する Rust のコーディング規約 |

**原則: unsafe ブロックは小さく・コメントは詳しく・テストは念入りに。**

---

## 演習

1. 境界チェックなしで `&[T]` の要素にアクセスする `unsafe fn` を定義し、その関数を安全な wrapper で包んでください。
2. 以下のコードの `unsafe` ブロックを可能な限り小さくリファクタリングしてください。

```rust
fn fill_with_index(v: &mut Vec<u32>) {
    unsafe {
        let len = v.len();
        println!("filling {} elements", len);
        let ptr = v.as_mut_ptr();
        for i in 0..len {
            *ptr.add(i) = i as u32;
        }
        let sum: u32 = v.iter().sum();
        println!("sum = {}", sum);
    }
}
```

3. `unsafe trait ByteRepr` を定義し、`u8`、`u32`、`f64` に対して安全に実装してください（`f64` の実装には注意が必要です）。
