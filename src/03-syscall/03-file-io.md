# 03: 高度なファイルI/O

## 概要

本節では、`std::fs`では制御できない低レベルのファイルI/O機能を扱います。
具体的には、`O_DIRECT`による直接I/O、`mmap`によるファイルマッピング、
`sendfile`による高速コピー、そしてLinux 5.1以降で利用可能な`io_uring`の概要を説明します。

```
アクセス方法の比較:
┌─────────────────────────────────────────────────────────────┐
│ read(2) / write(2)                                          │
│   → カーネルのページキャッシュを経由                         │
│   → データのコピーが2回発生（カーネル↔ユーザー空間）         │
├─────────────────────────────────────────────────────────────┤
│ O_DIRECT                                                    │
│   → ページキャッシュをバイパス                               │
│   → ディスクへの直接I/O（アライメント制約あり）              │
├─────────────────────────────────────────────────────────────┤
│ mmap                                                        │
│   → ページキャッシュをユーザー空間にマップ                   │
│   → コピー不要でポインタアクセス可能                         │
├─────────────────────────────────────────────────────────────┤
│ sendfile / splice                                           │
│   → カーネル内でのデータ転送（ゼロコピー）                   │
├─────────────────────────────────────────────────────────────┤
│ io_uring                                                    │
│   → 非同期I/O（Linux 5.1以降）                              │
│   → リングバッファを介したシステムコールバッチ処理            │
└─────────────────────────────────────────────────────────────┘
```

## open(2) フラグの詳細

### 主要なフラグ一覧

```rust
use libc::{
    O_RDONLY, O_WRONLY, O_RDWR,   // アクセスモード
    O_CREAT, O_TRUNC, O_APPEND,   // 作成・書き込みオプション
    O_EXCL,                        // 排他作成
    O_NONBLOCK,                    // ノンブロッキング
    O_CLOEXEC,                     // exec時にcloseする
    O_SYNC,                        // 書き込みを同期する
    O_DSYNC,                       // データのみ同期
};

// Linux限定フラグ
#[cfg(target_os = "linux")]
use libc::{
    O_DIRECT,   // ページキャッシュをバイパス（直接I/O）
    O_TMPFILE,  // 名前なしの一時ファイル作成
    O_PATH,     // パスのみ参照（ファイル内容にはアクセスしない）
};
```

### O_CLOEXEC の重要性

```rust
use libc::{O_RDONLY, O_CLOEXEC};
use std::ffi::CString;

fn open_with_cloexec(path: &str) -> std::io::Result<i32> {
    let cpath = CString::new(path)?;
    let fd = unsafe {
        // O_CLOEXECを付けることで、fork後のexecでfdが子プロセスに
        // 漏れることを防ぐ。競合状態のないアトミックな操作。
        libc::open(cpath.as_ptr(), O_RDONLY | O_CLOEXEC, 0)
    };
    if fd < 0 {
        return Err(std::io::Error::last_os_error());
    }
    Ok(fd)
}
```

## O_DIRECT による直接I/O

> **対応プラットフォーム**: Linux専用

`O_DIRECT`を使うと、ページキャッシュを完全にバイパスしてディスクに直接アクセスできます。
データベースや高性能ストレージシステムで使われる手法です。

### アライメント制約

```
O_DIRECTの制約:
┌────────────────────────────────────────────────────────┐
│ バッファのアドレス: 論理ブロックサイズ（通常512B）にアライン  │
│ 転送サイズ:         論理ブロックサイズの倍数                │
│ ファイルオフセット: 論理ブロックサイズの倍数                │
└────────────────────────────────────────────────────────┘
```

