# 非同期処理のパフォーマンス

Rust の非同期処理は `async`/`await` 構文と `Future` トレイトを中心に設計されており、tokio などのランタイムがタスクのスケジューリングを担う。本章では tokio の内部モデルを理解し、実際のパフォーマンスチューニングに必要な知識を体系的に学ぶ。

---

## 1. tokio のスレッドモデル（work-stealing scheduler）

### マルチスレッドランタイムの構成

```
┌─────────────────────────────────────────────────────────┐
│                   tokio ランタイム                        │
│                                                         │
│  スレッド1   スレッド2   スレッド3   スレッド4             │
│  ┌───────┐  ┌───────┐  ┌───────┐  ┌───────┐           │
│  │ Local │  │ Local │  │ Local │  │ Local │           │
│  │ Queue │  │ Queue │  │ Queue │  │ Queue │           │
│  └───┬───┘  └───┬───┘  └───┬───┘  └───┬───┘           │
│      │          │          │          │                 │
│  ────┴──────────┴──────────┴──────────┴────             │
│              Global Injection Queue                     │
└─────────────────────────────────────────────────────────┘
```

Work-stealing: スレッドのローカルキューが空になると、他スレッドのキューから末尾のタスクを「盗む」ことで CPU をアイドルにしない。

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust
use tokio::runtime::Builder;

fn configure_tokio_runtime() {
    // デフォルト: CPU コア数のスレッドを使うマルチスレッドランタイム
    let rt = Builder::new_multi_thread()
        .worker_threads(4)           // ワーカースレッド数を明示
        .thread_name("my-worker")    // スレッド名（デバッグに有用）
        .thread_stack_size(2 * 1024 * 1024) // スタックサイズ 2MB
        .enable_all()
        .build()
        .unwrap();

    rt.block_on(async {
        println!("ランタイム設定完了");
        // アクティブなスレッド数を確認
        println!("ワーカースレッド数: 4");
    });
}

#[tokio::main]
async fn basic_concurrent_tasks() {
    // tokio::spawn でタスクをランタイムに送る
    let task1 = tokio::spawn(async {
        tokio::time::sleep(std::time::Duration::from_millis(100)).await;
        println!("タスク1 完了");
        42u32
    });

    let task2 = tokio::spawn(async {
        tokio::time::sleep(std::time::Duration::from_millis(50)).await;
        println!("タスク2 完了");
        100u32
    });

    // 両タスクを並行して待つ
    let (r1, r2) = tokio::join!(task1, task2);
    println!("結果: {} + {} = {}", r1.unwrap(), r2.unwrap(),
        r1.unwrap() + r2.unwrap());
}
```

### シングルスレッドランタイム

I/O バウンドかつシングルスレッドで十分な場合（例: ツール実装）。

```rust
fn single_thread_runtime() {
    let rt = tokio::runtime::Builder::new_current_thread()
        .enable_all()
        .build()
        .unwrap();

    rt.block_on(async {
        println!("シングルスレッドランタイム");
        // !Send な型も使える
        let rc = std::rc::Rc::new(42);
        println!("Rc 値: {}", rc);
    });
}
```

---

## 2. CPU-bound タスクを `spawn_blocking` へ逃がす

非同期ランタイムのスレッドは I/O 待ちが主目的であり、CPU-bound な処理（重い計算・同期ブロッキング処理）をワーカースレッド上で実行するとランタイム全体がブロックされる。

```rust
use std::time::Duration;
use tokio::task;

// 問題のあるパターン: CPU-bound 処理をそのまま async に書く
async fn bad_heavy_compute(n: u64) -> u64 {
    // これはワーカースレッドをブロックし、他タスクの実行を妨げる!
    let mut sum = 0u64;
    for i in 0..n {
        sum = sum.wrapping_add(i);
    }
    sum
}

