# ロックフリーデータ構造

Mutex によるロックはシンプルで正確だが、高い競合状態ではスレッドのブロック・コンテキストスイッチ・優先度逆転などのオーバーヘッドが問題になる。ロックフリーデータ構造はアトミック操作のみでスレッド安全性を実現し、これらの問題を回避する。

---

## 1. ロックフリーとウェイトフリーの違い

| 特性 | 定義 | 特徴 |
|------|------|------|
| **ロックフリー** | システム全体として有限ステップで進捗が保証される | 一部のスレッドは飢餓になりうる |
| **ウェイトフリー** | 全スレッドが有限ステップで完了することが保証される | 最も強い保証・実装が複雑 |
| **オブストラクションフリー** | 他スレッドの干渉がなければ有限ステップで完了 | 最も弱い保証 |

```
強い保証
  ↑
Wait-Free (全スレッドが有限ステップで完了)
  |
Lock-Free (少なくとも1スレッドが有限ステップで完了)
  |
Obstruction-Free (単独実行なら有限ステップで完了)
  ↓
弱い保証
```

### ロックフリーが必要な場面

- リアルタイムシステム（ロック待ちによる遅延が許容できない）
- シグナルハンドラ内（mutex は async-signal-safe でない）
- 高スループットが要求されるデータパスのホットパス

---

## 2. `crossbeam` クレートのコレクション

```toml
# Cargo.toml
[dependencies]
crossbeam = "0.8"
crossbeam-channel = "0.5"
crossbeam-queue = "0.3"
```

### `SegQueue`: アンバウンド・ロックフリーキュー

セグメント化されたリンクリストで実装された MPMC（多対多）キュー。サイズ制限なし。

```rust
use crossbeam_queue::SegQueue;
use std::sync::Arc;
use std::thread;

fn seg_queue_example() {
    let queue: Arc<SegQueue<i32>> = Arc::new(SegQueue::new());
    let mut handles = vec![];

    // 複数の Producer スレッド
    for i in 0..4 {
        let q = Arc::clone(&queue);
        handles.push(thread::spawn(move || {
            for j in 0..25 {
                q.push(i * 25 + j);
            }
        }));
    }

    // 複数の Consumer スレッド
    let consumed = Arc::new(std::sync::atomic::AtomicUsize::new(0));
    for _ in 0..4 {
        let q = Arc::clone(&queue);
        let count = Arc::clone(&consumed);
        handles.push(thread::spawn(move || {
            loop {
                if q.pop().is_some() {
                    count.fetch_add(1, std::sync::atomic::Ordering::Relaxed);
                } else if count.load(std::sync::atomic::Ordering::Relaxed) >= 100 {
                    break;
                }
                std::hint::spin_loop();
            }
        }));
    }

    for h in handles { h.join().unwrap(); }
    println!("SegQueue: 全アイテムを処理しました");
}
```

### `ArrayQueue`: バウンド・ロックフリーキュー

固定サイズのリングバッファで実装。バックプレッシャーが必要な場合に使う。

```rust
use crossbeam_queue::ArrayQueue;
use std::sync::Arc;
use std::thread;
use std::time::Duration;

fn array_queue_example() {
    // 容量 64 のバウンドキュー
    let queue: Arc<ArrayQueue<String>> = Arc::new(ArrayQueue::new(64));

    let producer_q = Arc::clone(&queue);
    let producer = thread::spawn(move || {
        for i in 0..100 {
            let msg = format!("message-{}", i);
            loop {
                // キューが満杯なら push は Err を返す
                match producer_q.push(msg.clone()) {
                    Ok(_) => break,
                    Err(_) => {
                        // バックプレッシャー: 少し待って再試行
                        thread::sleep(Duration::from_micros(100));
                    }
                }
            }
        }
        println!("Producer: 全メッセージを送信完了");
    });

    let consumer_q = Arc::clone(&queue);
    let consumer = thread::spawn(move || {
        let mut received = 0;
        while received < 100 {
            if let Some(msg) = consumer_q.pop() {
                received += 1;
                if received % 10 == 0 {
                    println!("Consumer: {} 件受信 (最新: {})", received, msg);
                }
            } else {
                std::hint::spin_loop();
            }
        }
    });

    producer.join().unwrap();
    consumer.join().unwrap();
}
```

### `crossbeam-channel` の MPMC チャネル

