# Pin・自己参照構造体・PhantomData

Rust の所有権システムは値を自由にムーブできることを前提としています。
しかし非同期処理（`Future`）や自己参照構造体では、ムーブするとポインタが無効になるという問題があります。
`Pin<P>` と `PhantomData<T>` はこの問題を型システムレベルで解決するための仕組みです。

---

## 1. 自己参照構造体が安全でない理由

### ムーブと自己参照の問題

```text
ムーブ前:
┌─────────────────────────────────────┐
│ SelfRef { data: [1,2,3], ptr: ──────┼──→ data[0] (同じ構造体内)
└─────────────────────────────────────┘
アドレス: 0x1000

ムーブ後（メモリコピー）:
┌─────────────────────────────────────┐
│ SelfRef { data: [1,2,3], ptr: ──────┼──→ 0x1000 (古い場所！ 無効！)
└─────────────────────────────────────┘
アドレス: 0x2000        ↑ data はもう 0x2000 にある
```

値をムーブすると **メモリコピー** が行われます。
`ptr` フィールドが古いアドレスを指したままになり、ダングリングポインタが生まれます。

### 素朴な実装が危険である例

```rust
// これは安全でない自己参照構造体（実際には動作しない）
struct BadSelfRef {
    data: [i32; 3],
    ptr: *const i32, // data[0] を指すつもり
}

fn main() {
    let mut s = BadSelfRef {
        data: [10, 20, 30],
        ptr: std::ptr::null(),
    };

    // ptr を自分の data[0] に向ける
    s.ptr = s.data.as_ptr();
    println!("ムーブ前: ptr が指す値 = {}", unsafe { *s.ptr }); // 10

    // ここで s をムーブすると ptr が無効になる
    let s2 = s; // ムーブ発生: data は 0x???? にコピーされた
    // しかし ptr はコピー前のアドレスを指している

    // UB: ptr は無効なアドレスを指している可能性がある
    // println!("{}", unsafe { *s2.ptr }); // ← 未定義動作！
    let _ = s2;
}
```

---

## 2. `Pin<P>` の仕組み

`Pin<P>` は「このポインタが指す値をムーブしない」という保証をコンパイル時に強制します。

```text
Pin<P> のモデル

通常の P (例: Box<T>):          Pin<Box<T>>:
┌──────┐    ┌────────┐          ┌─────────┐   ┌────────┐
│ Box  │───>│   T    │          │ Pin     │──>│   T    │
│      │    │(移動可)│          │(Box<T>) │   │(固定!) │
└──────┘    └────────┘          └─────────┘   └────────┘
              ↑ムーブOK                          ↑ムーブ禁止

Pin は T へのアクセスを &T (不変参照) に制限する。
&mut T を得るには unsafe か T: Unpin が必要。
```

### `Unpin` トレイトと `!Unpin`

```rust
use std::marker::Unpin;

// Unpin とは「ムーブしても安全」という印（マーカートレイト）
// 標準ライブラリの型はほぼすべて Unpin を実装している

// i32, String, Vec<T> などは Unpin
fn show_unpin<T: Unpin>(_: &T) {
    println!("この型はムーブしても安全 (Unpin)");
}

fn main() {
    show_unpin(&42i32);
    show_unpin(&String::from("hello"));
    show_unpin(&vec![1, 2, 3]);
}
```

```rust
use std::marker::PhantomPinned;

// !Unpin にする: PhantomPinned を埋め込む
struct MyFuture {
    state: u32,
    _pin: PhantomPinned, // これにより MyFuture は !Unpin になる
}

// Unpin を実装していないことを確認（コンパイルエラーになるはずのコード）
fn require_unpin<T: Unpin>(_: T) {}

fn main() {
    let f = MyFuture { state: 0, _pin: PhantomPinned };
    // require_unpin(f); // コンパイルエラー: MyFuture は Unpin でない
    println!("MyFuture は !Unpin です");
    let _ = f;
}
```

### `Pin<P>` が提供する保証

```rust
use std::pin::Pin;

fn main() {
    let mut x = 42i32;

    // i32 は Unpin なので Pin<&mut i32> でも &mut i32 を安全に得られる
    let mut pinned: Pin<&mut i32> = Pin::new(&mut x);

    // Unpin な型は get_mut() で &mut T を取り出せる
    let r: &mut i32 = pinned.as_mut().get_mut();
    *r = 100;
    println!("x = {}", x); // 100

    // Unpin でない型からは get_mut() を呼べない（コンパイルエラー）
    // unsafe の get_unchecked_mut() を使う必要がある
}
```

