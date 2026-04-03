# プロセス生成・シグナル処理

Unixにおけるプロセス管理の核心は `fork`・`exec`・`waitpid` の3つのシステムコールにある。本章ではこれらをRustから直接呼び出す方法を学び、標準ライブラリの `std::process::Command` との違いを明確にしたうえで、シグナル処理・プロセス間通信まで踏み込む。

---

## 1. `fork` / `exec` / `waitpid` を libc で呼ぶ

### なぜ直接呼ぶのか

`std::process::Command` は安全で使いやすいが、内部で `posix_spawn` や `fork`+`exec` を使っており、細かい制御（ファイルディスクリプタの継承制御、子プロセスの namespace 変更、cgroup 設定など）が難しい。コンテナランタイムやプロセスマネージャを実装する場合は libc を直接呼ぶ必要がある。

```toml
# Cargo.toml
[dependencies]
libc = "0.2"
```

```rust
use libc::{fork, execvp, waitpid, WEXITSTATUS, WIFEXITED};
use std::ffi::CString;

fn main() {
    unsafe {
        let pid = fork();

        match pid {
            -1 => {
                eprintln!("fork に失敗しました");
                std::process::exit(1);
            }
            0 => {
                // 子プロセス: /bin/ls -la を実行する
                let cmd = CString::new("/bin/ls").unwrap();
                let arg0 = CString::new("ls").unwrap();
                let arg1 = CString::new("-la").unwrap();

                // execvp の argv は NULL 終端のポインタ配列
                let args: Vec<*const libc::c_char> = vec![
                    arg0.as_ptr(),
                    arg1.as_ptr(),
                    std::ptr::null(),
                ];

                execvp(cmd.as_ptr(), args.as_ptr());

                // execvp が返ってきた場合はエラー
                eprintln!("execvp に失敗しました");
                libc::_exit(1);
            }
            child_pid => {
                // 親プロセス: 子の終了を待つ
                let mut status: libc::c_int = 0;
                let waited = waitpid(child_pid, &mut status, 0);

                if waited == -1 {
                    eprintln!("waitpid に失敗しました");
                    return;
                }

                if WIFEXITED(status) {
                    println!("子プロセス (PID={}) が終了コード {} で終了しました",
                        child_pid, WEXITSTATUS(status));
                }
            }
        }
    }
}
```

### `fork` の重要な注意点

`fork` は「プロセス全体のコピー」を作る。マルチスレッドプログラムで `fork` を呼ぶと、子プロセスには呼び出したスレッドのみがコピーされ、他のスレッドが保持していた mutex が locked 状態のままになる危険がある。

```rust
// 危険なパターン: マルチスレッド環境での fork
use std::sync::Mutex;
use std::sync::Arc;

fn dangerous_fork_with_mutex() {
    let mutex = Arc::new(Mutex::new(0));
    let mutex_clone = mutex.clone();

    // 別スレッドが mutex を保持している間に fork すると...
    let _thread = std::thread::spawn(move || {
        let _guard = mutex_clone.lock().unwrap();
        std::thread::sleep(std::time::Duration::from_secs(10));
    });

    std::thread::sleep(std::time::Duration::from_millis(10));

    unsafe {
        let pid = libc::fork();
        if pid == 0 {
            // 子プロセスで mutex を取得しようとするとデッドロック!
            // let _g = mutex.lock().unwrap(); // ← 危険
            libc::_exit(0);
        }
        libc::waitpid(pid, std::ptr::null_mut(), 0);
    }
}
```

**原則: `fork` の後は速やかに `exec` を呼ぶ（fork-exec パターン）か、async-signal-safe な関数のみを使う。**

---

## 2. `std::process::Command` との違い・使い分け

| 観点 | `std::process::Command` | `fork` + `exec` (libc 直接) |
|------|--------------------------|------------------------------|
| 安全性 | Rust の安全性保証あり | `unsafe` ブロック必須 |
| 制御の細かさ | 限定的 | フルコントロール |
| ファイルディスクリプタ制御 | stdin/stdout/stderr のみ | 任意の FD を引き継ぐ/閉じる |
| 名前空間・cgroup | 不可 | `clone(2)` との組み合わせで可能 |
| 典型的ユースケース | 通常のコマンド実行 | コンテナ・デーモン・シェル実装 |

