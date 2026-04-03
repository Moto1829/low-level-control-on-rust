# 02: 直接システムコール

## 直接システムコールとは

`libc`クレートはC標準ライブラリを経由しますが、
インラインアセンブリまたは`syscall`クレートを使うことで、
Cランタイムを一切介さずにカーネルに直接リクエストを送ることができます。

```
アプローチ①: libc経由
Rustコード → libc (glibc/musl) → sys_call → カーネル

アプローチ②: 直接syscall
Rustコード → syscall命令 (ASM) → カーネル
               ↑ 本節のアプローチ
```

この手法が役立つ場面:
- musl や静的リンクでlibcが使えない環境
- コンテナランタイム・OSカーネル自体の実装
- libcのオーバーヘッドを完全に排除したいパフォーマンスクリティカルなコード
- セキュアな環境でlibcを信頼しない場合（seccompフィルタ等）

> **対応プラットフォーム**: 主にLinux x86_64を対象とします。
> macOSでもシステムコールは発行できますが、番号が異なり、
> AppleはABIの安定性を保証しないため実用上は非推奨です。

## syscall番号の調べ方

### Linux x86_64 の場合

```bash
# カーネルヘッダから確認
cat /usr/include/asm/unistd_64.h | grep "__NR_"

# ausyscall ツールを使う（audit-libs インストール済みの場合）
ausyscall --dump | head -20

# Pythonで確認
python3 -c "import sys; print(sys.platform)"
python3 -c "import ctypes; print(ctypes.cdll.LoadLibrary('libc.so.6').syscall.__doc__)"
```

### 主要なsyscall番号（Linux x86_64）

```
syscall番号  名前        説明
─────────────────────────────────────────────────────
0           read        ファイルディスクリプタから読み込む
1           write       ファイルディスクリプタに書き込む
2           open        ファイルを開く（古い形式）
3           close       ファイルディスクリプタを閉じる
9           mmap        メモリマッピングを作成する
10          mprotect    メモリ領域の保護を変更する
11          munmap      メモリマッピングを解除する
22          pipe        パイプを作成する
57          fork        プロセスを複製する
59          execve      プログラムを実行する
60          exit        プロセスを終了する
61          wait4       子プロセスを待つ
89          readdir     ディレクトリエントリを読む
257         openat      ディレクトリfd相対でファイルを開く
```

### Rustコードでsyscall番号を参照する

```rust
// syscall番号の定数定義（Linux x86_64）
mod nr {
    pub const READ:    u64 = 0;
    pub const WRITE:   u64 = 1;
    pub const OPEN:    u64 = 2;
    pub const CLOSE:   u64 = 3;
    pub const MMAP:    u64 = 9;
    pub const MUNMAP:  u64 = 11;
    pub const FORK:    u64 = 57;
    pub const EXECVE:  u64 = 59;
    pub const EXIT:    u64 = 60;
    pub const OPENAT:  u64 = 257;
}
```

## インラインアセンブリによる直接syscall

### x86_64 syscallのABI

```
レジスタ    役割
─────────────────────────────────
rax         syscall番号 / 戻り値
rdi         第1引数
rsi         第2引数
rdx         第3引数
r10         第4引数（通常はrcxだがsyscallではr10）
r8          第5引数
r9          第6引数
rcx, r11    カーネルが破壊する（呼び出し規約上）
```

### 基本的なsyscallラッパー