---

## 3. `Pin<Box<T>>` と `Pin<&mut T>` の作り方

### `Pin<Box<T>>` — ヒープ上に固定する

```rust
use std::pin::Pin;

struct Counter {
    count: u32,
    self_ptr: *const u32, // count を自己参照するポインタ
}

impl Counter {
    // Pin<Box<Self>> を返す専用のコンストラクタ
    fn new() -> Pin<Box<Self>> {
        let mut boxed = Box::new(Counter {
            count: 0,
            self_ptr: std::ptr::null(),
        });

        // Box の内部アドレスへのポインタをセット（ムーブ後は無効になるが Pin で防ぐ）
        let ptr = &boxed.count as *const u32;
        boxed.self_ptr = ptr;

        // Box::into_pin で Pin<Box<T>> に変換（unsafe 不要）
        Box::into_pin(boxed)
    }

    fn increment(self: Pin<&mut Self>) {
        // Pin<&mut Self> から &mut フィールドを得るには unsafe
        unsafe {
            let this = self.get_unchecked_mut();
            this.count += 1;
            // self_ptr は変わらず count を指している（ムーブしていないから）
        }
    }

    fn value(&self) -> u32 {
        // self_ptr 経由で count を読む（自己参照の例）
        unsafe { *self.self_ptr }
    }
}

fn main() {
    let mut counter = Counter::new();

    // Pin<Box<T>> の参照から Pin<&mut T> を得る
    counter.as_mut().increment();
    counter.as_mut().increment();
    counter.as_mut().increment();

    println!("count = {}", counter.count);    // 3
    println!("via ptr = {}", counter.value()); // 3（自己参照経由）
}
```

### `Pin<&mut T>` — スタック上に固定する

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

struct StackPinned {
    data: i32,
    _pin: PhantomPinned,
}

fn process(val: Pin<&mut StackPinned>) {
    // get_unchecked_mut は unsafe が必要
    let inner = unsafe { val.get_unchecked_mut() };
    inner.data *= 2;
}

fn main() {
    // スタック上の変数を Pin する（pin! マクロを使う方法）
    // std::pin::pin! は Rust 1.68 以降で使える
    let mut val = StackPinned { data: 21, _pin: PhantomPinned };

    // スタック固定: アドレスが取れた後はムーブ不可
    let pinned: Pin<&mut StackPinned> = unsafe { Pin::new_unchecked(&mut val) };
    // val はこれ以降ムーブできない（ただし Rust コンパイラは現時点では完全には強制しない）

    process(pinned);
    println!("data = {}", val.data); // 42
}
```

---

## 4. `unsafe impl !Unpin` で自己参照構造体を安全に実装する

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;
use std::ptr::NonNull;

/// 自己参照を持つ安全な構造体
pub struct SelfReferential {
    /// 実際のデータ
    data: String,
    /// data の先頭を指す自己参照ポインタ
    /// None = まだ初期化されていない
    self_ptr: Option<NonNull<u8>>,
    /// !Unpin にする
    _pin: PhantomPinned,
}

impl SelfReferential {
    /// ヒープ上に生成して Pin で固定する
    pub fn new(s: impl Into<String>) -> Pin<Box<Self>> {
        let this = SelfReferential {
            data: s.into(),
            self_ptr: None,
            _pin: PhantomPinned,
        };
        let mut boxed = Box::pin(this);

        // ムーブが起きないと保証された後で自己参照を設定する
        let ptr = NonNull::new(boxed.data.as_ptr() as *mut u8);
        unsafe {
            boxed.as_mut().get_unchecked_mut().self_ptr = ptr;
        }

        boxed
    }

    /// 自己参照ポインタ経由で最初の文字を取得
    pub fn first_char(&self) -> Option<char> {
        let ptr = self.self_ptr?.as_ptr();
        // SAFETY: self_ptr は self.data の先頭を指し、self は Pin で固定されている
        let byte = unsafe { *ptr };
        char::from_u32(byte as u32)
    }

    /// 通常の参照で data にアクセス（こちらが推奨）
    pub fn data(&self) -> &str {
        &self.data
    }
}

fn main() {
    let sr = SelfReferential::new("Hello, Pin!");

    println!("data = {:?}", sr.data());
    println!("first_char = {:?}", sr.first_char()); // Some('H')

    // sr はムーブできない（Pin に包まれているため）
    // let moved = *sr; // コンパイルエラー: !Unpin な型は Pin から取り出せない

    // Pin<Box<T>> はヒープを指すポインタ（Box）自体はムーブできる
    let sr2 = sr; // Box のアドレスが変わるだけ。中身のアドレスは変わらない
    println!("移動後も data = {:?}", sr2.data());
    println!("移動後も first_char = {:?}", sr2.first_char()); // 引き続き Some('H')
}
```

