# アセンブリによる局所最適化

Rustのインラインアセンブリ（`std::arch::asm!`）は、コンパイラが生成できない
あるいは最適化しきれない特定のCPU命令を直接記述するために使う。
この章では、タイマー・CPU情報取得・スピンループ・プリフェッチ・キャッシュ制御といった
実践的なユースケースを通じてインラインアセンブリの活用方法を解説する。

---

## RDTSC による高精度タイマーの実装

`RDTSC`（Read Time-Stamp Counter）は、CPUのタイムスタンプカウンタ（TSC）を読み取る命令である。
TSCはCPUクロックに基づくカウンタで、`std::time::Instant` よりも低オーバーヘッドな計測が可能。

```rust
use std::arch::asm;

/// CPU タイムスタンプカウンタを読み取る
///
/// # Safety
/// x86_64 専用。TSC がモノトニックでない環境（古いマルチコアCPUなど）では
/// 正確な計測ができない場合がある。
#[cfg(target_arch = "x86_64")]
pub unsafe fn rdtsc() -> u64 {
    let lo: u32;
    let hi: u32;
    asm!(
        "rdtsc",
        out("eax") lo,
        out("edx") hi,
        // rdtsc は eax と edx を使う
        options(nostack, nomem, preserves_flags)
    );
    ((hi as u64) << 32) | (lo as u64)
}

/// RDTSCP（シリアライズ付き RDTSC）
///
/// RDTSC と異なり、前の命令が完了してからカウンタを読む（より正確）。
/// プロセッサIDも同時に取得できる。
#[cfg(target_arch = "x86_64")]
pub unsafe fn rdtscp() -> (u64, u32) {
    let lo: u32;
    let hi: u32;
    let aux: u32; // プロセッサID（TSC_AUX MSR の値）
    asm!(
        "rdtscp",
        out("eax") lo,
        out("edx") hi,
        out("ecx") aux,
        options(nostack, nomem, preserves_flags)
    );
    (((hi as u64) << 32) | (lo as u64), aux)
}

/// RDTSC を使ったベンチマーク計測
///
/// # 使用例
/// ```
/// let cycles = measure_cycles(|| {
///     // 計測したい処理
///     (0..1000u64).sum::<u64>()
/// });
/// println!("サイクル数: {}", cycles);
/// ```
#[cfg(target_arch = "x86_64")]
pub fn measure_cycles<F: FnOnce() -> ()>(f: F) -> u64 {
    // CPUID でシリアライズ（投機実行を防ぐ）してから計測開始
    unsafe {
        // CPUID で前の命令をフラッシュ
        let _: u32;
        asm!(
            "cpuid",
            inout("eax") 0u32 => _,
            out("ebx") _,
            out("ecx") _,
            out("edx") _,
            options(nostack, nomem)
        );
        let start = rdtsc();

        f();

        // 計測終了後も RDTSCP でシリアライズ
        let (end, _) = rdtscp();
        end.wrapping_sub(start)
    }
}

#[cfg(test)]
mod rdtsc_tests {
    use super::*;

    #[test]
    #[cfg(target_arch = "x86_64")]
    fn test_rdtsc_increases() {
        unsafe {
            let t1 = rdtsc();
            // 少し処理を挟む
            std::hint::black_box(0u64);
            let t2 = rdtsc();
            // TSC は単調増加するはず
            assert!(t2 >= t1, "TSC が減少した: t1={}, t2={}", t1, t2);
        }
    }

    #[test]
    #[cfg(target_arch = "x86_64")]
    fn test_measure_cycles_positive() {
        let cycles = measure_cycles(|| {
            std::hint::black_box((0..1000u64).sum::<u64>());
        });
        assert!(cycles > 0, "サイクル数が0以下: {}", cycles);
        println!("1000要素の合計: {} サイクル", cycles);
    }
}
```

---

## `CPUID` 命令でCPU情報を取得する

`CPUID` はCPUのアーキテクチャ情報・機能フラグ・キャッシュ情報などを取得する命令である。

```rust
use std::arch::asm;

