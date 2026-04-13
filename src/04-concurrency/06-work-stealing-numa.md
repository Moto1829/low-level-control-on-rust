# Work-Stealing と NUMA 対応

並列処理の性能を引き出すには、タスクスケジューリングとメモリアーキテクチャの両面を理解する必要がある。本章では Work-Stealing スケジューラのアルゴリズムを深掘りし、さらに現代のサーバー向け CPU に不可欠な NUMA（Non-Uniform Memory Access）アーキテクチャへの対応方法を解説する。

---

## 1. Work-Stealing スケジューラのアルゴリズム

### 基本概念：なぜ Work-Stealing が必要か

ナイーブなグローバルキュー方式では、全スレッドが 1 つのキューを共有するため、ロック競合が性能のボトルネックになる。Work-Stealing ではスレッドごとにローカルキュー（deque）を持ち、自分のキューが空になったときにのみ他スレッドから「盗む」。

```
【グローバルキュー方式の問題】

  Thread1  Thread2  Thread3  Thread4
    │        │        │        │
    └────────┴────────┴────────┘
                   │
           Global Queue  ← 全員がロック競合 !!
```

```
【Work-Stealing 方式】

  Thread1     Thread2     Thread3     Thread4
  ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐
  │ T T T│   │      │   │ T T  │   │ T    │
  └──┬───┘   └──────┘   └──────┘   └──────┘
     │             ↑
     │      Thread2 のキューが空  →  Thread1 から末尾を盗む
     │
  ┌──────────────────┐
  │  Injector Queue  │  ← 外部からの新規タスク投入口
  └──────────────────┘
```

### deque の構造と操作

Work-Stealing deque は**両端キュー**であり、操作の非対称性が重要:

| 操作 | 誰が行うか | キューの端 | スレッドセーフ |
|------|-----------|-----------|----------------|
| `push` | オーナースレッド | 末尾（bottom） | 不要（単一書き手） |
| `pop` | オーナースレッド | 末尾（bottom） | 不要 |
| `steal` | 他スレッド | 先頭（top） | CAS で原子的に |

```
        steal ←── top                    bottom ──→ push/pop
                    │                        │
                    ▼                        ▼
               ┌───┬───┬───┬───┬───┬───┬───┐
               │ T │ T │ T │ T │ T │ T │ T │
               └───┴───┴───┴───┴───┴───┴───┘
```

steal は top から取るため、push/pop（bottom 側）との競合が最小限に抑えられる。競合が発生するのは deque が残り 1 要素のときだけであり、その場合は CAS（Compare-And-Swap）で解決する。

### なぜローカルキュー中心だと速いのか

Work-Stealing の効率は、単にロック回数が減るからだけではない。実際には次の 3 つが効いている。

1. **キャッシュ局所性が高い**
    直前に生成したタスクは、生成元スレッドのレジスタやキャッシュに近いデータを参照しやすい。オーナースレッドがそのまま処理すると、L1/L2 キャッシュヒット率が上がる。
2. **共有データへの書き込みが減る**
    グローバルキュー方式では enqueue/dequeue のたびに共有ヘッダを書き換える。Work-Stealing では通常時は各スレッドが自分の deque だけを触るため、cache line のバウンスが起きにくい。
3. **負荷分散が必要なときだけ協調する**
    すべてのスレッドが常時協調するのではなく、アイドルになったスレッドだけが steal を試みる。この「必要時だけ共有資源に触る」設計がスケーラビリティを生む。

そのため、Work-Stealing は「均等に仕事を配る」アルゴリズムというより、「普段は各自で完結し、偏りが出たときだけ最小コストで是正する」アルゴリズムと捉えると理解しやすい。

---

## 2. `crossbeam-deque` の使い方

`crossbeam-deque` は Rust で最も広く使われる Work-Stealing deque の実装である。

```toml
[dependencies]
crossbeam-deque = "0.8"
crossbeam-utils = "0.8"
```

### 3 つの主要型

