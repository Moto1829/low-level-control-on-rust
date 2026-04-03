# リングバッファ（循環キュー）

リングバッファ（Ring Buffer）は、固定サイズのメモリ領域を循環的に再利用するデータ構造です。
FIFO（先入れ先出し）キューを実装する際に、メモリの動的確保を避けながら高いスループットを実現できます。
ネットワークパケットバッファ、オーディオストリーム処理、ロックフリーな SPSC キューなどで広く使われています。

---

## リングバッファの仕組み

固定長の配列に `head`（読み出し位置）と `tail`（書き込み位置）の 2 つのインデックスを持ちます。
書き込みは `tail` が進み、読み出しは `head` が進みます。インデックスが配列末尾を超えると先頭に折り返します。

```
容量 8 のリングバッファ（4要素が入っている状態）

 インデックス:  0    1    2    3    4    5    6    7
               +----+----+----+----+----+----+----+----+
               |    |    | 10 | 20 | 30 | 40 |    |    |
               +----+----+----+----+----+----+----+----+
                          ^                   ^
                        head                tail
                      (読み出し位置)      (書き込み位置)

enqueue(50) → tail が 6 に移動
dequeue()   → head が 3 に移動、値 10 を返す

折り返しの例（tail が末尾を超えた場合）:

 インデックス:  0    1    2    3    4    5    6    7
               +----+----+----+----+----+----+----+----+
               | 70 | 80 | 10 | 20 | 30 | 40 | 50 | 60 |
               +----+----+----+----+----+----+----+----+
                          ^
                     head & tail (満杯)

 tail が 7 の次 → 0 に折り返す（循環）
```

---

## 固定長リングバッファを `unsafe` で実装

初期化されていないメモリを扱うために `MaybeUninit<T>` を使います。
これにより不要なゼロ初期化を避け、最大のパフォーマンスを得られます。

```rust
use std::mem::MaybeUninit;

/// 容量が 2 のべき乗である固定長リングバッファ
pub struct RingBuffer<T, const N: usize> {
    buf: [MaybeUninit<T>; N],
    head: usize, // 読み出し位置
    tail: usize, // 書き込み位置
    len: usize,  // 現在の要素数
}

impl<T, const N: usize> RingBuffer<T, N> {
    /// N は 2 のべき乗でなければならない
    pub fn new() -> Self {
        assert!(N.is_power_of_two(), "容量は 2 のべき乗でなければなりません");
        // SAFETY: MaybeUninit の配列を未初期化状態で生成する
        let buf = unsafe { MaybeUninit::uninit().assume_init() };
        Self { buf, head: 0, tail: 0, len: 0 }
    }

    /// バッファが満杯かどうか
    #[inline]
    pub fn is_full(&self) -> bool {
        self.len == N
    }

    /// バッファが空かどうか
    #[inline]
    pub fn is_empty(&self) -> bool {
        self.len == 0
    }

    /// 要素をエンキュー（末尾に追加）
    /// 満杯の場合は Err を返す
    pub fn push(&mut self, val: T) -> Result<(), T> {
        if self.is_full() {
            return Err(val);
        }
        // マスク演算でインデックスを折り返す（後述）
        let idx = self.tail & (N - 1);
        // SAFETY: idx は 0..N の範囲、かつこのスロットは未使用
        unsafe {
            self.buf[idx].write(val);
        }
        self.tail = self.tail.wrapping_add(1);
        self.len += 1;
        Ok(())
    }

    /// 要素をデキュー（先頭から取り出し）
    /// 空の場合は None を返す
    pub fn pop(&mut self) -> Option<T> {
        if self.is_empty() {
            return None;
        }
        let idx = self.head & (N - 1);
        // SAFETY: idx は有効な初期化済みスロット
        let val = unsafe { self.buf[idx].assume_init_read() };
        self.head = self.head.wrapping_add(1);
        self.len -= 1;
        Some(val)
    }

    /// 先頭要素への参照を返す（取り出しは行わない）
    pub fn peek(&self) -> Option<&T> {
        if self.is_empty() {
            return None;
        }
        let idx = self.head & (N - 1);
        // SAFETY: idx は有効な初期化済みスロット
        unsafe { Some(self.buf[idx].assume_init_ref()) }
    }
}

/// Drop 時に残っている要素を正しく解放する
impl<T, const N: usize> Drop for RingBuffer<T, N> {
    fn drop(&mut self) {
        while self.pop().is_some() {}
    }
}

fn main() {
    let mut rb: RingBuffer<i32, 8> = RingBuffer::new();

    for i in 0..8 {
        rb.push(i * 10).unwrap();
    }
    assert!(rb.push(999).is_err()); // 満杯

    println!("peek: {:?}", rb.peek()); // Some(0)

    while let Some(val) = rb.pop() {
        print!("{} ", val); // 0 10 20 30 40 50 60 70
    }
    println!();
}
```

