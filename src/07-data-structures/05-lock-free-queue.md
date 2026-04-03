# ロックフリーキューとスタック

ロックフリー（Lock-Free）なデータ構造は、`Mutex` を使わずにスレッド安全性を保証します。
少なくとも 1 つのスレッドが常に進行できることが保証されており、優先度逆転やデッドロックが発生しません。

---

## Michael-Scott キューのアルゴリズム解説

Michael-Scott キュー（MS Queue）は、1996 年に提案されたロックフリー FIFO キューです。
単方向リンクリストと CAS（Compare-And-Swap）操作で実装されます。

```
初期状態（ダミーノードあり）:

  head → [dummy] → NULL
  tail → [dummy]

enqueue(A):

  head → [dummy] → [A] → NULL
  tail →                 [A]     ← tail を CAS で更新

enqueue(B):

  head → [dummy] → [A] → [B] → NULL
  tail →                       [B]

dequeue() → A:

  head → [A] → [B] → NULL        ← head を CAS で [dummy] から [A] に更新
  tail →       [B]               （[dummy] が解放され [A] が新ダミーになる）
  戻り値: A の値
```

### アルゴリズムの要点

```
enqueue の手順:
  1. 新ノード node を確保
  2. tail.next を CAS(NULL → node)
     - 成功: tail を CAS(tail → node) で更新して完了
     - 失敗: 別スレッドが先に書いたので tail を advance してリトライ

dequeue の手順:
  1. head.next を読む（これが実データを持つ最初のノード）
  2. head を CAS(head → head.next) で更新
     - 成功: head.next の値を返す
     - 失敗: リトライ
```

---

## `crossbeam-queue` の `SegQueue` と `ArrayQueue`

```toml
# Cargo.toml
[dependencies]
crossbeam-queue = "0.3"
```

### `SegQueue` — 無制限キュー

```rust
use crossbeam_queue::SegQueue;
use std::sync::Arc;
use std::thread;

fn main() {
    // 容量無制限のロックフリーキュー
    let queue: Arc<SegQueue<u64>> = Arc::new(SegQueue::new());

    let producers: Vec<_> = (0..4).map(|producer_id| {
        let q = Arc::clone(&queue);
        thread::spawn(move || {
            for i in 0..1000u64 {
                q.push(producer_id * 1000 + i);
            }
        })
    }).collect();

    let consumer = {
        let q = Arc::clone(&queue);
        thread::spawn(move || {
            let mut sum = 0u64;
            let mut count = 0;
            while count < 4000 {
                if let Some(v) = q.pop() {
                    sum += v;
                    count += 1;
                } else {
                    std::hint::spin_loop();
                }
            }
            sum
        })
    };

    for p in producers { p.join().unwrap(); }
    println!("合計: {}", consumer.join().unwrap());
}
```

### `ArrayQueue` — 有界キュー（キャパシティ制限あり）

```rust
use crossbeam_queue::ArrayQueue;
use std::sync::Arc;
use std::thread;

fn main() {
    // 容量 256 の有界ロックフリーキュー
    let queue: Arc<ArrayQueue<String>> = Arc::new(ArrayQueue::new(256));

    let q_send = Arc::clone(&queue);
    let sender = thread::spawn(move || {
        for i in 0..500 {
            let msg = format!("msg_{}", i);
            loop {
                match q_send.push(msg.clone()) {
                    Ok(_) => break,
                    Err(_) => std::hint::spin_loop(), // 満杯なら待機
                }
            }
        }
    });

    let q_recv = Arc::clone(&queue);
    let receiver = thread::spawn(move || {
        let mut received = 0;
        while received < 500 {
            if let Some(msg) = q_recv.pop() {
                println!("受信: {}", msg);
                received += 1;
            } else {
                std::hint::spin_loop();
            }
        }
    });

    sender.join().unwrap();
    receiver.join().unwrap();
}
```

### `SegQueue` vs `ArrayQueue` の使い分け

```
SegQueue（セグメントリスト構造）:
  ✓ 容量制限なし
  ✓ 動的にメモリを確保
  ✗ ヒープ確保のオーバーヘッドあり
  ✗ キャッシュ非効率（ノードが散在）
  用途: バックプレッシャーが不要な場合

ArrayQueue（リングバッファ構造）:
  ✓ 固定メモリ、キャッシュ効率が良い
  ✓ 容量超過時に Err を返してバックプレッシャーを実現
  ✗ 容量を事前に決める必要がある
  用途: プロデューサー・コンシューマーの速度が釣り合う場合
```

---

## ABA プロブレムの具体例と対策

ABA 問題は CAS を使ったロックフリー実装で発生する古典的なバグです。