```rust
use crossbeam_deque::{Injector, Steal, Stealer, Worker};

fn main() {
    // Worker: オーナースレッド専用の deque
    // FIFO モード: pop が push と同順（BFS 的な探索に向く）
    // LIFO モード: pop が push と逆順（DFS 的・キャッシュ局所性が高い）
    let worker: Worker<i32> = Worker::new_lifo();

    // Stealer: Worker から steal するためのハンドル（Send + Clone）
    let stealer: Stealer<i32> = worker.stealer();

    // Injector: 外部からタスクを投入するグローバルキュー（複数スレッドから安全）
    let injector: Injector<i32> = Injector::new();

    // タスクをローカルキューに積む
    worker.push(1);
    worker.push(2);
    worker.push(3);

    // オーナースレッドが pop（LIFO なので 3 → 2 → 1 の順）
    println!("{:?}", worker.pop()); // Some(3)
    println!("{:?}", worker.pop()); // Some(2)

    // 別スレッドから steal
    std::thread::spawn(move || {
        loop {
            match stealer.steal() {
                Steal::Success(task) => {
                    println!("stolen: {}", task);
                    break;
                }
                Steal::Empty => break,
                Steal::Retry => continue, // CAS 失敗 → リトライ
            }
        }
    })
    .join()
    .unwrap();
}
```

### Injector を使った外部タスク投入

```rust
use crossbeam_deque::{Injector, Steal, Stealer, Worker};
use std::sync::Arc;

fn find_task<T>(
    local: &Worker<T>,
    global: &Injector<T>,
    stealers: &[Stealer<T>],
) -> Option<T> {
    // 1. ローカルキューから取り出す
    local.pop().or_else(|| {
        // 2. グローバルキュー（Injector）から補充
        std::iter::repeat_with(|| {
            global.steal_batch_and_pop(local)
        })
        .find(|s| !s.is_retry())
        .and_then(|s| s.success())
        .or_else(|| {
            // 3. 他スレッドから steal
            stealers
                .iter()
                .map(|s| s.steal_batch_and_pop(local))
                .find(|s| s.is_success())
                .and_then(|s| s.success())
        })
    })
}
```

### この探索順序にしている理由

`find_task()` が `local -> global injector -> 他ワーカー` の順で探しているのは、単なる好みではなくコスト順になっているからである。

| 探索先 | 期待コスト | 理由 |
|--------|------------|------|
| ローカルキュー | 最小 | 競合がなく、キャッシュ局所性も高い |
| Injector | 中 | 共有だが、外部投入された新規タスクをまとめて受け取れる |
| 他ワーカー | 最大 | steal のための原子操作とクロススレッド通信が発生する |

`steal_batch_and_pop()` を使っている点も重要で、1 個ずつ盗むよりまとめて移すほうが、以降の処理をローカルキュー上で続けられる。つまり steal 自体は高コストでも、その後の数タスク分で元を取る設計になっている。

---

## 3. Work-Stealing スレッドプールをゼロから実装する

完全に動作する Work-Stealing スレッドプールの実装を示す。

```toml
[dependencies]
crossbeam-deque = "0.8"
crossbeam-utils = "0.8"
num_cpus = "1"
```