---

## インデックス計算: マスク演算による最適化

容量 `N` を 2 のべき乗に限定することで、モジュロ演算（`%`）をビット AND（`&`）に置き換えられます。

```
通常のモジュロ:     idx = (idx + 1) % N      → 除算が必要（低速）
マスク演算:         idx = (idx + 1) & (N - 1) → ビット演算のみ（高速）

N = 8 の場合:
  N - 1 = 7 = 0b0000_0111

  idx = 6: 6 & 7 = 6  ✓
  idx = 7: 7 & 7 = 7  ✓
  idx = 8: 8 & 7 = 0  ← 折り返し！
  idx = 9: 9 & 7 = 1  ✓
```

また、`head` と `tail` を折り返さずに単調増加させ続ける手法も有効です。
64 ビット整数がオーバーフローするまでの時間は現実的に問題になりません。

```rust
// インデックス計算の比較
#[inline(always)]
fn mask(idx: usize, cap: usize) -> usize {
    idx & (cap - 1) // cap は 2 のべき乗
}
```

---

## `VecDeque` との実装・性能比較

標準ライブラリの `VecDeque<T>` は動的にサイズが変わるリングバッファです。
固定長の場合は手製実装の方が速い傾向があります。

```rust
use std::collections::VecDeque;
use std::time::Instant;

fn bench_vecdeque(n: usize) {
    let mut dq: VecDeque<u64> = VecDeque::with_capacity(n);
    let start = Instant::now();
    for i in 0..n as u64 {
        dq.push_back(i);
        if dq.len() > 1024 {
            dq.pop_front();
        }
    }
    println!("VecDeque: {:?}", start.elapsed());
}

fn bench_ring_buffer(n: usize) {
    let mut rb: RingBuffer<u64, 1024> = RingBuffer::new();
    let start = Instant::now();
    for i in 0..n as u64 {
        if rb.is_full() {
            rb.pop();
        }
        rb.push(i).unwrap();
    }
    println!("RingBuffer: {:?}", start.elapsed());
}

fn main() {
    let n = 10_000_000;
    bench_vecdeque(n);
    bench_ring_buffer(n);
}
```

```
典型的な結果（参考値）:
  VecDeque:    ~85 ms   （動的リサイズの可能性あり）
  RingBuffer:  ~40 ms   （固定長・MaybeUninit 使用）
```

---

## SPSC（シングルプロデューサー・シングルコンシューマー）への応用

1 つのスレッドが書き込み（producer）、別の 1 つのスレッドが読み出し（consumer）を行う場合、
`head` と `tail` を `AtomicUsize` にするだけでロックフリーな SPSC キューが実現できます。

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::mem::MaybeUninit;
use std::sync::Arc;

pub struct SpscQueue<T, const N: usize> {
    buf: Box<[MaybeUninit<T>; N]>,
    head: AtomicUsize,
    tail: AtomicUsize,
}

unsafe impl<T: Send, const N: usize> Send for SpscQueue<T, N> {}
unsafe impl<T: Send, const N: usize> Sync for SpscQueue<T, N> {}

