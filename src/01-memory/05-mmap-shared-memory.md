# mmap・共有メモリ・スタック計測

`mmap(2)` はプロセスのアドレス空間に任意のメモリ領域をマッピングするシステムコールです。
匿名マッピングによるヒープ代替・ファイルマッピング・プロセス間共有メモリ・実行可能コードの動的生成など、
OS レベルのメモリ操作の中心的な仕組みです。

---

## 1. `mmap(2)` の仕組みと用途

### アドレス空間のマッピングモデル

```text
プロセスの仮想アドレス空間

高アドレス
┌─────────────────────────────┐
│         スタック             │  ← スレッドごとに存在
│           ↓                 │
│                             │
│  mmap 領域（匿名・ファイル）  │  ← mmap で動的に追加
│                             │
│           ↑                 │
│         ヒープ               │  ← malloc/new が使う（brk/mmap）
├─────────────────────────────┤
│  BSS（未初期化グローバル）   │
│  データ（初期化済みグローバル）│
│  テキスト（コード）           │
└─────────────────────────────┘
低アドレス
```

### mmap の主な用途

| 用途 | フラグ | 説明 |
|------|--------|------|
| 匿名マッピング（プロセス専用） | `MAP_ANON \| MAP_PRIVATE` | ゼロ初期化されたページを確保 |
| ファイルマッピング（読み取り） | `MAP_SHARED` or `MAP_PRIVATE` | ファイル内容をメモリとして扱う |
| プロセス間共有メモリ | `MAP_SHARED` | 複数プロセスが同一物理ページを参照 |
| 実行可能メモリ（JIT） | `PROT_READ \| PROT_EXEC` | 動的生成コードを実行する |

---

## 2. `libc::mmap` での匿名メモリ確保

`cargo add libc` でlibc クレートを追加した上で、`mmap` を直接呼び出せます。

```rust
use libc::{
    mmap, munmap,
    MAP_ANON, MAP_PRIVATE, MAP_FAILED,
    PROT_READ, PROT_WRITE,
    c_void,
};
use std::ptr;

fn main() {
    let size: usize = 4 * 1024; // 4 KiB（1 ページ分）

    // mmap で匿名ページを確保する
    // addr:   NULL → カーネルがアドレスを決める
    // length: 確保サイズ
    // prot:   PROT_READ | PROT_WRITE → 読み書き可能
    // flags:  MAP_ANON | MAP_PRIVATE → 匿名・プロセス専用
    // fd:     -1（匿名マッピングでは使わない）
    // offset: 0
    let ptr = unsafe {
        mmap(
            ptr::null_mut(),
            size,
            PROT_READ | PROT_WRITE,
            MAP_ANON | MAP_PRIVATE,
            -1,
            0,
        )
    };

    if ptr == MAP_FAILED {
        eprintln!("mmap 失敗");
        return;
    }

    println!("マッピングアドレス: {:p}", ptr);

    // ゼロ初期化されているか確認
    let bytes = unsafe { std::slice::from_raw_parts(ptr as *const u8, size) };
    assert!(bytes.iter().all(|&b| b == 0), "mmap はゼロ初期化されている");
    println!("ゼロ初期化確認 OK");

    // 書き込みテスト
    unsafe {
        let slice = std::slice::from_raw_parts_mut(ptr as *mut u8, size);
        for (i, b) in slice.iter_mut().enumerate() {
            *b = (i % 256) as u8;
        }
        println!("先頭 4 バイト: {:?}", &slice[..4]);
    }

    // 必ず munmap で解放する（Drop トレイトはない）
    unsafe {
        if munmap(ptr, size) != 0 {
            eprintln!("munmap 失敗");
        }
    }
    println!("munmap 完了");
}
```

### ページサイズを確認する