```rust
// std::process::Command の使い方（高レベル）
use std::process::Command;

fn run_with_command() {
    let output = Command::new("ls")
        .arg("-la")
        .output()
        .expect("ls の実行に失敗しました");

    println!("終了コード: {}", output.status.code().unwrap_or(-1));
    println!("stdout:\n{}", String::from_utf8_lossy(&output.stdout));
}
```

```rust
// libc を使った低レベル制御: 特定の FD だけ子プロセスに渡す
use libc::{fork, execvp, close, dup2, STDOUT_FILENO};
use std::ffi::CString;
use std::os::unix::io::RawFd;

fn fork_with_fd_control(log_fd: RawFd) {
    unsafe {
        let pid = fork();
        if pid == 0 {
            // stdout を log_fd にリダイレクト
            dup2(log_fd, STDOUT_FILENO);
            // log_fd 自体は閉じる（dup2 でコピー済みなので不要）
            close(log_fd);

            let cmd = CString::new("/usr/bin/env").unwrap();
            let arg0 = CString::new("env").unwrap();
            let args: Vec<*const libc::c_char> = vec![arg0.as_ptr(), std::ptr::null()];
            execvp(cmd.as_ptr(), args.as_ptr());
            libc::_exit(1);
        }
        close(log_fd); // 親プロセス側でも閉じる
        libc::waitpid(pid, std::ptr::null_mut(), 0);
    }
}
```

---

## 3. シグナルハンドラの登録

### `signal` による簡易登録

```rust
use libc::{signal, SIGINT, SIG_DFL};

extern "C" fn handle_sigint(sig: libc::c_int) {
    // async-signal-safe な関数のみ呼べる
    // println! は使えない（malloc を呼ぶため）
    let msg = b"SIGINT を受信しました\n";
    unsafe {
        libc::write(
            libc::STDERR_FILENO,
            msg.as_ptr() as *const libc::c_void,
            msg.len(),
        );
    }
}

fn register_signal_handler() {
    unsafe {
        // SIGINT (Ctrl+C) のハンドラを登録
        signal(SIGINT, handle_sigint as libc::sighandler_t);
    }

    println!("Ctrl+C を押してください...");
    loop {
        std::thread::sleep(std::time::Duration::from_secs(1));
    }
}
```

### `sigaction` による精密な登録

`signal` は動作がプラットフォームによって異なる。本番コードでは `sigaction` を使うべきである。

```rust
use libc::{sigaction, sigset_t, SA_RESTART, SA_SIGINFO, SIGTERM};
use std::mem;

extern "C" fn handle_sigterm(
    sig: libc::c_int,
    info: *mut libc::siginfo_t,
    _ctx: *mut libc::c_void,
) {
    let msg = b"SIGTERM: シャットダウン中...\n";
    unsafe {
        libc::write(libc::STDERR_FILENO, msg.as_ptr() as *const _, msg.len());
        // info から送信元 PID を取得できる
        if !info.is_null() {
            let _sender_pid = (*info).si_pid();
        }
    }
}

fn register_sigterm_with_sigaction() {
    unsafe {
        let mut sa: sigaction = mem::zeroed();
        sa.sa_sigaction = handle_sigterm as libc::sighandler_t;
        // シグナルハンドラ中の再入を防ぐマスク設定
        libc::sigemptyset(&mut sa.sa_mask);
        libc::sigaddset(&mut sa.sa_mask, SIGTERM);
        // SA_SIGINFO: siginfo_t を受け取る
        // SA_RESTART: システムコールを自動的に再試行する
        sa.sa_flags = (SA_SIGINFO | SA_RESTART) as libc::c_int;

        if sigaction(SIGTERM, &sa, std::ptr::null_mut()) != 0 {
            eprintln!("sigaction の登録に失敗しました");
        }
    }
}
```

### アトミックフラグによる安全なシャットダウン

シグナルハンドラ内でできることは非常に限られる。推奨パターンはアトミックフラグを立てて、メインループで確認することである。

```rust
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;

static SHUTDOWN: AtomicBool = AtomicBool::new(false);

extern "C" fn handle_shutdown(_sig: libc::c_int) {
    // AtomicBool::store は async-signal-safe
    SHUTDOWN.store(true, Ordering::Relaxed);
}

fn graceful_shutdown_example() {
    unsafe {
        libc::signal(libc::SIGINT, handle_shutdown as libc::sighandler_t);
        libc::signal(libc::SIGTERM, handle_shutdown as libc::sighandler_t);
    }

    println!("サーバー起動。Ctrl+C で終了します。");

    while !SHUTDOWN.load(Ordering::Relaxed) {
        // メインループの処理
        std::thread::sleep(std::time::Duration::from_millis(100));
    }

    println!("グレースフルシャットダウン完了");
}
```

