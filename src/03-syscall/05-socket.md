# ソケットプログラミング

ソケットは Unix における汎用的な通信エンドポイントであり、TCP/UDP ネットワーク通信だけでなく Unix ドメインソケットによるプロセス間通信にも使われる。本章では libc を通じてソケット API を直接操作し、`std::net` との対比を通じて低レベル制御の意義を学ぶ。

---

## 1. ソケット API の基本フロー

### サーバー側の流れ

```
socket() → bind() → listen() → accept() → read/write → close()
```

### クライアント側の流れ

```
socket() → connect() → read/write → close()
```

---

## 2. `socket` / `bind` / `listen` / `accept` / `connect` の Rust からの呼び出し

```toml
# Cargo.toml
[dependencies]
libc = "0.2"
```

### TCP サーバーの最小実装

```rust
use libc::{
    socket, bind, listen, accept, read, write, close,
    sockaddr_in, AF_INET, SOCK_STREAM, INADDR_ANY,
    htons, htonl, SOL_SOCKET, SO_REUSEADDR, setsockopt,
};
use std::mem;

fn minimal_tcp_server(port: u16) {
    unsafe {
        // 1. ソケットの作成
        // AF_INET: IPv4, SOCK_STREAM: TCP, 0: 自動プロトコル選択
        let server_fd = socket(AF_INET, SOCK_STREAM, 0);
        if server_fd < 0 {
            eprintln!("socket の作成に失敗しました");
            return;
        }

        // 2. SO_REUSEADDR でポートの即時再利用を許可
        let opt: libc::c_int = 1;
        setsockopt(
            server_fd,
            SOL_SOCKET,
            SO_REUSEADDR,
            &opt as *const _ as *const libc::c_void,
            mem::size_of_val(&opt) as libc::socklen_t,
        );

        // 3. アドレスのバインド
        let addr = sockaddr_in {
            sin_family: AF_INET as libc::sa_family_t,
            sin_port: htons(port),
            sin_addr: libc::in_addr { s_addr: htonl(INADDR_ANY) },
            sin_zero: [0; 8],
        };

        if bind(
            server_fd,
            &addr as *const sockaddr_in as *const libc::sockaddr,
            mem::size_of::<sockaddr_in>() as libc::socklen_t,
        ) < 0 {
            eprintln!("bind に失敗しました (ポート {})", port);
            close(server_fd);
            return;
        }

        // 4. 接続待ちキューの設定 (backlog=128)
        if listen(server_fd, 128) < 0 {
            eprintln!("listen に失敗しました");
            close(server_fd);
            return;
        }

        println!("ポート {} でリッスン中...", port);

        // 5. クライアントの接続を受け入れ
        let mut client_addr: sockaddr_in = mem::zeroed();
        let mut addr_len = mem::size_of::<sockaddr_in>() as libc::socklen_t;

        let client_fd = accept(
            server_fd,
            &mut client_addr as *mut sockaddr_in as *mut libc::sockaddr,
            &mut addr_len,
        );

        if client_fd < 0 {
            eprintln!("accept に失敗しました");
            close(server_fd);
            return;
        }

        // クライアントの IP アドレスを表示
        let ip_bytes = client_addr.sin_addr.s_addr.to_ne_bytes();
        println!("クライアント接続: {}.{}.{}.{}:{}",
            ip_bytes[0], ip_bytes[1], ip_bytes[2], ip_bytes[3],
            u16::from_be(client_addr.sin_port));

        // 6. データの読み書き
        let mut buf = [0u8; 4096];
        let n = read(client_fd, buf.as_mut_ptr() as *mut libc::c_void, buf.len());
        if n > 0 {
            println!("受信 ({} バイト): {}",
                n, String::from_utf8_lossy(&buf[..n as usize]));
            // エコーバック
            write(client_fd, buf.as_ptr() as *const libc::c_void, n as usize);
        }

        close(client_fd);
        close(server_fd);
    }
}
```

### TCP クライアントの実装