---

## 5. `PhantomData<T>` の役割

`PhantomData<T>` は **コンパイル時にのみ存在する** ゼロサイズのフィールドです。
実際のメモリを消費せず、コンパイラに対して「この型は T を所有している」「このライフタイムに依存している」
「この型の変性（variance）はこうなっている」といった情報を伝えます。

```text
PhantomData<T> の実態

struct MyVec<T> {
    ptr: *mut T,
    len: usize,
    cap: usize,
    _marker: PhantomData<T>,  // 実行時サイズ: 0 バイト
}

size_of::<PhantomData<T>>() == 0（どんな T でも）
align_of::<PhantomData<T>>() == 1
```

### ゼロサイズであることを確認する

```rust
use std::marker::PhantomData;

struct Wrapper<T> {
    value: i32,
    _phantom: PhantomData<T>,
}

fn main() {
    use std::mem::size_of;
    println!("size_of::<Wrapper<u8>>()     = {}", size_of::<Wrapper<u8>>());   // 4
    println!("size_of::<Wrapper<String>>() = {}", size_of::<Wrapper<String>>()); // 4
    println!("size_of::<PhantomData<i32>>() = {}", size_of::<PhantomData<i32>>()); // 0

    // PhantomData はどんな型でも ZST（ゼロサイズ型）
    let _: PhantomData<String> = PhantomData;
    let _: PhantomData<*mut u8> = PhantomData;
}
```

---

## 6. `PhantomData` を使うべき場面

### 場面 1: 所有権の表明（Drop Check）

生ポインタを持つ構造体は、コンパイラが「T を所有しているかどうか」を判断できません。
`PhantomData<T>` を加えると「この構造体は T を所有している」と伝えられます。

```rust
use std::marker::PhantomData;

/// T を所有するカスタムコンテナ
struct MyBox<T> {
    ptr: *mut T,
    _owns: PhantomData<T>, // 「T を所有している」とコンパイラに伝える
}

impl<T> MyBox<T> {
    fn new(value: T) -> Self {
        let ptr = Box::into_raw(Box::new(value));
        MyBox { ptr, _owns: PhantomData }
    }

    fn get(&self) -> &T {
        unsafe { &*self.ptr }
    }
}

impl<T> Drop for MyBox<T> {
    fn drop(&mut self) {
        unsafe {
            // PhantomData<T> のおかげで Drop Checker は
            // T のドロップが MyBox のドロップより先に起きないことを保証できる
            drop(Box::from_raw(self.ptr));
        }
    }
}

fn main() {
    let b = MyBox::new(String::from("owned"));
    println!("{}", b.get());
    // b がドロップされると String も正しくドロップされる
}
```

### 場面 2: ライフタイムの伝搬

```rust
use std::marker::PhantomData;

/// 外部から借りたポインタ（所有しない）
struct BorrowedSlice<'a, T> {
    ptr: *const T,
    len: usize,
    /// 'a を持つ参照を「借りている」と伝える
    /// PhantomData<&'a T> = 「'a の間有効な T への参照を持っている」
    _lifetime: PhantomData<&'a T>,
}

impl<'a, T> BorrowedSlice<'a, T> {
    fn from_slice(s: &'a [T]) -> Self {
        BorrowedSlice {
            ptr: s.as_ptr(),
            len: s.len(),
            _lifetime: PhantomData,
        }
    }

    fn get(&self, index: usize) -> Option<&'a T> {
        if index < self.len {
            // SAFETY: 'a のライフタイム内なので ptr は有効
            Some(unsafe { &*self.ptr.add(index) })
        } else {
            None
        }
    }
}

fn main() {
    let data = vec![10, 20, 30, 40, 50];
    let borrowed = BorrowedSlice::from_slice(&data);

    println!("borrowed[2] = {:?}", borrowed.get(2)); // Some(30)
    println!("borrowed[9] = {:?}", borrowed.get(9)); // None

    // data がドロップされた後は borrowed を使えない（ライフタイムで保証）
    drop(data);
    // borrowed.get(0); // コンパイルエラー: data のライフタイムが切れている
}
```

