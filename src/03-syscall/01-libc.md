# 01: libcクレートの活用

## libcクレートとは

`libc`クレートは、C標準ライブラリおよびPOSIX APIの型・定数・関数宣言を
Rustから利用できるようにするFFIバインディングです。
`libc::open`・`libc::read`・`libc::write`・`libc::close`などを使うことで、
Rustの標準ライブラリを介さずにOSと直接対話できます。

```
Rustコード
    │
    ▼
libc::open(path, flags, mode)  ← extern "C" 宣言済み
    │
    ▼
glibc / musl / libc (プラットフォームのCライブラリ)
    │
    ▼
カーネルのsys_open (システムコール)
```

> **対応プラットフォーム**: Linux・macOS両対応（特記なき限り）

## 依存関係の追加

```toml
# Cargo.toml
[dependencies]
libc = "0.2"
```

## 基本的なopen / read / write / close

### open(2) の呼び出し

```rust
use libc::{c_int, O_RDONLY, O_WRONLY, O_CREAT, O_TRUNC};
use std::ffi::CString;

fn main() {
    // パスをCStringに変換（null終端文字列が必要）
    let path = CString::new("/tmp/hello.txt").expect("CString::new failed");

    // ファイルを書き込みモードで開く（なければ作成）
    // open(path, flags, mode) -> fd (負値はエラー)
    let fd: c_int = unsafe {
        libc::open(
            path.as_ptr(),
            O_WRONLY | O_CREAT | O_TRUNC,
            0o644, // rw-r--r--
        )
    };

    if fd < 0 {
        eprintln!("open failed: {}", std::io::Error::last_os_error());
        std::process::exit(1);
    }

    println!("ファイルディスクリプタ: {}", fd);

    // 必ずcloseする
    unsafe { libc::close(fd) };
}
```

### write(2) でデータを書き込む

```rust
use libc::{c_int, O_WRONLY, O_CREAT, O_TRUNC, SEEK_SET};
use std::ffi::CString;

fn write_to_file(path: &str, data: &[u8]) -> Result<(), String> {
    let cpath = CString::new(path).map_err(|e| e.to_string())?;

    let fd: c_int = unsafe {
        libc::open(cpath.as_ptr(), O_WRONLY | O_CREAT | O_TRUNC, 0o644)
    };
    if fd < 0 {
        return Err(format!("open: {}", std::io::Error::last_os_error()));
    }

    let mut written = 0usize;
    while written < data.len() {
        // write はシグナルで中断されることがあるため、ループで再試行する
        let ret = unsafe {
            libc::write(
                fd,
                data[written..].as_ptr() as *const libc::c_void,
                data.len() - written,
            )
        };
        if ret < 0 {
            unsafe { libc::close(fd) };
            return Err(format!("write: {}", std::io::Error::last_os_error()));
        }
        written += ret as usize;
    }

    unsafe { libc::close(fd) };
    Ok(())
}

fn main() {
    let data = b"Hello, low-level Rust!\n";
    match write_to_file("/tmp/libc_test.txt", data) {
        Ok(()) => println!("書き込み成功"),
        Err(e) => eprintln!("エラー: {}", e),
    }
}
```

### read(2) でデータを読み込む

```rust
use libc::{c_int, O_RDONLY};
use std::ffi::CString;

fn read_file(path: &str) -> Result<Vec<u8>, String> {
    let cpath = CString::new(path).map_err(|e| e.to_string())?;

    let fd: c_int = unsafe { libc::open(cpath.as_ptr(), O_RDONLY, 0) };
    if fd < 0 {
        return Err(format!("open: {}", std::io::Error::last_os_error()));
    }

    let mut buffer = Vec::new();
    let mut chunk = [0u8; 4096];

    loop {
        let ret = unsafe {
            libc::read(
                fd,
                chunk.as_mut_ptr() as *mut libc::c_void,
                chunk.len(),
            )
        };
        match ret {
            0 => break,          // EOF
            n if n < 0 => {
                unsafe { libc::close(fd) };
                return Err(format!("read: {}", std::io::Error::last_os_error()));
            }
            n => buffer.extend_from_slice(&chunk[..n as usize]),
        }
    }

    unsafe { libc::close(fd) };
    Ok(buffer)
}

fn main() {
    // まずファイルを作成
    let data = b"Hello, libc read test!\nLine 2\n";
    unsafe {
        let path = std::ffi::CString::new("/tmp/read_test.txt").unwrap();
        let fd = libc::open(
            path.as_ptr(),
            libc::O_WRONLY | libc::O_CREAT | libc::O_TRUNC,
            0o644,
        );
        libc::write(fd, data.as_ptr() as *const _, data.len());
        libc::close(fd);
    }

    match read_file("/tmp/read_test.txt") {
        Ok(bytes) => println!("読み込み内容:\n{}", String::from_utf8_lossy(&bytes)),
        Err(e) => eprintln!("エラー: {}", e),
    }
}
```

