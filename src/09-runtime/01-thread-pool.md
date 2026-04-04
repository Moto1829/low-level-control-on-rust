# 01: スレッドプールの自作

## スレッドプールとは何か

HTTPサーバーがリクエストごとにOSスレッドを生成すると、スレッド生成コスト（典型的には数百マイクロ秒）と大量のスタックメモリ消費が問題になります。スレッドプールは**あらかじめスレッドを作っておき、ジョブをキューに入れて再利用する**パターンです。

```
リクエスト1 ─┐
リクエスト2 ─┼──→ [ジョブキュー] ──→ [Worker0]
リクエスト3 ─┤                   ├──→ [Worker1]
リクエスト4 ─┘                   └──→ [Worker2]

スレッドは作り直さない — ジョブだけを流す
```

---

## 設計図

```
ThreadPool
  │
  ├── sender: mpsc::Sender<Message>   ← ジョブを送る側
  │
  └── workers: Vec<Worker>
        │
        └── Worker { id, thread: JoinHandle<()> }
                │
                └── loop {
                      let job = receiver.lock().unwrap().recv();
                      match job {
                        Ok(Message::Job(f)) => f(),
                        Ok(Message::Terminate) => break,
                        Err(_) => break,
                      }
                    }

Arc<Mutex<Receiver<Message>>>
  ↑ 複数Workerが同じreceiverを共有するために必要
```

データフロー:

```
execute(job)
    │
    ▼
sender.send(Message::Job(job))
    │
    ▼
[Channel Buffer]
    │
    ▼
Worker::thread が receiver.lock().recv() で取り出す
    │
    ▼
job()  ← クロージャを実行
```

---

## 基本実装

まず動く最小実装から始めます。

```rust
use std::sync::{Arc, Mutex};
use std::sync::mpsc;
use std::thread;

// ジョブの型エイリアス: Send + 'static が必要
type Job = Box<dyn FnOnce() + Send + 'static>;

enum Message {
    NewJob(Job),
    Terminate,
}

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Message>,
}

struct Worker {
    id: usize,
    thread: Option<thread::JoinHandle<()>>,
}
```

`FnOnce() + Send + 'static` の各制約の意味:
- `FnOnce()` — 一度だけ呼ばれるクロージャ
- `Send` — スレッドをまたいで送れる
- `'static` — スレッドが終わるまで生存保証が取れる（借用なし）

---

## Workerの実装

```rust
impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Message>>>) -> Worker {
        let thread = thread::Builder::new()
            .name(format!("worker-{}", id))
            .spawn(move || loop {
                // Mutex を取得してメッセージを受信
                // lock() が失敗するのは他スレッドがパニックした場合
                let message = receiver
                    .lock()
                    .expect("Mutex が汚染されています")
                    .recv();

                match message {
                    Ok(Message::NewJob(job)) => {
                        println!("Worker {} がジョブを実行します", id);
                        job();
                    }
                    Ok(Message::Terminate) => {
                        println!("Worker {} が終了します", id);
                        break;
                    }
                    Err(_) => {
                        // Sender がドロップされた = シャットダウン
                        println!("Worker {}: チャンネルが閉じました", id);
                        break;
                    }
                }
            })
            .expect("スレッドの生成に失敗しました");

        Worker {
            id,
            thread: Some(thread),
        }
    }
}
```

---

## ThreadPoolの実装

```rust
impl ThreadPool {
    /// スレッド数を指定してプールを生成する
    ///
    /// # Panics
    /// `size` が 0 の場合はパニックする
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0, "スレッド数は1以上が必要です");

        let (sender, receiver) = mpsc::channel();

        // 複数Workerでreceiverを共有するためにArc<Mutex>でラップ
        let receiver = Arc::new(Mutex::new(receiver));

        let mut workers = Vec::with_capacity(size);
        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }

        ThreadPool { workers, sender }
    }

    /// CPUコア数に合わせてスレッド数を自動決定する
    pub fn new_auto() -> ThreadPool {
        let num_threads = std::thread::available_parallelism()
            .map(|n| n.get())
            .unwrap_or(4);
        println!("スレッドプール: {} スレッドで起動", num_threads);
        ThreadPool::new(num_threads)
    }

    /// ジョブをスレッドプールに送信する
    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job = Box::new(f);
        self.sender
            .send(Message::NewJob(job))
            .expect("スレッドプールへのジョブ送信に失敗しました");
    }
}
```

---

## グレースフルシャットダウン

`Drop` トレイトを実装して、プールがスコープを外れたときにすべてのスレッドを正しく終了させます。