```rust
fn main() {
    // sysconf(_SC_PAGESIZE) でページサイズを取得
    let page_size = unsafe { libc::sysconf(libc::_SC_PAGESIZE) };
    println!("ページサイズ: {} バイト", page_size); // 通常 4096 (x86-64) または 16384 (Apple Silicon)
}
```

---

## 3. `MAP_SHARED` による複数プロセス間の共有メモリ

`MAP_SHARED` を使うと、親子プロセス（`fork` 後）やファイルを通じた複数プロセスが
**同一の物理ページ**を共有できます。

```text
     fork() 前
     ┌─────────────────────┐
     │ 親プロセス           │
     │  mmap(MAP_SHARED) ──┼──→ 物理ページ [   0   ]
     └─────────────────────┘

     fork() 後
     ┌─────────────────────┐
     │ 親プロセス           │
     │  ptr ───────────────┼──→ 物理ページ [  42  ] ← 子が書いた値を読める
     └─────────────────────┘
     ┌─────────────────────┐
     │ 子プロセス           │
     │  ptr ───────────────┼──↗  (同じ物理ページ)
     └─────────────────────┘
```

### 完全なサンプル: 親子プロセスで値を共有

```rust
use libc::{
    mmap, munmap, fork, waitpid,
    MAP_ANON, MAP_SHARED, MAP_FAILED,
    PROT_READ, PROT_WRITE,
    WIFEXITED,
};
use std::ptr;
use std::sync::atomic::{AtomicI32, Ordering};

fn main() {
    let size = std::mem::size_of::<AtomicI32>();

    // MAP_SHARED で共有ページを確保（fork より前に行う）
    let ptr = unsafe {
        mmap(
            ptr::null_mut(),
            size,
            PROT_READ | PROT_WRITE,
            MAP_SHARED | MAP_ANON,
            -1,
            0,
        )
    };
    assert_ne!(ptr, MAP_FAILED, "mmap 失敗");

    // AtomicI32 として初期化（アトミック操作で競合を防ぐ）
    let shared: &AtomicI32 = unsafe {
        let atomic_ptr = ptr as *mut AtomicI32;
        atomic_ptr.write(AtomicI32::new(0));
        &*atomic_ptr
    };

    println!("[親] fork 前の共有値: {}", shared.load(Ordering::SeqCst));

    let pid = unsafe { fork() };
    match pid {
        -1 => {
            eprintln!("fork 失敗");
        }
        0 => {
            // 子プロセス
            println!("[子] 共有メモリに 42 を書き込む");
            shared.store(42, Ordering::SeqCst);
            println!("[子] 書き込み完了、終了");
            unsafe { libc::_exit(0); }
        }
        child_pid => {
            // 親プロセス: 子の終了を待つ
            let mut status: i32 = 0;
            unsafe { waitpid(child_pid, &mut status, 0); }

            if unsafe { WIFEXITED(status) } {
                let value = shared.load(Ordering::SeqCst);
                println!("[親] 子プロセス終了後の共有値: {}", value);
                // → 42 が読める（MAP_SHARED のため）
            }
        }
    }

    unsafe { munmap(ptr, size); }
}
```

> **注意**: `MAP_PRIVATE` の場合、子プロセスが書き込むと CoW（Copy-on-Write）が発生し、
> 親プロセスのページとは別の物理ページが割り当てられます。値は共有されません。

---

## 4. `mprotect` でページの保護属性を変更する

`mprotect` は mmap で確保したページの保護属性（読み取り・書き込み・実行）を動的に変更します。
JIT コンパイラや書き込み保護ガードの実装に使います。

### 書き込み保護の設定と解除