## errno処理

システムコールが失敗した場合、エラーの原因は`errno`グローバル変数に設定されます。
Rustでは`std::io::Error::last_os_error()`または`libc::__errno_location()`で取得できます。

```rust
use libc::ENOENT;

/// errno を直接読む（Linux）
fn get_errno() -> i32 {
    unsafe { *libc::__errno_location() }
}

/// エラーコードから意味のあるメッセージを作る
fn check_open_error(path: &str) {
    let cpath = std::ffi::CString::new(path).unwrap();
    let fd = unsafe { libc::open(cpath.as_ptr(), libc::O_RDONLY, 0) };

    if fd < 0 {
        let errno = get_errno();
        let msg = match errno {
            libc::ENOENT  => "ファイルが存在しない",
            libc::EACCES  => "アクセス権限がない",
            libc::EMFILE  => "プロセスのFD上限に達した",
            libc::ENFILE  => "システム全体のFD上限に達した",
            libc::EISDIR  => "ディレクトリを指定した",
            _ => "その他のエラー",
        };
        eprintln!("open({}) 失敗 [errno={}]: {}", path, errno, msg);

        // strerror_r で人間向けメッセージを取得
        let mut buf = [0i8; 256];
        unsafe { libc::strerror_r(errno, buf.as_mut_ptr(), buf.len()) };
        let c_msg = unsafe { std::ffi::CStr::from_ptr(buf.as_ptr()) };
        eprintln!("strerror: {}", c_msg.to_string_lossy());
    }
}

fn main() {
    check_open_error("/nonexistent/path/file.txt");
    check_open_error("/etc/shadow"); // 通常は権限なし
}
```

### errno処理のユーティリティ関数

```rust
use std::io;

/// syscallの戻り値をResult<T>に変換するヘルパー
fn check_ret<T: PartialOrd + Default>(ret: T) -> io::Result<T> {
    if ret < T::default() {
        Err(io::Error::last_os_error())
    } else {
        Ok(ret)
    }
}

fn main() {
    let path = std::ffi::CString::new("/tmp/errno_test.txt").unwrap();
    let fd = unsafe {
        libc::open(
            path.as_ptr(),
            libc::O_WRONLY | libc::O_CREAT | libc::O_TRUNC,
            0o644,
        )
    };

    let fd = match check_ret(fd) {
        Ok(fd) => fd,
        Err(e) => {
            eprintln!("open failed: {}", e);
            return;
        }
    };

    let msg = b"errno test\n";
    let written = unsafe {
        libc::write(fd, msg.as_ptr() as *const _, msg.len())
    };

    match check_ret(written) {
        Ok(n) => println!("{} バイト書き込み", n),
        Err(e) => eprintln!("write failed: {}", e),
    }

    unsafe { libc::close(fd) };
}
```

## RAIIラッパーの作り方

`unsafe`ブロックを最小化し、リソースリークを防ぐために、
ファイルディスクリプタをRAIIでラップするのがベストプラクティスです。