// 正しいパターン: spawn_blocking でブロッキングスレッドプールに送る
async fn good_heavy_compute(n: u64) -> u64 {
    // spawn_blocking: 専用のブロッキングスレッドプールで実行
    // デフォルト最大 512 スレッド、設定変更可能
    task::spawn_blocking(move || {
        let mut sum = 0u64;
        for i in 0..n {
            sum = sum.wrapping_add(i);
        }
        sum
    })
    .await
    .unwrap()
}

#[tokio::main]
async fn spawn_blocking_example() {
    // 複数の重い計算を並行して実行
    let handles: Vec<_> = (0..4)
        .map(|i| tokio::spawn(good_heavy_compute(10_000_000 * (i + 1))))
        .collect();

    let results = futures::future::join_all(handles).await;
    for (i, r) in results.iter().enumerate() {
        println!("計算 {}: {}", i, r.as_ref().unwrap());
    }
}
```

### ブロッキングスレッドプールの設定

```rust
fn configure_blocking_pool() {
    let rt = tokio::runtime::Builder::new_multi_thread()
        .worker_threads(4)
        // ブロッキングスレッドの最大数 (デフォルト: 512)
        .max_blocking_threads(64)
        // ブロッキングスレッドのアイドル待機時間
        .thread_keep_alive(Duration::from_secs(30))
        .enable_all()
        .build()
        .unwrap();

    rt.block_on(async {
        println!("ブロッキングプール設定完了");
    });
}
```

### 同期 I/O ライブラリのラッピング

```rust
use tokio::task::spawn_blocking;
use std::io::Read;

async fn read_file_sync_wrapped(path: String) -> std::io::Result<String> {
    // std::fs は同期 I/O なので spawn_blocking でラップ
    spawn_blocking(move || {
        let mut file = std::fs::File::open(&path)?;
        let mut contents = String::new();
        file.read_to_string(&mut contents)?;
        Ok(contents)
    })
    .await
    .unwrap() // JoinError
}

// tokio::fs を使う方が望ましい（内部で spawn_blocking を使用）
async fn read_file_async(path: &str) -> std::io::Result<String> {
    tokio::fs::read_to_string(path).await
}
```

---

## 3. チャネルの選択とそれぞれのコスト

### tokio のチャネル種類

```rust
use tokio::sync::{mpsc, broadcast, watch, oneshot};

// mpsc: Multi-Producer Single-Consumer (最も一般的)
async fn mpsc_example() {
    // バウンドチャネル (キャパシティ 32)
    let (tx, mut rx) = mpsc::channel::<String>(32);

    let tx2 = tx.clone();

    tokio::spawn(async move {
        for i in 0..5 {
            tx.send(format!("producer1: {}", i)).await.unwrap();
        }
    });

    tokio::spawn(async move {
        for i in 0..5 {
            tx2.send(format!("producer2: {}", i)).await.unwrap();
        }
    });

    // Sender が全てドロップされるとチャネルが閉じる
    while let Some(msg) = rx.recv().await {
        println!("{}", msg);
    }
}

// broadcast: 全受信者に同じメッセージを配信
async fn broadcast_example() {
    // キャパシティ 16 (遅い受信者は古いメッセージを失う可能性)
    let (tx, _) = broadcast::channel::<String>(16);

    let mut rx1 = tx.subscribe();
    let mut rx2 = tx.subscribe();

    tx.send("全員に届くメッセージ".to_string()).unwrap();

    println!("rx1: {}", rx1.recv().await.unwrap());
    println!("rx2: {}", rx2.recv().await.unwrap());
}

// watch: 最新値のみを共有 (設定変更・状態共有に最適)
async fn watch_example() {
    let (tx, rx) = watch::channel("初期値".to_string());

    let mut rx_clone = rx.clone();

    tokio::spawn(async move {
        tokio::time::sleep(std::time::Duration::from_millis(50)).await;
        tx.send("更新された値".to_string()).unwrap();
    });

    // 変更を待つ
    rx_clone.changed().await.unwrap();
    println!("watch: {}", *rx_clone.borrow());
}

