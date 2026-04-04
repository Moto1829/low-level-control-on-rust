# 04: グリーンスレッド（コルーチン）の実装

## グリーンスレッドとOSスレッドの違い

```
OSスレッド:
  ┌─────────────────────────────────────────────────────┐
  │  カーネル空間                                        │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
  │  │ Thread0  │  │ Thread1  │  │ Thread2  │          │
  │  └──────────┘  └──────────┘  └──────────┘          │
  │  コンテキストスイッチ: システムコール + レジスタ保存 │
  │  スタックサイズ: 8MB (デフォルト)                   │
  │  生成コスト: ~数百マイクロ秒                        │
  └─────────────────────────────────────────────────────┘

グリーンスレッド (ユーザー空間スレッド):
  ┌─────────────────────────────────────────────────────┐
  │  ユーザー空間                                        │
  │  ┌──────────────────────────────────────────────┐  │
  │  │  Scheduler (ラウンドロビン等)                │  │
  │  │  ┌────────┐  ┌────────┐  ┌────────┐        │  │
  │  │  │ GreenT0│  │ GreenT1│  │ GreenT2│        │  │
  │  │  └────────┘  └────────┘  └────────┘        │  │
  │  └──────────────────────────────────────────────┘  │
  │  コンテキストスイッチ: レジスタ保存のみ (数ns)     │
  │  スタックサイズ: 8KB〜数百KB (設定可能)            │
  │  生成コスト: ~数マイクロ秒                         │
  └─────────────────────────────────────────────────────┘

OSスレッド1本の上で多数のグリーンスレッドをスケジュール
```

---

## コンテキストスイッチの仕組み

スレッドの切り替えとは「スタックポインタとレジスタを別のスレッドのものに差し替えること」です。

```
コンテキストスイッチの流れ (x86-64):

Thread A が実行中:
  rsp → [A's stack top]
  rbp → [A's stack frame]
  rip → [現在の命令]

switch(A → B) を呼ぶ:
  1. 呼び出し規約で保存すべきレジスタをA's stackにpush
     (rbx, r12, r13, r14, r15, rbp, rsp)
  2. Aの rsp を A.context.rsp に保存
  3. B.context.rsp を rsp にロード
  4. Bの stackからレジスタをpop
  5. ret → BがSwitchを呼んだ場所から再開

                Thread A                   Thread B
                  │                          │
            switch(A→B)                      │
                  │                          │
    [A's regs push]                          │
    [A->ctx.rsp = rsp]                       │
    [rsp = B->ctx.rsp]                       │
                  └──────────────────────────┘
                                        pop B's regs
                                        ret to B's saved rip
```

---

## x86-64 用コンテキスト構造体

```rust
// x86-64 System V AMD64 ABI の呼び出し規約:
// callee-saved registers: rbx, rbp, r12, r13, r14, r15
// + rsp (スタックポインタ)

#[repr(C)]
pub struct ThreadContext {
    pub rsp: u64,  // スタックポインタ — 必ず最初
    pub r15: u64,
    pub r14: u64,
    pub r13: u64,
    pub r12: u64,
    pub rbx: u64,
    pub rbp: u64,
}

impl ThreadContext {
    pub fn new() -> Self {
        ThreadContext {
            rsp: 0, r15: 0, r14: 0, r13: 0,
            r12: 0, rbx: 0, rbp: 0,
        }
    }
}
```

---

## `asm!` を使ったスタック切り替え

```rust
use std::arch::asm;

/// コンテキストスイッチ: old の状態を保存し、new に切り替える
///
/// # Safety
/// - `old` と `new` は有効な `ThreadContext` を指すこと
/// - `new` のスタックは正しく初期化されていること
/// - x86-64 Linux/macOS (System V AMD64 ABI) 専用
#[cfg(target_arch = "x86_64")]
pub unsafe fn switch(old: *mut ThreadContext, new: *const ThreadContext) {
    asm!(
        // 現在のレジスタを old に保存
        "mov [{old} + 0x00], rsp",
        "mov [{old} + 0x08], r15",
        "mov [{old} + 0x10], r14",
        "mov [{old} + 0x18], r13",
        "mov [{old} + 0x20], r12",
        "mov [{old} + 0x28], rbx",
        "mov [{old} + 0x30], rbp",
        // new のレジスタをロード
        "mov rsp, [{new} + 0x00]",
        "mov r15, [{new} + 0x08]",
        "mov r14, [{new} + 0x10]",
        "mov r13, [{new} + 0x18]",
        "mov r12, [{new} + 0x20]",
        "mov rbx, [{new} + 0x28]",
        "mov rbp, [{new} + 0x30]",
        // ret: new のスタックトップに積まれたアドレスへジャンプ
        "ret",
        old = in(reg) old,
        new = in(reg) new,
        // callee-saved を手動で操作するので clobber 宣言
        options(nostack),
    );
}
```

