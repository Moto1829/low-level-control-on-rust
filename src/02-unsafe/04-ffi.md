# FFI — C ライブラリを Rust から呼ぶ

FFI（Foreign Function Interface）を使うと、Rust から C の関数を呼び出したり、C から Rust の関数を呼び出したりできます。
既存の C ライブラリ資産を活用する場合や、OS レベルのシステムコールを直接叩く場合に必須の技術です。

---

## 1. `extern "C"` ブロックの書き方

C の関数を Rust から呼ぶには `extern "C"` ブロックで宣言します。

```rust
use std::os::raw::{c_int, c_double};

extern "C" {
    // C 標準ライブラリの abs 関数
    fn abs(x: c_int) -> c_int;

    // C 標準ライブラリの sqrt 関数
    fn sqrt(x: c_double) -> c_double;

    // C 標準ライブラリの exit 関数
    fn exit(status: c_int) -> !;
}

fn main() {
    let result = unsafe { abs(-42) };
    println!("abs(-42) = {}", result);

    let root = unsafe { sqrt(2.0) };
    println!("sqrt(2.0) = {:.6}", root);
}
```

コンパイル時に `-l` でリンクするライブラリを指定できます（`build.rs` または `#[link]` 属性で指定）。

```rust
// libc に直接リンクする例
#[link(name = "m")] // -lm (数学ライブラリ)
extern "C" {
    fn pow(base: f64, exp: f64) -> f64;
    fn ceil(x: f64) -> f64;
}

fn main() {
    unsafe {
        println!("2^10 = {}", pow(2.0, 10.0));
        println!("ceil(3.2) = {}", ceil(3.2));
    }
}
```

### 呼び出し規約の種類

| 規約 | 説明 |
|------|------|
| `"C"` | C のデフォルト呼び出し規約（最もよく使う） |
| `"stdcall"` | Windows API で使用（Win32） |
| `"system"` | プラットフォームに応じて `"C"` または `"stdcall"` に自動切り替え |
| `"Rust"` | Rust ネイティブ（ABI 安定でないため FFI 不可） |

---

## 2. `#[no_mangle]` で Rust の関数を C から呼ぶ

Rust はデフォルトで関数名をマングル（符号化）します。
C から呼べる名前でエクスポートするには `#[no_mangle]` と `extern "C"` を組み合わせます。

```rust
/// C から呼び出せる Rust 関数
#[no_mangle]
pub extern "C" fn rust_add(a: i32, b: i32) -> i32 {
    a + b
}

/// C から呼べるフィボナッチ関数
#[no_mangle]
pub extern "C" fn rust_fibonacci(n: u32) -> u64 {
    match n {
        0 => 0,
        1 => 1,
        _ => {
            let (mut a, mut b) = (0u64, 1u64);
            for _ in 2..=n {
                let c = a + b;
                a = b;
                b = c;
            }
            b
        }
    }
}
```

C 側のヘッダファイル（`rust_lib.h`）:

```c
#ifndef RUST_LIB_H
#define RUST_LIB_H

#include <stdint.h>

int32_t rust_add(int32_t a, int32_t b);
uint64_t rust_fibonacci(uint32_t n);

#endif
```

C 側の呼び出しコード（`main.c`）:

```c
#include <stdio.h>
#include "rust_lib.h"

int main(void) {
    printf("rust_add(3, 4) = %d\n", rust_add(3, 4));
    printf("rust_fibonacci(10) = %llu\n", rust_fibonacci(10));
    return 0;
}
```

`Cargo.toml` でスタティックライブラリとしてビルドする設定:

```toml
[lib]
name = "rust_lib"
crate-type = ["staticlib"]  # librust_lib.a を生成
```

---

## 3. C と Rust の型対応表

`std::os::raw`（または `libc` クレート）で C の型に対応する Rust 型を使います。

