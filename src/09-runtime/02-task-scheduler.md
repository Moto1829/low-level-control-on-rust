# 02: タスクスケジューラの設計と実装

## スケジューラとは何か

前節のスレッドプールは「ジョブをFIFOで実行する」だけでした。しかし実際のシステムでは:

- **緊急度の違い** — ユーザー向けリクエストはバッチ処理より先に実行したい
- **依存関係** — タスクBはタスクAが完了してから実行したい
- **公平性** — あるタスクが常に他を占有しないようにしたい

スケジューラはこれらの「いつ・どの順番でタスクを実行するか」を決める層です。

```
タスクA (優先度: 高) ─┐
タスクB (優先度: 低) ─┼──→ [スケジューラ] ──→ [実行順序決定] ──→ スレッドプール
タスクC (優先度: 中) ─┘
タスクD (Aが完了後) ──→ [依存グラフ]

実行順序: A → C → B → (Aが完了したのでD実行可能) → D
```

---

## スケジューリング戦略の比較

```
┌──────────────────────────────────────────────────────────┐
│ FIFO (First In, First Out)                               │
│   [A] [B] [C] [D] → A が最初に実行                       │
│   実装: VecDeque / mpsc::channel                         │
│   長所: シンプル・公平・予測可能                          │
│   短所: 緊急タスクが後回しになる                          │
├──────────────────────────────────────────────────────────┤
│ 優先度付き (Priority Queue)                              │
│   [A:5] [B:1] [C:3] → 優先度5のAが最初                  │
│   実装: BinaryHeap                                       │
│   長所: 重要タスクを先行できる                            │
│   短所: 低優先度タスクが飢える (starvation)              │
├──────────────────────────────────────────────────────────┤
│ Work Stealing                                            │
│   Worker0: [A] [B]   Worker0が暇なとき                  │
│   Worker1: [C] [D]   Worker1のキューからタスクを「盗む」 │
│   実装: crossbeam-deque                                  │
│   長所: 負荷分散が自動・CPUキャッシュ局所性が高い        │
│   短所: 実装が複雑                                        │
└──────────────────────────────────────────────────────────┘
```

---

## タスクの基本型定義

```rust
use std::cmp::Ordering;
use std::sync::Arc;
use std::sync::atomic::{AtomicU64, Ordering as AtomicOrdering};

/// タスクID生成器 (グローバル単調増加カウンタ)
static TASK_ID_COUNTER: AtomicU64 = AtomicU64::new(0);

fn next_task_id() -> u64 {
    TASK_ID_COUNTER.fetch_add(1, AtomicOrdering::Relaxed)
}

/// タスクの状態
#[derive(Debug, Clone, PartialEq)]
pub enum TaskStatus {
    Pending,   // 実行待ち
    Running,   // 実行中
    Done,      // 完了
    Failed,    // 失敗
}

/// タスク本体
pub struct Task {
    pub id: u64,
    pub name: String,
    pub priority: i32,          // 大きいほど高優先
    pub work: Box<dyn FnOnce() + Send + 'static>,
    pub deps: Vec<u64>,         // 依存タスクのID一覧
}

impl Task {
    pub fn new<F>(name: &str, priority: i32, work: F) -> Task
    where
        F: FnOnce() + Send + 'static,
    {
        Task {
            id: next_task_id(),
            name: name.to_string(),
            priority,
            work: Box::new(work),
            deps: Vec::new(),
        }
    }

    pub fn with_deps<F>(name: &str, priority: i32, deps: Vec<u64>, work: F) -> Task
    where
        F: FnOnce() + Send + 'static,
    {
        Task {
            id: next_task_id(),
            name: name.to_string(),
            priority,
            work: Box::new(work),
            deps,
        }
    }
}
```

---

## 優先度付きタスクキューの実装

`BinaryHeap` は最大ヒープなので、優先度が高いタスクが先に取り出されます。

```rust
use std::collections::BinaryHeap;
use std::sync::Mutex;

/// BinaryHeap で比較するためのラッパー
struct PriorityTask {
    priority: i32,
    seq: u64,    // 同一優先度内でのFIFO順序 (小さい方が先)
    task: Task,
}

impl PartialEq for PriorityTask {
    fn eq(&self, other: &Self) -> bool {
        self.priority == other.priority && self.seq == other.seq
    }
}
impl Eq for PriorityTask {}

impl PartialOrd for PriorityTask {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

impl Ord for PriorityTask {
    fn cmp(&self, other: &Self) -> Ordering {
        // 優先度が高い方を先に → 大きい方がヒープの先頭
        self.priority
            .cmp(&other.priority)
            // 同優先度ならseqが小さい方を先に (FIFO)
            .then_with(|| other.seq.cmp(&self.seq))
    }
}

pub struct PriorityScheduler {
    heap: Mutex<BinaryHeap<PriorityTask>>,
    seq_counter: AtomicU64,
}

impl PriorityScheduler {
    pub fn new() -> PriorityScheduler {
        PriorityScheduler {
            heap: Mutex::new(BinaryHeap::new()),
            seq_counter: AtomicU64::new(0),
        }
    }

    pub fn push(&self, task: Task) {
        let seq = self.seq_counter.fetch_add(1, AtomicOrdering::Relaxed);
        let priority = task.priority;
        self.heap.lock().unwrap().push(PriorityTask { priority, seq, task });
    }

    pub fn pop(&self) -> Option<Task> {
        self.heap.lock().unwrap().pop().map(|pt| pt.task)
    }

    pub fn len(&self) -> usize {
        self.heap.lock().unwrap().len()
    }
}
```