```rust
use libc::{socket, connect, write, read, close, AF_INET, SOCK_STREAM, htons, inet_pton};
use std::mem;

fn tcp_client(host: &str, port: u16, message: &str) {
    unsafe {
        let fd = socket(AF_INET, SOCK_STREAM, 0);
        if fd < 0 {
            eprintln!("socket の作成に失敗しました");
            return;
        }

        let mut addr: libc::sockaddr_in = mem::zeroed();
        addr.sin_family = AF_INET as libc::sa_family_t;
        addr.sin_port = htons(port);

        // 文字列 IP アドレスをバイナリに変換
        let host_cstr = std::ffi::CString::new(host).unwrap();
        inet_pton(
            AF_INET,
            host_cstr.as_ptr(),
            &mut addr.sin_addr as *mut _ as *mut libc::c_void,
        );

        if connect(
            fd,
            &addr as *const libc::sockaddr_in as *const libc::sockaddr,
            mem::size_of::<libc::sockaddr_in>() as libc::socklen_t,
        ) < 0 {
            eprintln!("接続に失敗しました: {}:{}", host, port);
            close(fd);
            return;
        }

        println!("{}:{} に接続しました", host, port);

        write(fd, message.as_ptr() as *const libc::c_void, message.len());

        let mut buf = [0u8; 4096];
        let n = read(fd, buf.as_mut_ptr() as *mut libc::c_void, buf.len());
        if n > 0 {
            println!("応答: {}", String::from_utf8_lossy(&buf[..n as usize]));
        }

        close(fd);
    }
}
```

---

## 3. `setsockopt` での TCP チューニング

### `TCP_NODELAY` (Nagle アルゴリズムの無効化)

```rust
use libc::{setsockopt, IPPROTO_TCP, TCP_NODELAY};

fn set_tcp_nodelay(fd: libc::c_int) {
    unsafe {
        let opt: libc::c_int = 1;
        let ret = setsockopt(
            fd,
            IPPROTO_TCP,
            TCP_NODELAY,
            &opt as *const _ as *const libc::c_void,
            std::mem::size_of_val(&opt) as libc::socklen_t,
        );
        if ret != 0 {
            eprintln!("TCP_NODELAY の設定に失敗しました");
        } else {
            println!("TCP_NODELAY を有効化: 小さなパケットを即座に送信します");
        }
    }
}
```

**TCP_NODELAY が有効な場面**: ゲームサーバー、リアルタイム通信、RPC フレームワークなど、レイテンシが最優先の場合。

### `SO_REUSEADDR` と `SO_REUSEPORT`

```rust
fn set_socket_reuse_options(fd: libc::c_int) {
    unsafe {
        let opt: libc::c_int = 1;

        // SO_REUSEADDR: TIME_WAIT 状態のポートを再利用可能にする
        setsockopt(fd, libc::SOL_SOCKET, libc::SO_REUSEADDR,
            &opt as *const _ as *const libc::c_void,
            std::mem::size_of_val(&opt) as libc::socklen_t);

        // SO_REUSEPORT: 複数プロセス/スレッドで同一ポートを共有
        // (Linux 3.9+, macOS 対応)
        setsockopt(fd, libc::SOL_SOCKET, libc::SO_REUSEPORT,
            &opt as *const _ as *const libc::c_void,
            std::mem::size_of_val(&opt) as libc::socklen_t);

        println!("SO_REUSEADDR, SO_REUSEPORT を設定しました");
    }
}
```

### 受信・送信バッファサイズの調整