| C の型 | Rust の型（`std::os::raw`） | `libc` クレートの型 |
|--------|--------------------------|-------------------|
| `int` | `c_int` | `libc::c_int` |
| `unsigned int` | `c_uint` | `libc::c_uint` |
| `long` | `c_long` | `libc::c_long` |
| `unsigned long` | `c_ulong` | `libc::c_ulong` |
| `long long` | `c_longlong` | `libc::c_longlong` |
| `char` | `c_char` | `libc::c_char` |
| `unsigned char` | `c_uchar` | `libc::c_uchar` |
| `short` | `c_short` | `libc::c_short` |
| `float` | `c_float` | `libc::c_float` |
| `double` | `c_double` | `libc::c_double` |
| `void *` | `*mut c_void` | `*mut libc::c_void` |
| `const void *` | `*const c_void` | — |
| `size_t` | `usize`（通常一致） | `libc::size_t` |
| `ptrdiff_t` | `isize`（通常一致） | `libc::ptrdiff_t` |
| `bool` (C99) | `bool` | — |
| `char *` (文字列) | `*const c_char` | — |

```rust
use std::os::raw::{c_int, c_uint, c_char, c_void, c_double};

extern "C" {
    // void* を受け取る汎用関数の例
    fn memset(s: *mut c_void, c: c_int, n: usize) -> *mut c_void;
    fn memcmp(s1: *const c_void, s2: *const c_void, n: usize) -> c_int;
    fn atof(s: *const c_char) -> c_double;
}

fn main() {
    let mut buf = [0u8; 8];
    unsafe {
        memset(buf.as_mut_ptr() as *mut c_void, 0xFF as c_int, buf.len());
    }
    println!("memset 後: {:?}", buf); // [255, 255, 255, 255, 255, 255, 255, 255]
}
```

---

## 4. `bindgen` で C ヘッダから Rust バインディングを自動生成する

手書きのバインディングはエラーが起きやすいため、`bindgen` で自動生成するのがベストプラクティスです。

### インストール

```bash
cargo install bindgen-cli
# または build.rs で使う場合
```

### 手動生成（CLI）

```bash
# zlib のヘッダからバインディングを生成
bindgen /usr/include/zlib.h \
    --allowlist-function "compress.*|uncompress.*|inflate.*|deflate.*" \
    --allowlist-type "z_stream.*" \
    --allowlist-var "Z_.*" \
    -o src/bindings.rs
```

### `build.rs` で自動生成する（推奨）

```toml
# Cargo.toml
[build-dependencies]
bindgen = "0.69"

[dependencies]
# バインディング生成後に必要なリンク指示は build.rs に書く
```

```rust
// build.rs
use std::path::PathBuf;

fn main() {
    // リンカへの指示
    println!("cargo:rustc-link-lib=z");

    let bindings = bindgen::Builder::default()
        .header("wrapper.h")
        // 生成する関数・型をフィルタ
        .allowlist_function("compress.*")
        .allowlist_function("uncompress.*")
        .allowlist_type("z_stream.*")
        .allowlist_var("Z_.*")
        // Rust の命名規則警告を抑制
        .raw_line("#![allow(non_upper_case_globals)]")
        .raw_line("#![allow(non_camel_case_types)]")
        .raw_line("#![allow(non_snake_case)]")
        .generate()
        .expect("バインディング生成失敗");

    let out_path = PathBuf::from(std::env::var("OUT_DIR").unwrap());
    bindings
        .write_to_file(out_path.join("bindings.rs"))
        .expect("バインディング書き込み失敗");
}
```

```c
// wrapper.h — bindgen に渡すラッパーヘッダ
#include <zlib.h>
```

生成されたバインディングの利用:

```rust
// src/lib.rs または src/main.rs
// bindgen が OUT_DIR に生成したファイルをインクルード
mod bindings {
    #![allow(non_upper_case_globals)]
    #![allow(non_camel_case_types)]
    #![allow(non_snake_case)]
    include!(concat!(env!("OUT_DIR"), "/bindings.rs"));
}
```