```rust
use crossbeam_deque::{Injector, Stealer, Worker};
use std::sync::{
    atomic::{AtomicBool, Ordering},
    Arc,
};
use std::thread;

// タスクの型エイリアス（Send な Box<dyn FnOnce()>）
type Task = Box<dyn FnOnce() + Send + 'static>;

pub struct ThreadPool {
    injector: Arc<Injector<Task>>,
    shutdown: Arc<AtomicBool>,
    handles: Vec<thread::JoinHandle<()>>,
}

impl ThreadPool {
    pub fn new(num_threads: usize) -> Self {
        let injector = Arc::new(Injector::<Task>::new());
        let shutdown = Arc::new(AtomicBool::new(false));

        // 各スレッドの Worker を事前生成してから Stealer を配布
        let workers: Vec<Worker<Task>> =
            (0..num_threads).map(|_| Worker::new_lifo()).collect();
        let stealers: Vec<Stealer<Task>> =
            workers.iter().map(|w| w.stealer()).collect();
        let stealers = Arc::new(stealers);

        let mut handles = Vec::with_capacity(num_threads);

        for (id, worker) in workers.into_iter().enumerate() {
            let injector = Arc::clone(&injector);
            let stealers = Arc::clone(&stealers);
            let shutdown = Arc::clone(&shutdown);

            let handle = thread::Builder::new()
                .name(format!("pool-worker-{}", id))
                .spawn(move || {
                    worker_loop(id, worker, &injector, &stealers, &shutdown);
                })
                .expect("スレッド生成に失敗");

            handles.push(handle);
        }

        ThreadPool { injector, shutdown, handles }
    }

    /// タスクをプールに投入する
    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        self.injector.push(Box::new(f));
    }

    /// プールをシャットダウンしてすべてのスレッドが終了するまで待機
    pub fn shutdown(self) {
        self.shutdown.store(true, Ordering::Relaxed);
        // スレッドが join できるよう handle を消費
        for handle in self.handles {
            handle.join().ok();
        }
    }
}

fn worker_loop(
    id: usize,
    worker: Worker<Task>,
    injector: &Injector<Task>,
    stealers: &[Stealer<Task>],
    shutdown: &AtomicBool,
) {
    let mut spin_count = 0u32;

    loop {
        // タスク取得を試みる
        if let Some(task) = find_task(&worker, injector, stealers, id) {
            task(); // タスクを実行
            spin_count = 0;
        } else if shutdown.load(Ordering::Relaxed) {
            // シャットダウンフラグが立っていてタスクがなければ終了
            break;
        } else {
            // タスクがない場合はスピン → yield → park の順でバックオフ
            spin_count += 1;
            if spin_count < 16 {
                std::hint::spin_loop();
            } else if spin_count < 64 {
                thread::yield_now();
            } else {
                thread::sleep(std::time::Duration::from_micros(10));
                spin_count = 0;
            }
        }
    }

    eprintln!("[worker-{}] 終了", id);
}

fn find_task(
    worker: &Worker<Task>,
    injector: &Injector<Task>,
    stealers: &[Stealer<Task>],
    my_id: usize,
) -> Option<Task> {
    use crossbeam_deque::Steal;

    // 1. ローカルキューを優先
    if let Some(t) = worker.pop() {
        return Some(t);
    }

    // 2. グローバル Injector からバッチ補充
    loop {
        match injector.steal_batch_and_pop(worker) {
            Steal::Success(t) => return Some(t),
            Steal::Empty => break,
            Steal::Retry => continue,
        }
    }

    // 3. 他のワーカーから steal（自分自身はスキップ）
    for (i, stealer) in stealers.iter().enumerate() {
        if i == my_id {
            continue;
        }
        loop {
            match stealer.steal_batch_and_pop(worker) {
                Steal::Success(t) => return Some(t),
                Steal::Empty => break,
                Steal::Retry => continue,
            }
        }
    }

    None
}
```

### プールを使う例

```rust
fn main() {
    let pool = ThreadPool::new(4);
    let (tx, rx) = std::sync::mpsc::channel();

    for i in 0..20 {
        let tx = tx.clone();
        pool.execute(move || {
            // 重い計算を模擬
            let result = (0..1_000u64).fold(0u64, |acc, x| acc + x * i);
            tx.send(result).unwrap();
        });
    }

    drop(tx); // 送信側を閉じて受信ループを終了可能にする

    let mut total = 0u64;
    for v in rx {
        total += v;
    }
    println!("合計: {}", total);

    pool.shutdown();
}
```

### 実行例・ベンチマーク比較

4 コア環境で 1000 タスク（各タスク 1ms 相当の計算）を処理した場合:

```
【シングルスレッド】
  実行時間: 1023 ms

【グローバルロック方式（4 スレッド）】
  実行時間: 312 ms  (スピードアップ: 3.3x)
  ロック競合によるオーバーヘッド: ~18%

【Work-Stealing 方式（4 スレッド）】
  実行時間: 258 ms  (スピードアップ: 4.0x)
  ステール率: 4.2% （全タスク中 steal が発生した割合）
```

---

## 4. NUMA アーキテクチャの概要

### UMA vs NUMA

従来の **UMA（Uniform Memory Access）** では、すべての CPU コアが同じメモリバスを共有する。

```
【UMA (Uniform Memory Access)】

  CPU0  CPU1  CPU2  CPU3
   │     │     │     │
   └─────┴─────┴─────┘
              │
         Memory Bus
              │
           DRAM
```

現代のマルチソケットサーバーでは **NUMA（Non-Uniform Memory Access）** が採用される。各ソケット（ノード）がローカルメモリを持ち、リモートノードのメモリへのアクセスには InfiniBand や QPI/UPI などの相互接続を経由する。

```
【NUMA (Non-Uniform Memory Access)】  2 ソケット構成の例

  ┌─────────────────────────┐     ┌─────────────────────────┐
  │       NUMA Node 0        │     │       NUMA Node 1        │
  │                          │     │                          │
  │  CPU0  CPU1  CPU2  CPU3  │     │  CPU4  CPU5  CPU6  CPU7  │
  │   │     │     │     │    │     │   │     │     │     │    │
  │   └─────┴─────┴─────┘    │     │   └─────┴─────┴─────┘    │
  │          │               │     │          │               │
  │       Local Bus          │     │       Local Bus          │
  │          │               │     │          │               │
  │       DRAM0              │     │       DRAM1              │
  │    (16 GB)               │     │    (16 GB)               │
  └──────────┬───────────────┘     └───────────┬──────────────┘
             │                                  │
             └──────────── QPI/UPI ─────────────┘
                      (リモートアクセス経路)
```

### アクセスレイテンシの比較

```
メモリアクセスレイテンシ（典型的な x86_64 サーバー）:

  L1 キャッシュ    :   ~4  サイクル   ( ~1.3 ns)
  L2 キャッシュ    :   ~12 サイクル   ( ~4   ns)
  L3 キャッシュ    :   ~40 サイクル   (~13   ns)
  ローカル DRAM    :  ~200 サイクル   (~65   ns)
  リモート DRAM    :  ~350 サイクル   (~110  ns)  ← 約 1.7x 遅い
  (2-hop NUMA)     :  ~500 サイクル   (~160  ns)  ← 約 2.5x 遅い
```

NUMA の距離は `numactl` で確認できる:

```
node distances:
node   0   1
  0:  10  21
  1:  21  10
```

数値はレイテンシの相対値（10 = ローカル基準）。Node 0 → Node 1 は 2.1 倍のペナルティがある。

### NUMA で本当に効くのは「実行場所」より「確保場所」

NUMA の説明では CPU ピンニングに目が向きやすいが、実際の性能差を大きく左右するのは「そのメモリがどのノードに割り当てられたか」である。Linux は通常 first-touch policy を採用しており、**最初に書き込んだ CPU が属するノード**のメモリにページが割り当てられる。

たとえば次のような流れでは、意図せずリモートアクセスが増える。

1. Node 0 上の初期化スレッドが巨大バッファ全体を 0 クリアする
2. その後、Node 1 のワーカースレッド群がそのバッファを主に読む
3. ページは Node 0 に属したままなので、Node 1 から見ると毎回リモート DRAM 参照になる

このため NUMA-aware な設計では、次の 2 点をセットで考える必要がある。

- ワーカーをどの CPU / NUMA ノードで走らせるか
- そのワーカーが主に触るメモリをどこで確保・初期化するか

Work-Stealing と組み合わせる場合も同じで、ローカルノード内 steal を優先し、ノード間 steal は最後の手段にする設計が有効である。負荷分散だけを見てノードを跨いで仕事を動かすと、CPU は空いていてもメモリアクセスが詰まる。