// oneshot: 一度だけ値を送受信 (RPC レスポンス等)
async fn oneshot_example() {
    let (tx, rx) = oneshot::channel::<u32>();

    tokio::spawn(async move {
        let result = 42u32; // 何らかの計算
        tx.send(result).unwrap();
    });

    let response = rx.await.unwrap();
    println!("oneshot: {}", response);
}
```

### チャネルのコスト比較

| チャネル | 送信コスト | 受信コスト | メモリ | 使いどころ |
|---------|-----------|-----------|--------|-----------|
| `mpsc::channel` (bounded) | O(1)、満杯時は await | O(1) | キャパシティ分 | 一般的なタスクキュー |
| `mpsc::unbounded_channel` | O(1)、ブロックなし | O(1) | 無制限（危険） | バックプレッシャー不要時 |
| `broadcast` | O(受信者数) | O(1) | キャパシティ × 受信者数 | イベント配信 |
| `watch` | O(受信者数) | O(1) | 最新値のみ | 設定・状態共有 |
| `oneshot` | O(1) | O(1) | 最小 | RPC、Future の完了通知 |

---

## 4. バックプレッシャーの実装パターン

バックプレッシャーとは「処理速度を超えた入力を受け付けないようにする」メカニズムである。

```rust
use tokio::sync::mpsc;
use std::time::Duration;

// パターン1: バウンドチャネルによる自然なバックプレッシャー
async fn bounded_channel_backpressure() {
    // キャパシティを超えると send が await でブロック
    let (tx, mut rx) = mpsc::channel::<Vec<u8>>(16);

    let producer = tokio::spawn(async move {
        for i in 0..100 {
            let data = vec![0u8; 1024]; // 1KB のペイロード
            // キューが満杯なら自動的に待機（バックプレッシャー）
            if tx.send(data).await.is_err() {
                println!("受信者がいなくなりました");
                break;
            }
            println!("送信: {}", i);
        }
    });

    let consumer = tokio::spawn(async move {
        while let Some(data) = rx.recv().await {
            // 低速な処理をシミュレート
            tokio::time::sleep(Duration::from_millis(10)).await;
            println!("処理: {} バイト", data.len());
        }
    });

    tokio::join!(producer, consumer);
}

// パターン2: セマフォによるコンカレント制限
use tokio::sync::Semaphore;
use std::sync::Arc;

async fn semaphore_rate_limiting() {
    // 同時実行を最大 5 に制限
    let semaphore = Arc::new(Semaphore::new(5));

    let mut handles = vec![];

    for i in 0..20 {
        let permit = Arc::clone(&semaphore);
        handles.push(tokio::spawn(async move {
            // セマフォが取得できるまで待つ
            let _permit = permit.acquire().await.unwrap();

            // 実際の処理（同時実行は最大 5）
            tokio::time::sleep(Duration::from_millis(100)).await;
            println!("タスク {} 完了", i);
            // _permit がドロップされるとセマフォが返却される
        }));
    }

    futures::future::join_all(handles).await;
}

