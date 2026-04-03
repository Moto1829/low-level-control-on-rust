# グローバルアロケータとカスタムアロケータ

Rustはデフォルトでシステムアロケータ（Linux では `ptmalloc`、macOS では `libmalloc`、Windows では `HeapAlloc`）を使用します。
しかし `GlobalAlloc` トレイトと `#[global_allocator]` 属性を利用することで、アロケータを自由に差し替えることができます。
本章では仕組みの理解から、サードパーティアロケータの導入、バンプアロケータのゼロからの実装まで扱います。

---

## 1. `GlobalAlloc` トレイトの仕組み

`GlobalAlloc` は `std::alloc` モジュールで定義されているトレイトです。

```rust
use std::alloc::Layout;

pub unsafe trait GlobalAlloc {
    /// メモリを確保する。失敗時はヌルポインタを返す。
    unsafe fn alloc(&self, layout: Layout) -> *mut u8;

    /// `alloc` で確保したメモリを解放する。
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout);

    /// ゼロ初期化済みのメモリを確保する（デフォルト実装あり）。
    unsafe fn alloc_zeroed(&self, layout: Layout) -> *mut u8 {
        let size = layout.size();
        let ptr = self.alloc(layout);
        if !ptr.is_null() {
            std::ptr::write_bytes(ptr, 0, size);
        }
        ptr
    }

    /// 既存ブロックをサイズ変更する（デフォルト実装あり）。
    unsafe fn realloc(
        &self,
        ptr: *mut u8,
        layout: Layout,
        new_size: usize,
    ) -> *mut u8 {
        let new_layout = Layout::from_size_align_unchecked(new_size, layout.align());
        let new_ptr = self.alloc(new_layout);
        if !new_ptr.is_null() {
            let copy_size = layout.size().min(new_size);
            std::ptr::copy_nonoverlapping(ptr, new_ptr, copy_size);
            self.dealloc(ptr, layout);
        }
        new_ptr
    }
}
```

### `Layout` とは

`Layout` はアロケーション要求のサイズとアラインメントを保持する型です。

```rust
use std::alloc::Layout;

fn main() {
    // u64 のレイアウト: size=8, align=8
    let layout = Layout::new::<u64>();
    println!("size={}, align={}", layout.size(), layout.align());

    // 要素数 16 の u32 配列のレイアウト
    let array_layout = Layout::array::<u32>(16).unwrap();
    println!("array size={}, align={}", array_layout.size(), array_layout.align());
}
```

---

## 2. `#[global_allocator]` でデフォルトアロケータを切り替える

`#[global_allocator]` 属性を付けた静的変数が、プログラム全体のアロケータになります。
切り替えは `main.rs`（またはライブラリ crate のルートファイル）に1か所書くだけです。

```rust
use std::alloc::{GlobalAlloc, Layout, System};

// System アロケータをそのままラップして呼び出しを計測する例
struct CountingAllocator;

static ALLOC_COUNT: std::sync::atomic::AtomicUsize =
    std::sync::atomic::AtomicUsize::new(0);

unsafe impl GlobalAlloc for CountingAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        ALLOC_COUNT.fetch_add(1, std::sync::atomic::Ordering::Relaxed);
        System.alloc(layout)
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        System.dealloc(ptr, layout);
    }
}

#[global_allocator]
static A: CountingAllocator = CountingAllocator;

fn main() {
    let _v: Vec<i32> = (0..100).collect();
    let count = ALLOC_COUNT.load(std::sync::atomic::Ordering::Relaxed);
    println!("アロケーション回数: {}", count);
}
```

> **注意**: `#[global_allocator]` はクレート全体に1つしか設定できません。複数設定するとコンパイルエラーになります。

---

## 3. jemalloc（`tikv-jemallocator`）への切り替え

jemalloc はメモリの断片化を抑えつつ高スループットを実現するアロケータです。
`tikv-jemallocator` クレートを使うと簡単に導入できます。

### `Cargo.toml`

```toml
[dependencies]
tikv-jemallocator = "0.6"
```

### `main.rs`