> **注意**: このコードは x86-64 Linux/macOS 専用です。Windows は異なるABI (rcx, rdx, r8, r9 が引数レジスタ、callee-savedも異なる) のため別実装が必要です。

---

## グリーンスレッドのスタック確保

```rust
use std::alloc::{alloc, dealloc, Layout};

const STACK_SIZE: usize = 1024 * 1024; // 1MB
const PAGE_SIZE: usize = 4096;

pub struct Stack {
    ptr: *mut u8,
    size: usize,
}

impl Stack {
    /// スタックを確保してガードページを設定する
    pub fn new(size: usize) -> Stack {
        assert!(size >= PAGE_SIZE * 2, "スタックが小さすぎます");
        assert!(size % PAGE_SIZE == 0, "スタックサイズはページサイズの倍数");

        unsafe {
            let layout = Layout::from_size_align(size, PAGE_SIZE).unwrap();
            let ptr = alloc(layout);
            if ptr.is_null() {
                panic!("スタックのアロケーションに失敗");
            }

            // ガードページの設定 (スタック底面 = アドレスが低い方)
            // mprotect で最初のページをアクセス不可にする
            #[cfg(unix)]
            {
                let ret = libc::mprotect(ptr as *mut libc::c_void, PAGE_SIZE, libc::PROT_NONE);
                if ret != 0 {
                    panic!("mprotect (guard page) に失敗");
                }
            }

            Stack { ptr, size }
        }
    }

    /// スタックのトップアドレス (高位アドレス) を返す
    /// x86-64 はスタックが下に向かって伸びるため、高位アドレスがトップ
    pub fn top(&self) -> *mut u8 {
        unsafe { self.ptr.add(self.size) }
    }
}

impl Drop for Stack {
    fn drop(&mut self) {
        unsafe {
            // ガードページの保護を解除してから解放
            #[cfg(unix)]
            libc::mprotect(
                self.ptr as *mut libc::c_void,
                PAGE_SIZE,
                libc::PROT_READ | libc::PROT_WRITE,
            );

            let layout = Layout::from_size_align(self.size, PAGE_SIZE).unwrap();
            dealloc(self.ptr, layout);
        }
    }
}
```

---

## グリーンスレッドの初期化

新しいグリーンスレッドが最初にどの関数を実行するかをスタックに設定します。

```rust
/// スレッド実行関数の型
type ThreadFn = fn();

/// グリーンスレッドのスタックを初期化する
///
/// スタックを以下のレイアウトに設定:
///
///  高位アドレス (スタックトップ)
///  ┌─────────────────┐
///  │  thread_start   │  ← retが返るアドレス (エントリポイント)
///  ├─────────────────┤  ← rsp はここを指す
///  │  (padding)      │  16バイトアライメント
///  └─────────────────┘
///  低位アドレス (ガードページ)
///
pub unsafe fn init_stack(stack: &Stack, f: ThreadFn) -> ThreadContext {
    let stack_top = stack.top() as usize;

    // x86-64: 関数呼び出し時に rsp は 16バイトアライメント
    // ret命令の前は8バイトオフセット (return addressがpushされるため)
    let rsp = (stack_top - 8) as *mut u64;

    // スレッドのエントリポイントをスタックトップに配置
    // switch() の最後の ret がこのアドレスにジャンプする
    *rsp = f as u64;

    ThreadContext {
        rsp: rsp as u64,
        r15: 0, r14: 0, r13: 0, r12: 0,
        rbx: 0, rbp: 0,
    }
}
```

---

## グリーンスレッドの状態と管理

