# 01. スレッド基礎

Rustの標準ライブラリが提供するスレッド機能を学びます。OSスレッドの生成から共有状態の安全な管理まで、`Send` / `Sync` トレイトによってコンパイル時に安全性が保証される仕組みを理解します。

## `std::thread::spawn` とクロージャ

スレッドは `std::thread::spawn` で生成します。クロージャが新しいスレッドで実行されます。

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        for i in 0..5 {
            println!("子スレッド: {}", i);
        }
    });

    for i in 0..5 {
        println!("メインスレッド: {}", i);
    }

    // スレッドの完了を待つ
    handle.join().unwrap();
}
```

**出力例** (順序は実行毎に変わります):

```
メインスレッド: 0
子スレッド: 0
メインスレッド: 1
子スレッド: 1
...
```

### `move` クロージャによる所有権の移動

スレッドのクロージャがスタック変数を参照する場合、変数の寿命がスレッドより短い可能性があります。`move` を付けて所有権をクロージャに移動します。

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3, 4, 5];

    // `move` なしだとコンパイルエラー:
    // error: closure may outlive the current function
    let handle = thread::spawn(move || {
        println!("データの合計: {}", data.iter().sum::<i32>());
        // ここで `data` の所有権が使われる
    });

    // この時点で `data` はスレッドに移動済みなので使えない
    // println!("{:?}", data); // コンパイルエラー

    handle.join().unwrap();
}
```

## `JoinHandle` によるスレッド管理

`spawn` は `JoinHandle<T>` を返します。`join()` を呼ぶことでスレッドの完了を待ち、戻り値を受け取れます。

```rust
use std::thread;

fn compute(n: u64) -> u64 {
    (1..=n).sum()
}

fn main() {
    let handles: Vec<_> = (1..=4)
        .map(|i| {
            thread::spawn(move || {
                let result = compute(i * 1_000_000);
                println!("スレッド {} 完了: {}", i, result);
                result
            })
        })
        .collect();

    let total: u64 = handles
        .into_iter()
        .map(|h| h.join().unwrap())
        .sum();

    println!("全スレッドの合計: {}", total);
}
```

### パニックの伝播

子スレッドがパニックした場合、`join()` は `Err` を返します。

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        panic!("意図的なパニック");
    });

    match handle.join() {
        Ok(val) => println!("成功: {:?}", val),
        Err(e)  => println!("スレッドがパニック: {:?}", e),
    }
    // → スレッドがパニック: Any { .. }
}
```

## スレッドビルダー (`Builder`)

`thread::Builder` を使うと名前やスタックサイズを指定できます。

```rust
use std::thread;

fn main() {
    let builder = thread::Builder::new()
        .name("worker-1".to_string())
        .stack_size(4 * 1024 * 1024); // 4MB

    let handle = builder.spawn(|| {
        let name = thread::current().name().unwrap_or("unnamed").to_string();
        println!("スレッド名: {}", name);

        // 再帰が深いアルゴリズムなどで大きなスタックが必要な場合
        recursive_function(10_000);
    }).unwrap();

    handle.join().unwrap();
}

fn recursive_function(depth: usize) {
    if depth == 0 { return; }
    recursive_function(depth - 1);
}
```

## 共有状態: `Arc<Mutex<T>>`

複数スレッドで同じデータを読み書きするには、`Arc`（参照カウントポインタ）と `Mutex`（相互排他ロック）を組み合わせます。

```
Arc<Mutex<T>>
│
├── clone() → スレッド1 → lock() → guard → データ操作
├── clone() → スレッド2 → lock() → 待機... → guard → データ操作
└── clone() → スレッド3 → lock() → 待機... → guard → データ操作
```

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0u64));
    let mut handles = vec![];

    for _ in 0..8 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            for _ in 0..1_000 {
                let mut num = counter.lock().unwrap();
                *num += 1;
                // `num` がスコープを抜けるとロックが自動解放される
            }
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("最終カウント: {}", *counter.lock().unwrap());
    // → 最終カウント: 8000
}
```