```rust
#[cfg(not(target_env = "msvc"))]
use tikv_jemallocator::Jemalloc;

#[cfg(not(target_env = "msvc"))]
#[global_allocator]
static GLOBAL: Jemalloc = Jemalloc;

fn main() {
    // 何も変えなくてよい。Vec・Box・String がすべて jemalloc を使う。
    let mut v: Vec<u8> = Vec::with_capacity(1024 * 1024);
    v.extend_from_slice(&[0u8; 1024 * 1024]);
    println!("確保バイト数: {}", v.len());
}
```

### jemalloc が有利なケース

| ケース | 理由 |
|--------|------|
| 多スレッドでの頻繁なアロケーション | スレッドローカルキャッシュで競合を減らす |
| 長時間稼働するサーバー | アリーナ方式でメモリ断片化を抑制 |
| 多様なサイズのアロケーション | サイズクラス管理が効率的 |

---

## 4. mimalloc（`mimalloc` クレート）への切り替え

mimalloc は Microsoft Research が開発した高速アロケータです。
Windows/Linux/macOS で動作し、特にスモールオブジェクトの速度に優れます。

### `Cargo.toml`

```toml
[dependencies]
mimalloc = { version = "0.1", default-features = false }
```

### `main.rs`

```rust
use mimalloc::MiMalloc;

#[global_allocator]
static GLOBAL: MiMalloc = MiMalloc;

fn main() {
    // 小さなオブジェクトを大量に生成するワークロードで効果的
    let strings: Vec<String> = (0..10_000)
        .map(|i| format!("item-{}", i))
        .collect();
    println!("生成した文字列数: {}", strings.len());
}
```

### 主なアロケータ比較

| アロケータ | 特徴 | 推奨ユースケース |
|------------|------|----------------|
| System（既定）| OS 標準、移植性最高 | 汎用 |
| jemalloc | 断片化対策、プロファイリング機能付き | 長時間稼働サーバー |
| mimalloc | 小オブジェクト超高速、低メモリオーバーヘッド | マイクロサービス、CLI ツール |
| rpmalloc | スレッドローカル完全分離 | 超低レイテンシ要件 |

---

## 5. バンプアロケータをゼロから実装する

バンプアロケータ（Arena アロケータとも呼ぶ）は最もシンプルなアロケータです。
確保は「ポインタをずらす（バンプする）」だけで行い、個別の解放はできません。
組み込みや短命なスコープ処理に適しています。

```rust
use std::alloc::{GlobalAlloc, Layout};
use std::cell::UnsafeCell;
use std::sync::atomic::{AtomicUsize, Ordering};

/// 固定バッファ上のバンプアロケータ
pub struct BumpAllocator<const N: usize> {
    buf: UnsafeCell<[u8; N]>,
    /// 次の確保開始位置（バッファ先頭からのオフセット）
    offset: AtomicUsize,
}

// シングルスレッド専用なので手動で Sync を実装（マルチスレッドには不向き）
unsafe impl<const N: usize> Sync for BumpAllocator<N> {}

impl<const N: usize> BumpAllocator<N> {
    pub const fn new() -> Self {
        Self {
            buf: UnsafeCell::new([0u8; N]),
            offset: AtomicUsize::new(0),
        }
    }

    /// 確保済みバイト数を返す
    pub fn used(&self) -> usize {
        self.offset.load(Ordering::Relaxed)
    }

    /// オフセットをリセットして全領域を再利用可能にする
    ///
    /// # Safety
    /// 以前に確保したポインタがすべて使用済みであることを呼び出し元が保証する必要があります。
    pub unsafe fn reset(&self) {
        self.offset.store(0, Ordering::Relaxed);
    }
}

unsafe impl<const N: usize> GlobalAlloc for BumpAllocator<N> {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let buf_ptr = self.buf.get() as usize;

        loop {
            let current = self.offset.load(Ordering::Acquire);

            // アラインメントを満たすようにオフセットを切り上げる
            let aligned = align_up(buf_ptr + current, layout.align());
            let new_offset = aligned + layout.size() - buf_ptr;

            if new_offset > N {
                // バッファ不足: ヌルポインタを返す
                return std::ptr::null_mut();
            }

            // CAS でオフセットを更新（他スレッドが競合した場合はリトライ）
            if self
                .offset
                .compare_exchange(current, new_offset, Ordering::AcqRel, Ordering::Acquire)
                .is_ok()
            {
                return aligned as *mut u8;
            }
        }
    }

    unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
        // バンプアロケータは個別解放をサポートしない（no-op）
    }
}

/// `addr` を `align` の倍数に切り上げる（`align` は2の冪であること）
fn align_up(addr: usize, align: usize) -> usize {
    (addr + align - 1) & !(align - 1)
}

// ---- 使用例 ----

const HEAP_SIZE: usize = 64 * 1024; // 64 KiB

#[global_allocator]
static BUMP: BumpAllocator<HEAP_SIZE> = BumpAllocator::new();

fn main() {
    println!("バンプアロケータ開始 (容量 {} bytes)", HEAP_SIZE);

    let v: Vec<u32> = (0..100u32).collect();
    println!("Vec<u32> 確保後 used={} bytes", BUMP.used());

    let s = String::from("Hello, Bump Allocator!");
    println!("String 確保後 used={} bytes", BUMP.used());
    println!("{}", s);
    println!("合計: {} / {} bytes 使用", BUMP.used(), HEAP_SIZE);

    drop(v); // dealloc は no-op なので used は減らない
    println!("drop 後も used={} bytes（解放されない）", BUMP.used());
}
```