```rust
#[cfg(target_os = "linux")]
use libc::O_DIRECT;
use std::alloc::{alloc, dealloc, Layout};
use std::ffi::CString;
use std::io;

/// アライメントされたバッファを確保するRAIIラッパー
struct AlignedBuffer {
    ptr: *mut u8,
    layout: Layout,
}

impl AlignedBuffer {
    /// alignment: 通常512または4096（デバイスのブロックサイズ）
    fn new(size: usize, alignment: usize) -> io::Result<Self> {
        let layout = Layout::from_size_align(size, alignment)
            .map_err(|e| io::Error::new(io::ErrorKind::InvalidInput, e))?;
        let ptr = unsafe { alloc(layout) };
        if ptr.is_null() {
            return Err(io::Error::new(io::ErrorKind::OutOfMemory, "alloc failed"));
        }
        // ゼロ初期化
        unsafe { std::ptr::write_bytes(ptr, 0, size) };
        Ok(AlignedBuffer { ptr, layout })
    }

    fn as_mut_ptr(&mut self) -> *mut u8 {
        self.ptr
    }

    fn as_slice(&self) -> &[u8] {
        unsafe { std::slice::from_raw_parts(self.ptr, self.layout.size()) }
    }
}

impl Drop for AlignedBuffer {
    fn drop(&mut self) {
        unsafe { dealloc(self.ptr, self.layout) };
    }
}

#[cfg(target_os = "linux")]
fn direct_io_write(path: &str, data: &[u8]) -> io::Result<()> {
    const BLOCK_SIZE: usize = 512;

    // データをブロックサイズにパディング
    let padded_size = (data.len() + BLOCK_SIZE - 1) & !(BLOCK_SIZE - 1);
    let mut buf = AlignedBuffer::new(padded_size, BLOCK_SIZE)?;

    // データをアライメントされたバッファにコピー
    unsafe {
        std::ptr::copy_nonoverlapping(data.as_ptr(), buf.as_mut_ptr(), data.len());
    }

    let cpath = CString::new(path)?;
    let fd = unsafe {
        libc::open(
            cpath.as_ptr(),
            libc::O_WRONLY | libc::O_CREAT | libc::O_TRUNC | O_DIRECT,
            0o644,
        )
    };
    if fd < 0 {
        return Err(io::Error::last_os_error());
    }

    let ret = unsafe {
        libc::write(
            fd,
            buf.as_mut_ptr() as *const libc::c_void,
            padded_size,
        )
    };
    unsafe { libc::close(fd) };

    if ret < 0 {
        return Err(io::Error::last_os_error());
    }
    println!("O_DIRECT書き込み: {} バイト（{}バイトパディング）", ret, padded_size);
    Ok(())
}

#[cfg(target_os = "linux")]
fn main() -> io::Result<()> {
    let data = b"O_DIRECT test data - must be block aligned!";
    direct_io_write("/tmp/direct_io_test.dat", data)?;
    Ok(())
}
```

## mmapによるファイルマッピング

`mmap`でファイルをメモリにマップすると、ポインタアクセスだけでファイルを読み書きできます。
大きなファイルをランダムアクセスする場合に特に効果的です。

```
mmapの仕組み:
                  仮想アドレス空間
                  ┌──────────────┐
                  │  スタック     │
                  ├──────────────┤
                  │   heap       │
                  ├──────────────┤
ptr ──────────→  │  mmapページ  │ ← ページフォルト時にカーネルが
                  │  (ファイル)  │   ディスクからページを読み込む
                  ├──────────────┤
                  │   テキスト   │
                  └──────────────┘
```

### 読み取り専用マッピング

```rust
use libc::{MAP_PRIVATE, PROT_READ};
use std::ffi::CString;
use std::io;

/// ファイル全体をメモリマップするRAIIラッパー
pub struct Mmap {
    ptr: *mut u8,
    len: usize,
}

impl Mmap {
    /// ファイルを読み取り専用でマップする（Linux/macOS対応）
    pub fn open_read(path: &str) -> io::Result<Self> {
        let cpath = CString::new(path)?;
        let fd = unsafe { libc::open(cpath.as_ptr(), libc::O_RDONLY, 0) };
        if fd < 0 {
            return Err(io::Error::last_os_error());
        }

        // ファイルサイズを取得
        let mut stat: libc::stat = unsafe { std::mem::zeroed() };
        let ret = unsafe { libc::fstat(fd, &mut stat) };
        if ret < 0 {
            unsafe { libc::close(fd) };
            return Err(io::Error::last_os_error());
        }
        let len = stat.st_size as usize;

        if len == 0 {
            unsafe { libc::close(fd) };
            return Err(io::Error::new(io::ErrorKind::InvalidInput, "空のファイル"));
        }

        let ptr = unsafe {
            libc::mmap(
                std::ptr::null_mut(),   // アドレス: カーネルに選ばせる
                len,
                PROT_READ,
                MAP_PRIVATE,
                fd,
                0,  // オフセット
            )
        };

        // fdはmmap後にcloseしてよい（マッピングは維持される）
        unsafe { libc::close(fd) };

        if ptr == libc::MAP_FAILED {
            return Err(io::Error::last_os_error());
        }

        Ok(Mmap { ptr: ptr as *mut u8, len })
    }

    pub fn as_slice(&self) -> &[u8] {
        unsafe { std::slice::from_raw_parts(self.ptr, self.len) }
    }

    pub fn len(&self) -> usize {
        self.len
    }
}

impl Drop for Mmap {
    fn drop(&mut self) {
        unsafe {
            libc::munmap(self.ptr as *mut libc::c_void, self.len);
        }
    }
}

// Mmap は複数スレッドから読み取れる（読み取り専用なのでSync）
unsafe impl Send for Mmap {}
unsafe impl Sync for Mmap {}
```