```rust
fn tune_socket_buffers(fd: libc::c_int) {
    unsafe {
        // 受信バッファを 256KB に設定
        let rcvbuf: libc::c_int = 256 * 1024;
        setsockopt(fd, libc::SOL_SOCKET, libc::SO_RCVBUF,
            &rcvbuf as *const _ as *const libc::c_void,
            std::mem::size_of_val(&rcvbuf) as libc::socklen_t);

        // 送信バッファを 256KB に設定
        let sndbuf: libc::c_int = 256 * 1024;
        setsockopt(fd, libc::SOL_SOCKET, libc::SO_SNDBUF,
            &sndbuf as *const _ as *const libc::c_void,
            std::mem::size_of_val(&sndbuf) as libc::socklen_t);

        // 実際に設定された値を確認（カーネルが倍にする場合がある）
        let mut actual_rcvbuf: libc::c_int = 0;
        let mut optlen = std::mem::size_of::<libc::c_int>() as libc::socklen_t;
        libc::getsockopt(fd, libc::SOL_SOCKET, libc::SO_RCVBUF,
            &mut actual_rcvbuf as *mut _ as *mut libc::c_void,
            &mut optlen);
        println!("実際の受信バッファサイズ: {} バイト", actual_rcvbuf);
    }
}
```

### `SO_KEEPALIVE` による接続死活監視

```rust
fn enable_keepalive(fd: libc::c_int) {
    unsafe {
        let opt: libc::c_int = 1;
        setsockopt(fd, libc::SOL_SOCKET, libc::SO_KEEPALIVE,
            &opt as *const _ as *const libc::c_void,
            std::mem::size_of_val(&opt) as libc::socklen_t);

        // Linux 固有: keepalive パラメータの細かい設定
        #[cfg(target_os = "linux")]
        {
            // アイドル後 60 秒で keepalive 開始
            let idle: libc::c_int = 60;
            setsockopt(fd, libc::IPPROTO_TCP, libc::TCP_KEEPIDLE,
                &idle as *const _ as *const libc::c_void,
                std::mem::size_of_val(&idle) as libc::socklen_t);

            // 10 秒間隔で再送
            let interval: libc::c_int = 10;
            setsockopt(fd, libc::IPPROTO_TCP, libc::TCP_KEEPINTVL,
                &interval as *const _ as *const libc::c_void,
                std::mem::size_of_val(&interval) as libc::socklen_t);

            // 5 回失敗したら切断
            let count: libc::c_int = 5;
            setsockopt(fd, libc::IPPROTO_TCP, libc::TCP_KEEPCNT,
                &count as *const _ as *const libc::c_void,
                std::mem::size_of_val(&count) as libc::socklen_t);
        }
    }
}
```

---

## 4. ノンブロッキングソケットと `poll` / `epoll` の基本

### ノンブロッキングモードの設定

```rust
use libc::{fcntl, F_GETFL, F_SETFL, O_NONBLOCK};

fn set_nonblocking(fd: libc::c_int) -> bool {
    unsafe {
        let flags = fcntl(fd, F_GETFL, 0);
        if flags < 0 {
            return false;
        }
        fcntl(fd, F_SETFL, flags | O_NONBLOCK) == 0
    }
}
```

### `poll` による I/O 多重化

```rust
use libc::{poll, pollfd, POLLIN, POLLOUT, POLLERR, POLLHUP};

fn poll_multiple_sockets(fds: &[libc::c_int]) {
    unsafe {
        let mut poll_fds: Vec<pollfd> = fds.iter().map(|&fd| pollfd {
            fd,
            events: POLLIN as libc::c_short,
            revents: 0,
        }).collect();

        // 1000ms タイムアウトで監視
        let ret = poll(
            poll_fds.as_mut_ptr(),
            poll_fds.len() as libc::nfds_t,
            1000,
        );

        match ret {
            -1 => eprintln!("poll エラー"),
            0  => println!("タイムアウト: 準備完了した FD なし"),
            n  => {
                println!("{} 個の FD が準備完了", n);
                for pfd in &poll_fds {
                    if pfd.revents & POLLIN as libc::c_short != 0 {
                        println!("FD {} が読み取り可能", pfd.fd);
                    }
                    if pfd.revents & POLLHUP as libc::c_short != 0 {
                        println!("FD {} が切断されました", pfd.fd);
                    }
                }
            }
        }
    }
}
```

### `epoll` による高効率 I/O 多重化 (Linux)

