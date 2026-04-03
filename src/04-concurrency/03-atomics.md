# Atomics とメモリオーダリング

マルチスレッドプログラミングにおいて、複数のスレッドが共有データにアクセスする際の正確な動作を理解するには、CPU・コンパイラの命令並び替え（リオーダリング）とメモリモデルの知識が不可欠である。Rust の `std::sync::atomic` は、Mutex を使わずにスレッド間で安全にデータを共有するための基本部品を提供する。

---

## 1. アトミック型の基本操作

### `AtomicUsize` と `AtomicBool`

```rust
use std::sync::atomic::{AtomicUsize, AtomicBool, Ordering};
use std::sync::Arc;
use std::thread;

fn basic_atomic_operations() {
    let counter = Arc::new(AtomicUsize::new(0));
    let flag = Arc::new(AtomicBool::new(false));

    let mut handles = vec![];

    for _ in 0..8 {
        let counter = Arc::clone(&counter);
        let flag = Arc::clone(&flag);

        handles.push(thread::spawn(move || {
            // fetch_add: 古い値を返しつつ加算（アトミック）
            let old = counter.fetch_add(1, Ordering::SeqCst);
            println!("加算前の値: {}", old);

            // store: 値を書き込む
            flag.store(true, Ordering::Release);

            // load: 値を読み取る
            let current = counter.load(Ordering::Acquire);
            println!("現在のカウンタ: {}", current);
        }));
    }

    for h in handles {
        h.join().unwrap();
    }

    println!("最終カウンタ: {}", counter.load(Ordering::SeqCst));
}
```

### `fetch_add` / `fetch_sub` / `fetch_and` / `fetch_or`

```rust
use std::sync::atomic::{AtomicUsize, AtomicU32, Ordering};

fn fetch_operations() {
    let x = AtomicUsize::new(10);

    // 加算: 10 -> 15, 戻り値 = 10（古い値）
    let old = x.fetch_add(5, Ordering::Relaxed);
    assert_eq!(old, 10);
    assert_eq!(x.load(Ordering::Relaxed), 15);

    // 減算: 15 -> 12, 戻り値 = 15
    let old = x.fetch_sub(3, Ordering::Relaxed);
    assert_eq!(old, 15);
    assert_eq!(x.load(Ordering::Relaxed), 12);

    // ビット OR: フラグのセットに便利
    let flags = AtomicU32::new(0b0001);
    flags.fetch_or(0b0100, Ordering::Relaxed); // 0b0101
    assert_eq!(flags.load(Ordering::Relaxed), 0b0101);

    // ビット AND: フラグのクリアに便利
    flags.fetch_and(!0b0001, Ordering::Relaxed); // 0b0100
    assert_eq!(flags.load(Ordering::Relaxed), 0b0100);
}
```

### `compare_exchange` と `compare_exchange_weak`

Compare-and-Swap (CAS) はロックフリーアルゴリズムの根幹をなす操作である。

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

fn compare_exchange_example() {
    let value = AtomicUsize::new(42);

    // compare_exchange(current, new, success_ordering, failure_ordering)
    // 現在値が current と等しければ new に置き換える
    match value.compare_exchange(
        42,              // 期待値
        100,             // 新しい値
        Ordering::SeqCst,  // 成功時の ordering
        Ordering::Relaxed, // 失敗時の ordering
    ) {
        Ok(old) => println!("CAS 成功: {} -> 100", old),
        Err(actual) => println!("CAS 失敗: 実際の値は {}", actual),
    }

    assert_eq!(value.load(Ordering::SeqCst), 100);

    // compare_exchange_weak: スプリアス失敗（偽の失敗）を許容する
    // ループ内では weak の方が効率的な場合がある
    let mut current = value.load(Ordering::Relaxed);
    loop {
        match value.compare_exchange_weak(
            current,
            current + 1,
            Ordering::SeqCst,
            Ordering::Relaxed,
        ) {
            Ok(_) => break,
            Err(actual) => {
                current = actual; // 最新値で再試行
            }
        }
    }
    println!("インクリメント後: {}", value.load(Ordering::SeqCst));
}
```

### `AtomicPtr`

```rust
use std::sync::atomic::{AtomicPtr, Ordering};
use std::ptr;