### デッドロックの例と回避

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn deadlock_example() {
    let lock_a = Arc::new(Mutex::new(0));
    let lock_b = Arc::new(Mutex::new(0));

    let la = Arc::clone(&lock_a);
    let lb = Arc::clone(&lock_b);

    // スレッド1: A を取得してから B を取得しようとする
    let t1 = thread::spawn(move || {
        let _a = la.lock().unwrap();
        thread::sleep(std::time::Duration::from_millis(10));
        let _b = lb.lock().unwrap(); // スレッド2がBを持っていると待機
    });

    // スレッド2: B を取得してから A を取得しようとする → デッドロック!
    let la2 = Arc::clone(&lock_a);
    let lb2 = Arc::clone(&lock_b);
    let t2 = thread::spawn(move || {
        let _b = lb2.lock().unwrap();
        thread::sleep(std::time::Duration::from_millis(10));
        let _a = la2.lock().unwrap(); // スレッド1がAを持っていると待機
    });

    // 解決策: 常に同じ順序でロックを取得する
    // lock_a → lock_b の順番を両スレッドで統一する
}
```

## 共有状態: `Arc<RwLock<T>>`

読み取りが多く書き込みが少ない場合は `RwLock` が効率的です。複数の読み取りを同時に許可し、書き込み時のみ排他します。

```
RwLock<T>
│
├── read()  → 複数スレッドが同時に読み取り可能
├── read()  ┘
└── write() → 排他ロック (読み取りも書き込みも他はブロック)
```

```rust
use std::sync::{Arc, RwLock};
use std::thread;
use std::time::Duration;

fn main() {
    let data = Arc::new(RwLock::new(vec![1, 2, 3, 4, 5]));
    let mut handles = vec![];

    // 複数の読み取りスレッド
    for i in 0..4 {
        let data = Arc::clone(&data);
        handles.push(thread::spawn(move || {
            let guard = data.read().unwrap();
            println!("読み取りスレッド {}: {:?}", i, *guard);
            // 複数スレッドが同時にここを実行できる
        }));
    }

    // 1つの書き込みスレッド
    {
        let data = Arc::clone(&data);
        handles.push(thread::spawn(move || {
            thread::sleep(Duration::from_millis(5));
            let mut guard = data.write().unwrap();
            guard.push(6);
            println!("書き込み完了: {:?}", *guard);
        }));
    }

    for h in handles {
        h.join().unwrap();
    }
}
```

### `Mutex` vs `RwLock` の使い分け

| 条件 | 選択 |
|---|---|
| 読み取りと書き込みが同程度 | `Mutex` |
| 読み取りが圧倒的に多い | `RwLock` |
| データが小さい (`Copy` 型) | `Atomic*` を検討 |
| Writer が増えると競合が増える | `Mutex` の方がシンプルで速い場合も |

## スレッドローカルストレージ

`thread_local!` マクロでスレッドごとに独立したデータを持てます。

```rust
use std::cell::RefCell;
use std::thread;

thread_local! {
    static BUFFER: RefCell<Vec<String>> = RefCell::new(Vec::new());
}

fn log(msg: &str) {
    BUFFER.with(|buf| {
        buf.borrow_mut().push(msg.to_string());
    });
}

fn flush() -> Vec<String> {
    BUFFER.with(|buf| buf.borrow().clone())
}

fn main() {
    let handles: Vec<_> = (0..3).map(|i| {
        thread::spawn(move || {
            log(&format!("スレッド{} メッセージ1", i));
            log(&format!("スレッド{} メッセージ2", i));
            flush()
        })
    }).collect();

    for h in handles {
        println!("{:?}", h.join().unwrap());
    }
    // 各スレッドが独立したバッファを持つ
}
```

## スレッドプールの考え方

毎回スレッドを生成するのはコストが高い（OSスレッド1本あたり数MB のスタック確保）。スレッドプールはあらかじめスレッドを生成しておき、タスクをキューに投入します。

```
タスクキュー
[Task1, Task2, Task3, Task4, Task5]
      ↓          ↓
  Worker1     Worker2    Worker3    Worker4
  (実行中)    (実行中)   (待機中)   (待機中)
```

### 簡単なスレッドプール実装

```rust
use std::sync::{Arc, Mutex};
use std::thread;

type Job = Box<dyn FnOnce() + Send + 'static>;

struct ThreadPool {
    workers: Vec<thread::JoinHandle<()>>,
    sender: std::sync::mpsc::Sender<Option<Job>>,
}