```rust
use crossbeam_channel::{bounded, unbounded, Sender, Receiver};
use std::thread;
use std::time::Duration;

fn crossbeam_channel_example() {
    // バウンドチャネル (容量 32)
    let (tx, rx) = bounded::<String>(32);

    // 複数 Producer
    let mut producers = vec![];
    for i in 0..4 {
        let tx = tx.clone();
        producers.push(thread::spawn(move || {
            for j in 0..10 {
                tx.send(format!("producer-{}: item-{}", i, j)).unwrap();
            }
        }));
    }

    // 複数 Consumer
    let mut consumers = vec![];
    for id in 0..2 {
        let rx = rx.clone();
        consumers.push(thread::spawn(move || {
            for msg in rx {
                println!("consumer-{}: {}", id, msg);
            }
        }));
    }

    for p in producers { p.join().unwrap(); }
    drop(tx); // 全 Sender をドロップするとチャネルが閉じる
    for c in consumers { c.join().unwrap(); }
}
```

---

## 3. ABA 問題の解説と対策

### ABA 問題とは

Compare-and-Swap (CAS) を使ったロックフリーアルゴリズムで発生する古典的な問題。

```
スレッド1: ポインタが A を指していることを確認
スレッド1: (停止)
スレッド2: A を取り出す
スレッド2: 新しいノード B を挿入
スレッド2: A を再挿入（メモリアドレスが同じ）
スレッド1: 再開。ポインタはまだ A → CAS 成功！
→ しかし「A の意味」が変わっているためデータ構造が壊れる
```

### 対策1: タグ付きポインタ (Tagged Pointer)

ポインタの下位ビットに「バージョンカウンタ」を埋め込み、A と A' を区別する。

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

/// タグ付きポインタ: ポインタ + 64ビットのバージョンカウンタ
/// x86_64 では仮想アドレスの上位ビットが未使用なため利用可能
struct TaggedPtr {
    // 下位 48 ビット: ポインタ, 上位 16 ビット: タグ
    value: AtomicUsize,
}

impl TaggedPtr {
    const TAG_SHIFT: usize = 48;
    const PTR_MASK: usize = (1 << 48) - 1;

    fn new(ptr: *mut u8, tag: usize) -> Self {
        let packed = (ptr as usize & Self::PTR_MASK) | (tag << Self::TAG_SHIFT);
        TaggedPtr { value: AtomicUsize::new(packed) }
    }

    fn load(&self, ord: Ordering) -> (*mut u8, usize) {
        let v = self.value.load(ord);
        let ptr = (v & Self::PTR_MASK) as *mut u8;
        let tag = v >> Self::TAG_SHIFT;
        (ptr, tag)
    }

    fn compare_exchange(
        &self,
        expected_ptr: *mut u8,
        expected_tag: usize,
        new_ptr: *mut u8,
        new_tag: usize,
        success: Ordering,
        failure: Ordering,
    ) -> Result<(), ()> {
        let expected = (expected_ptr as usize & Self::PTR_MASK)
            | (expected_tag << Self::TAG_SHIFT);
        let new = (new_ptr as usize & Self::PTR_MASK)
            | (new_tag << Self::TAG_SHIFT);

        self.value
            .compare_exchange(expected, new, success, failure)
            .map(|_| ())
            .map_err(|_| ())
    }
}

fn tagged_ptr_aba_prevention() {
    let mut data = 42u8;
    let ptr = TaggedPtr::new(&mut data as *mut u8, 0);

    let (p, tag) = ptr.load(Ordering::Acquire);
    println!("初期: ptr={:p}, tag={}", p, tag);

    // 同じポインタでも tag が違えば CAS が失敗する
    let result = ptr.compare_exchange(
        p, tag + 1, // 古いタグで試みる → 失敗するはず
        p, tag + 1,
        Ordering::SeqCst,
        Ordering::Relaxed,
    );
    println!("タグ不一致 CAS: {}", if result.is_ok() { "成功" } else { "失敗（正しい）" });
}
```

### 対策2: `crossbeam-epoch` によるハザードポインタ代替

```toml
[dependencies]
crossbeam-epoch = "0.9"
```

```rust
use crossbeam_epoch::{self as epoch, Atomic, Owned, Shared};
use std::sync::atomic::Ordering;

struct Stack<T> {
    head: Atomic<Node<T>>,
}

struct Node<T> {
    data: T,
    next: Atomic<Node<T>>,
}

impl<T> Stack<T> {
    fn new() -> Self {
        Stack { head: Atomic::null() }
    }

    fn push(&self, data: T) {
        // Guard: このスコープ内でのメモリ回収を遅延させる
        let guard = &epoch::pin();
        let mut node = Owned::new(Node {
            data,
            next: Atomic::null(),
        });

        loop {
            let head = self.head.load(Ordering::Relaxed, guard);
            node.next.store(head, Ordering::Relaxed);

            match self.head.compare_exchange(
                head,
                node,
                Ordering::Release,
                Ordering::Relaxed,
                guard,
            ) {
                Ok(_) => break,
                Err(e) => node = e.new,
            }
        }
    }

