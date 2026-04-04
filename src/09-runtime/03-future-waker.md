# 03: Futureとwaker — 非同期ランタイムの仕組み

## 非同期処理の核心問題

OSスレッドで「I/O待ち」をする場合、そのスレッドはスリープして何もできなくなります。100万のTCPコネクションを処理するなら、100万スレッドが必要になるでしょうか？それは現実的ではありません。

```
同期I/O (1スレッド = 1接続):                非同期I/O (1スレッド = N接続):

Thread0: [===作業===][...I/O待ち...][===作業===]    Thread0: [作業A][作業B][作業C]...
Thread1: [作業      ][    ブロック  ][...      ]             ↑ A がI/O待ちの間にBを処理
                                                             ↑ BがI/O待ちの間にCを処理

ムダが多い                                    効率的
```

Rustの `async`/`await` はこの問題を、**コルーチンとして関数を中断・再開できる状態機械**に変換することで解決します。その駆動エンジンが `Future` と `Waker` です。

---

## `Future` トレイトの定義

```rust
// std::future::Future の定義 (簡略版)
pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),    // 計算が完了した
    Pending,     // まだ完了していない。準備ができたら Waker を呼ぶ
}
```

```
poll() が呼ばれる
    │
    ├── Poll::Ready(value) → タスク完了
    │
    └── Poll::Pending      → I/O等を登録してReturnする
                              後で Waker::wake() を呼んで再度 poll してもらう
```

`poll` は非ブロッキングです。「まだできていない」なら即座に `Pending` を返し、準備ができたときに `Waker` 経由で通知します。

---

## Context と Waker

```
Context の役割:

  poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
                                     ↑
                                     └── Waker を1つ含む

  Waker の役割:
  ┌──────────────────────────────────────────────────────┐
  │  Future が「準備完了」になったとき wake() を呼ぶ     │
  │  ↓                                                   │
  │  Executor が該当タスクを再度キューに入れる           │
  │  ↓                                                   │
  │  次回のループで poll() が再度呼ばれる                │
  └──────────────────────────────────────────────────────┘
```

```rust
use std::task::{Context, Waker, Poll};
use std::pin::Pin;
use std::future::Future;

/// 最もシンプルなFuture: 即座に Ready を返す
struct Ready<T>(Option<T>);

impl<T: Unpin> Future for Ready<T> {
    type Output = T;

    fn poll(mut self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<T> {
        Poll::Ready(self.0.take().unwrap())
    }
}

/// 常に Pending を返す Future (絶対に完了しない)
struct Pending;

impl Future for Pending {
    type Output = ();

    fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<()> {
        // cx.waker() を保存して後で wake() を呼ばない限り
        // このタスクは永遠に再スケジュールされない
        Poll::Pending
    }
}
```

---

## Waker の低レイヤ実装 (`RawWakerVTable`)

`Waker` は実際には仮想関数テーブル (vtable) を使った手動vtableパターンです。`dyn Trait` を使わないのは、`Waker` がヒープアロケーション不要・サイズが固定という要件のためです。

```rust
use std::task::{RawWaker, RawWakerVTable, Waker};

// vtable に登録する4つの関数
unsafe fn clone_waker(data: *const ()) -> RawWaker {
    // data ポインタをインクリメントするなど参照カウントを増やす
    RawWaker::new(data, &VTABLE)
}

unsafe fn wake_waker(data: *const ()) {
    // data からスケジューラを復元してタスクを再キューに入れる
    println!("wake() が呼ばれました (data: {:p})", data);
}

unsafe fn wake_by_ref_waker(data: *const ()) {
    // wake と同様だが所有権を取らない版
    println!("wake_by_ref() が呼ばれました");
}

unsafe fn drop_waker(data: *const ()) {
    // data が指すリソースを解放する
    println!("Waker がドロップされました");
}

static VTABLE: RawWakerVTable = RawWakerVTable::new(
    clone_waker,
    wake_waker,
    wake_by_ref_waker,
    drop_waker,
);

fn create_noop_waker() -> Waker {
    let raw = RawWaker::new(std::ptr::null(), &VTABLE);
    // Safety: VTABLEの実装が正しい前提
    unsafe { Waker::from_raw(raw) }
}
```

`RawWakerVTable` の構造:

```
RawWaker {
    data: *const ()   ← タスクへのポインタなど任意のデータ
    vtable: &'static RawWakerVTable {
        clone:       fn(*const ()) -> RawWaker
        wake:        fn(*const ())   ← 所有権を取ってwake
        wake_by_ref: fn(*const ())   ← 参照でwake
        drop:        fn(*const ())   ← クリーンアップ
    }
}
```

---

## Futurを手動でpollする最小サンプル

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll, RawWaker, RawWakerVTable, Waker};

// noop waker (何もしないWaker) の生成
fn noop_waker() -> Waker {
    unsafe fn clone(p: *const ()) -> RawWaker { RawWaker::new(p, &VTABLE) }
    unsafe fn wake(_: *const ()) {}
    unsafe fn wake_by_ref(_: *const ()) {}
    unsafe fn drop(_: *const ()) {}
    static VTABLE: RawWakerVTable = RawWakerVTable::new(clone, wake, wake_by_ref, drop);
    unsafe { Waker::from_raw(RawWaker::new(std::ptr::null(), &VTABLE)) }
}