impl ThreadPool {
    fn new(size: usize) -> Self {
        let (sender, receiver) = std::sync::mpsc::channel::<Option<Job>>();
        let receiver = Arc::new(Mutex::new(receiver));
        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            let rx = Arc::clone(&receiver);
            let handle = thread::Builder::new()
                .name(format!("worker-{}", id))
                .spawn(move || loop {
                    let job = rx.lock().unwrap().recv().unwrap();
                    match job {
                        Some(job) => job(),
                        None => break, // シャットダウンシグナル
                    }
                })
                .unwrap();
            workers.push(handle);
        }

        ThreadPool { workers, sender }
    }

    fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        self.sender.send(Some(Box::new(f))).unwrap();
    }

    fn shutdown(self) {
        // 各ワーカーに終了シグナルを送る
        for _ in &self.workers {
            self.sender.send(None).unwrap();
        }
        for worker in self.workers {
            worker.join().unwrap();
        }
    }
}

fn main() {
    let pool = ThreadPool::new(4);

    for i in 0..10 {
        pool.execute(move || {
            println!("タスク {} をスレッド {:?} で実行", i, thread::current().name());
        });
    }

    pool.shutdown();
}
```

## `std::thread` と `rayon` の使い分け

| 用途 | 推奨 |
|---|---|
| タスク並列（各スレッドが異なる処理） | `std::thread` |
| データ並列（コレクションの並列変換） | `rayon` |
| スレッドのライフタイム管理が必要 | `std::thread` |
| 簡潔にイテレータを並列化したい | `rayon` |
| スレッド数の細かい制御 | `std::thread` + 自前プール |
| ワークスティーリングが欲しい | `rayon` |

```rust
// std::thread: タスク並列の例
fn task_parallel() {
    let h1 = thread::spawn(|| fetch_from_db());
    let h2 = thread::spawn(|| fetch_from_api());
    let (db_result, api_result) = (h1.join().unwrap(), h2.join().unwrap());
}

// rayon: データ並列の例
fn data_parallel(data: &[f64]) -> f64 {
    use rayon::prelude::*;
    data.par_iter().map(|x| x * x).sum()
}
```

## パフォーマンス比較

```rust
use std::time::Instant;
use std::thread;

const N: usize = 100_000_000;

fn single_thread_sum() -> u64 {
    (0..N as u64).sum()
}

fn multi_thread_sum(num_threads: usize) -> u64 {
    let chunk = N / num_threads;
    let handles: Vec<_> = (0..num_threads)
        .map(|i| {
            let start = (i * chunk) as u64;
            let end = if i == num_threads - 1 {
                N as u64
            } else {
                start + chunk as u64
            };
            thread::spawn(move || (start..end).sum::<u64>())
        })
        .collect();

    handles.into_iter().map(|h| h.join().unwrap()).sum()
}

fn main() {
    let t = Instant::now();
    let s = single_thread_sum();
    println!("シングルスレッド: {} ({:.2}ms)", s, t.elapsed().as_secs_f64() * 1000.0);

    for &n in &[2, 4, 8] {
        let t = Instant::now();
        let s = multi_thread_sum(n);
        println!("{}スレッド: {} ({:.2}ms)", n, s, t.elapsed().as_secs_f64() * 1000.0);
    }
}
```

**実行結果の例** (8コアマシン):

```
シングルスレッド: 4999999950000000 (312.45ms)
2スレッド:        4999999950000000 (158.12ms)
4スレッド:        4999999950000000 (81.34ms)
8スレッド:        4999999950000000 (43.22ms)
```

> **注意**: この例は CPU バウンドな単純な計算です。実際のアプリケーションでは、メモリ帯域幅やキャッシュの競合によってスケーリングが頭打ちになります。

## まとめ

- `thread::spawn` + `move` クロージャでスレッドに所有権を移動する
- `Arc<Mutex<T>>` で複数スレッドが安全に共有状態を変更できる
- 読み取り多用なら `Arc<RwLock<T>>` で並列読み取りを活用する
- デッドロックを避けるにはロックの取得順序を統一する
- スレッドプールでスレッド生成コストを削減する
- データ並列には次節の `rayon` を使うとコードがシンプルになる
