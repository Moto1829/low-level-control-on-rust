# 05: ミニ非同期ランタイムを作る

## 目標

tokio・async-std を**一切使わずに**、以下のものを動かします:

1. カスタム `Executor` — Futureのpollループ
2. カスタム `Reactor` — I/O完了の監視（epollベース）
3. カスタム `Waker` — ExecutorとReactorをつなぐ通知機構
4. タイマーFuture — 指定時間後に完了するFuture
5. TCPエコーサーバー — ミニランタイムで動く完全実装

---

## 3コンポーネントの設計図

```
  ┌──────────────────────────────────────────────────────────────┐
  │                     ミニ非同期ランタイム                      │
  │                                                              │
  │  ┌─────────────────────┐     wake(task_id)                  │
  │  │      Executor       │◄────────────────────────┐          │
  │  │  タスクキューのpoll  │                         │          │
  │  │  ループ              │                         │          │
  │  │  ready_queue:       │                         │          │
  │  │  VecDeque<TaskId>   │                         │          │
  │  └──────────┬──────────┘                         │          │
  │             │ poll()                              │          │
  │             ▼                                     │          │
  │  ┌─────────────────────┐     register_io()        │          │
  │  │      Task (Future)  │─────────────────────────►│         │
  │  │  async fn / impl    │                         │          │
  │  │  Future             │                         │          │
  │  └─────────────────────┘                  ┌──────┴──────┐   │
  │                                           │   Reactor   │   │
  │                                           │  epoll/     │   │
  │                                           │  kqueue     │   │
  │                                           │  I/O完了を  │   │
  │                                           │  監視       │   │
  │                                           └─────────────┘   │
  └──────────────────────────────────────────────────────────────┘

データフロー:
  1. Executor が Future::poll() を呼ぶ
  2. FutureがI/O待ちなら register_io(fd, waker) を Reactor に登録して Pending
  3. OSがI/O完了を通知 → Reactor が waker.wake() を呼ぶ
  4. wake() → Executor の ready_queue にタスクIDが入る
  5. Executor が再度 poll() を呼ぶ → Poll::Ready
```

---

## Executor の実装

```rust
use std::collections::{HashMap, VecDeque};
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, RawWaker, RawWakerVTable, Waker};

pub type TaskId = u64;
pub type BoxFuture = Pin<Box<dyn Future<Output = ()> + Send + 'static>>;

/// タスクキューの共有状態
struct ExecutorInner {
    tasks: HashMap<TaskId, BoxFuture>,
    ready_queue: VecDeque<TaskId>,
    next_id: TaskId,
}

pub struct Executor {
    inner: Arc<Mutex<ExecutorInner>>,
}

impl Executor {
    pub fn new() -> Executor {
        Executor {
            inner: Arc::new(Mutex::new(ExecutorInner {
                tasks: HashMap::new(),
                ready_queue: VecDeque::new(),
                next_id: 0,
            })),
        }
    }

    /// Futureをスポーンしてタスクキューに追加
    pub fn spawn<F>(&self, future: F) -> TaskId
    where
        F: Future<Output = ()> + Send + 'static,
    {
        let mut inner = self.inner.lock().unwrap();
        let id = inner.next_id;
        inner.next_id += 1;
        inner.tasks.insert(id, Box::pin(future));
        inner.ready_queue.push_back(id);
        id
    }

    /// メインのpollループ
    pub fn run(&self) {
        loop {
            // 実行可能なタスクIDを取り出す
            let task_id = {
                let mut inner = self.inner.lock().unwrap();
                if inner.tasks.is_empty() {
                    break; // 全タスク完了
                }
                inner.ready_queue.pop_front()
            };

            match task_id {
                None => {
                    // 実行可能なタスクがない = I/O待ち
                    // Reactor が wake() を呼ぶまで少し待つ
                    std::thread::sleep(std::time::Duration::from_millis(1));
                }
                Some(id) => {
                    // Waker を作成してタスクをpoll
                    let waker = self.create_waker(id);
                    let mut cx = Context::from_waker(&waker);

                    let mut inner = self.inner.lock().unwrap();
                    if let Some(future) = inner.tasks.get_mut(&id) {
                        match future.as_mut().poll(&mut cx) {
                            Poll::Ready(()) => {
                                inner.tasks.remove(&id);
                                println!("タスク {} 完了", id);
                            }
                            Poll::Pending => {
                                // Wakerが登録されたので待機
                            }
                        }
                    }
                }
            }
        }
    }

    /// タスクIDに紐付いたWakerを生成する
    fn create_waker(&self, task_id: TaskId) -> Waker {
        let data = Arc::new(WakerData {
            task_id,
            ready_queue: Arc::clone(&self.inner),
        });
        let raw = Arc::into_raw(data) as *const ();
        unsafe { Waker::from_raw(RawWaker::new(raw, &EXECUTOR_WAKER_VTABLE)) }
    }
}

struct WakerData {
    task_id: TaskId,
    ready_queue: Arc<Mutex<ExecutorInner>>,
}

fn wake_impl(data: *const ()) {
    let arc = unsafe { Arc::from_raw(data as *const WakerData) };
    arc.ready_queue
        .lock()
        .unwrap()
        .ready_queue
        .push_back(arc.task_id);
    // Arc が drop → refcount 減少
}

unsafe fn executor_wake(data: *const ()) {
    wake_impl(data);
}

unsafe fn executor_wake_by_ref(data: *const ()) {
    let arc = Arc::from_raw(data as *const WakerData);
    arc.ready_queue
        .lock()
        .unwrap()
        .ready_queue
        .push_back(arc.task_id);
    // by_ref: 所有権を返す
    let _ = Arc::into_raw(arc);
}

unsafe fn executor_clone(data: *const ()) -> RawWaker {
    let arc = Arc::from_raw(data as *const WakerData);
    let cloned = Arc::into_raw(Arc::clone(&arc));
    let _ = Arc::into_raw(arc);
    RawWaker::new(cloned as *const (), &EXECUTOR_WAKER_VTABLE)
}

unsafe fn executor_drop(data: *const ()) {
    drop(Arc::from_raw(data as *const WakerData));
}

static EXECUTOR_WAKER_VTABLE: RawWakerVTable = RawWakerVTable::new(
    executor_clone,
    executor_wake,
    executor_wake_by_ref,
    executor_drop,
);
```