### 読み書きマッピング（MAP_SHARED）

```rust
pub struct MmapMut {
    ptr: *mut u8,
    len: usize,
}

impl MmapMut {
    /// ファイルを読み書きモードでマップする（変更はファイルに反映）
    pub fn open_rw(path: &str) -> io::Result<Self> {
        let cpath = CString::new(path)?;
        let fd = unsafe {
            libc::open(cpath.as_ptr(), libc::O_RDWR, 0)
        };
        if fd < 0 {
            return Err(io::Error::last_os_error());
        }

        let mut stat: libc::stat = unsafe { std::mem::zeroed() };
        unsafe { libc::fstat(fd, &mut stat) };
        let len = stat.st_size as usize;

        let ptr = unsafe {
            libc::mmap(
                std::ptr::null_mut(),
                len,
                libc::PROT_READ | libc::PROT_WRITE,
                libc::MAP_SHARED,   // 変更がファイルに書き戻される
                fd,
                0,
            )
        };
        unsafe { libc::close(fd) };

        if ptr == libc::MAP_FAILED {
            return Err(io::Error::last_os_error());
        }

        Ok(MmapMut { ptr: ptr as *mut u8, len })
    }

    pub fn as_mut_slice(&mut self) -> &mut [u8] {
        unsafe { std::slice::from_raw_parts_mut(self.ptr, self.len) }
    }

    /// 変更をディスクに書き戻す
    pub fn flush(&self) -> io::Result<()> {
        let ret = unsafe {
            libc::msync(
                self.ptr as *mut libc::c_void,
                self.len,
                libc::MS_SYNC,  // MS_ASYNC で非同期フラッシュも可能
            )
        };
        if ret < 0 {
            return Err(io::Error::last_os_error());
        }
        Ok(())
    }
}

impl Drop for MmapMut {
    fn drop(&mut self) {
        unsafe {
            libc::munmap(self.ptr as *mut libc::c_void, self.len);
        }
    }
}

unsafe impl Send for MmapMut {}
```

### mmapの実践例：大きなファイルの特定部分を高速読み込み

```rust
fn mmap_demo() -> io::Result<()> {
    // テスト用ファイルを作成
    let path = "/tmp/mmap_test.bin";
    {
        let cpath = CString::new(path)?;
        let fd = unsafe {
            libc::open(
                cpath.as_ptr(),
                libc::O_WRONLY | libc::O_CREAT | libc::O_TRUNC,
                0o644,
            )
        };
        // 1MB のデータを書き込む
        let data = vec![0xABu8; 1024 * 1024];
        unsafe {
            libc::write(fd, data.as_ptr() as *const _, data.len());
            libc::close(fd);
        }
    }

    // mmapで開く
    let mmap = Mmap::open_read(path)?;
    println!("マップサイズ: {} バイト", mmap.len());

    // ポインタアクセスで任意位置を読み込む
    let slice = mmap.as_slice();
    println!("先頭4バイト: {:02X?}", &slice[..4]);
    println!("末尾4バイト: {:02X?}", &slice[slice.len()-4..]);

    // madvise でアクセスパターンをカーネルにヒントとして伝える
    unsafe {
        libc::madvise(
            mmap.ptr as *mut libc::c_void,
            mmap.len,
            libc::MADV_SEQUENTIAL, // シーケンシャルアクセスの場合
        );
    }

    Ok(())
}
```

## sendfileによる高速コピー

`sendfile`はカーネル内でファイルをコピーするため、
ユーザー空間へのデータコピーが発生しない「ゼロコピー」転送です。
Webサーバーが静的ファイルを高速に配信する際に使われています。

> **対応プラットフォーム**: Linux専用（macOSは引数が異なる）