```
ABA 問題の例（ロックフリースタック）:

初期状態:
  top → [A] → [B] → NULL

スレッド 1: pop() を試みる
  - top の値を読む: A
  - 一時停止（プリエンプション）

スレッド 2: pop() で A を取り出す → top = B
スレッド 2: pop() で B を取り出す → top = NULL
スレッド 2: push(A) で A を戻す  → top = A（同じアドレス！）

スレッド 1: CAS(top: A → B) を実行
  - top が A なので CAS 成功！ → top = B
  - しかし B はすでに解放済み → 未定義動作 💥

問題: アドレスが同じでも「中身が変わっている」かもしれない
```

### 対策 1: タグ付きポインタ（Tagged Pointer）

ポインタの下位ビット（アライメントで使われない）にバージョンカウンタを埋め込みます。

```rust
use std::sync::atomic::{AtomicU64, Ordering};

/// タグ付きポインタ: 上位48ビットがアドレス、下位16ビットがバージョン
struct TaggedPtr(AtomicU64);

impl TaggedPtr {
    fn new(ptr: *mut u8, tag: u16) -> Self {
        let val = (ptr as u64 & 0xFFFF_FFFF_FFFF) | ((tag as u64) << 48);
        Self(AtomicU64::new(val))
    }

    fn load(&self, ord: Ordering) -> (*mut u8, u16) {
        let val = self.0.load(ord);
        let ptr = (val & 0xFFFF_FFFF_FFFF) as *mut u8;
        let tag = (val >> 48) as u16;
        (ptr, tag)
    }

    fn compare_exchange(
        &self,
        old_ptr: *mut u8, old_tag: u16,
        new_ptr: *mut u8, new_tag: u16,
    ) -> bool {
        let old = (old_ptr as u64 & 0xFFFF_FFFF_FFFF) | ((old_tag as u64) << 48);
        let new = (new_ptr as u64 & 0xFFFF_FFFF_FFFF) | ((new_tag as u64) << 48);
        self.0.compare_exchange(old, new, Ordering::AcqRel, Ordering::Acquire).is_ok()
    }
}
```

### 対策 2: エポックベース GC（`crossbeam-epoch`）

`crossbeam` ライブラリが採用している手法です。スレッドが「エポック」に参加している間はメモリが解放されません。

```toml
[dependencies]
crossbeam-epoch = "0.9"
```

```rust
use crossbeam_epoch::{self as epoch, Atomic, Owned, Shared};
use std::sync::atomic::Ordering;

struct Node<T> {
    data: T,
    next: Atomic<Node<T>>,
}

// crossbeam-epoch が提供するハザードポインタ相当の保護機構
fn demo_epoch() {
    let guard = epoch::pin(); // 現在のエポックに参加

    // guard が有効な間はポインタが解放されない
    // guard が drop されるとエポックが進み、解放が可能になる
    drop(guard);
}
```

### 対策 3: ハザードポインタ

```
ハザードポインタの仕組み:

各スレッドは「今アクセス中のポインタ」をハザードポインタとして登録
  Thread 1: hazard[0] = ptr_A
  Thread 2: hazard[1] = ptr_B

解放時: 全スレッドのハザードポインタをチェック
  if ptr in any hazard pointer: 解放を延期
  else: 即座に解放

→ ABA 問題は起きない（参照中に解放されないため）
```

---

## ロックフリースタック（Treiber Stack）の実装

```rust
use std::sync::atomic::{AtomicPtr, Ordering};
use std::ptr;

struct Node<T> {
    data: T,
    next: *mut Node<T>,
}

/// Treiber スタック（ロックフリー）
pub struct TreiberStack<T> {
    top: AtomicPtr<Node<T>>,
}

unsafe impl<T: Send> Send for TreiberStack<T> {}
unsafe impl<T: Send> Sync for TreiberStack<T> {}

impl<T> TreiberStack<T> {
    pub fn new() -> Self {
        Self { top: AtomicPtr::new(ptr::null_mut()) }
    }

    pub fn push(&self, val: T) {
        let node = Box::into_raw(Box::new(Node {
            data: val,
            next: ptr::null_mut(),
        }));
        loop {
            let top = self.top.load(Ordering::Acquire);
            // SAFETY: node は直前に確保した有効なポインタ
            unsafe { (*node).next = top; }
            // CAS: top が変わっていなければ node を新しい先頭に
            match self.top.compare_exchange(
                top, node,
                Ordering::Release,
                Ordering::Relaxed,
            ) {
                Ok(_) => return,
                Err(_) => {} // 競合: リトライ
            }
        }
    }

    pub fn pop(&self) -> Option<T> {
        loop {
            let top = self.top.load(Ordering::Acquire);
            if top.is_null() {
                return None;
            }
            // SAFETY: top は push() で確保した有効なポインタ
            let next = unsafe { (*top).next };
            match self.top.compare_exchange(
                top, next,
                Ordering::Release,
                Ordering::Relaxed,
            ) {
                Ok(_) => {
                    // SAFETY: CAS 成功したのでこのスレッドが所有権を持つ
                    let node = unsafe { Box::from_raw(top) };
                    return Some(node.data);
                    // Box::drop で node のメモリが解放される
                    // ※ ABA 問題に注意: 本番実装では epoch GC を使うこと
                }
                Err(_) => {} // 競合: リトライ
            }
        }
    }
}

impl<T> Drop for TreiberStack<T> {
    fn drop(&mut self) {
        while self.pop().is_some() {}
    }
}

fn main() {
    use std::sync::Arc;
    use std::thread;

    let stack = Arc::new(TreiberStack::new());

    let handles: Vec<_> = (0..4).map(|i| {
        let s = Arc::clone(&stack);
        thread::spawn(move || {
            for j in 0..100 {
                s.push(i * 100 + j);
            }
        })
    }).collect();

    for h in handles { h.join().unwrap(); }

    let mut count = 0;
    while stack.pop().is_some() {
        count += 1;
    }
    println!("合計 {} 個の要素をポップ", count); // 400
}
```