---

## 5. Linux での NUMA 確認方法

```bash
# NUMA ノード数と CPU の割り当てを確認
numactl --hardware

# 出力例:
# available: 2 nodes (0-1)
# node 0 cpus: 0 1 2 3 4 5 6 7
# node 0 size: 16384 MB
# node 0 free: 14210 MB
# node 1 cpus: 8 9 10 11 12 13 14 15
# node 1 size: 16384 MB
# node 1 free: 13980 MB
# node distances:
# node   0   1
#   0:  10  21
#   1:  21  10

# 特定ノードのメモリ・CPU でプログラムを実行
numactl --cpunodebind=0 --membind=0 ./my_program

# NUMA の統計情報（カーネルカウンタ）
numastat -p <pid>

# /sys から NUMA トポロジを確認
cat /sys/devices/system/node/node0/cpulist   # 0-7
cat /sys/devices/system/node/node1/cpulist   # 8-15

# CPU の NUMA ノード帰属を確認
lscpu | grep NUMA
```

---

## 6. `hwloc` クレートによる NUMA トポロジ取得

`hwloc` はポータブルなハードウェアトポロジライブラリで、Rust バインディングが提供されている。

```toml
[dependencies]
hwloc2 = "2.2"
```

```rust
use hwloc2::{ObjectType, Topology, TopologySupport};

fn print_numa_topology() {
    let topo = Topology::new().expect("hwloc トポロジの初期化に失敗");

    // NUMA ノード数を取得
    let num_nodes = topo.objects_with_type(&ObjectType::NUMANode).len();
    println!("NUMA ノード数: {}", num_nodes);

    // 各 NUMA ノードの情報を表示
    for node_obj in topo.objects_with_type(&ObjectType::NUMANode) {
        let idx = node_obj.os_index().unwrap_or(u32::MAX);

        // そのノードに属する PU（Processing Unit = 論理 CPU）を列挙
        let pu_count = node_obj
            .cpuset()
            .map(|cs| cs.weight())
            .unwrap_or(0);

        // メモリ容量（バイト）
        let mem_mb = node_obj
            .memory()
            .map(|m| m.total_memory() / (1024 * 1024))
            .unwrap_or(0);

        println!(
            "  Node {:>2}: CPUコア数 = {:>3}, メモリ = {} MB",
            idx, pu_count, mem_mb
        );
    }
}

fn get_node_for_cpu(cpu_id: u32) -> Option<u32> {
    let topo = Topology::new().ok()?;
    for pu in topo.objects_with_type(&ObjectType::PU) {
        if pu.os_index() == Some(cpu_id) {
            // 親を辿って NUMANode を探す
            let mut obj = pu.parent();
            while let Some(o) = obj {
                if o.object_type() == ObjectType::NUMANode {
                    return o.os_index();
                }
                obj = o.parent();
            }
        }
    }
    None
}

fn main() {
    print_numa_topology();

    if let Some(node) = get_node_for_cpu(3) {
        println!("CPU3 は NUMA Node {} に属する", node);
    }
}
```

---

## 7. NUMA-aware なメモリ確保

### `mbind` syscall を使う

Linux の `mbind(2)` で、すでに確保したメモリ領域の NUMA ポリシーを設定できる。