/// CPUID の結果
#[derive(Debug, Clone, Copy)]
pub struct CpuidResult {
    pub eax: u32,
    pub ebx: u32,
    pub ecx: u32,
    pub edx: u32,
}

/// CPUID 命令を実行する
///
/// # 引数
/// - `leaf`: EAX に渡す葉番号
/// - `subleaf`: ECX に渡す副葉番号（不要なら 0）
#[cfg(target_arch = "x86_64")]
pub fn cpuid(leaf: u32, subleaf: u32) -> CpuidResult {
    let eax: u32;
    let ebx: u32;
    let ecx: u32;
    let edx: u32;
    unsafe {
        asm!(
            // rbx はCallee-saved register なので退避・復元が必要
            "push rbx",
            "cpuid",
            "mov {ebx_out:e}, ebx",
            "pop rbx",
            inout("eax") leaf => eax,
            inout("ecx") subleaf => ecx,
            ebx_out = out(reg) ebx,
            out("edx") edx,
            options(nostack, nomem)
        );
    }
    CpuidResult { eax, ebx, ecx, edx }
}

/// CPU ベンダー文字列を取得する（例: "GenuineIntel", "AuthenticAMD"）
#[cfg(target_arch = "x86_64")]
pub fn cpu_vendor() -> String {
    let result = cpuid(0, 0);
    // ベンダー文字列は EBX・EDX・ECX の順に格納される（各4バイト）
    let mut vendor = [0u8; 12];
    vendor[0..4].copy_from_slice(&result.ebx.to_le_bytes());
    vendor[4..8].copy_from_slice(&result.edx.to_le_bytes());
    vendor[8..12].copy_from_slice(&result.ecx.to_le_bytes());
    String::from_utf8_lossy(&vendor).to_string()
}

/// CPU が AVX2 をサポートしているか確認する
#[cfg(target_arch = "x86_64")]
pub fn has_avx2() -> bool {
    // CPUID leaf=7, subleaf=0 の EBX ビット5 が AVX2
    let result = cpuid(7, 0);
    (result.ebx >> 5) & 1 == 1
}

/// CPU が AVX-512F をサポートしているか確認する
#[cfg(target_arch = "x86_64")]
pub fn has_avx512f() -> bool {
    // CPUID leaf=7, subleaf=0 の EBX ビット16 が AVX-512F
    let result = cpuid(7, 0);
    (result.ebx >> 16) & 1 == 1
}

/// L1 データキャッシュのサイズを取得する（バイト単位）
#[cfg(target_arch = "x86_64")]
pub fn l1_data_cache_size() -> Option<u32> {
    // CPUID leaf=4 でキャッシュ情報を取得（Intel）
    for subleaf in 0..16 {
        let r = cpuid(4, subleaf);
        // bits[4:0]: キャッシュタイプ (1=データ, 2=命令, 3=統合)
        let cache_type = r.eax & 0x1F;
        if cache_type == 0 { break; }  // これ以上キャッシュはない
        if cache_type == 1 {           // データキャッシュ
            let level = (r.eax >> 5) & 0x7;
            if level == 1 {            // L1
                // サイズ = (ways+1) * (partitions+1) * (line_size+1) * (sets+1)
                let ways       = ((r.ebx >> 22) & 0x3FF) + 1;
                let partitions = ((r.ebx >> 12) & 0x3FF) + 1;
                let line_size  = (r.ebx & 0xFFF) + 1;
                let sets       = r.ecx + 1;
                return Some(ways * partitions * line_size * sets);
            }
        }
    }
    None
}