---

## Reactor の実装（epollベース）

```rust
#[cfg(target_os = "linux")]
mod reactor {
    use super::*;
    use std::os::unix::io::RawFd;
    use std::collections::HashMap;
    use std::sync::Mutex;

    /// I/O イベントの種類
    #[derive(Clone, Copy)]
    pub enum Interest {
        Readable,
        Writable,
    }

    /// Reactor: epollでI/Oイベントを監視する
    pub struct Reactor {
        epoll_fd: RawFd,
        // fd → (interest, waker)
        waiters: Mutex<HashMap<RawFd, (Interest, Waker)>>,
    }

    impl Reactor {
        pub fn new() -> Reactor {
            let epoll_fd = unsafe {
                let fd = libc::epoll_create1(libc::EPOLL_CLOEXEC);
                if fd < 0 { panic!("epoll_create1 失敗"); }
                fd
            };
            Reactor {
                epoll_fd,
                waiters: Mutex::new(HashMap::new()),
            }
        }

        /// fdのI/OイベントをReactorに登録する
        pub fn register(&self, fd: RawFd, interest: Interest, waker: Waker) {
            let events = match interest {
                Interest::Readable => libc::EPOLLIN | libc::EPOLLET,
                Interest::Writable => libc::EPOLLOUT | libc::EPOLLET,
            };

            let mut event = libc::epoll_event {
                events: events as u32,
                u64: fd as u64, // fdをユーザーデータとして埋め込む
            };

            let ret = unsafe {
                libc::epoll_ctl(self.epoll_fd, libc::EPOLL_CTL_ADD, fd, &mut event)
            };
            if ret < 0 {
                // すでに登録済みなら MOD
                unsafe {
                    libc::epoll_ctl(self.epoll_fd, libc::EPOLL_CTL_MOD, fd, &mut event);
                }
            }

            self.waiters.lock().unwrap().insert(fd, (interest, waker));
        }

        /// イベントを待機してWakerを呼び出す (別スレッドで実行)
        pub fn wait_loop(&self) {
            const MAX_EVENTS: usize = 64;
            let mut events = vec![libc::epoll_event { events: 0, u64: 0 }; MAX_EVENTS];

            loop {
                let n = unsafe {
                    libc::epoll_wait(
                        self.epoll_fd,
                        events.as_mut_ptr(),
                        MAX_EVENTS as i32,
                        -1, // タイムアウトなし
                    )
                };

                if n < 0 {
                    let err = std::io::Error::last_os_error();
                    if err.kind() == std::io::ErrorKind::Interrupted { continue; }
                    eprintln!("epoll_wait エラー: {}", err);
                    break;
                }

                for i in 0..n as usize {
                    let fd = events[i].u64 as RawFd;
                    if let Some((_, waker)) = self.waiters.lock().unwrap().remove(&fd) {
                        // I/O が完了 → タスクを再スケジュール
                        waker.wake();
                    }
                }
            }
        }
    }

    impl Drop for Reactor {
        fn drop(&mut self) {
            unsafe { libc::close(self.epoll_fd); }
        }
    }
}
```