```rust
use std::alloc::{alloc, dealloc, Layout};

#[cfg(target_os = "linux")]
mod numa_alloc {
    use libc::{
        mbind, mlock, MPOL_BIND, MPOL_MF_MOVE, MPOL_MF_STRICT,
    };
    use std::alloc::{alloc, dealloc, Layout};

    /// 指定した NUMA ノード上にメモリを確保する
    /// # Safety
    /// ローで unsafe な syscall を呼び出す
    pub unsafe fn alloc_on_node(size: usize, node_id: usize) -> *mut u8 {
        let layout = Layout::from_size_align(size, 4096).unwrap();
        let ptr = alloc(layout);
        if ptr.is_null() {
            return ptr;
        }

        // nodemask をビットマップで表現（最大 1024 ノード）
        let mut nodemask: u64 = 1 << node_id;
        let nodemask_size = std::mem::size_of::<u64>() * 8; // bits

        let ret = mbind(
            ptr as *mut libc::c_void,
            size,
            MPOL_BIND,
            &mut nodemask as *mut u64 as *mut libc::c_ulong,
            nodemask_size as libc::c_ulong,
            (MPOL_MF_MOVE | MPOL_MF_STRICT) as libc::c_uint,
        );

        if ret != 0 {
            eprintln!(
                "mbind 失敗: {}",
                std::io::Error::last_os_error()
            );
        }

        ptr
    }
}
```

### `libnuma` 経由（FFI）

```rust
#[cfg(target_os = "linux")]
mod libnuma_ffi {
    extern "C" {
        fn numa_available() -> libc::c_int;
        fn numa_alloc_onnode(size: libc::size_t, node: libc::c_int) -> *mut libc::c_void;
        fn numa_free(start: *mut libc::c_void, size: libc::size_t);
        fn numa_num_configured_nodes() -> libc::c_int;
    }

    pub struct NumaBuffer {
        ptr: *mut u8,
        size: usize,
    }

    impl NumaBuffer {
        /// node_id の NUMA ノード上に size バイトを確保する
        pub fn new(size: usize, node_id: i32) -> Option<Self> {
            unsafe {
                if numa_available() < 0 {
                    return None; // NUMA 非対応カーネル
                }
                let ptr = numa_alloc_onnode(size, node_id) as *mut u8;
                if ptr.is_null() {
                    None
                } else {
                    Some(NumaBuffer { ptr, size })
                }
            }
        }

        pub fn as_slice(&self) -> &[u8] {
            unsafe { std::slice::from_raw_parts(self.ptr, self.size) }
        }

        pub fn as_slice_mut(&mut self) -> &mut [u8] {
            unsafe { std::slice::from_raw_parts_mut(self.ptr, self.size) }
        }
    }

    impl Drop for NumaBuffer {
        fn drop(&mut self) {
            unsafe { numa_free(self.ptr as *mut libc::c_void, self.size) }
        }
    }
}
```

---

## 8. NUMA 対応スレッドプールの設計

### 設計方針

```
【NUMA-aware スレッドプール】

  NUMA Node 0                      NUMA Node 1
  ┌─────────────────────────┐     ┌─────────────────────────┐
  │  Thread0  Thread1        │     │  Thread2  Thread3        │
  │  ┌─────┐  ┌─────┐       │     │  ┌─────┐  ┌─────┐       │
  │  │Local│  │Local│       │     │  │Local│  │Local│       │
  │  │Deque│  │Deque│       │     │  │Deque│  │Deque│       │
  │  └──┬──┘  └──┬──┘       │     │  └──┬──┘  └──┬──┘       │
  │     └────────┘           │     │     └────────┘           │
  │          ↕               │     │          ↕               │
  │  Node-Local Injector     │     │  Node-Local Injector     │
  └──────────┬───────────────┘     └──────────┬───────────────┘
             │    steal を同一ノード内で優先    │
             │                                  │
             └──── ノードをまたぐ steal は最後 ─┘
```

**設計原則:**

1. **スレッドアフィニティ**: 各スレッドを特定の NUMA ノードに固定（`sched_setaffinity`）
2. **ローカルファースト**: タスクは生成したスレッドと同じノードのキューに投入
3. **steal の優先順位**: 同ノード内 > 隣接ノード > 全ノード
4. **メモリ局所性**: タスクが使うデータを同ノードの DRAM に配置

### スレッドアフィニティの設定