```rust
impl Drop for ThreadPool {
    fn drop(&mut self) {
        println!("全Workerに終了メッセージを送信します...");

        // 1) Terminate メッセージをWorker数だけ送る
        for _ in &self.workers {
            self.sender
                .send(Message::Terminate)
                .expect("Terminateの送信に失敗しました");
        }

        // 2) 各Workerのスレッドが終了するまで待つ
        for worker in &mut self.workers {
            println!("Worker {} の終了を待っています...", worker.id);
            if let Some(thread) = worker.thread.take() {
                thread.join().expect("Workerスレッドのjoinに失敗しました");
            }
        }

        println!("全Workerが正常に終了しました");
    }
}
```

シャットダウンの流れ:

```
drop(pool) が呼ばれる
    │
    ├─ Terminate × N を送信
    │      ↓
    │  Worker0: Terminate受信 → break → スレッド終了
    │  Worker1: Terminate受信 → break → スレッド終了
    │  Worker2: Terminate受信 → break → スレッド終了
    │
    └─ worker.thread.join() × N
           ↓
       全スレッドが終了するまでブロック
```

---

## パニックしたワーカーの再起動

実際のサーバーではジョブがパニックすることがあります。パニックが起きてもプールが縮小しないよう、ワーカーを再起動する設計を実装します。

```rust
use std::sync::atomic::{AtomicBool, Ordering};

pub struct RobustThreadPool {
    workers: Vec<RobustWorker>,
    sender: mpsc::Sender<Message>,
    receiver: Arc<Mutex<mpsc::Receiver<Message>>>,
    running: Arc<AtomicBool>,
}

struct RobustWorker {
    id: usize,
    thread: Option<thread::JoinHandle<()>>,
}

impl RobustWorker {
    fn spawn(
        id: usize,
        receiver: Arc<Mutex<mpsc::Receiver<Message>>>,
        running: Arc<AtomicBool>,
    ) -> RobustWorker {
        let thread = thread::Builder::new()
            .name(format!("robust-worker-{}", id))
            .spawn(move || {
                while running.load(Ordering::Relaxed) {
                    // パニックしても外側のスレッドが検知できるように
                    let result = std::panic::catch_unwind(std::panic::AssertUnwindSafe(|| {
                        let message = {
                            let lock = receiver.lock().expect("Mutex汚染");
                            lock.recv()
                        };
                        match message {
                            Ok(Message::NewJob(job)) => job(),
                            Ok(Message::Terminate) | Err(_) => {
                                running.store(false, Ordering::Relaxed);
                            }
                        }
                    }));

                    if let Err(e) = result {
                        eprintln!(
                            "Worker {} がパニックしました: {:?}。再試行します。",
                            id, e
                        );
                        // パニック後もループを継続 → 次のジョブを処理
                    }
                }
            })
            .expect("スレッド生成失敗");

        RobustWorker {
            id,
            thread: Some(thread),
        }
    }
}
```

---

## 動作確認サンプル

```rust
fn main() {
    // 4スレッドのプールを作成
    let pool = ThreadPool::new(4);

    for i in 0..8 {
        pool.execute(move || {
            println!(
                "ジョブ {} を {:?} で実行",
                i,
                std::thread::current().name().unwrap_or("unnamed")
            );
            // 重い処理のシミュレーション
            std::thread::sleep(std::time::Duration::from_millis(100));
        });
    }

    println!("全ジョブをキューに投入しました");
    // drop(pool) が呼ばれてグレースフルシャットダウン
}
```

実行例（出力は順不同）:

```
全ジョブをキューに投入しました
Worker 0 がジョブを実行します
ジョブ 0 を worker-0 で実行
Worker 1 がジョブを実行します
ジョブ 1 を worker-1 で実行
...
全Workerに終了メッセージを送信します...
Worker 0 が終了します
...
全Workerが正常に終了しました
```

---

## スレッド数の自動決定

```rust
use std::thread::available_parallelism;

fn optimal_thread_count() -> usize {
    match available_parallelism() {
        Ok(n) => {
            let count = n.get();
            println!("論理CPUコア数: {}", count);

            // I/Oバウンドなら2〜4倍、CPUバウンドならコア数そのまま
            // ここではCPUバウンドを想定
            count
        }
        Err(e) => {
            eprintln!("CPU数の取得に失敗 ({}), デフォルト4を使用", e);
            4
        }
    }
}

fn main() {
    let count = optimal_thread_count();
    let pool = ThreadPool::new(count);

    // I/OバウンドならCPUコア数の2倍程度が最適なことが多い
    let io_pool = ThreadPool::new(count * 2);

    pool.execute(|| println!("CPUバウンドタスク"));
    io_pool.execute(|| println!("I/Oバウンドタスク"));
}
```