```
通常のread+write:                  sendfile:
───────────────────────────────    ───────────────────────────────
アプリ    カーネル    ディスク      アプリ    カーネル    ディスク
 read() →                           sendfile() →
          ← ページキャッシュへ                  ← ページキャッシュへ
  ← データコピー                             → ソケットバッファへ
  write() →                                   → ネットワーク送信
          → ソケットバッファへ
          → ネットワーク送信

コピー回数: 2回                     コピー回数: 0〜1回
```

```rust
#[cfg(target_os = "linux")]
fn sendfile_copy(src_path: &str, dst_path: &str) -> io::Result<u64> {
    let src_cpath = CString::new(src_path)?;
    let dst_cpath = CString::new(dst_path)?;

    let src_fd = unsafe { libc::open(src_cpath.as_ptr(), libc::O_RDONLY, 0) };
    if src_fd < 0 {
        return Err(io::Error::last_os_error());
    }

    // コピー先ファイルを作成
    let dst_fd = unsafe {
        libc::open(
            dst_cpath.as_ptr(),
            libc::O_WRONLY | libc::O_CREAT | libc::O_TRUNC,
            0o644,
        )
    };
    if dst_fd < 0 {
        unsafe { libc::close(src_fd) };
        return Err(io::Error::last_os_error());
    }

    // ソースのサイズを取得
    let mut stat: libc::stat = unsafe { std::mem::zeroed() };
    unsafe { libc::fstat(src_fd, &mut stat) };
    let file_size = stat.st_size as usize;

    let mut total_sent = 0usize;
    let mut offset: libc::off_t = 0;

    // 大きなファイルは複数回に分けて送信
    while total_sent < file_size {
        let to_send = std::cmp::min(file_size - total_sent, 1024 * 1024 * 128); // 128MB単位
        let ret = unsafe {
            libc::sendfile(dst_fd, src_fd, &mut offset, to_send)
        };
        if ret < 0 {
            unsafe {
                libc::close(src_fd);
                libc::close(dst_fd);
            }
            return Err(io::Error::last_os_error());
        }
        total_sent += ret as usize;
    }

    unsafe {
        libc::close(src_fd);
        libc::close(dst_fd);
    }

    Ok(total_sent as u64)
}

#[cfg(target_os = "linux")]
fn main() -> io::Result<()> {
    // テスト用ファイル作成
    std::fs::write("/tmp/sendfile_src.txt", b"sendfile test content\n".repeat(1000))?;

    let bytes = sendfile_copy("/tmp/sendfile_src.txt", "/tmp/sendfile_dst.txt")?;
    println!("sendfileで {} バイトコピー完了", bytes);
    Ok(())
}
```

## io_uring の概要

> **対応プラットフォーム**: Linux 5.1以降専用

`io_uring`はLinux 5.1で導入された非同期I/Oフレームワークです。
従来の`epoll`+ノンブロッキングI/Oや`aio`と比べて、
システムコールのオーバーヘッドを大幅に削減できます。

### アーキテクチャ

```
io_uringの仕組み:
┌──────────────────────────────────────────────────────────────┐
│                    ユーザー空間                               │
│  ┌─────────────────────────┐  ┌────────────────────────────┐ │
│  │  SQ (Submission Queue)  │  │  CQ (Completion Queue)     │ │
│  │  ┌───┬───┬───┬───┐      │  │  ┌───┬───┬───┬───┐        │ │
│  │  │SQE│SQE│SQE│SQE│      │  │  │CQE│CQE│CQE│CQE│        │ │
│  │  └───┴───┴───┴───┘      │  │  └───┴───┴───┴───┘        │ │
│  └──────────────────────────┘  └────────────────────────────┘ │
│          ↓ io_uring_enter()              ↑ 完了通知            │
└──────────────────────────────────────────────────────────────┘
                   ↕ 共有メモリ（mmap）
┌──────────────────────────────────────────────────────────────┐
│                    カーネル空間                               │
│         非同期I/Oワーカースレッドが処理を実行                  │
└──────────────────────────────────────────────────────────────┘
```

### Crateを使ったio_uringサンプル

```toml
# Cargo.toml（Linux専用）
[target.'cfg(target_os = "linux")'.dependencies]
io-uring = "0.6"
```