    fn pop(&self) -> Option<T> {
        let guard = &epoch::pin();

        loop {
            let head = self.head.load(Ordering::Acquire, guard);

            match unsafe { head.as_ref() } {
                None => return None,
                Some(h) => {
                    let next = h.next.load(Ordering::Relaxed, guard);

                    if self.head
                        .compare_exchange(
                            head,
                            next,
                            Ordering::Release,
                            Ordering::Relaxed,
                            guard,
                        )
                        .is_ok()
                    {
                        unsafe {
                            // Epoch GC によって安全なタイミングでメモリを解放
                            guard.defer_destroy(head);
                            return Some(std::ptr::read(&h.data));
                        }
                    }
                }
            }
        }
    }
}
```

---

## 4. `Arc` のアトミック参照カウントのコストと `crossbeam-epoch`

### `Arc` のクローンコスト

```rust
use std::sync::Arc;
use std::sync::atomic::Ordering;

fn arc_clone_cost() {
    let data = Arc::new(vec![1, 2, 3, 4, 5]);

    // Arc::clone はアトミックなインクリメントを行う
    // 競合が激しい場合は SeqCst に近いコストがかかる
    let clones: Vec<Arc<Vec<i32>>> = (0..1000)
        .map(|_| Arc::clone(&data))
        .collect();

    println!("参照カウント: {}", Arc::strong_count(&data));
    drop(clones);
    println!("解放後の参照カウント: {}", Arc::strong_count(&data));
}
```

### `crossbeam-epoch` の利点

`Arc` は参照カウントのアトミック更新が常に必要だが、`crossbeam-epoch` は「エポック」の概念でメモリ回収タイミングを遅延させ、ホットパスのアトミック操作を削減する。

```rust
use crossbeam_epoch::{self as epoch, Atomic, Owned};
use std::sync::atomic::Ordering;

struct SharedData {
    value: Atomic<Vec<i32>>,
}

impl SharedData {
    fn new(v: Vec<i32>) -> Self {
        SharedData {
            value: Atomic::new(v),
        }
    }

    // 読み取り: Arc::clone なしで参照を取得
    fn read_snapshot(&self) -> Vec<i32> {
        let guard = &epoch::pin();
        let snapshot = self.value.load(Ordering::Acquire, guard);
        unsafe { snapshot.deref().clone() }
    }

    // 書き込み: アトミックに新しいデータに交換
    fn update(&self, new_data: Vec<i32>) {
        let guard = &epoch::pin();
        let old = self.value.swap(
            Owned::new(new_data),
            Ordering::AcqRel,
            guard,
        );
        // 他スレッドがアクセスしなくなった後に解放
        unsafe { guard.defer_destroy(old); }
    }
}
```

---

## 5. `dashmap` によるロックフリー HashMap

`dashmap` は内部をシャードに分割し、シャードごとの RwLock でロックの競合を最小化した並行 HashMap である。

```toml
[dependencies]
dashmap = "5"
```

```rust
use dashmap::DashMap;
use std::sync::Arc;
use std::thread;

fn dashmap_example() {
    let map: Arc<DashMap<String, u64>> = Arc::new(DashMap::new());
    let mut handles = vec![];

    // 複数スレッドから同時に書き込み
    for i in 0..8 {
        let map = Arc::clone(&map);
        handles.push(thread::spawn(move || {
            for j in 0..100 {
                let key = format!("key-{}", j % 20); // キーを意図的に重複させる
                map.entry(key)
                    .and_modify(|v| *v += 1)
                    .or_insert(1);
            }
        }));
    }

    for h in handles { h.join().unwrap(); }

    // 各キーの最終カウント
    let mut total = 0u64;
    for entry in map.iter() {
        total += *entry.value();
    }
    println!("DashMap: 総カウント = {} (期待値: 800)", total);
}

// DashMap を使ったキャッシュパターン
fn cache_example() {
    use std::time::{Duration, Instant};

    let cache: Arc<DashMap<u64, (String, Instant)>> = Arc::new(DashMap::new());
    let ttl = Duration::from_secs(60);

    let cache_clone = Arc::clone(&cache);

    // TTL 付きキャッシュへの読み書き
    let lookup = move |key: u64| -> Option<String> {
        if let Some(entry) = cache_clone.get(&key) {
            if entry.value().1.elapsed() < ttl {
                return Some(entry.value().0.clone());
            }
        }
        None
    };

    // キャッシュに値を挿入
    cache.insert(1, ("cached_value".to_string(), Instant::now()));

    match lookup(1) {
        Some(v) => println!("キャッシュヒット: {}", v),
        None => println!("キャッシュミス"),
    }
}
```

---

## 6. ベンチマーク: Mutex 版 vs ロックフリー版の比較

```toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "lock_free_bench"
harness = false
```

```rust
// benches/lock_free_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion, BenchmarkId};
use std::sync::{Arc, Mutex};
use crossbeam_queue::SegQueue;
use std::thread;