```rust
// Linux x86_64 専用
#[cfg(all(target_os = "linux", target_arch = "x86_64"))]
mod syscall_raw {
    use std::arch::asm;

    /// 引数なしのsyscall
    pub unsafe fn syscall0(nr: u64) -> i64 {
        let ret: i64;
        asm!(
            "syscall",
            in("rax") nr,
            out("rcx") _,
            out("r11") _,
            lateout("rax") ret,
            options(nostack),
        );
        ret
    }

    /// 引数1つのsyscall
    pub unsafe fn syscall1(nr: u64, a1: u64) -> i64 {
        let ret: i64;
        asm!(
            "syscall",
            in("rax") nr,
            in("rdi") a1,
            out("rcx") _,
            out("r11") _,
            lateout("rax") ret,
            options(nostack),
        );
        ret
    }

    /// 引数3つのsyscall（read/writeなど）
    pub unsafe fn syscall3(nr: u64, a1: u64, a2: u64, a3: u64) -> i64 {
        let ret: i64;
        asm!(
            "syscall",
            in("rax") nr,
            in("rdi") a1,
            in("rsi") a2,
            in("rdx") a3,
            out("rcx") _,
            out("r11") _,
            lateout("rax") ret,
            options(nostack),
        );
        ret
    }

    /// 引数6つのsyscall（mmapなど）
    pub unsafe fn syscall6(
        nr: u64,
        a1: u64, a2: u64, a3: u64,
        a4: u64, a5: u64, a6: u64,
    ) -> i64 {
        let ret: i64;
        asm!(
            "syscall",
            in("rax") nr,
            in("rdi") a1,
            in("rsi") a2,
            in("rdx") a3,
            in("r10") a4,
            in("r8")  a5,
            in("r9")  a6,
            out("rcx") _,
            out("r11") _,
            lateout("rax") ret,
            options(nostack),
        );
        ret
    }
}
```

### syscall戻り値のエラーチェック

Linuxのsyscallは、エラーの場合に`-1`ではなく`-errno`（負のerrno値）を返します。
libcのラッパーとは異なるので注意が必要です。

```rust
use std::io;

/// syscall戻り値をio::Resultに変換
fn syscall_result(ret: i64) -> io::Result<i64> {
    if ret < 0 {
        // 戻り値が負 → -ret がerrno
        Err(io::Error::from_raw_os_error(-ret as i32))
    } else {
        Ok(ret)
    }
}
```

### write syscallの実装例

```rust
#[cfg(all(target_os = "linux", target_arch = "x86_64"))]
fn main() {
    use std::io;

    // write(1, buf, len) でstdoutに書き込む
    let msg = b"Hello from direct syscall!\n";

    let ret = unsafe {
        syscall_raw::syscall3(
            1,                           // NR_write
            1,                           // fd = stdout
            msg.as_ptr() as u64,         // buf
            msg.len() as u64,            // count
        )
    };

    match syscall_result(ret) {
        Ok(n) => eprintln!("{} バイト書き込み", n),
        Err(e) => eprintln!("write syscall失敗: {}", e),
    }
}
```

## mmapのsyscall呼び出し例

`mmap`は最も引数が多いsyscallの一つです（6引数）。
匿名マッピング（ファイルに紐付かないメモリ領域）を確保する例を示します。

```
mmap(addr, length, prot, flags, fd, offset)
  │
  ├─ addr   : 0 = カーネルが適切なアドレスを選ぶ
  ├─ length : 確保するバイト数
  ├─ prot   : PROT_READ | PROT_WRITE など
  ├─ flags  : MAP_ANONYMOUS | MAP_PRIVATE など
  ├─ fd     : -1 (匿名マッピングの場合)
  └─ offset : 0 (匿名マッピングの場合)
```

### mmap / munmap の定数

```rust
// mmap用の定数（Linux x86_64）
const PROT_NONE:    u64 = 0x0;
const PROT_READ:    u64 = 0x1;
const PROT_WRITE:   u64 = 0x2;
const PROT_EXEC:    u64 = 0x4;

const MAP_SHARED:   u64 = 0x01;
const MAP_PRIVATE:  u64 = 0x02;
const MAP_ANONYMOUS: u64 = 0x20;

const MAP_FAILED: *mut u8 = usize::MAX as *mut u8; // (void*)-1
```

### 匿名mmapの実装