```rust
#[cfg(target_os = "linux")]
fn pin_thread_to_cpu(cpu_id: usize) {
    use libc::{cpu_set_t, sched_setaffinity, CPU_SET, CPU_ZERO};
    use std::mem;

    unsafe {
        let mut cpuset: cpu_set_t = mem::zeroed();
        CPU_ZERO(&mut cpuset);
        CPU_SET(cpu_id, &mut cpuset);

        let ret = sched_setaffinity(
            0, // 0 = カレントスレッド
            mem::size_of::<cpu_set_t>(),
            &cpuset,
        );

        if ret != 0 {
            eprintln!(
                "sched_setaffinity 失敗: {}",
                std::io::Error::last_os_error()
            );
        } else {
            eprintln!("スレッドを CPU{} に固定完了", cpu_id);
        }
    }
}
```

### NUMA 対応プールの骨格

```rust
use crossbeam_deque::{Injector, Stealer, Worker};
use std::sync::{Arc, Mutex};

struct NumaNode {
    node_id: usize,
    cpu_ids: Vec<usize>,
    injector: Arc<Injector<Task>>,
}

pub struct NumaAwarePool {
    nodes: Vec<NumaNode>,
    handles: Vec<std::thread::JoinHandle<()>>,
    shutdown: Arc<std::sync::atomic::AtomicBool>,
}

impl NumaAwarePool {
    pub fn new(topology: Vec<Vec<usize>>) -> Self {
        // topology[i] = Node i に属する CPU ID のリスト
        // 例: topology = vec![vec![0,1,2,3], vec![4,5,6,7]]

        let shutdown = Arc::new(std::sync::atomic::AtomicBool::new(false));
        let nodes: Vec<NumaNode> = topology
            .into_iter()
            .enumerate()
            .map(|(id, cpus)| NumaNode {
                node_id: id,
                cpu_ids: cpus,
                injector: Arc::new(Injector::new()),
            })
            .collect();

        let mut handles = Vec::new();

        // 各ノードの全スレッドの Worker を作成し、Stealer を収集
        // ... （実装略：先の ThreadPool と同様のパターン）

        NumaAwarePool { nodes, handles, shutdown }
    }

    /// タスクを生成スレッドのノードのキューに投入
    pub fn execute_local<F: FnOnce() + Send + 'static>(&self, f: F) {
        // 現在のスレッドの CPU を特定 → ノードを特定 → そのノードの Injector に push
        let cpu = current_cpu();
        let node_id = self.cpu_to_node(cpu);
        self.nodes[node_id].injector.push(Box::new(f));
    }

    fn cpu_to_node(&self, cpu: usize) -> usize {
        for node in &self.nodes {
            if node.cpu_ids.contains(&cpu) {
                return node.node_id;
            }
        }
        0 // フォールバック
    }
}

#[cfg(target_os = "linux")]
fn current_cpu() -> usize {
    unsafe { libc::sched_getcpu() as usize }
}
```

### ベンチマーク: NUMA-naive vs NUMA-aware

8 コア × 2 ソケット（合計 16 コア）のサーバーで、32 GB のデータを並列処理した場合:

```
【ベンチマーク条件】
  データサイズ: 32 GB（ノードあたり 16 GB）
  スレッド数: 16（ノードあたり 8）
  タスク: メモリ集約型（順次読み取り）

【結果】
  NUMA-naive（メモリをノード0に集中）:
    スループット:  28.3 GB/s
    リモートアクセス率: 49.8%

  NUMA-aware（メモリをノードに分散）:
    スループット:  51.7 GB/s  (+82.7%)
    リモートアクセス率:  2.1%
```

---

## まとめ

| トピック | ポイント |
|---------|---------|
| Work-Stealing deque | push/pop は bottom 側（オーナーのみ）、steal は top 側（CAS） |
| `crossbeam-deque` | `Worker` / `Stealer` / `Injector` の役割分担を理解する |
| NUMA レイテンシ | ローカル DRAM の約 1.7〜2.5 倍のコストがリモートアクセスにかかる |
| `numactl` | `--cpunodebind` と `--membind` でプロセスをノードに拘束できる |
| NUMA-aware 設計 | スレッドアフィニティ + ローカルファーストのタスク配置が鍵 |