```rust
use libc::{
    mmap, munmap, mprotect,
    MAP_ANON, MAP_PRIVATE, MAP_FAILED,
    PROT_READ, PROT_WRITE, PROT_NONE,
};
use std::ptr;

fn main() {
    let page_size = unsafe { libc::sysconf(libc::_SC_PAGESIZE) as usize };
    let size = page_size * 2; // 2 ページ確保

    let ptr = unsafe {
        mmap(
            ptr::null_mut(),
            size,
            PROT_READ | PROT_WRITE,
            MAP_ANON | MAP_PRIVATE,
            -1,
            0,
        )
    };
    assert_ne!(ptr, MAP_FAILED);

    let slice = unsafe { std::slice::from_raw_parts_mut(ptr as *mut u8, size) };

    // まず書き込む
    slice[0] = 0xAB;
    println!("書き込み後: 0x{:02X}", slice[0]);

    // 2 ページ目を書き込み禁止にする
    let second_page = unsafe { (ptr as *mut u8).add(page_size) as *mut libc::c_void };
    let ret = unsafe { mprotect(second_page, page_size, PROT_READ) };
    assert_eq!(ret, 0, "mprotect 失敗");
    println!("2 ページ目を読み取り専用に変更");

    // 1 ページ目は書き込める
    slice[0] = 0xCD;
    println!("1 ページ目への書き込み: OK (0x{:02X})", slice[0]);

    // 2 ページ目への書き込みは SIGSEGV を引き起こす（コメントアウト）
    // slice[page_size] = 0xFF; // ← ここでプロセスがクラッシュする

    // 書き込み禁止を解除
    let ret = unsafe { mprotect(second_page, page_size, PROT_READ | PROT_WRITE) };
    assert_eq!(ret, 0);
    println!("保護を解除、再び書き込み可能");
    slice[page_size] = 0xFF;
    println!("2 ページ目への書き込み: OK (0x{:02X})", slice[page_size]);

    unsafe { munmap(ptr, size); }
}
```

### 実行可能メモリへの変換（JIT の基礎）

```rust
use libc::{
    mmap, munmap, mprotect,
    MAP_ANON, MAP_PRIVATE, MAP_FAILED,
    PROT_READ, PROT_WRITE, PROT_EXEC,
};
use std::ptr;

fn main() {
    let page_size = unsafe { libc::sysconf(libc::_SC_PAGESIZE) as usize };

    // まず書き込み可能なメモリを確保（EXEC は最初付けない）
    let ptr = unsafe {
        mmap(
            ptr::null_mut(),
            page_size,
            PROT_READ | PROT_WRITE,
            MAP_ANON | MAP_PRIVATE,
            -1,
            0,
        )
    };
    assert_ne!(ptr, MAP_FAILED);

    // x86-64 の機械語: mov eax, 42; ret
    // 0xB8, 0x2A, 0x00, 0x00, 0x00  → mov eax, 42
    // 0xC3                           → ret
    let code: &[u8] = &[0xB8, 0x2A, 0x00, 0x00, 0x00, 0xC3];

    // 書き込み可能な段階でコードをコピー
    unsafe {
        let dst = std::slice::from_raw_parts_mut(ptr as *mut u8, page_size);
        dst[..code.len()].copy_from_slice(code);
    }
    println!("機械語コードを書き込み完了");

    // W^X: 書き込みを外して実行可能にする
    let ret = unsafe { mprotect(ptr, page_size, PROT_READ | PROT_EXEC) };
    assert_eq!(ret, 0, "mprotect (EXEC) 失敗");
    println!("実行可能属性に変更");

    // 関数ポインタとしてキャストして呼び出す
    #[cfg(target_arch = "x86_64")]
    {
        let func: unsafe fn() -> i32 = unsafe { std::mem::transmute(ptr) };
        let result = unsafe { func() };
        println!("JIT 実行結果: {} (期待値: 42)", result);
    }
    #[cfg(not(target_arch = "x86_64"))]
    println!("このサンプルは x86-64 専用です");

    unsafe { munmap(ptr, page_size); }
}
```

---

## 5. `msync` / `munmap` による明示的なフラッシュと解放