fn main() {
    #[cfg(target_arch = "x86_64")]
    {
        println!("CPU ベンダー: {}", cpu_vendor());
        println!("AVX2 サポート: {}", has_avx2());
        println!("AVX-512F サポート: {}", has_avx512f());
        if let Some(size) = l1_data_cache_size() {
            println!("L1 データキャッシュ: {} KB", size / 1024);
        }
    }
}
```

実行例:
```bash
cargo run --release
# CPU ベンダー: GenuineIntel
# AVX2 サポート: true
# AVX-512F サポート: false
# L1 データキャッシュ: 32 KB
```

---

## `PAUSE` 命令によるスピンループ最適化

スピンロック（busy-wait）の実装では、単純なループよりも `PAUSE` 命令を挿入すると
消費電力の低減とパイプラインの効率化につながる。
`PAUSE` はCPUに「スピンループ中である」ことを知らせ、不要なメモリオーダー違反の検出を抑制する。

```rust
use std::arch::asm;
use std::sync::atomic::{AtomicBool, Ordering};

/// CPU に対してスピンループ中であることを通知する
///
/// x86_64: PAUSE 命令を発行
/// その他: コンパイラバリアとして機能
#[inline(always)]
pub fn cpu_pause() {
    #[cfg(target_arch = "x86_64")]
    unsafe {
        asm!("pause", options(nostack, nomem, preserves_flags));
    }
    #[cfg(not(target_arch = "x86_64"))]
    {
        std::hint::spin_loop(); // 他アーキテクチャでの代替
    }
}

/// PAUSE を使ったスピンロックの実装
pub struct SpinLock {
    locked: AtomicBool,
}

impl SpinLock {
    pub const fn new() -> Self {
        Self {
            locked: AtomicBool::new(false),
        }
    }

    /// ロックを取得するまでスピン
    pub fn lock(&self) {
        // 指数バックオフ付きスピンループ
        let mut backoff = 1u32;
        loop {
            // まず楽観的に読み取り（キャッシュラインを共有モードで保持）
            if !self.locked.load(Ordering::Relaxed) {
                // CAS でロックを試みる
                if self.locked
                    .compare_exchange(false, true, Ordering::Acquire, Ordering::Relaxed)
                    .is_ok()
                {
                    return;
                }
            }

            // ロック取得失敗: PAUSE でバックオフ
            for _ in 0..backoff {
                cpu_pause();
            }

            // バックオフを指数的に増加（上限あり）
            if backoff < 64 {
                backoff *= 2;
            }
        }
    }

    /// ロックを解放する
    ///
    /// # Safety
    /// ロックを保持しているスレッドのみが呼び出すこと
    pub fn unlock(&self) {
        self.locked.store(false, Ordering::Release);
    }
}

/// PAUSE ベースの待機ループ
///
/// フラグが真になるまでスピン（イベント待機など）
pub fn spin_wait_until(flag: &AtomicBool) {
    while !flag.load(Ordering::Acquire) {
        cpu_pause();
    }
}

#[cfg(test)]
mod spinlock_tests {
    use super::*;
    use std::sync::Arc;
    use std::thread;

    #[test]
    fn test_spinlock_mutual_exclusion() {
        let lock = Arc::new(SpinLock::new());
        let counter = Arc::new(std::sync::atomic::AtomicU64::new(0));

        let mut handles = vec![];
        for _ in 0..4 {
            let lock = Arc::clone(&lock);
            let counter = Arc::clone(&counter);
            handles.push(thread::spawn(move || {
                for _ in 0..1000 {
                    lock.lock();
                    let val = counter.load(Ordering::Relaxed);
                    counter.store(val + 1, Ordering::Relaxed);
                    lock.unlock();
                }
            }));
        }

        for h in handles {
            h.join().unwrap();
        }

        assert_eq!(counter.load(Ordering::Relaxed), 4 * 1000);
    }
}
```

---

## `PREFETCHNTA`・`PREFETCHT0` によるプリフェッチ

プリフェッチ命令はメモリのデータをキャッシュに事前ロードし、
ロード遅延（メモリレイテンシ）を隠蔽する。

| 命令 | 動作 |
|------|------|
| `PREFETCHT0` | L1・L2・L3 キャッシュにプリフェッチ |
| `PREFETCHT1` | L2・L3 キャッシュにプリフェッチ |
| `PREFETCHT2` | L3 キャッシュにプリフェッチ |
| `PREFETCHNTA` | Non-Temporal（キャッシュを汚染しない）プリフェッチ |

```rust
use std::arch::asm;