```rust
use libc::{c_int, O_RDONLY, O_WRONLY, O_CREAT, O_TRUNC};
use std::ffi::CString;
use std::io;

/// ファイルディスクリプタのRAIIラッパー
pub struct OwnedFd {
    fd: c_int,
}

impl OwnedFd {
    /// ファイルを開いてOwnedFdを返す
    pub fn open(path: &str, flags: c_int, mode: u32) -> io::Result<Self> {
        let cpath = CString::new(path)
            .map_err(|e| io::Error::new(io::ErrorKind::InvalidInput, e))?;

        let fd = unsafe { libc::open(cpath.as_ptr(), flags, mode) };
        if fd < 0 {
            return Err(io::Error::last_os_error());
        }
        Ok(OwnedFd { fd })
    }

    /// 内部のfdを返す（unsafeな操作に渡す用）
    pub fn as_raw_fd(&self) -> c_int {
        self.fd
    }

    /// データを書き込む
    pub fn write_all(&self, data: &[u8]) -> io::Result<()> {
        let mut written = 0usize;
        while written < data.len() {
            let ret = unsafe {
                libc::write(
                    self.fd,
                    data[written..].as_ptr() as *const libc::c_void,
                    data.len() - written,
                )
            };
            if ret < 0 {
                let err = io::Error::last_os_error();
                // EINTR（シグナルによる中断）の場合は再試行
                if err.kind() == io::ErrorKind::Interrupted {
                    continue;
                }
                return Err(err);
            }
            written += ret as usize;
        }
        Ok(())
    }

    /// データを読み込む
    pub fn read(&self, buf: &mut [u8]) -> io::Result<usize> {
        loop {
            let ret = unsafe {
                libc::read(
                    self.fd,
                    buf.as_mut_ptr() as *mut libc::c_void,
                    buf.len(),
                )
            };
            if ret < 0 {
                let err = io::Error::last_os_error();
                if err.kind() == io::ErrorKind::Interrupted {
                    continue; // EINTRは再試行
                }
                return Err(err);
            }
            return Ok(ret as usize);
        }
    }

    /// ファイルサイズを取得する
    pub fn file_size(&self) -> io::Result<u64> {
        let mut stat: libc::stat = unsafe { std::mem::zeroed() };
        let ret = unsafe { libc::fstat(self.fd, &mut stat) };
        if ret < 0 {
            return Err(io::Error::last_os_error());
        }
        Ok(stat.st_size as u64)
    }
}

/// Dropトレイトで自動closeを保証
impl Drop for OwnedFd {
    fn drop(&mut self) {
        unsafe {
            libc::close(self.fd);
        }
    }
}

fn main() -> io::Result<()> {
    // 書き込み
    {
        let fd = OwnedFd::open(
            "/tmp/raii_test.txt",
            O_WRONLY | O_CREAT | O_TRUNC,
            0o644,
        )?;
        fd.write_all(b"RAII wrapper test\n")?;
        fd.write_all(b"This file will be closed automatically.\n")?;
        println!("書き込み完了 (fd={})", fd.as_raw_fd());
        // ここでfdがスコープを外れると自動的にclose
    }

    // 読み込み
    {
        let fd = OwnedFd::open("/tmp/raii_test.txt", O_RDONLY, 0)?;
        let size = fd.file_size()?;
        println!("ファイルサイズ: {} バイト", size);

        let mut buf = vec![0u8; size as usize];
        let n = fd.read(&mut buf)?;
        println!("読み込み内容:\n{}", String::from_utf8_lossy(&buf[..n]));
    }

    Ok(())
}
```

### Read / Write トレイトの実装

```rust
use std::io::{self, Read, Write};

impl Read for OwnedFd {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        OwnedFd::read(self, buf)
    }
}

impl Write for OwnedFd {
    fn write(&mut self, buf: &[u8]) -> io::Result<usize> {
        loop {
            let ret = unsafe {
                libc::write(
                    self.fd,
                    buf.as_ptr() as *const libc::c_void,
                    buf.len(),
                )
            };
            if ret < 0 {
                let err = io::Error::last_os_error();
                if err.kind() == io::ErrorKind::Interrupted {
                    continue;
                }
                return Err(err);
            }
            return Ok(ret as usize);
        }
    }

    fn flush(&mut self) -> io::Result<()> {
        // fdレベルではflushは不要（バッファリングなし）
        // ただしfsyncが必要な場合は:
        // let ret = unsafe { libc::fsync(self.fd) };
        Ok(())
    }
}
```

## まとめ

| 操作 | libc関数 | Rustラッパーのポイント |
|------|----------|----------------------|
| ファイルを開く | `libc::open` | `CString`でパス変換、戻り値を`Result`に |
| 書き込み | `libc::write` | EINTRに対応したループ、部分書き込みの処理 |
| 読み込み | `libc::read` | EINTRに対応、戻り値0はEOF |
| 閉じる | `libc::close` | `Drop`トレイトで自動close |
| エラー確認 | `errno` / `last_os_error()` | `io::Result`に変換するヘルパーを作る |

次節では、libcを介さず直接システムコールを発行する方法を学びます。