// パターン3: try_send による非ブロッキング送信と廃棄
async fn try_send_with_drop() {
    let (tx, mut rx) = mpsc::channel::<String>(8);

    let producer = tokio::spawn(async move {
        for i in 0..50 {
            match tx.try_send(format!("item-{}", i)) {
                Ok(_) => {}
                Err(mpsc::error::TrySendError::Full(_)) => {
                    println!("キューが満杯: item-{} を廃棄", i);
                }
                Err(mpsc::error::TrySendError::Closed(_)) => break,
            }
        }
    });

    let consumer = tokio::spawn(async move {
        while let Some(item) = rx.recv().await {
            tokio::time::sleep(Duration::from_millis(20)).await;
            println!("受信: {}", item);
        }
    });

    tokio::time::sleep(Duration::from_millis(500)).await;
    tokio::join!(producer, consumer);
}
```

---

## 5. `async` のオーバーヘッド（Future・Poll・waker）

### Future の内部構造

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll, Waker};
use std::time::{Duration, Instant};
use std::sync::{Arc, Mutex};

// 手動実装した Future: 指定時間後に完了するタイマー
struct ManualTimer {
    deadline: Instant,
    waker: Arc<Mutex<Option<Waker>>>,
}

impl ManualTimer {
    fn new(duration: Duration) -> Self {
        let deadline = Instant::now() + duration;
        let waker: Arc<Mutex<Option<Waker>>> = Arc::new(Mutex::new(None));
        let waker_clone = Arc::clone(&waker);

        // バックグラウンドスレッドで waker を呼び出す
        std::thread::spawn(move || {
            let sleep_time = deadline.saturating_duration_since(Instant::now());
            std::thread::sleep(sleep_time);
            // タスクを再スケジュール
            if let Some(w) = waker_clone.lock().unwrap().take() {
                w.wake();
            }
        });

        ManualTimer { deadline, waker }
    }
}

impl Future for ManualTimer {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if Instant::now() >= self.deadline {
            Poll::Ready(())
        } else {
            // Waker を保存してタイムアウト後に wake を呼べるようにする
            *self.waker.lock().unwrap() = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}

async fn manual_future_usage() {
    println!("タイマー開始");
    ManualTimer::new(Duration::from_millis(100)).await;
    println!("タイマー完了");
}
```

### async 関数のステートマシン展開

`async fn` はコンパイラが自動的にステートマシンに変換する。各 `.await` ポイントが状態の境界になる。

```rust
// この async 関数は...
async fn example_async(x: u32) -> u32 {
    let a = tokio::time::sleep(Duration::from_millis(10)).await;
    let b = tokio::time::sleep(Duration::from_millis(10)).await;
    x + 42
}

// コンパイラが概念的に生成するステートマシン (簡略化)
enum ExampleFuture {
    State0 { x: u32 },
    State1 { x: u32, sleep1: tokio::time::Sleep },
    State2 { x: u32, sleep2: tokio::time::Sleep },
    Done,
}

// Future のサイズ = 全状態変数の最大サイズ
// → 深いネストや大きな変数はスタックフレームのコスト増大につながる
fn check_future_size() {
    use std::future::Future;

    // Future のサイズを確認
    println!("example_async Future のサイズ: {} bytes",
        std::mem::size_of_val(&example_async(0)));

    // 大きな配列を持つ Future は注意
    async fn large_future() {
        let _buf = [0u8; 65536]; // 64KB スタック割り当て
        tokio::time::sleep(Duration::from_millis(1)).await;
    }

    println!("large_future のサイズ: {} bytes",
        std::mem::size_of_val(&large_future()));
}
```

### オーバーヘッドを減らすテクニック

```rust
// 大きなデータは Box::pin または Arc でヒープに置く
async fn optimized_large_data() {
    // スタック上の大きな配列の代わりにヒープを使う
    let buf: Box<[u8; 65536]> = Box::new([0u8; 65536]);
    tokio::time::sleep(Duration::from_millis(1)).await;
    println!("バッファサイズ: {}", buf.len());
}

// タスクの粒度: 細かすぎる spawn はオーバーヘッドになる
async fn task_granularity_example() {
    // 悪い例: 小さい処理を大量に spawn
    let bad_handles: Vec<_> = (0..10000)
        .map(|i| tokio::spawn(async move { i * 2 }))
        .collect();

    // 良い例: バッチ処理で spawn 数を削減
    let good_handles: Vec<_> = (0..10)
        .map(|batch| {
            tokio::spawn(async move {
                (batch * 1000..(batch + 1) * 1000)
                    .map(|i| i * 2)
                    .sum::<usize>()
            })
        })
        .collect();

    let _bad_results = futures::future::join_all(bad_handles).await;
    let _good_results = futures::future::join_all(good_handles).await;
}
```

---

## 6. `tokio-console` でのプロファイリング

`tokio-console` は tokio タスクのリアルタイム監視ツール。タスクの待機時間・ポーリング回数・ビジーループを可視化できる。