fn atomic_ptr_example() {
    // 共有ポインタの安全なスワップ
    let mut data1 = Box::new(42u32);
    let mut data2 = Box::new(100u32);

    let ptr = AtomicPtr::new(data1.as_mut() as *mut u32);

    // CAS でポインタを交換
    let old = ptr.compare_exchange(
        data1.as_mut() as *mut u32,
        data2.as_mut() as *mut u32,
        Ordering::SeqCst,
        Ordering::Relaxed,
    );

    match old {
        Ok(p) => println!("ポインタ交換成功: 旧値={}", unsafe { *p }),
        Err(p) => println!("ポインタ交換失敗: 現在値={}", unsafe { *p }),
    }

    let current = ptr.load(Ordering::SeqCst);
    println!("現在のポインタが指す値: {}", unsafe { *current });
}
```

---

## 2. `Ordering` の種類と意味

メモリオーダリングは「このアトミック操作を実行する際、他のメモリ操作との順序関係をどう保証するか」を指定する。

### 5 種類の Ordering

| Ordering | 読み取り | 書き込み | 特徴 |
|----------|---------|---------|------|
| `Relaxed` | 可能 | 可能 | 順序保証なし・最も高速 |
| `Acquire` | 可能 | 不可 | 以降の読み書きが前に来ないことを保証 |
| `Release` | 不可 | 可能 | 以前の読み書きが後に行かないことを保証 |
| `AcqRel` | 可能 | 可能 | Acquire + Release の組み合わせ |
| `SeqCst` | 可能 | 可能 | 全スレッド間で一貫した順序を保証・最も遅い |

```rust
use std::sync::atomic::{AtomicUsize, AtomicBool, Ordering};
use std::sync::Arc;
use std::thread;

// Relaxed: 順序保証不要な単純なカウンタ
fn relaxed_counter_example() {
    let counter = Arc::new(AtomicUsize::new(0));
    let mut handles = vec![];

    for _ in 0..1000 {
        let c = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            // 他の操作との順序は問わない; 単に正確に加算されれば良い
            c.fetch_add(1, Ordering::Relaxed);
        }));
    }

    for h in handles { h.join().unwrap(); }
    println!("Relaxed カウンタ: {}", counter.load(Ordering::Relaxed));
}
```

---

## 3. happens-before 関係とメモリモデル

**happens-before** とは「操作 A が操作 B より先に完了することが保証される」関係を指す。

```
スレッド1                       スレッド2
─────────────────────────────   ──────────────────────────────
data = compute_result()         loop { if ready.load(Acquire) { break; } }
ready.store(true, Release)      // ここで data を安全に読める
```

### Acquire/Release ペアの仕組み

```
Release store より前の全ての書き込み
    ↓ happens-before
Acquire load より後の全ての読み取り
```

```rust
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;
use std::thread;

fn happens_before_demo() {
    let data: Arc<std::cell::UnsafeCell<Vec<i32>>> =
        Arc::new(std::cell::UnsafeCell::new(Vec::new()));
    let ready = Arc::new(AtomicBool::new(false));

    let data_clone = Arc::clone(&data);
    let ready_clone = Arc::clone(&ready);

    let producer = thread::spawn(move || {
        // データを準備する
        unsafe {
            let v = &mut *data_clone.get();
            v.push(1);
            v.push(2);
            v.push(3);
        }
        // Release store: 上記の書き込みが store より前に完了することを保証
        ready_clone.store(true, Ordering::Release);
    });

    let consumer = thread::spawn(move || {
        // Acquire load: load 以降の読み取りが load より後であることを保証
        // ready が true になるまでスピン
        while !ready.load(Ordering::Acquire) {
            std::hint::spin_loop();
        }
        // ここで data の読み取りが安全 (happens-before 関係が成立)
        let v = unsafe { &*data.get() };
        println!("consumer が読んだデータ: {:?}", v);
    });

    producer.join().unwrap();
    consumer.join().unwrap();
}
```

---

## 4. Acquire/Release ペアによるフラグ同期パターン

ワンショット通知パターンは、あるスレッドが「準備完了」を別スレッドに伝える典型的なユースケースである。

```rust
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;
use std::thread;
use std::time::Duration;