---

## 5. 文字列の受け渡し（`CStr` / `CString`）

C の文字列はヌル終端（`\0` で終わる）バイト列であり、Rust の `str` / `String` とは異なります。

### Rust → C: `CString` を使う

```rust
use std::ffi::{CString, CStr};
use std::os::raw::c_char;

extern "C" {
    // C の puts 関数（文字列を標準出力に書く）
    fn puts(s: *const c_char) -> i32;
}

fn main() {
    // CString: ヌル終端バイト列を所有する Rust 型
    let message = CString::new("Hello from Rust via C!").expect("CString 生成失敗");

    // as_ptr() で *const c_char を得て C 関数に渡す
    unsafe {
        puts(message.as_ptr());
    }
    // message がここでドロップ → メモリ解放
}
```

### C → Rust: `CStr` を使う

```rust
use std::ffi::CStr;
use std::os::raw::c_char;

/// C から渡された文字列ポインタを Rust の &str に変換する
///
/// # Safety
/// - ptr は有効なヌル終端文字列を指していること
/// - ptr の指すメモリはこの関数の呼び出し中有効であること
unsafe fn c_string_to_str(ptr: *const c_char) -> &'static str {
    CStr::from_ptr(ptr)
        .to_str()
        .expect("UTF-8 でない C 文字列")
}

// C から呼ばれる関数の例
#[no_mangle]
pub extern "C" fn print_c_string(s: *const c_char) {
    if s.is_null() {
        println!("(null)");
        return;
    }
    let rust_str = unsafe { CStr::from_ptr(s).to_string_lossy() };
    println!("Rust から受け取った文字列: {}", rust_str);
}

fn main() {
    // テスト: Rust 内で CString を作って C 関数風に渡す
    use std::ffi::CString;
    let s = CString::new("テスト文字列").unwrap();
    print_c_string(s.as_ptr());
}
```

### 文字列変換のパターンまとめ

| 変換方向 | 使う型・メソッド |
|----------|----------------|
| `String` → `*const c_char` | `CString::new(s)?.as_ptr()` |
| `*const c_char` → `&str` | `CStr::from_ptr(ptr).to_str()?` |
| `*const c_char` → `String` | `CStr::from_ptr(ptr).to_string_lossy().into_owned()` |
| `&[u8]` (ヌル終端含む) → `&CStr` | `CStr::from_bytes_with_nul(bytes)?` |

---

## 6. 所有権境界の管理

FFI を跨いだメモリ管理で最も重要なルールは
**「確保したアロケータと同じアロケータで解放する」** ことです。

### C が確保したメモリは C（または対応する API）で解放する

```rust
use std::os::raw::{c_char, c_int};
use std::ffi::CStr;

extern "C" {
    // 例: C ライブラリが文字列を確保して返す関数
    // char* get_version_string(void);
    // void free_string(char* s);
    fn get_version_string() -> *mut c_char;
    fn free_string(s: *mut c_char);
}

fn get_version() -> String {
    unsafe {
        let ptr = get_version_string();
        assert!(!ptr.is_null());

        // C ポインタから Rust の String にコピーする
        let version = CStr::from_ptr(ptr).to_string_lossy().into_owned();

        // C ライブラリの解放関数を使う（Rust の dealloc ではない！）
        free_string(ptr);

        version
    }
}
```

### Rust が確保したメモリを C に渡す場合