/// アドレスを L1 キャッシュにプリフェッチする
///
/// ストリーミング処理など、繰り返しアクセスするデータに使う。
#[cfg(target_arch = "x86_64")]
#[inline(always)]
pub unsafe fn prefetch_t0<T>(ptr: *const T) {
    asm!(
        "prefetcht0 [{ptr}]",
        ptr = in(reg) ptr,
        options(nostack, nomem, preserves_flags)
    );
}

/// Non-Temporal プリフェッチ（キャッシュを汚染しない）
///
/// 一度だけ読むデータ（ストリーミング読み取り）に使う。
/// キャッシュを他のデータで汚染しないため、
/// 作業データのキャッシュ効率を維持できる。
#[cfg(target_arch = "x86_64")]
#[inline(always)]
pub unsafe fn prefetch_nta<T>(ptr: *const T) {
    asm!(
        "prefetchnta [{ptr}]",
        ptr = in(reg) ptr,
        options(nostack, nomem, preserves_flags)
    );
}

/// プリフェッチを使ったメモリコピー（大きなバッファ向け）
///
/// 次に処理するキャッシュラインを先行してプリフェッチすることで
/// メモリ帯域を効率的に活用する。
#[cfg(target_arch = "x86_64")]
pub fn memcpy_prefetch(dst: &mut [u8], src: &[u8]) {
    assert_eq!(dst.len(), src.len());

    const CACHE_LINE: usize = 64;
    const PREFETCH_DISTANCE: usize = 8; // 8キャッシュライン（512バイト）先をプリフェッチ

    let len = src.len();
    let prefetch_ahead = PREFETCH_DISTANCE * CACHE_LINE;

    unsafe {
        for i in (0..len).step_by(CACHE_LINE) {
            // 先読み: PREFETCH_DISTANCE キャッシュライン先のアドレスをプリフェッチ
            if i + prefetch_ahead < len {
                prefetch_t0(src.as_ptr().add(i + prefetch_ahead));
            }
            // 現在のキャッシュラインをコピー
            let copy_len = (CACHE_LINE).min(len - i);
            dst[i..i + copy_len].copy_from_slice(&src[i..i + copy_len]);
        }
    }
}

/// ストリーミング読み取り（読んだデータをキャッシュに残さない）
#[cfg(target_arch = "x86_64")]
pub fn streaming_sum(data: &[f32]) -> f32 {
    const CACHE_LINE: usize = 64;
    const F32_PER_LINE: usize = CACHE_LINE / 4; // 16個
    const PREFETCH_DISTANCE: usize = 4;

    let mut sum = 0.0f32;
    unsafe {
        for (i, chunk) in data.chunks(F32_PER_LINE).enumerate() {
            // 先のキャッシュラインをNTAプリフェッチ（一度きりの読み取り）
            let ahead = i + PREFETCH_DISTANCE;
            if ahead * F32_PER_LINE < data.len() {
                prefetch_nta(data.as_ptr().add(ahead * F32_PER_LINE));
            }
            for &x in chunk {
                sum += x;
            }
        }
    }
    sum
}
```

---

## `CLFLUSH` によるキャッシュライン無効化

`CLFLUSH` はキャッシュライン（通常64バイト）をキャッシュから追い出し、
メモリに書き戻す命令。永続メモリ（NVDIMM）への書き込み保証や、
キャッシュ関連のタイミング攻撃研究（Flush+Reloadなど）で使われる。

```rust
use std::arch::asm;