```rust
#[cfg(target_os = "linux")]
fn io_uring_read_example() -> io::Result<()> {
    use io_uring::{IoUring, opcode, types};
    use std::os::unix::io::AsRawFd;

    const BUF_SIZE: usize = 4096;
    let file = std::fs::File::open("/etc/hostname")?;
    let mut buf = vec![0u8; BUF_SIZE];

    // io_uringリングを作成（キューサイズ=4）
    let mut ring = IoUring::new(4)?;

    // Submission Queue Entry (SQE) を準備
    let sqe = opcode::Read::new(
        types::Fd(file.as_raw_fd()),
        buf.as_mut_ptr(),
        buf.len() as u32,
    )
    .offset(0)  // ファイルの先頭から
    .build()
    .user_data(0x42); // 後で識別するためのタグ

    // SQにエントリを追加
    unsafe { ring.submission().push(&sqe) }
        .map_err(|e| io::Error::new(io::ErrorKind::Other, e.to_string()))?;

    // カーネルにサブミット＆完了を1件待つ
    ring.submit_and_wait(1)?;

    // Completion Queue Entry (CQE) を読む
    let cqe = ring.completion().next()
        .ok_or_else(|| io::Error::new(io::ErrorKind::Other, "no CQE"))?;

    let result = cqe.result();
    if result < 0 {
        return Err(io::Error::from_raw_os_error(-result));
    }

    println!("io_uring読み込み: {} バイト", result);
    println!("内容: {}", String::from_utf8_lossy(&buf[..result as usize]).trim());
    Ok(())
}
```

### io_uringバッチ処理（複数ファイルの並列読み込み）

```rust
#[cfg(target_os = "linux")]
fn io_uring_batch_read(paths: &[&str]) -> io::Result<Vec<Vec<u8>>> {
    use io_uring::{IoUring, opcode, types};
    use std::os::unix::io::AsRawFd;

    const BUF_SIZE: usize = 8192;
    let queue_size = paths.len().next_power_of_two() as u32;

    let mut ring = IoUring::new(queue_size)?;
    let mut files = Vec::new();
    let mut buffers: Vec<Vec<u8>> = (0..paths.len())
        .map(|_| vec![0u8; BUF_SIZE])
        .collect();

    // ファイルを開く
    for path in paths {
        files.push(std::fs::File::open(path)?);
    }

    // 全ファイルのSQEを一度にサブミット
    {
        let mut sq = ring.submission();
        for (i, file) in files.iter().enumerate() {
            let sqe = opcode::Read::new(
                types::Fd(file.as_raw_fd()),
                buffers[i].as_mut_ptr(),
                BUF_SIZE as u32,
            )
            .offset(0)
            .build()
            .user_data(i as u64); // インデックスをタグとして使用
            unsafe { sq.push(&sqe) }
                .map_err(|e| io::Error::new(io::ErrorKind::Other, e.to_string()))?;
        }
    }

    // 全件完了を待つ
    ring.submit_and_wait(paths.len())?;

    let mut results = vec![Vec::new(); paths.len()];
    for cqe in ring.completion() {
        let idx = cqe.user_data() as usize;
        let n = cqe.result();
        if n < 0 {
            eprintln!("ファイル[{}] 読み込み失敗: {}", idx, n);
            continue;
        }
        results[idx] = buffers[idx][..n as usize].to_vec();
    }

    Ok(results)
}

#[cfg(target_os = "linux")]
fn main() -> io::Result<()> {
    // io_uring バッチ読み込み
    let paths = vec!["/etc/hostname", "/etc/os-release"];
    let results = io_uring_batch_read(&paths)?;
    for (path, data) in paths.iter().zip(results.iter()) {
        println!("{}:\n  {}", path, String::from_utf8_lossy(data).trim());
    }
    Ok(())
}
```

## パフォーマンス比較の指針

| 手法 | スループット | レイテンシ | CPU使用率 | 用途 |
|------|------------|-----------|-----------|------|
| read/write (通常) | 中 | 中 | 低 | 一般的な用途 |
| O_DIRECT | 高 | 低（ディスク依存） | 低 | データベース・ストレージ |
| mmap | 高 | 低 | 低 | ランダムアクセス・大ファイル |
| sendfile | 高 | 低 | 非常に低 | ファイル転送・Webサーバー |
| io_uring | 最高 | 最低 | 低 | 高I/O負荷サーバー |

次節では、`fork`・`exec`・`waitpid`を使ったプロセス管理を学びます。