`epoll` は `poll` と異なり O(1) で監視できるため、多数の接続を扱うサーバーに適している。

```rust
#[cfg(target_os = "linux")]
fn epoll_server_loop(server_fd: libc::c_int) {
    use libc::{
        epoll_create1, epoll_ctl, epoll_wait,
        epoll_event, EPOLLIN, EPOLLET,
        EPOLL_CTL_ADD, EPOLL_CTL_DEL,
    };

    unsafe {
        // epoll インスタンスを作成
        let epfd = epoll_create1(0);
        if epfd < 0 {
            eprintln!("epoll_create1 に失敗しました");
            return;
        }

        // サーバーソケットを監視リストに追加
        let mut ev = epoll_event {
            events: (EPOLLIN | EPOLLET) as u32,
            u64: server_fd as u64,
        };
        epoll_ctl(epfd, EPOLL_CTL_ADD, server_fd, &mut ev);

        let mut events = vec![std::mem::zeroed::<epoll_event>(); 64];

        loop {
            // イベントを待つ (-1 = 無期限待機)
            let nfds = epoll_wait(
                epfd,
                events.as_mut_ptr(),
                events.len() as libc::c_int,
                -1,
            );

            for i in 0..nfds as usize {
                let fd = events[i].u64 as libc::c_int;

                if fd == server_fd {
                    // 新規接続
                    let client_fd = libc::accept(
                        server_fd,
                        std::ptr::null_mut(),
                        std::ptr::null_mut(),
                    );
                    if client_fd >= 0 {
                        set_nonblocking(client_fd);
                        let mut cev = epoll_event {
                            events: (EPOLLIN | EPOLLET) as u32,
                            u64: client_fd as u64,
                        };
                        epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &mut cev);
                        println!("新規クライアント: FD={}", client_fd);
                    }
                } else {
                    // 既存クライアントからのデータ
                    let mut buf = [0u8; 4096];
                    let n = libc::read(
                        fd,
                        buf.as_mut_ptr() as *mut libc::c_void,
                        buf.len(),
                    );
                    if n <= 0 {
                        // 接続終了
                        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, std::ptr::null_mut());
                        libc::close(fd);
                    } else {
                        // エコーバック
                        libc::write(fd, buf.as_ptr() as *const libc::c_void, n as usize);
                    }
                }
            }
        }
    }
}
```

---

## 5. `std::net` との対比

| 観点 | `std::net` | libc 直接 |
|------|-----------|-----------|
| 安全性 | 完全に安全 | `unsafe` 必須 |
| ポータビリティ | 高い | OS 依存コードが増える |
| ソケットオプション | 限定的 | `setsockopt` で全て制御可能 |
| I/O 多重化 | 非対応（tokio に委譲） | `poll`/`epoll`/`kqueue` 直接制御 |
| ノンブロッキング | `set_nonblocking` のみ | 細粒度の制御 |

```rust
// std::net での TCP サーバー（高レベル・安全）
use std::net::{TcpListener, TcpStream};
use std::io::{Read, Write};

fn std_net_server() {
    let listener = TcpListener::bind("0.0.0.0:8080")
        .expect("バインドに失敗しました");

    println!("ポート 8080 でリッスン中 (std::net)");

    for stream in listener.incoming() {
        match stream {
            Ok(mut client) => {
                std::thread::spawn(move || {
                    handle_client_std(&mut client);
                });
            }
            Err(e) => eprintln!("接続エラー: {}", e),
        }
    }
}

fn handle_client_std(stream: &mut TcpStream) {
    let mut buf = [0u8; 4096];
    loop {
        match stream.read(&mut buf) {
            Ok(0) => break, // 接続終了
            Ok(n) => {
                if stream.write_all(&buf[..n]).is_err() {
                    break;
                }
            }
            Err(_) => break,
        }
    }
}
```

---

## 6. シンプルな TCP エコーサーバーの完全実装

スレッドモデルによる TCP エコーサーバーの完全な実装例を示す。