/// 指定アドレスのキャッシュラインを無効化・フラッシュする
///
/// # Safety
/// ptr は有効なメモリを指していること。
/// この命令の後、SFENCE でメモリオーダーを保証することが推奨される。
#[cfg(target_arch = "x86_64")]
#[inline(always)]
pub unsafe fn clflush<T>(ptr: *const T) {
    asm!(
        "clflush [{ptr}]",
        ptr = in(reg) ptr,
        options(nostack, preserves_flags)
    );
}

/// SFENCE: ストア命令のメモリオーダーを保証する
#[cfg(target_arch = "x86_64")]
#[inline(always)]
pub unsafe fn sfence() {
    asm!("sfence", options(nostack, nomem, preserves_flags));
}

/// 永続メモリへのデータ書き込みを保証する
///
/// 通常の write はキャッシュに留まる可能性があるが、
/// CLFLUSH + SFENCE でメモリへの書き込みを保証する。
#[cfg(target_arch = "x86_64")]
pub fn persist_data(data: &[u8]) {
    const CACHE_LINE: usize = 64;
    unsafe {
        // 各キャッシュラインをフラッシュ
        for i in (0..data.len()).step_by(CACHE_LINE) {
            clflush(data.as_ptr().add(i));
        }
        // 最後のキャッシュラインもフラッシュ
        if data.len() % CACHE_LINE != 0 {
            clflush(data.as_ptr().add(data.len() - 1));
        }
        // 全フラッシュが完了するまで待機
        sfence();
    }
}

/// CLFLUSH を使ったキャッシュアクセス時間計測（教育目的）
///
/// キャッシュヒット vs キャッシュミスのサイクル数を比較する。
#[cfg(target_arch = "x86_64")]
pub fn measure_cache_latency(data: &[u8]) -> (u64, u64) {
    assert!(!data.is_empty());

    unsafe {
        // 1. キャッシュにロード（アクセスして暖める）
        let _ = std::hint::black_box(data[0]);

        // キャッシュヒット時のアクセス時間を計測
        let t1 = crate::rdtsc();
        let _ = std::hint::black_box(std::ptr::read_volatile(&data[0]));
        let t2 = crate::rdtsc();
        let cached_latency = t2 - t1;

        // 2. CLFLUSH でキャッシュから追い出す
        clflush(data.as_ptr());
        asm!("mfence", options(nostack, nomem, preserves_flags)); // メモリバリア

        // キャッシュミス時のアクセス時間を計測
        let t3 = crate::rdtsc();
        let _ = std::hint::black_box(std::ptr::read_volatile(&data[0]));
        let t4 = crate::rdtsc();
        let uncached_latency = t4 - t3;

        (cached_latency, uncached_latency)
    }
}
```

---

## Rustコンパイラが生成したコードとのケーススタディ比較

### ケーススタディ: 整数の絶対値

コンパイラが最適化する例と、手動アセンブリによる実装を比較する。

```rust
/// コンパイラによる実装（最適化後は CMOV 命令になる）
pub fn abs_compiler(x: i64) -> i64 {
    x.abs()
}

/// 条件分岐による実装（最適化前のイメージ）
pub fn abs_branch(x: i64) -> i64 {
    if x < 0 { -x } else { x }
}

/// ビット操作による実装（分岐なし）
pub fn abs_bitwise(x: i64) -> i64 {
    let mask = x >> 63; // 負なら -1（全ビット1）、正なら 0
    (x ^ mask) - mask
}

/// インラインアセンブリによる実装（CMOV を明示的に使用）
#[cfg(target_arch = "x86_64")]
pub fn abs_asm(x: i64) -> i64 {
    let result: i64;
    unsafe {
        use std::arch::asm;
        asm!(
            "mov {tmp}, {x}",   // tmp = x
            "neg {tmp}",        // tmp = -x（CF=1 if x!=0, SF が設定される）
            "cmovl {tmp}, {x}", // if x < 0（元の x が負）: tmp = -x、else: tmp = x
            // 注: neg後のフラグで判断するのではなく、元の符号を使う
            x = in(reg) x,
            tmp = out(reg) result,
            options(pure, nomem, nostack, preserves_flags)
        );
    }
    // 上記は説明用。実際は compiler が i64::abs() を最適化する
    // より正確な実装:
    let mask = x >> 63;
    (x + mask) ^ mask
}
```

### ケーススタディ: ビットカウント（popcount）

```rust
/// Rust 標準の実装（コンパイラが POPCNT 命令に最適化する）
pub fn popcount_std(x: u64) -> u32 {
    x.count_ones()
}