ファイルマッピング（`MAP_SHARED`）では、書き込んだ内容はカーネルのページキャッシュにあります。
`msync` を呼ぶと変更をディスクに即座に反映させられます。

```text
プロセスの書き込み
    ↓
ページキャッシュ（RAM）← msync なしではここで止まる場合がある
    ↓  msync(MS_SYNC)
ディスク上のファイル
```

### ファイルマッピングのサンプル

```rust
use libc::{
    mmap, munmap, msync,
    MAP_SHARED, MAP_FAILED,
    PROT_READ, PROT_WRITE,
    MS_SYNC, MS_ASYNC,
    open, ftruncate, close,
    O_RDWR, O_CREAT,
};
use std::ffi::CString;
use std::ptr;

fn main() {
    let path = CString::new("/tmp/mmap_test.bin").unwrap();
    let file_size: usize = 4096;

    // ファイルを開く（存在しなければ作成）
    let fd = unsafe { open(path.as_ptr(), O_RDWR | O_CREAT, 0o644u32) };
    assert!(fd >= 0, "ファイルオープン失敗");

    // ファイルサイズを設定（mmap の前にサイズを確保しておく必要がある）
    unsafe { ftruncate(fd, file_size as i64); }

    // ファイルを MAP_SHARED でマッピング
    let ptr = unsafe {
        mmap(
            ptr::null_mut(),
            file_size,
            PROT_READ | PROT_WRITE,
            MAP_SHARED,
            fd,
            0,
        )
    };
    assert_ne!(ptr, MAP_FAILED, "mmap 失敗");
    unsafe { close(fd); } // mmap 後は fd を閉じてよい

    // ファイルにデータを書き込む（メモリへの書き込みがそのままファイルに反映）
    let slice = unsafe { std::slice::from_raw_parts_mut(ptr as *mut u8, file_size) };
    let message = b"Hello, mmap file!";
    slice[..message.len()].copy_from_slice(message);
    println!("データ書き込み完了: {:?}", std::str::from_utf8(&slice[..message.len()]).unwrap());

    // MS_SYNC: msync が戻るまでにディスクへの書き込みを完了させる
    // MS_ASYNC: 非同期でフラッシュを開始する（戻り値は即座）
    let ret = unsafe { msync(ptr, file_size, MS_SYNC) };
    assert_eq!(ret, 0, "msync 失敗");
    println!("msync(MS_SYNC) 完了 → /tmp/mmap_test.bin に書き込まれた");

    // マッピングを解放
    unsafe { munmap(ptr, file_size); }
    println!("munmap 完了");

    // 確認: 通常の read でファイル内容を読む
    let content = std::fs::read("/tmp/mmap_test.bin").unwrap();
    println!("ファイル先頭: {:?}", std::str::from_utf8(&content[..message.len()]).unwrap());

    // クリーンアップ
    std::fs::remove_file("/tmp/mmap_test.bin").ok();
}
```

### `msync` のフラグ

| フラグ | 挙動 |
|--------|------|
| `MS_SYNC` | ディスクへの書き込み完了まで呼び出しをブロックする |
| `MS_ASYNC` | フラッシュを非同期で開始してすぐ戻る |
| `MS_INVALIDATE` | 他のマッピングのキャッシュを無効化して最新を読む |

---

## 6. スタック使用量の計測

### `/proc/self/status` から VmStk を読む（Linux）

```rust
#[cfg(target_os = "linux")]
fn read_vm_stack_kb() -> Option<u64> {
    use std::fs;
    let status = fs::read_to_string("/proc/self/status").ok()?;
    for line in status.lines() {
        if line.starts_with("VmStk:") {
            // "VmStk:       132 kB" → 132 を取り出す
            let kb: u64 = line
                .split_whitespace()
                .nth(1)?
                .parse()
                .ok()?;
            return Some(kb);
        }
    }
    None
}

#[cfg(target_os = "linux")]
fn main() {
    if let Some(kb) = read_vm_stack_kb() {
        println!("スタック使用量 (VmStk): {} kB", kb);
    } else {
        println!("/proc/self/status を読み取れませんでした");
    }
}

#[cfg(not(target_os = "linux"))]
fn main() {
    println!("/proc/self/status は Linux 専用です");
}
```