---

## カスタム Waker の実装（クロスプラットフォーム版）

```rust
use std::sync::{Arc, Mutex};
use std::collections::VecDeque;
use std::task::{RawWaker, RawWakerVTable, Waker};

/// チャンネルを使ったシンプルなWaker
/// epollが使えない環境でもテスト可能
pub struct ChannelWaker {
    task_id: u64,
    queue: Arc<Mutex<VecDeque<u64>>>,
}

impl ChannelWaker {
    pub fn new(task_id: u64, queue: Arc<Mutex<VecDeque<u64>>>) -> Waker {
        let data = Arc::into_raw(Arc::new(ChannelWaker { task_id, queue })) as *const ();
        unsafe { Waker::from_raw(RawWaker::new(data, &CHANNEL_VTABLE)) }
    }

    fn enqueue(&self) {
        self.queue.lock().unwrap().push_back(self.task_id);
    }
}

unsafe fn cw_clone(data: *const ()) -> RawWaker {
    let arc = Arc::from_raw(data as *const ChannelWaker);
    let cloned = Arc::into_raw(Arc::clone(&arc));
    std::mem::forget(arc); // refcount を戻す
    RawWaker::new(cloned as *const (), &CHANNEL_VTABLE)
}

unsafe fn cw_wake(data: *const ()) {
    let arc = Arc::from_raw(data as *const ChannelWaker);
    arc.enqueue();
    // arc が drop → refcount 減少
}

unsafe fn cw_wake_by_ref(data: *const ()) {
    let arc = Arc::from_raw(data as *const ChannelWaker);
    arc.enqueue();
    std::mem::forget(arc);
}

unsafe fn cw_drop(data: *const ()) {
    drop(Arc::from_raw(data as *const ChannelWaker));
}

static CHANNEL_VTABLE: RawWakerVTable = RawWakerVTable::new(cw_clone, cw_wake, cw_wake_by_ref, cw_drop);
```

---

## タイマーFutureの実装

### スレッドスリープベース（シンプル版）

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll};
use std::thread;
use std::time::{Duration, Instant};

pub struct TimerFuture {
    state: Arc<Mutex<TimerState>>,
}

struct TimerState {
    completed: bool,
    waker: Option<std::task::Waker>,
}

impl TimerFuture {
    /// `duration` 後に完了するFutureを作る
    pub fn new(duration: Duration) -> TimerFuture {
        let state = Arc::new(Mutex::new(TimerState {
            completed: false,
            waker: None,
        }));

        // バックグラウンドスレッドでタイマーを動かす
        let state_clone = Arc::clone(&state);
        thread::spawn(move || {
            thread::sleep(duration);
            let mut s = state_clone.lock().unwrap();
            s.completed = true;
            // Executor に「準備完了」を通知
            if let Some(waker) = s.waker.take() {
                waker.wake();
            }
        });

        TimerFuture { state }
    }
}

impl Future for TimerFuture {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        let mut state = self.state.lock().unwrap();
        if state.completed {
            Poll::Ready(())
        } else {
            // Wakerを登録 (タイマースレッドが呼ぶ)
            state.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}
```

---

## Oneshot チャンネルFuture

```rust
use std::sync::mpsc;

/// 一度だけ値を送受信できる非同期チャンネル
pub struct OneshotReceiver<T> {
    inner: Arc<Mutex<OneshotInner<T>>>,
}

pub struct OneshotSender<T> {
    inner: Arc<Mutex<OneshotInner<T>>>,
}

struct OneshotInner<T> {
    value: Option<T>,
    waker: Option<Waker>,
}

pub fn oneshot<T>() -> (OneshotSender<T>, OneshotReceiver<T>) {
    let inner = Arc::new(Mutex::new(OneshotInner {
        value: None,
        waker: None,
    }));
    (
        OneshotSender { inner: Arc::clone(&inner) },
        OneshotReceiver { inner },
    )
}

impl<T: Send + 'static> OneshotSender<T> {
    pub fn send(self, value: T) {
        let mut inner = self.inner.lock().unwrap();
        inner.value = Some(value);
        if let Some(waker) = inner.waker.take() {
            waker.wake();
        }
    }
}