---

## 完全動作するスレッドプール実装（rayon不使用）

以下は単一ファイルでコンパイル・実行できる完全実装です。

```rust
use std::sync::{Arc, Mutex};
use std::sync::mpsc;
use std::thread;

type Job = Box<dyn FnOnce() + Send + 'static>;

enum Message {
    NewJob(Job),
    Terminate,
}

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: Option<mpsc::Sender<Message>>,
}

struct Worker {
    id: usize,
    thread: Option<thread::JoinHandle<()>>,
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Message>>>) -> Worker {
        let thread = thread::spawn(move || loop {
            let message = receiver.lock().unwrap().recv();
            match message {
                Ok(Message::NewJob(job)) => job(),
                Ok(Message::Terminate) | Err(_) => break,
            }
        });
        Worker { id, thread: Some(thread) }
    }
}

impl ThreadPool {
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);
        let (sender, receiver) = mpsc::channel();
        let receiver = Arc::new(Mutex::new(receiver));
        let workers = (0..size)
            .map(|id| Worker::new(id, Arc::clone(&receiver)))
            .collect();
        ThreadPool { workers, sender: Some(sender) }
    }

    pub fn execute<F: FnOnce() + Send + 'static>(&self, f: F) {
        if let Some(ref sender) = self.sender {
            sender.send(Message::NewJob(Box::new(f))).unwrap();
        }
    }
}

impl Drop for ThreadPool {
    fn drop(&mut self) {
        // senderをdropしてチャンネルを閉じる
        drop(self.sender.take());
        for worker in &mut self.workers {
            if let Some(t) = worker.thread.take() {
                t.join().unwrap();
            }
        }
    }
}

fn main() {
    let pool = ThreadPool::new(
        thread::available_parallelism()
            .map(|n| n.get())
            .unwrap_or(4),
    );

    let results = Arc::new(Mutex::new(Vec::new()));

    for i in 0..12 {
        let results = Arc::clone(&results);
        pool.execute(move || {
            let result = i * i;
            results.lock().unwrap().push((i, result));
        });
    }

    // pool がドロップされるまで待つ (グレースフルシャットダウン)
    drop(pool);

    let mut results = Arc::try_unwrap(results).unwrap().into_inner().unwrap();
    results.sort_by_key(|&(i, _)| i);
    for (i, r) in results {
        println!("{} の二乗 = {}", i, r);
    }
}
```

---

## ベンチマーク比較

```rust
use std::time::Instant;

fn benchmark_thread_pool() {
    const JOBS: usize = 1000;
    const WORK_US: u64 = 100; // 各ジョブ100マイクロ秒の作業

    // 毎回スレッドを生成する場合
    let start = Instant::now();
    let mut handles = Vec::new();
    for _ in 0..JOBS {
        handles.push(thread::spawn(|| {
            std::thread::sleep(std::time::Duration::from_micros(WORK_US));
        }));
    }
    for h in handles { h.join().unwrap(); }
    let naive_duration = start.elapsed();

    // スレッドプールを使う場合
    let pool = ThreadPool::new(8);
    let start = Instant::now();
    let counter = Arc::new(Mutex::new(0usize));
    for _ in 0..JOBS {
        let counter = Arc::clone(&counter);
        pool.execute(move || {
            std::thread::sleep(std::time::Duration::from_micros(WORK_US));
            *counter.lock().unwrap() += 1;
        });
    }
    drop(pool);
    let pool_duration = start.elapsed();

    println!("毎回スレッド生成: {:?}", naive_duration);
    println!("スレッドプール:   {:?}", pool_duration);
    println!(
        "速度比: {:.1}x 高速",
        naive_duration.as_secs_f64() / pool_duration.as_secs_f64()
    );
}
```

---

## まとめ

| 設計要素 | 実装方法 | 役割 |
|---------|---------|------|
| ジョブキュー | `mpsc::channel` | ワーカー間でジョブを分配 |
| 共有レシーバ | `Arc<Mutex<Receiver>>` | 複数スレッドから安全にアクセス |
| シャットダウン | `Message::Terminate` + `Drop` | リソースのリーク防止 |
| スレッド数決定 | `available_parallelism()` | ハードウェアに最適化 |
| パニック耐性 | `catch_unwind` | ワーカーの自動回復 |

次節では、このスレッドプールの上で動くタスクスケジューラを実装し、「どのジョブをいつ実行するか」の戦略を学びます。