```rust
#[derive(Debug, PartialEq)]
enum GreenState {
    Ready,    // 実行可能
    Running,  // 実行中
    Finished, // 完了
}

pub struct GreenThread {
    id: usize,
    state: GreenState,
    context: ThreadContext,
    stack: Option<Stack>,
}

impl GreenThread {
    fn new_main() -> GreenThread {
        // メインスレッド用: スタックはOSが管理するので不要
        GreenThread {
            id: 0,
            state: GreenState::Running,
            context: ThreadContext::new(),
            stack: None,
        }
    }

    fn new(id: usize, f: ThreadFn) -> GreenThread {
        let stack = Stack::new(STACK_SIZE);
        let context = unsafe { init_stack(&stack, f) };
        GreenThread {
            id,
            state: GreenState::Ready,
            context,
            stack: Some(stack),
        }
    }
}
```

---

## ラウンドロビンスケジューラの実装

```rust
use std::cell::UnsafeCell;

// スケジューラはスレッドローカルに保持 (マルチスレッドで使わない)
thread_local! {
    static SCHEDULER: UnsafeCell<Option<Scheduler>> = UnsafeCell::new(None);
}

pub struct Scheduler {
    threads: Vec<GreenThread>,
    current: usize,
}

impl Scheduler {
    fn new() -> Scheduler {
        // スレッド0はメインスレッド
        let main_thread = GreenThread::new_main();
        Scheduler {
            threads: vec![main_thread],
            current: 0,
        }
    }

    /// 新しいグリーンスレッドをスポーンする
    fn spawn(&mut self, f: ThreadFn) {
        let id = self.threads.len();
        self.threads.push(GreenThread::new(id, f));
        println!("グリーンスレッド {} を生成しました", id);
    }

    /// 次のReady状態スレッドにスイッチする (ラウンドロビン)
    fn yield_now(&mut self) {
        let old = self.current;

        // 次のReady/Runningスレッドを探す
        let next = (old + 1..self.threads.len())
            .chain(0..old)
            .find(|&i| {
                matches!(self.threads[i].state, GreenState::Ready | GreenState::Running)
            });

        match next {
            None => {
                // 全スレッドがFinishedならメインに戻る
                if self.current != 0 {
                    self.switch_to(0);
                }
            }
            Some(next_id) => {
                if next_id != old {
                    self.switch_to(next_id);
                }
            }
        }
    }

    fn switch_to(&mut self, next: usize) {
        let old = self.current;
        self.current = next;

        if self.threads[old].state == GreenState::Running {
            self.threads[old].state = GreenState::Ready;
        }
        self.threads[next].state = GreenState::Running;

        let old_ctx = &mut self.threads[old].context as *mut ThreadContext;
        let new_ctx = &self.threads[next].context as *const ThreadContext;

        unsafe { switch(old_ctx, new_ctx); }
    }

    /// 現在のスレッドを完了状態にして他へ切り替える
    fn thread_return(&mut self) {
        let current = self.current;
        println!("グリーンスレッド {} が完了", current);
        self.threads[current].state = GreenState::Finished;

        // 次のスレッドへ
        let next = self.threads.iter().enumerate()
            .find(|(i, t)| *i != current && t.state == GreenState::Ready)
            .map(|(i, _)| i)
            .unwrap_or(0);

        self.switch_to(next);
    }
}

/// グリーンスレッドから呼ぶ yield
pub fn green_yield() {
    SCHEDULER.with(|s| {
        let sched = unsafe { &mut *s.get() };
        if let Some(ref mut s) = sched {
            s.yield_now();
        }
    });
}
```

---

## 動作確認サンプル

```rust
fn thread_a() {
    println!("[A] ステップ1");
    green_yield();
    println!("[A] ステップ2");
    green_yield();
    println!("[A] ステップ3");
    // スレッドAが終了
    SCHEDULER.with(|s| {
        let sched = unsafe { &mut *s.get() };
        if let Some(ref mut s) = sched {
            s.thread_return();
        }
    });
}

fn thread_b() {
    println!("[B] ステップ1");
    green_yield();
    println!("[B] ステップ2");
    SCHEDULER.with(|s| {
        let sched = unsafe { &mut *s.get() };
        if let Some(ref mut s) = sched {
            s.thread_return();
        }
    });
}

fn main() {
    // スケジューラを初期化
    SCHEDULER.with(|s| {
        let sched = unsafe { &mut *s.get() };
        *sched = Some(Scheduler::new());
        if let Some(ref mut s) = sched {
            s.spawn(thread_a);
            s.spawn(thread_b);
        }
    });

    println!("[Main] スケジューラ開始");

    // メインがyieldして他スレッドを起動
    green_yield();
    green_yield();
    green_yield();

    println!("[Main] 全スレッド完了");
}
```