impl<T> Future for OneshotReceiver<T> {
    type Output = T;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<T> {
        let mut inner = self.inner.lock().unwrap();
        match inner.value.take() {
            Some(v) => Poll::Ready(v),
            None => {
                inner.waker = Some(cx.waker().clone());
                Poll::Pending
            }
        }
    }
}
```

---

## ミニランタイムで動くTCPエコーサーバー

tokioを使わず、標準ライブラリの`TcpListener`を非ブロッキングモードで使い、ミニランタイム上で動かします。

```rust
use std::io::{self, Read, Write};
use std::net::{TcpListener, TcpStream};
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::collections::VecDeque;
use std::task::{Context, Poll};

// ── 非ブロッキングAcceptFuture ────────────────────────────────

struct AcceptFuture {
    listener: Arc<TcpListener>,
}

impl Future for AcceptFuture {
    type Output = io::Result<(TcpStream, std::net::SocketAddr)>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        match self.listener.accept() {
            Ok(result) => Poll::Ready(Ok(result)),
            Err(e) if e.kind() == io::ErrorKind::WouldBlock => {
                // まだ接続がない。Wakerを登録して待つ。
                // (実際のランタイムではepollに登録するが、ここではスピンポーリング)
                cx.waker().wake_by_ref();
                Poll::Pending
            }
            Err(e) => Poll::Ready(Err(e)),
        }
    }
}

// ── 非ブロッキングReadFuture ──────────────────────────────────

struct ReadFuture {
    stream: Arc<Mutex<TcpStream>>,
    buf: Vec<u8>,
}

impl Future for ReadFuture {
    type Output = io::Result<Vec<u8>>;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let mut stream = self.stream.lock().unwrap();
        let mut tmp = [0u8; 4096];
        match stream.read(&mut tmp) {
            Ok(0) => Poll::Ready(Ok(Vec::new())), // 接続終了
            Ok(n) => {
                self.buf.extend_from_slice(&tmp[..n]);
                Poll::Ready(Ok(self.buf.clone()))
            }
            Err(e) if e.kind() == io::ErrorKind::WouldBlock => {
                cx.waker().wake_by_ref(); // スピンポーリング
                Poll::Pending
            }
            Err(e) => Poll::Ready(Err(e)),
        }
    }
}

// ── エコーサーバーのロジック ─────────────────────────────────

async fn handle_client(stream: TcpStream) {
    stream.set_nonblocking(true).unwrap();
    let stream = Arc::new(Mutex::new(stream));

    println!("クライアント接続: {:?}", stream.lock().unwrap().peer_addr().ok());

    loop {
        let read_future = ReadFuture {
            stream: Arc::clone(&stream),
            buf: Vec::new(),
        };

        match read_future.await {
            Ok(data) if data.is_empty() => {
                println!("クライアントが切断しました");
                break;
            }
            Ok(data) => {
                println!("受信: {} バイト", data.len());
                // エコーバック
                if let Ok(mut s) = stream.lock() {
                    let _ = s.write_all(&data);
                }
            }
            Err(e) => {
                eprintln!("読み取りエラー: {}", e);
                break;
            }
        }
    }
}

async fn run_server(addr: &str) {
    let listener = TcpListener::bind(addr).unwrap();
    listener.set_nonblocking(true).unwrap();
    println!("TCPエコーサーバー起動: {}", addr);

    let listener = Arc::new(listener);

    loop {
        let accept_future = AcceptFuture { listener: Arc::clone(&listener) };
        match accept_future.await {
            Ok((stream, addr)) => {
                println!("新しい接続: {}", addr);
                // 本来はここで spawn してクライアントを並行処理
                handle_client(stream).await;
            }
            Err(e) => {
                eprintln!("Accept エラー: {}", e);
                break;
            }
        }
    }
}
```

---

## シンプルなブロッキングExecutorで動かす

```rust
/// 単一タスクをブロッキングで実行するシンプルなExecutor
pub fn block_on<F: Future>(future: F) -> F::Output {
    // noop waker: タスクが1つしかないのでwakeは何もしなくていい
    let waker = noop_waker();
    let mut cx = Context::from_waker(&waker);
    let mut future = Box::pin(future);

    loop {
        match future.as_mut().poll(&mut cx) {
            Poll::Ready(output) => return output,
            Poll::Pending => {
                // スピンポーリング (非効率だが動作する)
                std::hint::spin_loop();
            }
        }
    }
}