### 場面 3: 変性（Variance）の制御

```rust
use std::marker::PhantomData;

// 共変（covariant）: PhantomData<T> または PhantomData<&'a T>
// T のサブタイプを受け入れる
struct Covariant<T> {
    _marker: PhantomData<T>,
}

// 反変（contravariant）: PhantomData<fn(T)>
// T のスーパータイプを受け入れる
struct Contravariant<T> {
    _marker: PhantomData<fn(T)>,
}

// 不変（invariant）: PhantomData<fn(T) -> T> または PhantomData<*mut T>
// 型が完全に一致しなければならない
struct Invariant<T> {
    _marker: PhantomData<fn(T) -> T>,
}

fn main() {
    // 変性の具体例（ライフタイムで示す）
    fn takes_covariant<'short>(x: Covariant<&'short str>) {
        let _ = x;
    }

    let long_lived = String::from("hello");
    let long_ref: &str = &long_lived;
    let c: Covariant<&str> = Covariant { _marker: PhantomData };

    // 共変: より長いライフタイムを持つ参照を短いライフタイムとして渡せる
    takes_covariant(c);
    println!("変性の制御は PhantomData で行う");
}
```

---

## 7. 実例: raw ポインタを持つイテレータに `PhantomData` を付ける

```rust
use std::marker::PhantomData;

/// スライスを生ポインタで走査するイテレータ
pub struct RawIter<'a, T> {
    ptr: *const T,          // 現在位置
    end: *const T,          // 終端（ptr == end で終了）
    /// 'a の間有効な T のスライスを借りていることをコンパイラに伝える
    /// これがないと:
    ///   1. ライフタイムが伝搬されず use-after-free が検出されない
    ///   2. T が参照型のとき変性が正しく設定されない
    _marker: PhantomData<&'a T>,
}

impl<'a, T> RawIter<'a, T> {
    pub fn new(slice: &'a [T]) -> Self {
        let ptr = slice.as_ptr();
        let end = unsafe { ptr.add(slice.len()) };
        RawIter { ptr, end, _marker: PhantomData }
    }
}

impl<'a, T> Iterator for RawIter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        if self.ptr == self.end {
            return None;
        }
        // SAFETY: ptr は end に達するまで有効なスライス要素を指す
        let item = unsafe { &*self.ptr };
        self.ptr = unsafe { self.ptr.add(1) };
        Some(item)
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        let len = (self.end as usize - self.ptr as usize) / std::mem::size_of::<T>();
        (len, Some(len))
    }
}

// ExactSizeIterator も実装できる
impl<'a, T> ExactSizeIterator for RawIter<'a, T> {}

fn main() {
    let data = vec![1u32, 2, 3, 4, 5];
    let iter = RawIter::new(&data);

    // size_hint が正確
    println!("残り要素数: {}", iter.len());

    let collected: Vec<_> = RawIter::new(&data).collect();
    println!("収集: {:?}", collected);

    // 合計
    let sum: u32 = RawIter::new(&data).copied().sum();
    println!("合計: {}", sum); // 15

    // ライフタイム保護の確認
    // let iter = {
    //     let local = vec![10, 20, 30];
    //     RawIter::new(&local) // コンパイルエラー: local より長生きできない
    // };
    // iter.next(); // ← PhantomData がなければこれを防げない
}
```

---

## 8. `PhantomPinned` との組み合わせ

`std::marker::PhantomPinned` は `!Unpin` を実装するための専用の `PhantomData` です。

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;
use std::marker::PhantomData;