```rust
use std::os::raw::c_char;
use std::ffi::CString;

/// C に文字列バッファを渡し、C が書き込んだ後 Rust で解放する
fn fill_buffer_example() {
    let mut buf: Vec<u8> = vec![0u8; 256];

    // C 関数にバッファを貸し出す（所有権は移転しない）
    // c_fill_buffer(buf.as_mut_ptr() as *mut c_char, buf.len() as _);

    // 使い終わったら Rust の Vec として通常通りドロップ
    drop(buf);
}

/// Box で確保したメモリを C に「売る」場合の解放エクスポート
#[no_mangle]
pub extern "C" fn rust_alloc_buffer(size: usize) -> *mut u8 {
    let mut v: Vec<u8> = vec![0u8; size];
    let ptr = v.as_mut_ptr();
    std::mem::forget(v); // Vec のドロップを止めて生ポインタだけ渡す
    ptr
}

/// rust_alloc_buffer で確保した領域を解放する（C から呼ぶ）
#[no_mangle]
pub unsafe extern "C" fn rust_free_buffer(ptr: *mut u8, size: usize) {
    if ptr.is_null() { return; }
    drop(Vec::from_raw_parts(ptr, size, size));
}
```

### 所有権境界のルール一覧

| シナリオ | 解放する側 | 注意点 |
|---------|-----------|--------|
| C が `malloc` → Rust に渡す | C（`free`） | Rust の `drop` / `dealloc` を呼ばない |
| Rust が `Box::new` → C に渡す | Rust（`Box::from_raw`） | C が `free` しないように設計する |
| Rust が `Vec` → C に渡す | Rust（`Vec::from_raw_parts`） | `forget` + 専用解放関数をエクスポート |
| スタック上の値のポインタ | 不要（自動） | C がポインタを保存して後で使わないこと |

---

## 7. 実例: `libz`（zlib）の圧縮関数を呼ぶ

zlib の `compress` と `uncompress` を FFI 経由で呼ぶ完全なサンプルです。

### `Cargo.toml`

```toml
[package]
name = "zlib-ffi-example"
version = "0.1.0"
edition = "2021"

[dependencies]
# libc クレートで c_int 等の型を使う
libc = "0.2"
```

### `build.rs`

```rust
fn main() {
    // zlib をリンクする（macOS/Linux では通常インストール済み）
    println!("cargo:rustc-link-lib=z");
}
```

### `src/main.rs`

```rust
use libc::{c_int, c_ulong, c_char};
use std::ffi::CStr;

// zlib の定数
const Z_OK: c_int = 0;
const Z_BUF_ERROR: c_int = -5;

extern "C" {
    /// データを圧縮する
    /// dest:       圧縮後データを書き込むバッファ
    /// dest_len:   入力: dest のサイズ / 出力: 実際の圧縮後サイズ
    /// source:     圧縮対象データ
    /// source_len: 圧縮対象のバイト数
    fn compress(
        dest: *mut u8,
        dest_len: *mut c_ulong,
        source: *const u8,
        source_len: c_ulong,
    ) -> c_int;

    /// データを展開する
    fn uncompress(
        dest: *mut u8,
        dest_len: *mut c_ulong,
        source: *const u8,
        source_len: c_ulong,
    ) -> c_int;

    /// エラーコードをメッセージ文字列に変換する
    fn zError(err: c_int) -> *const c_char;
}

/// zlib の compress を Rust でラップする
fn zlib_compress(input: &[u8]) -> Result<Vec<u8>, String> {
    // 圧縮後の最大サイズ = 元サイズ + 0.1% + 12 bytes（zlib の保証）
    let mut dest_len: c_ulong = (input.len() as c_ulong * 11 / 10) + 12;
    let mut dest: Vec<u8> = vec![0u8; dest_len as usize];

    let ret = unsafe {
        compress(
            dest.as_mut_ptr(),
            &mut dest_len,
            input.as_ptr(),
            input.len() as c_ulong,
        )
    };

    if ret != Z_OK {
        let msg = unsafe {
            CStr::from_ptr(zError(ret))
                .to_string_lossy()
                .into_owned()
        };
        return Err(format!("compress 失敗 (code={}): {}", ret, msg));
    }

    dest.truncate(dest_len as usize);
    Ok(dest)
}

/// zlib の uncompress を Rust でラップする
fn zlib_decompress(input: &[u8], expected_size: usize) -> Result<Vec<u8>, String> {
    let mut dest_len: c_ulong = expected_size as c_ulong;
    let mut dest: Vec<u8> = vec![0u8; dest_len as usize];

    let ret = unsafe {
        uncompress(
            dest.as_mut_ptr(),
            &mut dest_len,
            input.as_ptr(),
            input.len() as c_ulong,
        )
    };

    if ret == Z_BUF_ERROR {
        return Err("バッファが小さすぎます。expected_size を増やしてください".to_string());
    }
    if ret != Z_OK {
        let msg = unsafe {
            CStr::from_ptr(zError(ret))
                .to_string_lossy()
                .into_owned()
        };
        return Err(format!("uncompress 失敗 (code={}): {}", ret, msg));
    }

    dest.truncate(dest_len as usize);
    Ok(dest)
}

fn main() {
    let original = b"Hello, zlib! This is a test of FFI-based compression in Rust. \
                     Repeated data compresses well: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa";

    println!("元データ: {} bytes", original.len());
    println!("内容: {}", std::str::from_utf8(original).unwrap());

    // 圧縮
    let compressed = zlib_compress(original).expect("圧縮失敗");
    println!(
        "\n圧縮後: {} bytes ({:.1}% に削減)",
        compressed.len(),
        compressed.len() as f64 / original.len() as f64 * 100.0
    );

    // 展開
    let decompressed = zlib_decompress(&compressed, original.len())
        .expect("展開失敗");

    println!("\n展開後: {} bytes", decompressed.len());
    println!("一致確認: {}", original.as_ref() == decompressed.as_slice());
    println!("内容: {}", std::str::from_utf8(&decompressed).unwrap());
}
```