struct OneShot {
    ready: AtomicBool,
}

impl OneShot {
    fn new() -> Self {
        OneShot { ready: AtomicBool::new(false) }
    }

    // 送信側: Release で書き込み前の操作を公開
    fn signal(&self) {
        self.ready.store(true, Ordering::Release);
    }

    // 受信側: Acquire で書き込み後の操作と happens-before を確立
    fn wait(&self) {
        while !self.ready.load(Ordering::Acquire) {
            std::hint::spin_loop();
        }
    }
}

fn oneshot_flag_example() {
    let shot = Arc::new(OneShot::new());
    let shot2 = Arc::clone(&shot);

    let t1 = thread::spawn(move || {
        println!("スレッド1: 重い処理を実行中...");
        thread::sleep(Duration::from_millis(100));
        println!("スレッド1: 完了。シグナルを送信");
        shot2.signal();
    });

    let t2 = thread::spawn(move || {
        println!("スレッド2: シグナルを待機中...");
        shot.wait();
        println!("スレッド2: シグナルを受信！処理を継続");
    });

    t1.join().unwrap();
    t2.join().unwrap();
}
```

---

## 5. スピンロックを Atomics で実装する

```rust
use std::sync::atomic::{AtomicBool, Ordering};
use std::cell::UnsafeCell;
use std::ops::{Deref, DerefMut};

/// シンプルなスピンロック実装
pub struct SpinLock<T> {
    locked: AtomicBool,
    data: UnsafeCell<T>,
}

// 複数スレッドからの共有を許可
unsafe impl<T: Send> Sync for SpinLock<T> {}
unsafe impl<T: Send> Send for SpinLock<T> {}

pub struct SpinGuard<'a, T> {
    lock: &'a SpinLock<T>,
}

impl<T> SpinLock<T> {
    pub fn new(data: T) -> Self {
        SpinLock {
            locked: AtomicBool::new(false),
            data: UnsafeCell::new(data),
        }
    }

    pub fn lock(&self) -> SpinGuard<T> {
        loop {
            // Acquire: ロック取得後の読み書きが前に来ないことを保証
            if self.locked
                .compare_exchange_weak(
                    false,
                    true,
                    Ordering::Acquire,
                    Ordering::Relaxed,
                )
                .is_ok()
            {
                return SpinGuard { lock: self };
            }
            // スピン中は CPU にヒントを出してパイプラインの無駄を減らす
            std::hint::spin_loop();
        }
    }
}

impl<'a, T> Deref for SpinGuard<'a, T> {
    type Target = T;
    fn deref(&self) -> &T {
        unsafe { &*self.lock.data.get() }
    }
}

impl<'a, T> DerefMut for SpinGuard<'a, T> {
    fn deref_mut(&mut self) -> &mut T {
        unsafe { &mut *self.lock.data.get() }
    }
}

impl<'a, T> Drop for SpinGuard<'a, T> {
    fn drop(&mut self) {
        // Release: ロック解放前の読み書きが後に行かないことを保証
        self.lock.locked.store(false, Ordering::Release);
    }
}

// 使用例
fn spinlock_example() {
    use std::sync::Arc;
    use std::thread;

    let lock = Arc::new(SpinLock::new(0usize));
    let mut handles = vec![];

    for _ in 0..8 {
        let l = Arc::clone(&lock);
        handles.push(thread::spawn(move || {
            let mut guard = l.lock();
            *guard += 1;
            // guard がスコープを抜けると自動的にロック解放
        }));
    }

    for h in handles { h.join().unwrap(); }
    println!("最終値: {}", *lock.lock()); // 8
}
```

---

## 6. よくある間違い

### 間違い1: Relaxed の過剰使用

```rust
use std::sync::atomic::{AtomicBool, AtomicUsize, Ordering};
use std::sync::Arc;
use std::thread;

static DATA: AtomicUsize = AtomicUsize::new(0);
static READY: AtomicBool = AtomicBool::new(false);