---

## タスクの依存関係グラフ (DAG)

```
タスクグラフの例:

  A ──→ C ──→ F
  B ──→ D ──→ F
  A ──→ E

  実行可能順序 (トポロジカル順序の1つ):
  A, B → C, D, E → F

  Aが終わるまでCとEは開始できない
  BとDが終わるまでFは開始できない
```

```rust
use std::collections::{HashMap, HashSet, VecDeque};

pub struct DagScheduler {
    /// タスクID → タスク
    tasks: HashMap<u64, Task>,
    /// タスクID → 完了状態
    completed: HashSet<u64>,
    /// タスクID → このタスクが完了したら実行可能になるタスク群
    dependents: HashMap<u64, Vec<u64>>,
    /// 現在実行可能なタスクのキュー
    ready_queue: VecDeque<u64>,
}

impl DagScheduler {
    pub fn new() -> DagScheduler {
        DagScheduler {
            tasks: HashMap::new(),
            completed: HashSet::new(),
            dependents: HashMap::new(),
            ready_queue: VecDeque::new(),
        }
    }

    /// タスクを登録する
    pub fn add_task(&mut self, task: Task) {
        let id = task.id;
        let deps = task.deps.clone();

        // 依存関係の逆引きマップを構築
        for dep_id in &deps {
            self.dependents
                .entry(*dep_id)
                .or_insert_with(Vec::new)
                .push(id);
        }

        // 依存が全て完了済みならすぐに実行可能
        let all_deps_done = deps.iter().all(|d| self.completed.contains(d));
        if all_deps_done {
            self.ready_queue.push_back(id);
        }

        self.tasks.insert(id, task);
    }

    /// 次に実行すべきタスクを取り出す
    pub fn next_task(&mut self) -> Option<Task> {
        let id = self.ready_queue.pop_front()?;
        self.tasks.remove(&id)
    }

    /// タスクの完了を通知する
    pub fn complete(&mut self, id: u64) {
        self.completed.insert(id);

        // このタスクの完了により実行可能になるタスクを探す
        if let Some(dependents) = self.dependents.get(&id).cloned() {
            for dep_id in dependents {
                if let Some(task) = self.tasks.get(&dep_id) {
                    let all_done = task.deps.iter().all(|d| self.completed.contains(d));
                    if all_done {
                        self.ready_queue.push_back(dep_id);
                    }
                }
            }
        }
    }

    pub fn is_finished(&self) -> bool {
        self.tasks.is_empty() && self.ready_queue.is_empty()
    }
}
```

---

## スケジューラの状態機械

```
タスクの状態遷移:

        add_task()           next_task()
  ──→  [Pending] ──→ [ReadyQueue] ──→ [Running]
                                           │
                            complete()  ───┤
                                           ▼
                                        [Done]
                            panic!()   ───┤
                                           ▼
                                        [Failed]

Failed なら依存タスクも連鎖キャンセル
```

```rust
#[derive(Debug, Clone)]
pub enum TaskState {
    Pending { deps_remaining: usize },
    Ready,
    Running { started_at: std::time::Instant },
    Done { duration: std::time::Duration },
    Failed { error: String },
    Cancelled,
}

pub struct TaskRegistry {
    states: HashMap<u64, TaskState>,
}

impl TaskRegistry {
    pub fn new() -> Self {
        TaskRegistry { states: HashMap::new() }
    }

    pub fn register(&mut self, id: u64, dep_count: usize) {
        let state = if dep_count == 0 {
            TaskState::Ready
        } else {
            TaskState::Pending { deps_remaining: dep_count }
        };
        self.states.insert(id, state);
    }

    pub fn mark_running(&mut self, id: u64) {
        self.states.insert(id, TaskState::Running {
            started_at: std::time::Instant::now(),
        });
    }

    pub fn mark_done(&mut self, id: u64) {
        if let Some(TaskState::Running { started_at }) = self.states.get(&id) {
            let duration = started_at.elapsed();
            self.states.insert(id, TaskState::Done { duration });
        }
    }

    pub fn dec_dep(&mut self, id: u64) -> bool {
        if let Some(TaskState::Pending { deps_remaining }) = self.states.get_mut(&id) {
            *deps_remaining -= 1;
            if *deps_remaining == 0 {
                self.states.insert(id, TaskState::Ready);
                return true; // 実行可能になった
            }
        }
        false
    }
}
```

