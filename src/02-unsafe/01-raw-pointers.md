# 01: 生ポインタ (Raw Pointers)

## 生ポインタとは

Rust には C/C++ と同様の生ポインタ型が存在します。

| 型 | 意味 |
|----|------|
| `*const T` | T への読み取り専用の生ポインタ |
| `*mut T` | T への読み書き可能な生ポインタ |

生ポインタは Rust の参照（`&T` / `&mut T`）とは根本的に異なります。

```
参照 (&T / &mut T)                生ポインタ (*const T / *mut T)
┌──────────────────────────┐      ┌──────────────────────────────────┐
│ 常に有効なメモリを指す    │      │ null でもよい                     │
│ 生存期間がコンパイル時保証│      │ ダングリングでもよい              │
│ エイリアシング規則を守る  │      │ 複数の可変ポインタが共存できる    │
│ 自動デリファレンス        │      │ デリファレンスは unsafe が必要    │
└──────────────────────────┘      └──────────────────────────────────┘
```

---

## 生ポインタの作成

生ポインタを**作るだけ**は safe なコードでも可能です。
危険なのは**デリファレンス（間接参照）する**ときです。

### 参照から生ポインタへ

```rust
fn main() {
    let x: i32 = 42;
    let y: i32 = 100;

    // &x を *const i32 にキャスト（safe）
    let ptr_const: *const i32 = &x;

    // &mut y を *mut i32 にキャスト（safe）
    let mut z: i32 = 200;
    let ptr_mut: *mut i32 = &mut z;

    println!("ptr_const のアドレス: {:p}", ptr_const);
    println!("ptr_mut のアドレス:   {:p}", ptr_mut);

    // デリファレンスは unsafe
    unsafe {
        println!("*ptr_const = {}", *ptr_const); // 42
        println!("*ptr_mut   = {}", *ptr_mut);   // 200

        *ptr_mut = 999;
        println!("z after write: {}", z);         // 999
    }
}
```

### アドレス値から生ポインタへ

```rust
fn main() {
    // 整数リテラルからポインタを作る（非常に危険）
    let dangerous: *const i32 = 0x12345678 as *const i32;

    // アドレスを表示するだけなら safe
    println!("address: {:p}", dangerous);

    // デリファレンスは unsafe（セグフォルト・UB の可能性大）
    // unsafe { println!("{}", *dangerous); } // 絶対にやらない
}
```

---

## 参照と生ポインタの違いを体感する

```rust
fn main() {
    let mut data = vec![1, 2, 3];

    // 参照では同時に可変・不変参照は持てない
    // let r1 = &data;
    // let r2 = &mut data; // コンパイルエラー!

    // 生ポインタなら（作るだけなら）コンパイラは通す
    let ptr_const: *const Vec<i32> = &data;
    let ptr_mut: *mut Vec<i32> = &mut data;

    // ただし、両方を同時にデリファレンスして使うのは UB
    unsafe {
        // これは技術的には未定義動作（UB）になりうる
        // println!("{:?} {:?}", *ptr_const, *ptr_mut);

        // 安全な使い方: 一方だけ使う
        println!("via const ptr: {:?}", &*ptr_const);
        (*ptr_mut).push(4);
        println!("via mut ptr: {:?}", &*ptr_mut);
    }
}
```

---

## ヌルポインタ

`*const T` / `*mut T` は null になれます。デリファレンス前に必ずチェックが必要です。

```rust
use std::ptr;

fn safe_deref(ptr: *const i32) -> Option<i32> {
    if ptr.is_null() {
        None
    } else {
        // SAFETY: null チェック済み。ポインタが有効なメモリを指すことは
        //         呼び出し側が保証しなければならない
        Some(unsafe { *ptr })
    }
}

fn main() {
    let value: i32 = 42;
    let valid_ptr: *const i32 = &value;
    let null_ptr: *const i32 = ptr::null();

    println!("{:?}", safe_deref(valid_ptr)); // Some(42)
    println!("{:?}", safe_deref(null_ptr));  // None

    // null ポインタの作り方
    let null_mut: *mut i32 = ptr::null_mut();
    println!("is null: {}", null_mut.is_null()); // true
}
```