期待出力:

```
[Main] スケジューラ開始
[A] ステップ1
[B] ステップ1
[A] ステップ2
[B] ステップ2
[A] ステップ3
グリーンスレッド 1 が完了
グリーンスレッド 2 が完了
[Main] 全スレッド完了
```

---

## スタックオーバーフロー検出

```rust
// UNIX系ではシグナルハンドラでガードページへのアクセスを検知できる
#[cfg(unix)]
fn setup_stack_overflow_handler() {
    use std::mem;

    extern "C" fn sigsegv_handler(sig: libc::c_int) {
        eprintln!("スタックオーバーフローを検出しました! (シグナル {})", sig);
        eprintln!("グリーンスレッドのスタックが枯渇した可能性があります");
        std::process::exit(1);
    }

    unsafe {
        let mut sa: libc::sigaction = mem::zeroed();
        sa.sa_sigaction = sigsegv_handler as libc::sighandler_t;
        sa.sa_flags = libc::SA_SIGINFO;

        libc::sigaction(libc::SIGSEGV, &sa, std::ptr::null_mut());
        libc::sigaction(libc::SIGBUS, &sa, std::ptr::null_mut());
    }
}

// スタック使用量のチェック関数
fn check_stack_usage(stack: &Stack) -> usize {
    // スタックは高位アドレスから低位に伸びる
    // ガードページの直上から初期rspまでが使用済み領域
    unsafe {
        let guard_end = stack.ptr.add(PAGE_SIZE) as usize;
        let stack_top = stack.top() as usize;

        // スタックの使用済みバイト数を推定
        // (実際のrspを取得するにはアセンブリが必要)
        stack_top - guard_end // 最大使用可能サイズ
    }
}
```

---

## `async`/`await` との関係

グリーンスレッドと `async`/`await` の違いを明確にします。

```
グリーンスレッド:                  async/await:
┌─────────────────────────┐       ┌─────────────────────────┐
│ スタックをそのまま保存  │       │ ヒープ上の状態機械      │
│ どこでもyield可能       │       │ .await ポイントでのみ中断│
│ スタックを大量消費      │       │ スタック消費が少ない     │
│ コンテキストスイッチが  │       │ Futureのpollは安価       │
│ 比較的重い (数十ns)     │       │ (数ns)                   │
│                         │       │                          │
│ 例: Go言語のgoroutine   │       │ 例: tokio, async-std     │
└─────────────────────────┘       └─────────────────────────┘

どちらも「OSスレッドを1本使いまわして多数の並行処理を実現」する点は同じ
```

```rust
// Rust の async/await はグリーンスレッドではなく
// ゼロコスト抽象化で実装された状態機械

// グリーンスレッドの場合: スタック全体を保存
fn green_thread_example() {
    let x = 1;
    let y = 2;
    let z = x + y; // これらの変数はスタックに存在
    green_yield();  // スタック全体がコンテキストとして保存される
    println!("{}", z); // z はスタックから復元される
}

// async/await の場合: 必要な状態だけをヒープに保存
async fn async_example() {
    let x = 1;
    let y = 2;
    let z = x + y;
    some_future().await; // z だけが状態機械のフィールドとして保存
    println!("{}", z);
}
```

---

## まとめ

| 項目 | 説明 |
|------|------|
| コンテキストスイッチ | x86-64の callee-saved レジスタ + rsp を保存/復元 |
| スタック確保 | `alloc` でヒープから確保、`mprotect` でガードページ設定 |
| スタック初期化 | エントリ関数のアドレスをスタックトップに配置 |
| スケジューリング | `switch()` でrspを差し替えるだけ |
| オーバーフロー検出 | ガードページへのアクセスが SIGSEGV を発生させる |

次節では、これまで学んだ全コンポーネント（スレッドプール・スケジューラ・Future/Waker）を統合し、tokioを使わずに動く非同期ランタイムを完成させます。