/// カウントダウンFuture: N回pollされたらReadyを返す
struct Countdown {
    remaining: u32,
}

impl Future for Countdown {
    type Output = &'static str;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<&'static str> {
        if self.remaining == 0 {
            Poll::Ready("カウントダウン完了!")
        } else {
            self.remaining -= 1;
            println!("  あと {} 回", self.remaining);
            // 即座に再pollしてほしいので wake を呼ぶ
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

fn main() {
    let mut future = Box::pin(Countdown { remaining: 3 });
    let waker = noop_waker();
    let mut cx = Context::from_waker(&waker);

    loop {
        match future.as_mut().poll(&mut cx) {
            Poll::Ready(msg) => {
                println!("完了: {}", msg);
                break;
            }
            Poll::Pending => {
                println!("まだPending...");
            }
        }
    }
}
```

出力:

```
  あと 2 回
まだPending...
  あと 1 回
まだPending...
  あと 0 回
まだPending...
完了: カウントダウン完了!
```

---

## `async`/`await` が展開される状態機械

コンパイラは `async fn` を状態機械に変換します。実際にどのように展開されるかを示します。

```rust
// これを書いたとき:
async fn my_async_fn() -> u32 {
    let x = some_future().await;
    let y = another_future(x).await;
    x + y
}
```

コンパイラが生成する（概念的な）コード:

```rust
// コンパイラが生成する状態機械 (概念コード)
enum MyAsyncFnState {
    Start,
    WaitingForSomeFuture {
        future: SomeFuture,
    },
    WaitingForAnotherFuture {
        x: u32,
        future: AnotherFuture,
    },
    Done,
}

struct MyAsyncFn {
    state: MyAsyncFnState,
}

impl Future for MyAsyncFn {
    type Output = u32;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<u32> {
        loop {
            match &mut self.state {
                MyAsyncFnState::Start => {
                    let future = some_future();
                    self.state = MyAsyncFnState::WaitingForSomeFuture { future };
                }
                MyAsyncFnState::WaitingForSomeFuture { future } => {
                    // Safety: self がPinされているので future も移動しない
                    let future = unsafe { Pin::new_unchecked(future) };
                    match future.poll(cx) {
                        Poll::Ready(x) => {
                            let future = another_future(x);
                            self.state = MyAsyncFnState::WaitingForAnotherFuture { x, future };
                        }
                        Poll::Pending => return Poll::Pending,
                    }
                }
                MyAsyncFnState::WaitingForAnotherFuture { x, future } => {
                    let future = unsafe { Pin::new_unchecked(future) };
                    match future.poll(cx) {
                        Poll::Ready(y) => {
                            let result = *x + y;
                            self.state = MyAsyncFnState::Done;
                            return Poll::Ready(result);
                        }
                        Poll::Pending => return Poll::Pending,
                    }
                }
                MyAsyncFnState::Done => panic!("完了済みのFutureをpollしました"),
            }
        }
    }
}
```

---

## `Pin` の役割

`Future` は状態機械として自己参照構造体になることがあります。自己参照構造体はメモリ上で移動させてはいけません。`Pin<&mut Self>` はそれを保証します。

```rust
// 自己参照の例: Future が自分自身の別フィールドへの参照を持つ
struct SelfRef {
    data: String,
    ptr: *const String,  // data への生ポインタ
}

// これを移動 (move) すると ptr が古いアドレスを指したまま → UB!
// Pin<&mut Self> はメモリ上の移動を防ぐ

use std::pin::Pin;
use std::marker::PhantomPinned;

struct NeverMove {
    data: u32,
    _pinned: PhantomPinned,  // このマーカーでUnpinを実装させない
}

fn use_pinned() {
    let mut val = NeverMove { data: 42, _pinned: PhantomPinned };
    // Box::pin でヒープに固定
    let mut pinned = Box::pin(val);

    // Pin<&mut NeverMove> 経由でしかアクセスできない
    let pin_ref: Pin<&mut NeverMove> = pinned.as_mut();
    // pin_ref.data = 100; // 直接変更できる
}
```

---

## `pin_mut!` マクロの役割

スタック上でFutureをpinするための便利マクロです。

```rust
use std::pin::pin;

async fn example() {
    let future = async { 42u32 };

    // pin! マクロでスタック上のFutureを固定
    // (tokioではpin_mut!マクロが同等の機能を提供)
    let mut pinned = pin!(future);

    // 固定されたFutureを複数回pollできる
    let waker = noop_waker();
    let mut cx = Context::from_waker(&waker);
    let _ = pinned.as_mut().poll(&mut cx);
}

// pin! を使わない場合の手動固定
fn manual_pin() {
    let mut future = async { 42u32 };
    // Safety: このブロックを抜けるまでfutureを移動しない
    let pinned = unsafe { Pin::new_unchecked(&mut future) };
}
```

---

## チャンネルを使ったWakerの実装

実際のランタイムでは、`Waker` はタスクをスケジューラのキューに戻す機能を持ちます。

```rust
use std::sync::{Arc, Mutex};
use std::sync::mpsc;
use std::task::{RawWaker, RawWakerVTable, Waker};
use std::future::Future;
use std::pin::Pin;

/// タスクの識別子とキューへの参照を持つWakerデータ
struct TaskWaker {
    task_id: u64,
    // タスクIDをExecutorのキューに送るチャンネル
    sender: mpsc::Sender<u64>,
}

impl TaskWaker {
    fn into_waker(self) -> Waker {
        let data = Arc::into_raw(Arc::new(self)) as *const ();
        unsafe { Waker::from_raw(RawWaker::new(data, &TASK_WAKER_VTABLE)) }
    }
}

unsafe fn task_wake(data: *const ()) {
    // Arc を復元して送信
    let arc = Arc::from_raw(data as *const TaskWaker);
    arc.sender.send(arc.task_id).ok();
    // Arc がここでdrop → refcountが減る (from_rawで所有権取得済み)
}

unsafe fn task_wake_by_ref(data: *const ()) {
    let arc = Arc::from_raw(data as *const TaskWaker);
    arc.sender.send(arc.task_id).ok();
    // by_ref は所有権を取らないので clone して leakする
    let _ = Arc::into_raw(Arc::clone(&arc));
    let _ = Arc::into_raw(arc); // 元のArcも戻す
}

unsafe fn task_clone(data: *const ()) -> RawWaker {
    let arc = Arc::from_raw(data as *const TaskWaker);
    let cloned = Arc::into_raw(Arc::clone(&arc));
    let _ = Arc::into_raw(arc); // 元を戻す
    RawWaker::new(cloned as *const (), &TASK_WAKER_VTABLE)
}

unsafe fn task_drop(data: *const ()) {
    drop(Arc::from_raw(data as *const TaskWaker));
}

static TASK_WAKER_VTABLE: RawWakerVTable = RawWakerVTable::new(
    task_clone,
    task_wake,
    task_wake_by_ref,
    task_drop,
);
```

---

## Futureのポーリングループ（シンプルなExecutor）

```rust
use std::collections::HashMap;
use std::future::Future;
use std::pin::Pin;
use std::sync::mpsc;
use std::task::{Context, Poll};

type BoxFuture = Pin<Box<dyn Future<Output = ()> + Send>>;

struct SimpleExecutor {
    tasks: HashMap<u64, BoxFuture>,
    sender: mpsc::Sender<u64>,
    receiver: mpsc::Receiver<u64>,
    next_id: u64,
}

impl SimpleExecutor {
    fn new() -> Self {
        let (sender, receiver) = mpsc::channel();
        SimpleExecutor {
            tasks: HashMap::new(),
            sender,
            receiver,
            next_id: 0,
        }
    }

    fn spawn<F: Future<Output = ()> + Send + 'static>(&mut self, future: F) -> u64 {
        let id = self.next_id;
        self.next_id += 1;
        self.tasks.insert(id, Box::pin(future));
        self.sender.send(id).unwrap();
        id
    }

    fn run(&mut self) {
        while let Ok(task_id) = self.receiver.recv() {
            if let Some(future) = self.tasks.get_mut(&task_id) {
                let waker = TaskWaker {
                    task_id,
                    sender: self.sender.clone(),
                }.into_waker();
                let mut cx = Context::from_waker(&waker);

                match future.as_mut().poll(&mut cx) {
                    Poll::Ready(()) => {
                        println!("タスク {} 完了", task_id);
                        self.tasks.remove(&task_id);
                    }
                    Poll::Pending => {
                        // Waker が登録されたので wake() が呼ばれるまで待つ
                    }
                }
            }

            // キューが空になったら終了
            if self.tasks.is_empty() {
                break;
            }
        }
    }
}
```

---

## まとめ

```
非同期処理の全体像:

  async fn foo() { ... }
       │
       ▼ コンパイラが変換
  impl Future for FooState { fn poll(...) }
       │
       ▼ Executorが駆動
  loop {
    task_id = queue.recv()
    match future.poll(&mut cx) {
      Ready(v) → タスク完了
      Pending  → Wakerを登録してスキップ
                 後で Waker::wake() により queue に task_id が入る
    }
  }
       │
       ▼ Reactorが通知
  epoll/kqueue → I/O完了 → waker.wake() → Executorのキューにtask_id
```

| 概念 | 役割 |
|------|------|
| `Future::poll` | 「今すぐできるか？」を確認する |
| `Poll::Pending` | まだできていない、Wakerを登録した |
| `Poll::Ready(v)` | 完了、結果はv |
| `Waker::wake()` | 「準備できた！もう一度pollして」という通知 |
| `Pin` | 自己参照Futureがメモリ上で移動しないことを保証 |
| `Context` | pollに渡される現在のWaker |

次節では、OSスレッドよりも軽量なグリーンスレッドをアセンブリを使って実装します。