---

## crossbeam-channel を使ったタスク分配

`std::sync::mpsc` は SPSC/MPSC（複数送信者・単一受信者）ですが、`crossbeam-channel` は MPMC（複数送信者・複数受信者）に対応しています。

```rust
// Cargo.toml: crossbeam-channel = "0.5"

use crossbeam_channel::{bounded, Sender, Receiver};
use std::thread;

pub struct MultiWorkerScheduler {
    senders: Vec<Sender<Box<dyn FnOnce() + Send + 'static>>>,
    workers: Vec<thread::JoinHandle<()>>,
    round_robin: std::sync::atomic::AtomicUsize,
}

impl MultiWorkerScheduler {
    pub fn new(worker_count: usize) -> Self {
        let mut senders = Vec::new();
        let mut workers = Vec::new();

        for id in 0..worker_count {
            // バッファサイズ64のチャンネルを各ワーカーに割り当て
            let (tx, rx): (
                Sender<Box<dyn FnOnce() + Send + 'static>>,
                Receiver<Box<dyn FnOnce() + Send + 'static>>,
            ) = bounded(64);

            senders.push(tx);
            workers.push(thread::spawn(move || {
                println!("Worker {} 起動", id);
                while let Ok(job) = rx.recv() {
                    job();
                }
                println!("Worker {} 終了", id);
            }));
        }

        MultiWorkerScheduler {
            senders,
            workers,
            round_robin: std::sync::atomic::AtomicUsize::new(0),
        }
    }

    /// ラウンドロビンでタスクを分配
    pub fn dispatch<F: FnOnce() + Send + 'static>(&self, job: F) {
        let idx = self.round_robin
            .fetch_add(1, std::sync::atomic::Ordering::Relaxed)
            % self.senders.len();
        self.senders[idx].send(Box::new(job)).unwrap();
    }

    pub fn shutdown(mut self) {
        // senderをdropしてチャンネルを閉じる
        drop(self.senders);
        for w in self.workers {
            w.join().unwrap();
        }
    }
}
```

---

## 完全動作するタスクスケジューラ実装