// 間違いパターン: Relaxed を使うと happens-before が成立しない
fn wrong_relaxed_sync() {
    let producer = thread::spawn(|| {
        DATA.store(42, Ordering::Relaxed);
        // Relaxed: DATA の書き込みが READY の書き込みより先に見える保証がない!
        READY.store(true, Ordering::Relaxed);
    });

    let consumer = thread::spawn(|| {
        while !READY.load(Ordering::Relaxed) {
            std::hint::spin_loop();
        }
        // DATA が 42 である保証がない（コンパイラ/CPU が並び替える可能性）
        let value = DATA.load(Ordering::Relaxed);
        // value が 0 の可能性がある!
        println!("値: {}", value);
    });

    producer.join().unwrap();
    consumer.join().unwrap();
}

// 正しいパターン: Release/Acquire を使う
fn correct_release_acquire_sync() {
    let producer = thread::spawn(|| {
        DATA.store(42, Ordering::Relaxed); // DATA は Relaxed でも良い
        // Release: 上記の書き込みが store より前であることを保証
        READY.store(true, Ordering::Release);
    });

    let consumer = thread::spawn(|| {
        // Acquire: 下記の読み取りが load より後であることを保証
        while !READY.load(Ordering::Acquire) {
            std::hint::spin_loop();
        }
        // ここで DATA は必ず 42
        let value = DATA.load(Ordering::Relaxed);
        assert_eq!(value, 42);
        println!("値: {}", value);
    });

    producer.join().unwrap();
    consumer.join().unwrap();
}
```

### 間違い2: SeqCst の過剰使用（パフォーマンス低下）

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

static COUNTER: AtomicUsize = AtomicUsize::new(0);

// SeqCst は全スレッド間の全順序を保証するため重い
fn slow_seqcst_counter() {
    // 1000万回の SeqCst カウントアップ（低速）
    for _ in 0..10_000_000 {
        COUNTER.fetch_add(1, Ordering::SeqCst);
    }
}

// 単純なカウンタなら Relaxed で十分
fn fast_relaxed_counter() {
    // 1000万回の Relaxed カウントアップ（高速）
    for _ in 0..10_000_000 {
        COUNTER.fetch_add(1, Ordering::Relaxed);
    }
    // 最終値を読む際だけ SeqCst を使う必要もない
    let _ = COUNTER.load(Ordering::Relaxed);
}
```

### 間違い3: アトミック操作の複合を非アトミックに扱う

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

static VALUE: AtomicUsize = AtomicUsize::new(0);

// 間違い: 「読んで書く」が非アトミック（TOCTOU 競合）
fn non_atomic_read_modify_write() {
    let v = VALUE.load(Ordering::Relaxed);
    // ここで別スレッドが VALUE を変更する可能性がある!
    VALUE.store(v + 1, Ordering::Relaxed);
}

// 正しい: fetch_add でアトミックに更新
fn atomic_read_modify_write() {
    VALUE.fetch_add(1, Ordering::Relaxed);
}

// CAS ループ: より複雑な条件付き更新
fn conditional_update(threshold: usize) {
    loop {
        let current = VALUE.load(Ordering::Relaxed);
        if current >= threshold {
            break; // 閾値以上なら更新しない
        }
        if VALUE.compare_exchange_weak(
            current,
            current + 1,
            Ordering::Relaxed,
            Ordering::Relaxed,
        ).is_ok() {
            break;
        }
        // スプリアス失敗または競合 → 再試行
    }
}
```

---

## まとめ

| Ordering | 使いどころ |
|----------|-----------|
| `Relaxed` | 独立したカウンタ。他の操作との順序関係が不要な場合 |
| `Acquire` | ロック取得・フラグの load。以降の読み書きを守る |
| `Release` | ロック解放・フラグの store。以前の読み書きを公開する |
| `AcqRel` | fetch_add 等の read-modify-write でロック的な役割を持つ場合 |
| `SeqCst` | 全スレッド間で一貫した順序が必要な場合（デフォルト選択肢だが遅い） |

**選択指針**: まず `SeqCst` で正しく動くコードを書き、プロファイリングでボトルネックを確認してから `Acquire`/`Release` に落とす。`Relaxed` は意味が明確な場合のみ使う。