### `getrlimit` でスタックの上限を取得する

```rust
use libc::{getrlimit, rlimit, RLIMIT_STACK};

fn main() {
    let mut lim = rlimit { rlim_cur: 0, rlim_max: 0 };
    let ret = unsafe { getrlimit(RLIMIT_STACK, &mut lim) };
    assert_eq!(ret, 0, "getrlimit 失敗");

    let cur_mib = lim.rlim_cur as f64 / 1024.0 / 1024.0;
    let max_mib = if lim.rlim_max == libc::RLIM_INFINITY {
        f64::INFINITY
    } else {
        lim.rlim_max as f64 / 1024.0 / 1024.0
    };

    println!("スタック上限 (ソフト): {:.1} MiB", cur_mib);
    println!("スタック上限 (ハード): {} MiB",
        if max_mib.is_infinite() { "無制限".to_string() } else { format!("{:.1}", max_mib) }
    );
    // 典型的な出力: ソフト 8.0 MiB, ハード 無制限（Linux）
    //              ソフト 8.0 MiB, ハード 64.0 MiB（macOS）
}
```

### スタックポインタをインラインアセンブリで読み取る

```rust
/// 現在のスタックポインタ（RSP レジスタ）を返す（x86-64 専用）
#[cfg(target_arch = "x86_64")]
fn get_stack_pointer() -> usize {
    let rsp: usize;
    unsafe {
        std::arch::asm!(
            "mov {}, rsp",
            out(reg) rsp,
        );
    }
    rsp
}

#[cfg(target_arch = "x86_64")]
fn measure_stack_usage() {
    let outer_sp = get_stack_pointer();
    println!("外側の RSP: 0x{:016X}", outer_sp);

    // 再帰関数内でスタックポインタがどれだけ動くか計測
    fn recursive(depth: usize, outer_sp: usize) {
        let inner_sp = get_stack_pointer();
        let used = outer_sp.saturating_sub(inner_sp);
        println!("深さ {:3}: RSP=0x{:016X}, 使用量={} バイト", depth, inner_sp, used);
        if depth < 5 {
            recursive(depth + 1, outer_sp);
        }
    }

    recursive(1, outer_sp);
}

#[cfg(target_arch = "x86_64")]
fn main() {
    measure_stack_usage();
}

#[cfg(not(target_arch = "x86_64"))]
fn main() {
    println!("このサンプルは x86-64 専用です");
}
```

### スタック使用量を関数単位で計測するラッパー

```rust
#[cfg(target_arch = "x86_64")]
fn measure<F: FnOnce() -> R, R>(label: &str, f: F) -> R {
    let sp_before: usize;
    unsafe { std::arch::asm!("mov {}, rsp", out(reg) sp_before); }

    let result = f();

    let sp_after: usize;
    unsafe { std::arch::asm!("mov {}, rsp", out(reg) sp_after); }

    // 呼び出し後は RSP が戻っているはずだが、最大使用量はわからない
    // ここでは呼び出し時点でのフレームサイズの下限を示す
    let frame_delta = sp_before.saturating_sub(sp_after);
    println!("[{}] フレームサイズの下限: {} バイト", label, frame_delta);
    result
}

#[cfg(target_arch = "x86_64")]
fn main() {
    let _ = measure("large_array", || {
        // スタックに大きな配列を確保
        let arr = [0u8; 1024];
        arr.iter().sum::<u8>()
    });
}

#[cfg(not(target_arch = "x86_64"))]
fn main() {}
```

---

## 7. スタックオーバーフローを意図的に引き起こす実験