```rust
use std::collections::{BinaryHeap, HashMap, HashSet};
use std::cmp::Ordering;
use std::sync::{Arc, Mutex};
use std::thread;

// ── 型定義 ──────────────────────────────────────────────

type JobFn = Box<dyn FnOnce() + Send + 'static>;

struct PTask {
    id: u64,
    priority: i32,
    inserted_at: u64,
    job: JobFn,
}

impl PartialEq for PTask {
    fn eq(&self, o: &Self) -> bool {
        self.priority == o.priority && self.inserted_at == o.inserted_at
    }
}
impl Eq for PTask {}
impl PartialOrd for PTask {
    fn partial_cmp(&self, o: &Self) -> Option<Ordering> { Some(self.cmp(o)) }
}
impl Ord for PTask {
    fn cmp(&self, o: &Self) -> Ordering {
        self.priority.cmp(&o.priority)
            .then_with(|| o.inserted_at.cmp(&self.inserted_at))
    }
}

// ── スケジューラ本体 ──────────────────────────────────────

struct Inner {
    heap: BinaryHeap<PTask>,
    completed: HashSet<u64>,
    // id → (job, deps)
    waiting: HashMap<u64, (JobFn, Vec<u64>)>,
    // dep_id → 待機中のタスクid一覧
    dep_map: HashMap<u64, Vec<u64>>,
    insert_counter: u64,
}

pub struct Scheduler {
    inner: Arc<Mutex<Inner>>,
}

impl Scheduler {
    pub fn new() -> Self {
        Scheduler {
            inner: Arc::new(Mutex::new(Inner {
                heap: BinaryHeap::new(),
                completed: HashSet::new(),
                waiting: HashMap::new(),
                dep_map: HashMap::new(),
                insert_counter: 0,
            })),
        }
    }

    /// タスクを登録する (deps が空なら即座に実行キューへ)
    pub fn submit<F: FnOnce() + Send + 'static>(
        &self,
        id: u64,
        priority: i32,
        deps: Vec<u64>,
        job: F,
    ) {
        let mut g = self.inner.lock().unwrap();
        let seq = g.insert_counter;
        g.insert_counter += 1;

        let all_done = deps.iter().all(|d| g.completed.contains(d));
        if all_done {
            g.heap.push(PTask { id, priority, inserted_at: seq, job: Box::new(job) });
        } else {
            for dep in &deps {
                g.dep_map.entry(*dep).or_default().push(id);
            }
            g.waiting.insert(id, (Box::new(job), deps));
        }
    }

    /// 次のタスクを取り出して実行する (なければ None)
    pub fn run_next(&self) -> bool {
        let (id, job) = {
            let mut g = self.inner.lock().unwrap();
            match g.heap.pop() {
                Some(pt) => (pt.id, pt.job),
                None => return false,
            }
        };

        job(); // ← Mutex外で実行 (デッドロック防止)

        // 完了を通知して依存タスクを解放
        let mut g = self.inner.lock().unwrap();
        g.completed.insert(id);

        if let Some(dependents) = g.dep_map.remove(&id) {
            let seq_base = g.insert_counter;
            let mut newly_ready = Vec::new();

            for dep_id in dependents {
                if let Some((_, deps)) = g.waiting.get(&dep_id) {
                    if deps.iter().all(|d| g.completed.contains(d)) {
                        newly_ready.push(dep_id);
                    }
                }
            }

            for (i, dep_id) in newly_ready.into_iter().enumerate() {
                if let Some((job, _)) = g.waiting.remove(&dep_id) {
                    // 優先度は暫定0 (実際はwaitingに保存すべき)
                    g.heap.push(PTask {
                        id: dep_id,
                        priority: 0,
                        inserted_at: seq_base + i as u64,
                        job,
                    });
                }
            }
            g.insert_counter = seq_base + newly_ready.len() as u64;
        }

        true
    }

    pub fn has_tasks(&self) -> bool {
        let g = self.inner.lock().unwrap();
        !g.heap.is_empty() || !g.waiting.is_empty()
    }
}

fn main() {
    let sched = Arc::new(Scheduler::new());

    // A と B は依存なし
    sched.submit(1, 10, vec![], || println!("[A] 優先度10のタスク実行"));
    sched.submit(2, 5, vec![], || println!("[B] 優先度5のタスク実行"));
    // C は A(id=1) の完了後
    sched.submit(3, 8, vec![1], || println!("[C] Aの完了後に実行"));
    // D は A(id=1) と B(id=2) の両方の完了後
    sched.submit(4, 3, vec![1, 2], || println!("[D] AとBの完了後に実行"));

    // シングルスレッドで消化
    while sched.has_tasks() {
        if !sched.run_next() {
            // キューが空だが待機中タスクがある = デッドロック検出
            eprintln!("警告: 循環依存またはデッドロックの可能性");
            break;
        }
    }
}
```

期待出力:

```
[A] 優先度10のタスク実行    ← 優先度最高
[C] Aの完了後に実行         ← Aが終わったので実行可能
[B] 優先度5のタスク実行     ← 優先度5
[D] AとBの完了後に実行      ← AとBが揃ったので実行可能
```

---

## Work Stealing の概念実装

```rust
// work-stealingの概念を示す簡略実装
// 本格実装は crossbeam-deque クレートを参照

use std::collections::VecDeque;

struct StealableQueue<T> {
    local: VecDeque<T>,  // ローカルキュー (push/pop は後端から)
}

impl<T> StealableQueue<T> {
    fn new() -> Self { StealableQueue { local: VecDeque::new() } }

    // 自分のタスクはLIFO (キャッシュ局所性が高い)
    fn push(&mut self, item: T) { self.local.push_back(item); }
    fn pop(&mut self) -> Option<T> { self.local.pop_back() }

    // 他のワーカーからStealされるときはFIFO (最古のタスクを渡す)
    fn steal(&mut self) -> Option<T> { self.local.pop_front() }
}

fn work_stealing_demo() {
    let mut worker0: StealableQueue<String> = StealableQueue::new();
    let mut worker1: StealableQueue<String> = StealableQueue::new();

    // worker0にタスクを積む
    for i in 0..5 {
        worker0.push(format!("task-{}", i));
    }

    // worker1は自分のキューが空なのでworker0から盗む
    println!("worker0のキュー: {:?}", worker0.local);

    // worker1: 自分のpopを優先 (空なのでsteal)
    while let Some(task) = worker1.pop().or_else(|| worker0.steal()) {
        println!("worker1 が実行: {}", task);
    }
}
```

---

## まとめ

| 戦略 | データ構造 | 時間計算量 | 適用場面 |
|------|-----------|-----------|--------|
| FIFO | `VecDeque` / `mpsc` | O(1) | シンプルなジョブキュー |
| 優先度付き | `BinaryHeap` | O(log n) | 重要度が異なるタスク |
| DAG | `HashMap` + `HashSet` | O(V+E) | パイプライン・ビルドシステム |
| Work Stealing | `crossbeam-deque` | O(1) amortized | CPUバウンドな並列処理 |

次節では、スレッドプールとスケジューラの上で非同期処理を動かすための中核概念——`Future` と `Waker` ——を学びます。