---

## ベンチマーク: `Mutex<VecDeque>` vs `SegQueue` vs `ArrayQueue`

```toml
[dependencies]
crossbeam-queue = "0.3"

[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }
```

```rust
// benches/queue_bench.rs
use criterion::{criterion_group, criterion_main, Criterion, BenchmarkId};
use crossbeam_queue::{SegQueue, ArrayQueue};
use std::sync::{Arc, Mutex};
use std::collections::VecDeque;
use std::thread;

const N: usize = 100_000;
const THREADS: usize = 4;

fn bench_mutex_vecdeque(c: &mut Criterion) {
    c.bench_function("Mutex<VecDeque>", |b| {
        b.iter(|| {
            let q = Arc::new(Mutex::new(VecDeque::new()));
            let handles: Vec<_> = (0..THREADS).map(|_| {
                let q = Arc::clone(&q);
                thread::spawn(move || {
                    for i in 0..N / THREADS {
                        q.lock().unwrap().push_back(i);
                    }
                })
            }).collect();
            for h in handles { h.join().unwrap(); }
            let mut q = q.lock().unwrap();
            while q.pop_front().is_some() {}
        });
    });
}

fn bench_seg_queue(c: &mut Criterion) {
    c.bench_function("SegQueue", |b| {
        b.iter(|| {
            let q = Arc::new(SegQueue::new());
            let handles: Vec<_> = (0..THREADS).map(|_| {
                let q = Arc::clone(&q);
                thread::spawn(move || {
                    for i in 0..N / THREADS {
                        q.push(i);
                    }
                })
            }).collect();
            for h in handles { h.join().unwrap(); }
            while q.pop().is_some() {}
        });
    });
}

fn bench_array_queue(c: &mut Criterion) {
    c.bench_function("ArrayQueue", |b| {
        b.iter(|| {
            let q = Arc::new(ArrayQueue::new(N * 2));
            let handles: Vec<_> = (0..THREADS).map(|_| {
                let q = Arc::clone(&q);
                thread::spawn(move || {
                    for i in 0..N / THREADS {
                        while q.push(i).is_err() {
                            std::hint::spin_loop();
                        }
                    }
                })
            }).collect();
            for h in handles { h.join().unwrap(); }
            while q.pop().is_some() {}
        });
    });
}

criterion_group!(benches, bench_mutex_vecdeque, bench_seg_queue, bench_array_queue);
criterion_main!(benches);
```

```bash
cargo bench
```

```
典型的な結果（4 スレッド, 100,000 ops, 参考値）:

Mutex<VecDeque>   time: [28.5 ms  29.1 ms  29.8 ms]
SegQueue          time: [12.3 ms  12.7 ms  13.1 ms]  ← 約 2.3x 高速
ArrayQueue        time: [ 8.1 ms   8.4 ms   8.7 ms]  ← 約 3.5x 高速

競合が激しいほどロックフリーの優位性が増す
```

---

## 各手法の選択基準

```
シングルプロデューサー・シングルコンシューマー:
  → ringbuf::HeapRb  （最速、アトミック操作のみ）

複数プロデューサー・複数コンシューマー（容量制限不要）:
  → crossbeam_queue::SegQueue

複数プロデューサー・複数コンシューマー（バックプレッシャーが必要）:
  → crossbeam_queue::ArrayQueue

シンプルさ優先（性能要件が緩い）:
  → std::sync::Mutex<VecDeque<T>>
```

---

## まとめ

| 実装 | スレッドモデル | ABA 対策 | スループット |
|---|---|---|---|
| Mutex\<VecDeque\> | 任意 | 不要 | 低 |
| Treiber Stack | 任意 | epoch GC 推奨 | 中 |
| MS Queue | 任意 | epoch GC | 高 |
| SegQueue | 任意 | epoch GC (内蔵) | 高 |
| ArrayQueue | 任意 | タグ付きポインタ | 最高 |
| SpscQueue | SPSC のみ | 不要 | 最高 |