### セットアップ

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
console-subscriber = "0.3"

# tokio-console のインストール
# cargo install tokio-console
```

```rust
// main.rs
fn main() {
    // tokio-console の subscriber を初期化（最初に呼ぶ）
    console_subscriber::init();

    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async_main());
}

async fn async_main() {
    // タスクに名前を付けると tokio-console で識別しやすい
    let task1 = tokio::task::Builder::new()
        .name("data-processor")
        .spawn(data_processor())
        .unwrap();

    let task2 = tokio::task::Builder::new()
        .name("http-handler")
        .spawn(http_handler())
        .unwrap();

    tokio::join!(task1, task2);
}

async fn data_processor() {
    loop {
        tokio::time::sleep(Duration::from_millis(100)).await;
        // 重い処理
    }
}

async fn http_handler() {
    loop {
        tokio::time::sleep(Duration::from_millis(50)).await;
    }
}
```

### よくあるパフォーマンス問題と tokio-console での診断

```rust
// 問題1: ビジーループ (tokio-console で "busy" 時間が長い)
async fn busy_wait_bad() {
    loop {
        if some_condition() {
            break;
        }
        // yield しないと他タスクが実行されない!
        // tokio-console でタスクのビジー率が 100% になる
    }
}

fn some_condition() -> bool { false }

// 修正: yield_now で他タスクに制御を渡す
async fn busy_wait_good() {
    loop {
        if some_condition() {
            break;
        }
        tokio::task::yield_now().await; // スケジューラに制御を返す
    }
}

// 問題2: ブロッキング操作 (tokio-console で poll 時間が長い)
async fn blocking_in_async_bad() {
    std::thread::sleep(Duration::from_millis(100)); // ブロッキング!
}

async fn blocking_in_async_good() {
    tokio::time::sleep(Duration::from_millis(100)).await; // 非同期
}

// 計測: タスクの実行時間を手動で記録
async fn measured_task(name: &str) {
    let start = std::time::Instant::now();

    // 処理...
    tokio::time::sleep(Duration::from_millis(50)).await;

    let elapsed = start.elapsed();
    if elapsed > Duration::from_millis(10) {
        // 閾値を超えたらログに記録
        tracing::warn!(
            task = name,
            elapsed_ms = elapsed.as_millis(),
            "タスクが予想より長くかかりました"
        );
    }
}
```

### `tracing` との連携

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

```rust
use tracing::{info, instrument, span, Level};

#[instrument] // 関数に自動でスパンを付与
async fn instrumented_handler(request_id: u64) {
    info!("リクエスト処理開始");

    let result = tokio::task::spawn_blocking(move || {
        // CPU-bound 処理
        request_id * 2
    }).await.unwrap();

    info!(result = result, "リクエスト処理完了");
}

fn setup_tracing() {
    use tracing_subscriber::{fmt, EnvFilter};

    // RUST_LOG=info cargo run でログレベルを制御
    fmt()
        .with_env_filter(EnvFilter::from_default_env())
        .with_target(false)
        .compact()
        .init();
}

#[tokio::main]
async fn main_with_tracing() {
    setup_tracing();

    for i in 0..5 {
        instrumented_handler(i).await;
    }
}
```

---

## まとめ

| テクニック | 効果 | 適用場面 |
|-----------|------|---------|
| `spawn_blocking` | CPU-bound をブロッキングプールに逃がす | 重い計算、同期 I/O |
| バウンドチャネル | 自然なバックプレッシャー | Producer-Consumer |
| `Semaphore` | 並行数の制限 | 外部 API レート制限 |
| `yield_now` | ビジーループの回避 | ポーリングループ |
| `tokio-console` | リアルタイムタスク監視 | パフォーマンス診断 |
| `#[instrument]` | タスクのトレーシング | デバッグ・可観測性 |

**最重要の原則**: async タスク内で絶対にブロッキング操作をしない。疑わしい処理は `spawn_blocking` に逃がし、`tokio-console` と `tracing` で実際の動作を測定して最適化する。