---

## ダングリングポインタ

ダングリングポインタとは、**解放済みまたはスコープを外れたメモリを指すポインタ**です。
これをデリファレンスするのは最も危険な未定義動作の一つです。

```rust
fn dangling_example() -> *const i32 {
    let x = 42; // スタック上に確保
    &x as *const i32
    // ← この関数が返った時点で x はスコープを外れ、ポインタは dangling になる
}

fn main() {
    let ptr = dangling_example();

    // ptr はすでに無効なメモリを指している！
    // unsafe {
    //     println!("{}", *ptr); // 未定義動作: クラッシュ・ゴミ値・etc
    // }

    println!("dangling ptr address: {:p}", ptr); // アドレス表示は safe だが...

    // 参照では同じことをするとコンパイルエラーになる
    // fn dangling_ref() -> &i32 {
    //     let x = 42;
    //     &x  // error[E0106]: missing lifetime specifier
    // }
}
```

**コンパイラによる保護の比較:**

```
&T (参照)          *const T (生ポインタ)
   ↓                      ↓
コンパイル時に         実行時まで
ダングリングを          検知されない
検知する              → UB / クラッシュ
```

---

## アドレス演算とポインタ演算

C の `ptr + n` に相当する操作は Rust では `add`、`offset`、`sub` メソッドで行います。

```rust
fn main() {
    let arr: [i32; 5] = [10, 20, 30, 40, 50];
    let base: *const i32 = arr.as_ptr();

    unsafe {
        // add(n): n 要素分進む（型のサイズ分のバイトを加算）
        for i in 0..5 {
            let elem = *base.add(i);
            println!("arr[{}] = {} (addr: {:p})", i, elem, base.add(i));
        }
    }
}
```

実行結果の例:
```
arr[0] = 10 (addr: 0x7ffd12345600)
arr[1] = 20 (addr: 0x7ffd12345604)  ← 4 バイト (i32) 進む
arr[2] = 30 (addr: 0x7ffd12345608)
arr[3] = 40 (addr: 0x7ffd1234560c)
arr[4] = 50 (addr: 0x7ffd12345610)
```

### ポインタ演算の安全性要件

`add(n)` が安全に使えるためには以下が満たされている必要があります。

```
条件1: base.add(n) が指すメモリは、同じ「アロケーション」の中にある
条件2: 結果のアドレスが usize の範囲を超えない
条件3: isize にキャストしてもオーバーフローしない（offset の場合）
```

```rust
fn pointer_arithmetic_safe() {
    let data = vec![1u8, 2, 3, 4, 5, 6, 7, 8];
    let ptr = data.as_ptr();

    unsafe {
        // OK: 同一アロケーション内
        let third = ptr.add(2);
        assert_eq!(*third, 3);

        // OK: 一つ過ぎたアドレスまでは有効（デリファレンスは不可）
        let end = ptr.add(data.len());
        println!("end ptr: {:p}", end);

        // NG: アロケーション外へ出てデリファレンス -> UB
        // let beyond = ptr.add(data.len() + 1);
        // println!("{}", *beyond); // 未定義動作!
    }
}

fn main() {
    pointer_arithmetic_safe();
}
```

---

## ポインタから参照への変換

```rust
fn main() {
    let x: i32 = 100;
    let ptr: *const i32 = &x;

    // *const T から &T へ
    // SAFETY: ptr は有効な i32 を指し、生存期間 'a の間有効
    let reference: &i32 = unsafe { &*ptr };
    println!("{}", reference); // 100

    let mut y: i32 = 200;
    let ptr_mut: *mut i32 = &mut y;

    // *mut T から &mut T へ
    // SAFETY: ptr_mut は有効な i32 を指し、他に可変参照が存在しない
    let ref_mut: &mut i32 = unsafe { &mut *ptr_mut };
    *ref_mut = 999;
    println!("{}", y); // 999
}
```