### バンプアロケータの特性

- **確保**: O(1) — アトミックな加算のみ
- **解放**: O(1) — no-op
- **リセット**: O(1) — オフセットをゼロに戻すだけ
- **制約**: 個別解放不可・バッファサイズ固定

---

## 6. アロケータの性能測定

### `criterion` を使ったマイクロベンチマーク

```toml
# Cargo.toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "alloc_bench"
harness = false
```

```rust
// benches/alloc_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn bench_vec_alloc(c: &mut Criterion) {
    c.bench_function("Vec<u8> 1MiB 確保と解放", |b| {
        b.iter(|| {
            let v: Vec<u8> = vec![black_box(0u8); 1024 * 1024];
            drop(v);
        })
    });
}

fn bench_many_small_alloc(c: &mut Criterion) {
    c.bench_function("String x 10000 確保と解放", |b| {
        b.iter(|| {
            let v: Vec<String> = (0..10_000)
                .map(|i| black_box(format!("item-{}", i)))
                .collect();
            drop(v);
        })
    });
}

criterion_group!(benches, bench_vec_alloc, bench_many_small_alloc);
criterion_main!(benches);
```

```bash
# アロケータを切り替えながら計測
cargo bench
```

### `std::alloc::GlobalAlloc` を使った手動計測

```rust
use std::alloc::{alloc, dealloc, Layout};
use std::time::Instant;

fn measure_alloc_dealloc(iterations: usize) {
    let layout = Layout::array::<u8>(4096).unwrap();
    let start = Instant::now();

    for _ in 0..iterations {
        unsafe {
            let ptr = alloc(layout);
            assert!(!ptr.is_null());
            dealloc(ptr, layout);
        }
    }

    let elapsed = start.elapsed();
    println!(
        "{} 回の alloc/dealloc: {:?} ({:.2} ns/op)",
        iterations,
        elapsed,
        elapsed.as_nanos() as f64 / iterations as f64
    );
}

fn main() {
    measure_alloc_dealloc(100_000);
}
```

### Valgrind / heaptrack による計測

```bash
# heaptrack でアロケーション全体を可視化する（Linux）
heaptrack ./target/release/my_program
heaptrack_gui heaptrack.my_program.*.zst

# DHAT（Valgrind）でヒープ使用の「目的別」分析
valgrind --tool=dhat ./target/release/my_program
```

---

## まとめ

| 操作 | 方法 |
|------|------|
| アロケータを切り替える | `#[global_allocator]` + `GlobalAlloc` 実装 |
| 高スループットが必要 | `tikv-jemallocator` or `mimalloc` |
| 低レイテンシ・組み込み | バンプアロケータを自作 |
| 性能測定 | `criterion` ベンチマーク or `heaptrack` |

カスタムアロケータは「どこに・どう確保されるか」を完全に制御する手段です。
本番投入前には必ずベンチマークと ASan（AddressSanitizer）で安全性を確認してください。