impl<T, const N: usize> SpscQueue<T, N> {
    pub fn new() -> Arc<Self> {
        assert!(N.is_power_of_two());
        Arc::new(Self {
            // SAFETY: MaybeUninit の配列は未初期化で安全
            buf: Box::new(unsafe { MaybeUninit::uninit().assume_init() }),
            head: AtomicUsize::new(0),
            tail: AtomicUsize::new(0),
        })
    }

    /// プロデューサー側: 要素を書き込む
    pub fn push(&self, val: T) -> Result<(), T> {
        let tail = self.tail.load(Ordering::Relaxed);
        let head = self.head.load(Ordering::Acquire); // consumer の更新を観測
        if tail.wrapping_sub(head) == N {
            return Err(val); // 満杯
        }
        let idx = tail & (N - 1);
        unsafe {
            (self.buf[idx].as_ptr() as *mut T).write(val);
        }
        self.tail.store(tail.wrapping_add(1), Ordering::Release); // consumer に公開
        Ok(())
    }

    /// コンシューマー側: 要素を読み出す
    pub fn pop(&self) -> Option<T> {
        let head = self.head.load(Ordering::Relaxed);
        let tail = self.tail.load(Ordering::Acquire); // producer の更新を観測
        if head == tail {
            return None; // 空
        }
        let idx = head & (N - 1);
        let val = unsafe { self.buf[idx].assume_init_read() };
        self.head.store(head.wrapping_add(1), Ordering::Release); // producer に公開
        Some(val)
    }
}

fn main() {
    let queue: Arc<SpscQueue<u64, 1024>> = SpscQueue::new();
    let q_producer = Arc::clone(&queue);
    let q_consumer = Arc::clone(&queue);

    let producer = std::thread::spawn(move || {
        for i in 0..100_000u64 {
            loop {
                if q_producer.push(i).is_ok() {
                    break;
                }
                std::hint::spin_loop();
            }
        }
    });

    let consumer = std::thread::spawn(move || {
        let mut sum = 0u64;
        for _ in 0..100_000 {
            loop {
                if let Some(v) = q_consumer.pop() {
                    sum += v;
                    break;
                }
                std::hint::spin_loop();
            }
        }
        sum
    });

    producer.join().unwrap();
    let sum = consumer.join().unwrap();
    println!("sum = {}", sum); // 4999950000
}
```

### メモリオーダリングの解説

```
Producer                        Consumer
---------                       ---------
buf[tail].write(val)            buf[head].read()
tail.store(Release)  ──────→   tail.load(Acquire)
                               head.store(Release) ──→ head.load(Acquire)

Release: それ以前の全書き込みを公開
Acquire: それ以降の全読み込みが公開済みデータを観測
```

---

## `ringbuf` クレートの紹介

実用的な SPSC リングバッファが必要な場合は `ringbuf` クレートが便利です。

```toml
# Cargo.toml
[dependencies]
ringbuf = "0.4"
```

```rust
use ringbuf::HeapRb;

fn main() {
    // 容量 1024 のヒープ確保リングバッファ
    let rb = HeapRb::<i32>::new(1024);
    let (mut producer, mut consumer) = rb.split();

    std::thread::scope(|s| {
        s.spawn(|| {
            for i in 0..500 {
                while producer.push(i).is_err() {
                    std::hint::spin_loop();
                }
            }
        });

        s.spawn(|| {
            let mut count = 0;
            while count < 500 {
                if let Some(v) = consumer.pop() {
                    println!("received: {}", v);
                    count += 1;
                }
            }
        });
    });
}
```

`ringbuf` の主な特徴:
- ヒープ（`HeapRb`）・静的（`StaticRb`）両対応
- スライスベースの一括転送 API（`push_slice` / `pop_slice`）
- `no_std` 環境にも対応

---

## まとめ

| 比較項目 | 手製 RingBuffer | VecDeque | ringbuf |
|---|---|---|---|
| メモリ確保 | ゼロ（スタック） | ヒープ（動的） | ヒープ（固定） |
| 容量の変更 | 不可 | 可 | 不可 |
| SPSC 対応 | AtomicUsize で可 | Mutex 必要 | 標準機能 |
| no_std | 可 | 不可 | 可 |
| 実装コスト | 高 | なし | 低 |