const NUM_THREADS: usize = 8;
const OPS_PER_THREAD: usize = 10_000;

fn bench_mutex_queue(c: &mut Criterion) {
    c.bench_function("mutex_queue", |b| {
        b.iter(|| {
            let queue: Arc<Mutex<Vec<i32>>> = Arc::new(Mutex::new(Vec::new()));
            let mut handles = vec![];

            for i in 0..NUM_THREADS {
                let q = Arc::clone(&queue);
                handles.push(thread::spawn(move || {
                    for j in 0..OPS_PER_THREAD {
                        let mut lock = q.lock().unwrap();
                        lock.push(black_box((i * OPS_PER_THREAD + j) as i32));
                    }
                }));
            }

            for h in handles { h.join().unwrap(); }
            let total = queue.lock().unwrap().len();
            assert_eq!(total, NUM_THREADS * OPS_PER_THREAD);
        });
    });
}

fn bench_lockfree_queue(c: &mut Criterion) {
    c.bench_function("lockfree_seg_queue", |b| {
        b.iter(|| {
            let queue: Arc<SegQueue<i32>> = Arc::new(SegQueue::new());
            let mut handles = vec![];

            for i in 0..NUM_THREADS {
                let q = Arc::clone(&queue);
                handles.push(thread::spawn(move || {
                    for j in 0..OPS_PER_THREAD {
                        q.push(black_box((i * OPS_PER_THREAD + j) as i32));
                    }
                }));
            }

            for h in handles { h.join().unwrap(); }
            assert_eq!(queue.len(), NUM_THREADS * OPS_PER_THREAD);
        });
    });
}

criterion_group!(benches, bench_mutex_queue, bench_lockfree_queue);
criterion_main!(benches);
```

### 典型的なベンチマーク結果（参考値）

```
スレッド数: 8, 操作数: 10,000/スレッド (合計 80,000 操作)

mutex_queue          time: [45.2 ms 46.1 ms 47.3 ms]
lockfree_seg_queue   time: [12.8 ms 13.1 ms 13.6 ms]

改善率: 約 3.5x
```

**注意**: ロックフリーが常に速いわけではない。

- 低競合: Mutex の方が速い場合がある（シンプルなアトミック操作のみ）
- 高競合・多スレッド: ロックフリーが有利
- メモリ使用量: ロックフリー構造はガベージコレクション相当の処理が必要

```rust
// 実際の性能比較を簡易実装で行う例
use std::time::Instant;
use std::sync::{Arc, Mutex};
use crossbeam_queue::ArrayQueue;

fn simple_benchmark() {
    const THREADS: usize = 8;
    const OPS: usize = 100_000;

    // Mutex 版
    let mutex_queue = Arc::new(Mutex::new(std::collections::VecDeque::new()));
    let start = Instant::now();
    {
        let mut handles = vec![];
        for _ in 0..THREADS {
            let q = Arc::clone(&mutex_queue);
            handles.push(std::thread::spawn(move || {
                for i in 0..OPS {
                    let mut lock = q.lock().unwrap();
                    lock.push_back(i);
                    if lock.len() > 1000 {
                        lock.pop_front();
                    }
                }
            }));
        }
        for h in handles { h.join().unwrap(); }
    }
    let mutex_time = start.elapsed();

    // ロックフリー版
    let lf_queue = Arc::new(ArrayQueue::new(2000));
    let start = Instant::now();
    {
        let mut handles = vec![];
        for _ in 0..THREADS {
            let q = Arc::clone(&lf_queue);
            handles.push(std::thread::spawn(move || {
                for i in 0..OPS {
                    let _ = q.push(i);
                    q.pop();
                }
            }));
        }
        for h in handles { h.join().unwrap(); }
    }
    let lf_time = start.elapsed();

    println!("Mutex:      {:?}", mutex_time);
    println!("Lock-free:  {:?}", lf_time);
    println!("改善率: {:.2}x", mutex_time.as_secs_f64() / lf_time.as_secs_f64());
}
```

---

## まとめ

| データ構造 | クレート | 特徴 | 使いどころ |
|-----------|---------|------|-----------|
| `SegQueue` | crossbeam | アンバウンド MPMC キュー | 汎用タスクキュー |
| `ArrayQueue` | crossbeam | バウンド MPMC キュー | バックプレッシャーが必要な場合 |
| `DashMap` | dashmap | シャード化並行 HashMap | 高競合な KV ストア |
| Epoch-based | crossbeam-epoch | メモリ安全なロックフリー構造の基盤 | カスタム構造の実装 |

ロックフリーはツールであり、目的ではない。常にプロファイリングで効果を測定してから採用すること。