> **警告**: 以下のコードはプロセスをクラッシュ（SIGSEGV/stack overflow）させます。
> `cargo run` で実行すると `thread 'main' has overflowed its stack` と表示されます。

### パターン 1: 無限再帰

```rust
fn infinite_recurse(n: u64) -> u64 {
    // スタックフレームを消費し続ける
    // n を使うことで最適化による末尾再帰除去を防ぐ
    let local = [0u8; 256]; // フレームごとに 256 バイト消費
    let _ = local;
    infinite_recurse(n + 1) + n // 末尾再帰でないので展開されない
}

fn main() {
    // スタックオーバーフローを意図的に発生させる
    // Rust は "thread 'main' has overflowed its stack" を出力して abort する
    println!("スタックオーバーフロー実験開始...");
    let _ = infinite_recurse(0);
}
```

### パターン 2: スタックオーバーフローをスレッドで安全に観察する

```rust
use std::thread;

fn main() {
    // スタックサイズを 64 KiB に制限したスレッドを生成
    let builder = thread::Builder::new()
        .name("small_stack".to_string())
        .stack_size(64 * 1024); // 64 KiB

    let handle = builder.spawn(|| {
        fn recurse(n: usize) -> usize {
            let local = [0u8; 512]; // フレームごと 512 バイト
            let _ = local;
            recurse(n + 1) + n
        }
        recurse(0)
    });

    // スレッドがパニック（スタックオーバーフロー）した場合を捕捉する
    match handle.unwrap().join() {
        Ok(v) => println!("完了（ありえないはず）: {}", v),
        Err(e) => println!("スレッドがパニック: {:?}", e),
    }
    // 出力例: thread 'small_stack' has overflowed its stack
    //         スレッドがパニック: Any { .. }
    println!("メインスレッドは生存している（スレッド隔離のおかげ）");
}
```

### スタックオーバーフローが起きる仕組み

```text
スタック（高アドレス → 低アドレスへ伸びる）

┌─────────────────────────────┐ ← スタック上端（スレッド生成時に確保）
│  フレーム 1 (main)           │
│  フレーム 2 (infinite_recurse n=0) │ 256 バイト
│  フレーム 3 (infinite_recurse n=1) │ 256 バイト
│  フレーム 4 (infinite_recurse n=2) │ 256 バイト
│         ...                 │
│  フレーム N                  │
├─────────────────────────────┤ ← スタック下端（ガードページ）
│  GUARD PAGE (mprotect NONE) │ ← ここへのアクセスで SIGSEGV 発生
└─────────────────────────────┘
```

OS はスタックの末端に **ガードページ**（`PROT_NONE` の mmap ページ）を設置しています。
スタックポインタがガードページを超えてアクセスしようとした瞬間に `SIGSEGV` が発生し、
Rust のランタイムがそれを捕捉して `"has overflowed its stack"` メッセージを出力します。

---

## まとめ

| 操作 | 関数 | 用途 |
|------|------|------|
| メモリ確保 | `mmap(MAP_ANON \| MAP_PRIVATE)` | プロセス専用の匿名ページ |
| 共有メモリ | `mmap(MAP_SHARED)` | fork 後の親子プロセス間共有 |
| 保護属性変更 | `mprotect` | 書き込み保護・実行可能化 |
| ディスク同期 | `msync(MS_SYNC)` | ファイルマッピングの即時フラッシュ |
| 解放 | `munmap` | マッピングの解放（忘れるとリーク） |
| スタック上限確認 | `getrlimit(RLIMIT_STACK)` | ソフト/ハード上限の取得 |
| スタックポインタ | インラインアセンブリ `mov {}, rsp` | 現在の RSP 値を取得 |

`mmap` は非常に強力ですが、`Drop` トレイトを持たないため**必ず `munmap` を呼ぶ**必要があります。
RAII でラップするか、`nix` クレートの `MmapMut` のような安全な抽象を使うことを推奨します。