### 実行結果の例

```
元データ: 121 bytes
内容: Hello, zlib! This is a test of FFI-based compression in Rust. ...

圧縮後: 62 bytes (51.2% に削減)

展開後: 121 bytes
一致確認: true
内容: Hello, zlib! This is a test of FFI-based compression in Rust. ...
```

---

## 8. FFI の安全な設計指針

### Unsafe を最小化するラッパーパターン

```rust
// unsafe な FFI 宣言を専用モジュールに隔離する
mod ffi {
    use libc::{c_int, c_ulong};

    extern "C" {
        pub fn compress(
            dest: *mut u8,
            dest_len: *mut c_ulong,
            source: *const u8,
            source_len: c_ulong,
        ) -> c_int;
    }
}

// 公開 API は safe な Rust 関数として提供する
pub fn compress(input: &[u8]) -> Option<Vec<u8>> {
    // バリデーション・unsafe の呼び出し・エラーハンドリング
    // をこの関数内に閉じ込める
    todo!()
}
```

### チェックリスト

- `extern "C"` 宣言の型は C ヘッダと完全に一致しているか
- ポインタが null でないことを確認してから使っているか
- C 側で確保したメモリは C 側の `free` 関数で解放しているか
- Rust の参照から作った生ポインタのライフタイムを超えて使っていないか
- マルチスレッドで呼ぶ場合に C 関数がスレッドセーフかどうか確認したか
- `bindgen` を使って型ミスマッチを防いでいるか

---

## まとめ

| トピック | 方法 |
|---------|------|
| C 関数を Rust から呼ぶ | `extern "C"` ブロック + `unsafe` |
| Rust 関数を C からエクスポート | `#[no_mangle]` + `pub extern "C" fn` |
| 型対応 | `std::os::raw` または `libc` クレート |
| バインディング自動生成 | `bindgen` CLI または `build.rs` |
| 文字列の受け渡し | `CString`（Rust→C）/ `CStr`（C→Rust） |
| 所有権管理 | 確保側が解放する・専用 free 関数をエクスポート |
| 実例 | zlib の `compress` / `uncompress` |

FFI は Rust の安全保証が及ばない領域です。
コードレビュー・Valgrind・AddressSanitizer などのツールを活用して、メモリ安全性を丁寧に検証してください。