---

## 4. `SIGCHLD` の処理とゾンビプロセスの回避

子プロセスが終了しても親が `waitpid` を呼ばないと、子のエントリがプロセステーブルに残る（ゾンビプロセス）。

```rust
use std::sync::atomic::{AtomicBool, Ordering};

static CHILD_EXITED: AtomicBool = AtomicBool::new(false);

extern "C" fn handle_sigchld(_sig: libc::c_int) {
    CHILD_EXITED.store(true, Ordering::Relaxed);
}

fn avoid_zombie_processes() {
    unsafe {
        // SIGCHLD ハンドラを登録
        libc::signal(libc::SIGCHLD, handle_sigchld as libc::sighandler_t);

        // 複数の子プロセスを生成
        for i in 0..3 {
            let pid = libc::fork();
            if pid == 0 {
                // 子プロセス: 少し待って終了
                std::thread::sleep(std::time::Duration::from_millis(100 * (i + 1)));
                libc::_exit(i as libc::c_int);
            }
        }

        // メインループ: SIGCHLD を受信したら waitpid で回収
        loop {
            if CHILD_EXITED.swap(false, Ordering::Relaxed) {
                // WNOHANG: ブロックせずに終了した子だけを回収
                loop {
                    let mut status = 0;
                    let pid = libc::waitpid(-1, &mut status, libc::WNOHANG);
                    if pid <= 0 {
                        break; // 回収できる子がいなくなった
                    }
                    if libc::WIFEXITED(status) {
                        println!("子プロセス PID={} が終了コード {} で終了",
                            pid, libc::WEXITSTATUS(status));
                    }
                }
            }
            std::thread::sleep(std::time::Duration::from_millis(50));
        }
    }
}
```

**`SA_NOCLDWAIT` フラグを使う方法**: カーネルに子の回収を任せ、ゾンビを完全に防ぐ。

```rust
fn use_sa_nocldwait() {
    unsafe {
        let mut sa: libc::sigaction = std::mem::zeroed();
        sa.sa_sigaction = libc::SIG_DFL;
        sa.sa_flags = libc::SA_NOCLDWAIT as libc::c_int;
        libc::sigaction(libc::SIGCHLD, &sa, std::ptr::null_mut());
        // 以降、子プロセスはゾンビにならない（waitpid は -1 を返す）
    }
}
```

---

## 5. `pipe(2)` によるプロセス間通信

```rust
use libc::{pipe, fork, read, write, close, STDIN_FILENO, STDOUT_FILENO};

fn pipe_example() {
    unsafe {
        // fds[0] = 読み取り端, fds[1] = 書き込み端
        let mut fds = [0i32; 2];
        if pipe(fds.as_mut_ptr()) == -1 {
            eprintln!("pipe の作成に失敗しました");
            return;
        }

        let [read_fd, write_fd] = fds;

        let pid = fork();
        match pid {
            -1 => eprintln!("fork に失敗しました"),
            0 => {
                // 子プロセス: 書き込み側
                close(read_fd); // 読み取り端を閉じる

                let message = b"Hello from child process!\n";
                write(
                    write_fd,
                    message.as_ptr() as *const libc::c_void,
                    message.len(),
                );
                close(write_fd);
                libc::_exit(0);
            }
            _parent => {
                // 親プロセス: 読み取り側
                close(write_fd); // 書き込み端を閉じる

                let mut buf = [0u8; 256];
                let n = read(
                    read_fd,
                    buf.as_mut_ptr() as *mut libc::c_void,
                    buf.len(),
                );
                close(read_fd);

                if n > 0 {
                    let received = std::str::from_utf8(&buf[..n as usize]).unwrap_or("");
                    print!("親プロセスが受信: {}", received);
                }

                libc::waitpid(pid, std::ptr::null_mut(), 0);
            }
        }
    }
}
```

### 双方向通信: `socketpair`

単方向の `pipe` の代わりに `socketpair` を使うと双方向通信が可能になる。