/// 自己参照 + 借用ライフタイム + !Unpin を組み合わせた構造体
struct AsyncBuffer<'a> {
    /// 実際のデータ
    buffer: Vec<u8>,
    /// buffer の一部を指す自己参照スライス（Pin で固定されるまで設定しない）
    slice: *const [u8],
    /// 'a のライフタイムを持つ外部データへの参照
    external: &'a str,
    /// !Unpin にする（自己参照があるためムーブ不可）
    _pin: PhantomPinned,
    /// 'a ライフタイムを伝搬させる（&'a str があれば不要だが明示例として）
    _lifetime: PhantomData<&'a ()>,
}

impl<'a> AsyncBuffer<'a> {
    fn new(external: &'a str, capacity: usize) -> Pin<Box<Self>> {
        let this = AsyncBuffer {
            buffer: Vec::with_capacity(capacity),
            slice: std::ptr::slice_from_raw_parts(std::ptr::null(), 0),
            external,
            _pin: PhantomPinned,
            _lifetime: PhantomData,
        };
        let mut pinned = Box::pin(this);

        // Pin された後に自己参照を設定
        unsafe {
            let this_mut = pinned.as_mut().get_unchecked_mut();
            // buffer の現在のデータ範囲を指すスライスを設定
            let buf_ptr = this_mut.buffer.as_ptr();
            let buf_len = this_mut.buffer.len();
            this_mut.slice = std::ptr::slice_from_raw_parts(buf_ptr, buf_len);
        }

        pinned
    }

    fn push(self: Pin<&mut Self>, byte: u8) {
        unsafe {
            let this = self.get_unchecked_mut();
            this.buffer.push(byte);
            // バッファが再アロケートされた場合、slice を更新する必要がある
            let buf_ptr = this.buffer.as_ptr();
            let buf_len = this.buffer.len();
            this.slice = std::ptr::slice_from_raw_parts(buf_ptr, buf_len);
        }
    }

    fn external_data(&self) -> &str {
        self.external
    }

    fn buffer_len(&self) -> usize {
        unsafe { (*self.slice).len() }
    }
}

fn main() {
    let label = String::from("my-buffer");
    let mut buf = AsyncBuffer::new(&label, 16);

    buf.as_mut().push(10);
    buf.as_mut().push(20);
    buf.as_mut().push(30);

    println!("external = {:?}", buf.external_data());
    println!("buffer_len = {}", buf.buffer_len());
    println!("buffer = {:?}", buf.buffer);

    // Pin<Box<T>> 自体は移動できる（中身のアドレスは変わらない）
    let buf2 = buf;
    println!("移動後も external = {:?}", buf2.external_data()); // 正常
}
```

### `Pin` / `PhantomData` / `PhantomPinned` の使い分け

```text
自己参照構造体を作る
    ↓
PhantomPinned を埋め込む
    → !Unpin になる（ムーブするとコンパイルエラー）

Pin<Box<T>> でヒープに固定する
    → ムーブしても Box のアドレスだけ変わり、T のアドレスは不変

PhantomData<T> を使う場面:
  1. *const T / *mut T を持ちつつ「T を所有する」と伝えたい
     → PhantomData<T>（所有権・Drop Check）
  2. 'a のライフタイムに依存すると伝えたい
     → PhantomData<&'a T>（ライフタイム伝搬）
  3. 反変にしたい
     → PhantomData<fn(T)>
  4. 不変にしたい
     → PhantomData<*mut T> または PhantomData<fn(T) -> T>
```

---

## まとめ

| 機能 | 役割 | 典型的な使い方 |
|------|------|--------------|
| `Pin<Box<T>>` | ヒープ上の値を固定 | `Box::pin(value)` |
| `Pin<&mut T>` | スタック上の値を固定 | `unsafe { Pin::new_unchecked(&mut val) }` |
| `Unpin` | ムーブしても安全な型の印 | 自動実装（ほとんどの型） |
| `PhantomPinned` | `!Unpin` にする | フィールドに `_pin: PhantomPinned` を追加 |
| `PhantomData<T>` | 所有権をコンパイラに伝える | `*mut T` を持つ構造体に `PhantomData<T>` |
| `PhantomData<&'a T>` | ライフタイムを伝搬させる | 生ポインタとライフタイムを組み合わせる |
| `PhantomData<fn(T)>` | 反変にする | コールバック型を保持する構造体 |

`Pin` と `PhantomData` は Rust の非同期処理（`Future` / `async-await`）の基盤でもあります。
これらを正しく理解することで、`unsafe` コードの安全性を型レベルで保証する設計ができるようになります。