/// インラインアセンブリで POPCNT を明示的に使う例
#[cfg(target_arch = "x86_64")]
pub fn popcount_asm(x: u64) -> u64 {
    let result: u64;
    unsafe {
        use std::arch::asm;
        asm!(
            "popcnt {result}, {x}",
            x = in(reg) x,
            result = out(reg) result,
            options(pure, nomem, nostack, preserves_flags)
        );
    }
    result
}

/// 比較: どちらが速いかをサイクル数で計測
#[cfg(target_arch = "x86_64")]
pub fn benchmark_popcount() {
    let test_values: Vec<u64> = (0..1000).collect();

    let cycles_std = measure_cycles(|| {
        let _: u32 = test_values.iter().map(|&x| x.count_ones()).sum();
        std::hint::black_box(());
    });

    let cycles_asm = measure_cycles(|| {
        let _: u64 = test_values.iter().map(|&x| popcount_asm(x)).sum();
        std::hint::black_box(());
    });

    println!("popcount_std: {} サイクル", cycles_std);
    println!("popcount_asm: {} サイクル", cycles_asm);
    println!("→ 多くの場合、コンパイラが生成するコードとほぼ同等");
}

fn main() {
    #[cfg(target_arch = "x86_64")]
    {
        // CPUID でCPU情報を表示
        println!("=== CPU情報 ===");
        println!("ベンダー: {}", cpu_vendor());
        println!("AVX2: {}", has_avx2());

        // キャッシュレイテンシ計測
        let data = vec![0u8; 1024];
        let (cached, uncached) = measure_cache_latency(&data);
        println!("\n=== キャッシュレイテンシ ===");
        println!("キャッシュヒット: {} サイクル", cached);
        println!("キャッシュミス:  {} サイクル", uncached);
        println!("比率: {:.1}x", uncached as f64 / cached as f64);

        // popcount 比較
        println!("\n=== popcount ベンチマーク ===");
        benchmark_popcount();
    }
}
```

アセンブリ生成の確認:

```bash
# リリースビルドでアセンブリを出力
RUSTFLAGS="-C target-cpu=native" cargo rustc --release -- --emit=asm

# popcnt 命令が使われているか確認
grep -n "popcnt\|bsf\|lzcnt" target/release/deps/*.s

# Compiler Explorer での確認
# https://godbolt.org に以下のオプションで貼り付け:
# -C opt-level=3 -C target-cpu=haswell
```

---

## まとめ

| 命令 | 用途 | Rustでの代替 |
|------|------|-------------|
| `RDTSC` | 高精度タイマー | `std::time::Instant`（低精度） |
| `CPUID` | CPU機能検出 | `is_x86_feature_detected!` マクロ |
| `PAUSE` | スピンループ最適化 | `std::hint::spin_loop()` |
| `PREFETCHT0` | キャッシュへの事前ロード | 自動プリフェッチに任せる |
| `PREFETCHNTA` | キャッシュ汚染なし読み取り | `_mm_prefetch`（std::arch） |
| `CLFLUSH` | キャッシュラインの無効化 | なし（永続メモリ等で必要） |

インラインアセンブリはコンパイラが生成するコードで不十分な場合の最終手段である。
まず `std::hint::spin_loop()`・`is_x86_feature_detected!`・`std::time::Instant` など
Rustの標準機能で解決できないか確認してから、インラインアセンブリを検討すること。