```rust
fn socketpair_example() {
    unsafe {
        let mut fds = [0i32; 2];
        // AF_UNIX, SOCK_STREAM で双方向パイプを作成
        libc::socketpair(
            libc::AF_UNIX,
            libc::SOCK_STREAM,
            0,
            fds.as_mut_ptr(),
        );

        let [parent_fd, child_fd] = fds;

        let pid = libc::fork();
        if pid == 0 {
            // 子プロセス
            close(parent_fd);

            let req = b"PING";
            write(child_fd, req.as_ptr() as *const _, req.len());

            let mut resp = [0u8; 64];
            let n = read(child_fd, resp.as_mut_ptr() as *mut _, resp.len());
            if n > 0 {
                println!("子: 受信 = {}", std::str::from_utf8(&resp[..n as usize]).unwrap_or(""));
            }
            close(child_fd);
            libc::_exit(0);
        } else {
            // 親プロセス
            close(child_fd);

            let mut req = [0u8; 64];
            let n = read(parent_fd, req.as_mut_ptr() as *mut _, req.len());
            if n > 0 {
                println!("親: 受信 = {}", std::str::from_utf8(&req[..n as usize]).unwrap_or(""));
                let resp = b"PONG";
                write(parent_fd, resp.as_ptr() as *const _, resp.len());
            }
            close(parent_fd);
            libc::waitpid(pid, std::ptr::null_mut(), 0);
        }
    }
}
```

---

## 6. `nix` クレートを使った安全なラッパー例

`nix` クレートは libc のシステムコールを Rust らしい型安全なインターフェースでラップする。

```toml
# Cargo.toml
[dependencies]
nix = { version = "0.27", features = ["process", "signal", "unistd"] }
```

```rust
use nix::unistd::{fork, ForkResult, execvp, pipe, read, write, close};
use nix::sys::wait::{waitpid, WaitStatus, WaitPidFlag};
use nix::sys::signal::{signal, Signal, SigHandler};
use std::ffi::CString;

fn nix_fork_exec_example() -> nix::Result<()> {
    let (read_fd, write_fd) = pipe()?;

    match unsafe { fork()? } {
        ForkResult::Child => {
            // 子プロセス
            close(read_fd)?;

            let msg = b"hello from nix child";
            write(write_fd, msg)?;
            close(write_fd)?;

            let cmd = CString::new("/bin/echo").unwrap();
            let args = vec![
                CString::new("echo").unwrap(),
                CString::new("nix exec example").unwrap(),
            ];
            execvp(&cmd, &args)?;
            unreachable!()
        }
        ForkResult::Parent { child } => {
            // 親プロセス
            close(write_fd)?;

            let mut buf = [0u8; 128];
            let n = read(read_fd, &mut buf)?;
            println!("受信: {}", std::str::from_utf8(&buf[..n]).unwrap_or(""));
            close(read_fd)?;

            // waitpid で子の終了を待つ
            match waitpid(child, None)? {
                WaitStatus::Exited(pid, code) => {
                    println!("PID={} が終了コード {} で終了", pid, code);
                }
                WaitStatus::Signaled(pid, sig, _) => {
                    println!("PID={} がシグナル {:?} で終了", pid, sig);
                }
                _ => {}
            }
        }
    }

    Ok(())
}
```

```rust
// nix によるシグナルハンドラ登録
use nix::sys::signal::{self, Signal, SigAction, SigSet, SaFlags};

extern "C" fn nix_sigint_handler(_: libc::c_int) {
    let msg = b"\nSIGINT を受信しました (nix)\n";
    unsafe { libc::write(libc::STDERR_FILENO, msg.as_ptr() as *const _, msg.len()); }
}

fn register_with_nix() -> nix::Result<()> {
    let sa = SigAction::new(
        signal::SigHandler::Handler(nix_sigint_handler),
        SaFlags::SA_RESTART,
        SigSet::empty(),
    );
    unsafe { signal::sigaction(Signal::SIGINT, &sa)? };
    println!("SIGINT ハンドラを登録しました");
    Ok(())
}
```

---

## まとめ

| システムコール | 役割 | 注意点 |
|---------------|------|--------|
| `fork` | プロセスの複製 | マルチスレッド環境では危険。fork-exec パターンを徹底する |
| `exec` 族 | プロセスイメージの置き換え | 失敗時は `_exit` で終了すること |
| `waitpid` | 子プロセスの回収 | `WNOHANG` で非ブロッキング回収、ゾンビ防止 |
| `signal` / `sigaction` | シグナルハンドラ登録 | ハンドラ内は async-signal-safe 関数のみ |
| `pipe` | 一方向 IPC | 双方向なら `socketpair` を使う |

`nix` クレートを使うことで `unsafe` ブロックを最小化しつつ、低レベルなプロセス制御を実現できる。