```rust
use std::net::{TcpListener, TcpStream};
use std::io::{BufRead, BufReader, Write};
use std::thread;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;
use libc::{setsockopt, SOL_SOCKET, SO_REUSEADDR, IPPROTO_TCP, TCP_NODELAY};
use std::os::unix::io::AsRawFd;

static CONNECTION_COUNT: AtomicUsize = AtomicUsize::new(0);

fn handle_echo_client(mut stream: TcpStream, id: usize) {
    let peer = stream.peer_addr()
        .map(|a| a.to_string())
        .unwrap_or_else(|_| "不明".to_string());

    println!("[接続 {}] {} が接続しました", id, peer);

    // ソケットオプションを低レベルで設定
    let fd = stream.as_raw_fd();
    unsafe {
        let opt: libc::c_int = 1;
        setsockopt(fd, IPPROTO_TCP, TCP_NODELAY,
            &opt as *const _ as *const libc::c_void,
            std::mem::size_of_val(&opt) as libc::socklen_t);
    }

    let reader = BufReader::new(stream.try_clone().expect("クローンに失敗"));
    let mut bytes_transferred = 0usize;

    for line in reader.lines() {
        match line {
            Ok(text) => {
                let response = format!("ECHO: {}\n", text);
                bytes_transferred += response.len();
                if stream.write_all(response.as_bytes()).is_err() {
                    break;
                }
                // "quit" で接続を閉じる
                if text.trim() == "quit" {
                    let _ = stream.write_all(b"Goodbye!\n");
                    break;
                }
            }
            Err(_) => break,
        }
    }

    let count = CONNECTION_COUNT.fetch_sub(1, Ordering::Relaxed);
    println!("[接続 {}] {} が切断しました ({} バイト転送, 残り {} 接続)",
        id, peer, bytes_transferred, count - 1);
}

fn run_echo_server(port: u16) -> std::io::Result<()> {
    let listener = TcpListener::bind(format!("0.0.0.0:{}", port))?;

    // SO_REUSEADDR を libc で設定
    let server_fd = listener.as_raw_fd();
    unsafe {
        let opt: libc::c_int = 1;
        setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR,
            &opt as *const _ as *const libc::c_void,
            std::mem::size_of_val(&opt) as libc::socklen_t);
    }

    println!("TCP エコーサーバー起動: ポート {}", port);
    println!("接続コマンド例: nc localhost {}", port);

    let mut conn_id = 0usize;

    for stream in listener.incoming() {
        match stream {
            Ok(client) => {
                conn_id += 1;
                let id = conn_id;
                let count = CONNECTION_COUNT.fetch_add(1, Ordering::Relaxed) + 1;
                println!("現在の接続数: {}", count);

                thread::spawn(move || {
                    handle_echo_client(client, id);
                });
            }
            Err(e) => {
                eprintln!("accept エラー: {}", e);
            }
        }
    }

    Ok(())
}

fn main() {
    run_echo_server(8080).unwrap();
}
```

### 動作確認方法

```bash
# サーバーを起動
cargo run

# 別のターミナルで接続テスト
nc localhost 8080
Hello, World!
# => ECHO: Hello, World!
quit
# => Goodbye!
```

---

## まとめ

| API | 用途 | 特徴 |
|-----|------|------|
| `socket` | ソケット作成 | AF_INET/AF_UNIX, SOCK_STREAM/SOCK_DGRAM を指定 |
| `setsockopt` | ソケットオプション | TCP_NODELAY, SO_REUSEADDR, バッファサイズ等 |
| `poll` | I/O 多重化 | シンプル、O(n) 走査 |
| `epoll` (Linux) | 高効率 I/O 多重化 | O(1)、大量接続向け |
| `std::net` | 高レベル API | 安全・簡単だがオプション制御が限定的 |

低レベルのソケット API を使うことで、バッファサイズや Nagle アルゴリズムといったネットワークスタックの細部を制御でき、レイテンシやスループットを最適化できる。