fn noop_waker() -> Waker {
    unsafe fn clone(p: *const ()) -> RawWaker { RawWaker::new(p, &VTABLE) }
    unsafe fn wake(_: *const ()) {}
    unsafe fn wake_by_ref(_: *const ()) {}
    unsafe fn drop(_: *const ()) {}
    static VTABLE: RawWakerVTable = RawWakerVTable::new(clone, wake, wake_by_ref, drop);
    unsafe { Waker::from_raw(RawWaker::new(std::ptr::null(), &VTABLE)) }
}

fn main() {
    // タイマーFutureのテスト
    block_on(async {
        println!("1秒待ちます...");
        TimerFuture::new(std::time::Duration::from_secs(1)).await;
        println!("完了!");
    });

    // Oneshot チャンネルのテスト
    block_on(async {
        let (tx, rx) = oneshot::<String>();

        std::thread::spawn(move || {
            std::thread::sleep(std::time::Duration::from_millis(100));
            tx.send("Hello from another thread!".to_string());
        });

        let msg = rx.await;
        println!("受信: {}", msg);
    });
}
```

---

## マルチタスクExecutorの実装

```rust
use std::sync::mpsc;

/// 複数タスクを並行実行できるExecutor
pub struct MultiTaskExecutor {
    ready_tx: mpsc::Sender<TaskId>,
    ready_rx: mpsc::Receiver<TaskId>,
    tasks: HashMap<TaskId, BoxFuture>,
    next_id: TaskId,
}

impl MultiTaskExecutor {
    pub fn new() -> Self {
        let (tx, rx) = mpsc::channel();
        MultiTaskExecutor {
            ready_tx: tx,
            ready_rx: rx,
            tasks: HashMap::new(),
            next_id: 0,
        }
    }

    pub fn spawn<F: Future<Output = ()> + Send + 'static>(&mut self, f: F) -> TaskId {
        let id = self.next_id;
        self.next_id += 1;
        self.tasks.insert(id, Box::pin(f));
        self.ready_tx.send(id).unwrap();
        id
    }

    pub fn run_until_complete(&mut self) {
        while !self.tasks.is_empty() {
            // タイムアウト付きで待機
            let id = match self.ready_rx.recv_timeout(std::time::Duration::from_millis(100)) {
                Ok(id) => id,
                Err(mpsc::RecvTimeoutError::Timeout) => continue,
                Err(mpsc::RecvTimeoutError::Disconnected) => break,
            };

            if let Some(future) = self.tasks.get_mut(&id) {
                let waker = {
                    let tx = self.ready_tx.clone();
                    make_waker(id, tx)
                };
                let mut cx = Context::from_waker(&waker);

                if let Poll::Ready(()) = future.as_mut().poll(&mut cx) {
                    self.tasks.remove(&id);
                }
            }
        }
    }
}

fn make_waker(task_id: TaskId, tx: mpsc::Sender<TaskId>) -> Waker {
    let data = Arc::into_raw(Arc::new((task_id, Mutex::new(tx)))) as *const ();

    unsafe fn clone_fn(data: *const ()) -> RawWaker {
        let arc = Arc::from_raw(data as *const (TaskId, Mutex<mpsc::Sender<TaskId>>));
        let cloned = Arc::into_raw(Arc::clone(&arc));
        std::mem::forget(arc);
        RawWaker::new(cloned as *const (), &MAKE_VTABLE)
    }
    unsafe fn wake_fn(data: *const ()) {
        let arc = Arc::from_raw(data as *const (TaskId, Mutex<mpsc::Sender<TaskId>>));
        arc.1.lock().unwrap().send(arc.0).ok();
    }
    unsafe fn wake_by_ref_fn(data: *const ()) {
        let arc = Arc::from_raw(data as *const (TaskId, Mutex<mpsc::Sender<TaskId>>));
        arc.1.lock().unwrap().send(arc.0).ok();
        std::mem::forget(arc);
    }
    unsafe fn drop_fn(data: *const ()) {
        drop(Arc::from_raw(data as *const (TaskId, Mutex<mpsc::Sender<TaskId>>)));
    }

    static MAKE_VTABLE: RawWakerVTable =
        RawWakerVTable::new(clone_fn, wake_fn, wake_by_ref_fn, drop_fn);

    unsafe { Waker::from_raw(RawWaker::new(data, &MAKE_VTABLE)) }
}