---

## `as_ptr` / `as_mut_ptr` パターン

標準ライブラリの多くの型は `as_ptr()` / `as_mut_ptr()` を提供しています。

```rust
fn main() {
    let mut v: Vec<u8> = vec![0u8; 8];

    // Vec の内部バッファへの生ポインタ
    let ptr: *mut u8 = v.as_mut_ptr();

    unsafe {
        // 各バイトを直接書き込む
        for i in 0..8 {
            ptr.add(i).write(i as u8 * 2);
        }
    }

    println!("{:?}", v); // [0, 2, 4, 6, 8, 10, 12, 14]

    // String の内部バッファ
    let s = String::from("hello");
    let byte_ptr: *const u8 = s.as_ptr();
    unsafe {
        for i in 0..s.len() {
            print!("{}", *byte_ptr.add(i) as char);
        }
    }
    println!(); // hello
}
```

---

## `ptr::read` と `ptr::write`

デリファレンスより安全な読み書きには `ptr::read` / `ptr::write` を使います。

```rust
use std::ptr;

fn main() {
    let src: i32 = 42;
    let mut dst: i32 = 0;

    unsafe {
        // ptr::read: コピーを返す（ムーブセマンティクスを無視した生コピー）
        let val = ptr::read(&src as *const i32);
        println!("read: {}", val);

        // ptr::write: ドロップを呼ばずに書き込む
        ptr::write(&mut dst as *mut i32, val);
        println!("dst after write: {}", dst);
    }

    // read と write の使い分け
    // - ptr::read: 初期化されていないメモリから値を取り出すとき
    // - ptr::write: まだ初期化されていないメモリに書き込むとき（既存値のドロップを避ける）

    // ptr::copy: memcpy 相当
    let src_arr: [u8; 4] = [1, 2, 3, 4];
    let mut dst_arr: [u8; 4] = [0; 4];
    unsafe {
        ptr::copy(
            src_arr.as_ptr(),
            dst_arr.as_mut_ptr(),
            4, // count (要素数、バイト数ではない)
        );
    }
    println!("{:?}", dst_arr); // [1, 2, 3, 4]

    // ptr::copy_nonoverlapping: memmove でなく memcpy（重複不可だが高速）
    let mut buf: [u8; 8] = [1, 2, 3, 4, 5, 6, 7, 8];
    unsafe {
        // 前半4バイトを後半4バイトにコピー（重複しない）
        let (left, right) = buf.split_at_mut(4);
        ptr::copy_nonoverlapping(left.as_ptr(), right.as_mut_ptr(), 4);
    }
    println!("{:?}", buf); // [1, 2, 3, 4, 1, 2, 3, 4]
}
```

---

## まとめ: 生ポインタのチェックリスト

生ポインタをデリファレンスする前に、以下をすべて確認してください。

```
□ ポインタは null でないか？      → is_null() でチェック
□ ポインタは有効なメモリを指すか？ → アロケーションの範囲内か確認
□ ポインタは適切にアラインされているか？ → T のアラインメント要件を満たすか
□ ポインタが指す値は正しく初期化されているか？ → 未初期化メモリの読み取りは UB
□ 可変ポインタの場合、他に参照やポインタが同じメモリを使っていないか？
□ ポインタの生存期間は十分か？    → ダングリングになっていないか
```

これらが満たされていることを `// SAFETY:` コメントで明記する習慣をつけましょう。

---

## 演習

1. `i32` の配列を生ポインタで逆順に読み取るプログラムを書いてください。
2. ダングリングポインタが発生するコードを意図的に書き、なぜ危険かコメントで説明してください。
3. `Vec<T>` の `as_ptr()` と `len()` を使い、スライスを使わずに全要素の合計を計算してください。