```rust
#[cfg(all(target_os = "linux", target_arch = "x86_64"))]
fn mmap_anonymous(size: usize) -> std::io::Result<*mut u8> {
    let page_size = 4096usize;
    // ページサイズに切り上げ
    let aligned_size = (size + page_size - 1) & !(page_size - 1);

    let ret = unsafe {
        syscall_raw::syscall6(
            9,              // NR_mmap
            0,              // addr = NULL（カーネルに選ばせる）
            aligned_size as u64,
            PROT_READ | PROT_WRITE,
            MAP_ANONYMOUS | MAP_PRIVATE,
            u64::MAX,       // fd = -1 as u64
            0,              // offset = 0
        )
    };

    let addr = syscall_result(ret)? as usize as *mut u8;

    // MAP_FAILEDチェック
    if addr == MAP_FAILED {
        return Err(std::io::Error::last_os_error());
    }

    Ok(addr)
}

#[cfg(all(target_os = "linux", target_arch = "x86_64"))]
fn munmap(addr: *mut u8, size: usize) -> std::io::Result<()> {
    let ret = unsafe {
        syscall_raw::syscall2_helper(
            11, // NR_munmap
            addr as u64,
            size as u64,
        )
    };
    syscall_result(ret).map(|_| ())
}
```

### mmapを使った実際のサンプル

```rust
#[cfg(all(target_os = "linux", target_arch = "x86_64"))]
fn main() -> std::io::Result<()> {
    const SIZE: usize = 8192; // 2ページ分

    // メモリを確保
    let ptr = mmap_anonymous(SIZE)?;
    println!("mmap成功: アドレス = {:p}", ptr);

    // 書き込み
    let data = b"Direct mmap memory!\0";
    unsafe {
        std::ptr::copy_nonoverlapping(data.as_ptr(), ptr, data.len());
    }

    // 読み込み確認
    let read_str = unsafe {
        std::ffi::CStr::from_ptr(ptr as *const i8)
            .to_string_lossy()
    };
    println!("読み込み: {}", read_str);

    // 解放
    unsafe {
        let ret = syscall_raw::syscall3(
            11,          // NR_munmap
            ptr as u64,
            SIZE as u64,
            0,
        );
        syscall_result(ret)?;
    }
    println!("munmap成功");

    Ok(())
}
```

## syscallクレートを使う方法

インラインASMを自前で書く代わりに、`syscall`クレートを使うこともできます。

```toml
# Cargo.toml
[target.'cfg(target_os = "linux")'.dependencies]
syscall = "0.3"
```

```rust
#[cfg(target_os = "linux")]
fn main() {
    use syscall::syscall;

    // write(stdout, msg, len)
    let msg = "Hello via syscall crate!\n";
    unsafe {
        syscall!(WRITE, 1u64, msg.as_ptr() as u64, msg.len() as u64);
    }

    // getpid()
    let pid = unsafe { syscall!(GETPID) };
    println!("PID: {}", pid);
}
```

## vDSO（Virtual Dynamic Shared Object）

頻繁に呼ばれるsyscall（`gettimeofday`、`clock_gettime`など）は、
カーネルが`vDSO`という特殊なメモリ領域を介して**ユーザー空間**で実行します。
これによりコンテキストスイッチのコストがほぼゼロになります。

```
通常のsyscall:         vDSOを介したsyscall:
────────────────────   ─────────────────────────────────
ユーザー空間           ユーザー空間
  syscall命令            call [vDSO内の関数]
  ↓ (特権遷移)             ↓ (特権遷移なし！)
カーネル空間           vDSOページ（カーネルがマップ）
  sys_clock_gettime      clock_gettime実装（カーネルが保守）
  ↓ (特権遷移)
ユーザー空間
```

```rust
// clock_gettime は vDSO 経由で高速に実行される
fn get_time_ns() -> u64 {
    let mut ts = libc::timespec {
        tv_sec: 0,
        tv_nsec: 0,
    };
    unsafe {
        libc::clock_gettime(libc::CLOCK_MONOTONIC, &mut ts);
    }
    ts.tv_sec as u64 * 1_000_000_000 + ts.tv_nsec as u64
}
```

## まとめ

| 手法 | メリット | デメリット | 用途 |
|------|----------|-----------|------|
| `libc`クレート | 移植性高い・エラー処理済み | libcへの依存 | 一般的な低レベルプログラミング |
| インラインASM | 依存なし・最速 | アーキテクチャ固定・複雑 | OSカーネル・静的バイナリ |
| `syscall`クレート | ASMより簡潔 | マクロの透明性が低い | プロトタイピング |

次節では、これらの知識を応用して高度なファイルI/O操作を実装します。