fn main() {
    let mut executor = MultiTaskExecutor::new();

    executor.spawn(async {
        println!("タスク A: 開始");
        TimerFuture::new(std::time::Duration::from_millis(100)).await;
        println!("タスク A: 完了");
    });

    executor.spawn(async {
        println!("タスク B: 開始");
        TimerFuture::new(std::time::Duration::from_millis(50)).await;
        println!("タスク B: 完了 (Aより先)");
    });

    executor.spawn(async {
        println!("タスク C: 即座に完了");
    });

    executor.run_until_complete();
    println!("全タスク完了");
}
```

期待出力:

```
タスク A: 開始
タスク B: 開始
タスク C: 即座に完了
タスク B: 完了 (Aより先)
タスク A: 完了
全タスク完了
```

---

## tokio との設計比較

```
┌──────────────────────────────────────────────────────────────────┐
│                     設計比較                                      │
├────────────────────┬──────────────────┬──────────────────────────┤
│ 要素               │ ミニランタイム   │ tokio                    │
├────────────────────┼──────────────────┼──────────────────────────┤
│ Executor           │ 単一スレッド     │ work-stealing マルチスレ│
│                    │ pollループ       │ ッドスケジューラ         │
├────────────────────┼──────────────────┼──────────────────────────┤
│ Reactor            │ なし/epoll単純版 │ mio (epoll/kqueue/IOCP)  │
│                    │                  │ プラットフォーム抽象化   │
├────────────────────┼──────────────────┼──────────────────────────┤
│ Waker              │ Arc<Mutex<Queue>>│ Arc-free 最適化版        │
│                    │ 手動vtable       │ 参照カウントなし         │
├────────────────────┼──────────────────┼──────────────────────────┤
│ タイマー           │ 別スレッドsleep  │ time-wheel (ヒープ不要)  │
├────────────────────┼──────────────────┼──────────────────────────┤
│ I/O抽象化          │ 生syscall        │ AsyncRead / AsyncWrite   │
│                    │                  │ トレイト                 │
├────────────────────┼──────────────────┼──────────────────────────┤
│ spawn              │ Box::pin 手動    │ JoinHandle + 自動Pin     │
└────────────────────┴──────────────────┴──────────────────────────┘
```

tokioの内部実装 (概念):

```rust
// tokio の Executor (概念コード)
struct TokioRuntime {
    // work-stealing デキュー (CPU数分)
    local_queues: Vec<crossbeam_deque::Worker<Task>>,
    // グローバルキュー (スティール元)
    global_queue: crossbeam_deque::Injector<Task>,
    // I/OポーリングのReactor (mioのEventLoop)
    reactor: mio::Poll,
    // タイマーホイール
    timer: TimerWheel,
}
```

---

## まとめ

この章で作ったミニランタイムの全体像:

```
[async fn] → [状態機械 (Future)] → [Executor::spawn]
                                          │
                              ┌───────────┴───────────┐
                              │  pollループ            │
                              │  ┌──────────────────┐  │
                              │  │ ready_queue から  │  │
                              │  │ task_id を取得   │  │
                              │  └────────┬─────────┘  │
                              │           │ poll()      │
                              │           ▼             │
                              │  ┌──────────────────┐  │
                              │  │ Future::poll     │  │
                              │  │ Ready → 完了     │  │
                              │  │ Pending → 待機   │  │
                              │  └──────────────────┘  │
                              └───────────────────────┘
                                          ▲
                                    waker.wake()
                                          │
                              ┌───────────┴───────────┐
                              │  TimerFuture          │
                              │  別スレッドでsleep後  │
                              │  wake() を呼ぶ        │
                              └───────────────────────┘
```

| コンポーネント | 実装方法 | tokioでの対応 |
|--------------|---------|-------------|
| Executor | `mpsc::channel` + pollループ | work-stealing scheduler |
| Waker | `Arc` + 手動vtable | 最適化済みArc-freeWaker |
| Reactor | epoll / スピンポーリング | mio (cross-platform) |
| TimerFuture | 別スレッドsleep | time-wheel |
| spawn | `Box::pin` | `JoinHandle` |

この実装をベースにすることで、tokioのソースコードを読む際にも「何のためのコードか」が明確に理解できるようになります。ランタイムを自作することで、非同期Rustの全レイヤが繋がって見えてきます。
