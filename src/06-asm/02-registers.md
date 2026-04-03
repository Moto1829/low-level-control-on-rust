# 6.2 レジスタと呼び出し規約

## x86-64 レジスタの全体像

x86-64（AMD64）アーキテクチャには多種多様なレジスタがあります。インラインアセンブリを正しく書くためには、各レジスタの役割と制約を理解することが不可欠です。

```
x86-64 レジスタ体系

  ┌─────────────────────────────────────────────────────────────────┐
  │ 汎用レジスタ（General Purpose Registers）                        │
  │  64-bit  32-bit  16-bit  8-bit(lo)  8-bit(hi)                   │
  │  rax     eax     ax      al         ah                           │
  │  rbx     ebx     bx      bl         bh                           │
  │  rcx     ecx     cx      cl         ch                           │
  │  rdx     edx     dx      dl         dh                           │
  │  rsi     esi     si      sil        ─                            │
  │  rdi     edi     di      dil        ─                            │
  │  rsp     esp     sp      spl        ─   ← スタックポインタ       │
  │  rbp     ebp     bp      bpl        ─   ← フレームポインタ       │
  │  r8      r8d     r8w     r8b        ─   ← 新規（x86-64追加）    │
  │  r9      r9d     r9w     r9b        ─                            │
  │  r10     r10d    r10w    r10b       ─                            │
  │  r11     r11d    r11w    r11b       ─                            │
  │  r12     r12d    r12w    r12b       ─                            │
  │  r13     r13d    r13w    r13b       ─                            │
  │  r14     r14d    r14w    r14b       ─                            │
  │  r15     r15d    r15w    r15b       ─                            │
  ├─────────────────────────────────────────────────────────────────┤
  │ 特殊レジスタ                                                      │
  │  rip   ─ 命令ポインタ（プログラムカウンタ）                       │
  │  rflags ─ フラグレジスタ（CF, ZF, SF, OF, PF, DF 等）           │
  ├─────────────────────────────────────────────────────────────────┤
  │ SIMDレジスタ                                                      │
  │  xmm0〜xmm15   ─ 128-bit（SSE）                                  │
  │  ymm0〜ymm15   ─ 256-bit（AVX）xmm の上位拡張                   │
  │  zmm0〜zmm31   ─ 512-bit（AVX-512）                              │
  │  k0〜k7        ─ AVX-512 マスクレジスタ                          │
  ├─────────────────────────────────────────────────────────────────┤
  │ セグメントレジスタ                                                 │
  │  cs, ss, ds, es, fs, gs                                          │
  │  （現代のLinux/macOSではfs/gsのみ実用的）                         │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 汎用レジスタの詳細

### レジスタのサイズとエイリアス

x86-64では1つの物理レジスタを異なるサイズで参照できます。

```
64-bit レジスタ RAX の内部構造:

  ┌──────────────────────────────────────────────────────────────┐
  │ 63                           31            15      7        0│
  │                              │            │       │         │
  │◄─────────────── RAX (64-bit) ──────────────────────────────►│
  │                              │◄─── EAX (32-bit) ──────────►│
  │                              │            │◄── AX (16-bit)►│
  │                              │            │◄─AH►│◄──AL────►│
  └──────────────────────────────────────────────────────────────┘
```

重要な挙動: **32-bitへの書き込みは上位32bitをゼロクリアする**

```rust
use std::arch::asm;

fn register_aliasing() {
    let rax_val: u64;
    unsafe {
        asm!(
            "mov rax, 0xDEADBEEF_12345678",  // RAX に 64-bit 値を書く
            "mov eax, 0x0000FFFF",           // EAX に書くと上位32bitがゼロになる！
            out("rax") rax_val,
            options(nostack),
        );
    }
    // rax_val は 0x0000FFFF（上位32bitがゼロクリアされた）
    println!("rax = {:#x}", rax_val); // 0xffff
}
```

これとは対照的に、**16-bitや8-bitへの書き込みは上位ビットに影響を与えない**:

```rust
use std::arch::asm;

fn partial_write() {
    let rax_val: u64;
    unsafe {
        asm!(
            "mov rax, 0xDEADBEEF_CAFEBABE",
            "mov ax, 0x1234",  // 下位16bitのみ書き換え（上位48bitは不変）
            out("rax") rax_val,
            options(nostack),
        );
    }
    println!("rax = {:#x}", rax_val); // 0xdeadbeef_cafe1234
}
```

---

## 明示的レジスタ指定

特定のレジスタを名指しでオペランドに使うには、`in("rax")` のように文字列でレジスタ名を指定します。

```rust
use std::arch::asm;

fn explicit_rax() -> u64 {
    let result: u64;
    unsafe {
        asm!(
            "mov rax, 0xCAFEBABE",
            out("rax") result,       // rax を明示的に出力に使う
            options(nomem, nostack),
        );
    }
    result
}
```

### 明示的指定が必要な場面

特定の命令は特定のレジスタを暗黙的に使います。そのような命令では明示的に宣言する必要があります。

```rust
use std::arch::asm;

/// mul 命令: rax * src → rdx:rax（128-bit 積）
fn wide_multiply(a: u64, b: u64) -> (u64, u64) {
    let lo: u64;  // rax（積の下位64bit）
    let hi: u64;  // rdx（積の上位64bit）
    unsafe {
        asm!(
            "mul {b}",           // rax * b → rdx:rax
            b = in(reg) b,
            inout("rax") a => lo,  // rax が入力かつ出力（下位）
            out("rdx") hi,         // rdx が出力（上位）
            options(pure, nomem, nostack),
        );
    }
    (lo, hi)
}

fn main() {
    let a: u64 = 0xFFFF_FFFF_FFFF_FFFF;
    let b: u64 = 2;
    let (lo, hi) = wide_multiply(a, b);
    println!("lo = {:#x}", lo);  // 0xfffffffffffffffе
    println!("hi = {:#x}", hi);  // 0x1
}
```

### 明示的レジスタが必要な主な命令

```
┌────────────────┬────────────────────────────────────────────────┐
│ 命令           │ 暗黙的に使うレジスタ                            │
├────────────────┼────────────────────────────────────────────────┤
│ mul / imul     │ rax（入力）, rdx:rax（出力）                    │
│ div / idiv     │ rdx:rax（被除数入力）, rax（商）, rdx（余り）   │
│ push / pop     │ rsp（スタックポインタ）                         │
│ call / ret     │ rsp, rip                                        │
│ shl/shr n, cl  │ rcx（シフト量）                                 │
│ loop           │ rcx（カウンタ）                                  │
│ rep movsb 等   │ rsi（ソース）, rdi（目的）, rcx（カウント）     │
│ rdtsc          │ eax（下位）, edx（上位）                        │
│ cpuid          │ eax/ebx/ecx/edx（入出力）                       │
│ syscall        │ rax（番号）, rdi/rsi/rdx/r10/r8/r9（引数）     │
└────────────────┴────────────────────────────────────────────────┘
```

---

## クロバーリスト

**クロバー（clobber）** とは「アセンブリが書き換えるが、Rustの変数として使わないレジスタ」です。クロバーを宣言することで、コンパイラはそのレジスタを保存・復元するコードを生成します。

### クロバーの宣言方法

`out("reg") _` と書くことで「このレジスタを書き換えるが値は使わない」と宣言します。

```rust
use std::arch::asm;

fn with_clobber(val: u64) -> u64 {
    let result: u64;
    unsafe {
        asm!(
            "mov rax, {input}",   // rax を作業レジスタとして使用
            "imul rax, rax, 3",   // rax = rax * 3
            "mov {output}, rax",
            input = in(reg) val,
            output = out(reg) result,
            out("rax") _,         // rax をクロバーとして宣言
            options(nomem, nostack),
        );
    }
    result
}
```

### フラグレジスタのクロバー

多くの算術・論理命令はフラグレジスタを更新します。`preserves_flags` を使わない場合、コンパイラはフラグが変更されると見なします。

```rust
use std::arch::asm;

fn flags_clobber_example(a: u64, b: u64) -> u64 {
    let result: u64;
    unsafe {
        asm!(
            "add {0}, {1}",
            // add命令はCF, OF, SF, ZF, AF, PFを変更する
            // preserves_flags を指定しないことでコンパイラに伝わる
            inout(reg) a => result,
            in(reg) b,
            options(nomem, nostack),
            // preserves_flags を書かない = フラグを変更すると宣言
        );
    }
    result
}
```

### メモリクロバー

アセンブリがメモリを読み書きする場合、`nomem` を省略します。すると、コンパイラはアセンブリの前後でメモリの再ロードを行います。

```rust
use std::arch::asm;

fn memory_barrier() {
    unsafe {
        asm!(
            "mfence",  // 全メモリ操作の完了を保証するフェンス命令
            options(nostack),
            // nomem を書かない = メモリに影響すると伝える
        );
    }
}
```

---

## フラグレジスタ（RFLAGS）

フラグレジスタは算術・論理演算の結果を記録するビットフィールドです。

```
RFLAGS レジスタのビット構成（主要なもの）:

  Bit 0  CF  キャリーフラグ    ─ 符号なし演算のオーバーフロー
  Bit 2  PF  パリティフラグ   ─ 結果の下位8bitの1の個数が偶数なら1
  Bit 4  AF  補助キャリー      ─ BCD演算用
  Bit 6  ZF  ゼロフラグ        ─ 結果がゼロなら1
  Bit 7  SF  符号フラグ        ─ 結果の最上位ビット（負なら1）
  Bit 8  TF  トラップフラグ   ─ シングルステップ実行
  Bit 9  IF  割り込みフラグ   ─ 外部割り込みの許可
  Bit 10 DF  方向フラグ        ─ 文字列操作の方向（0=増加方向）
  Bit 11 OF  オーバーフローフラグ ─ 符号あり演算のオーバーフロー
```

フラグレジスタを直接読み取る例:

```rust
use std::arch::asm;

/// RFLAGS レジスタの値を取得する
fn read_rflags() -> u64 {
    let flags: u64;
    unsafe {
        asm!(
            "pushfq",         // RFLAGS をスタックにプッシュ
            "pop {0}",        // スタックからポップ
            out(reg) flags,
            options(nomem),   // スタック操作するので nostack は指定しない
        );
    }
    flags
}

fn main() {
    let flags = read_rflags();
    let if_flag = (flags >> 9) & 1; // 割り込みフラグ
    let df_flag = (flags >> 10) & 1; // 方向フラグ
    println!("RFLAGS = {:#018x}", flags);
    println!("IF (割り込み許可) = {}", if_flag);
    println!("DF (方向フラグ)   = {}", df_flag);
}
```

---

## System V AMD64 ABI（呼び出し規約）

**ABI（Application Binary Interface）** は関数呼び出しの規約です。Linux・macOSのx86-64では **System V AMD64 ABI** が使われます。

### 引数渡しの規約

```
整数・ポインタ引数の順番（最大6個）:

  第1引数  ─ rdi
  第2引数  ─ rsi
  第3引数  ─ rdx
  第4引数  ─ rcx
  第5引数  ─ r8
  第6引数  ─ r9
  第7引数以降 ─ スタック（右から左の順でプッシュ）

浮動小数点・SIMD引数の順番（最大8個）:
  xmm0〜xmm7

戻り値:
  整数・ポインタ ─ rax（, rdx）
  浮動小数点     ─ xmm0（, xmm1）
```

### Caller-saved と Callee-saved レジスタ

```
┌────────────────────────────────┬────────────────────────────────┐
│ Caller-saved（呼び出し元が保存） │ Callee-saved（呼び出し先が保存）│
│ （関数呼び出し後に値が保証されない）│ （関数呼び出し後も値が保持）    │
├────────────────────────────────┼────────────────────────────────┤
│ rax                            │ rbx                            │
│ rcx                            │ rbp                            │
│ rdx                            │ r12                            │
│ rsi                            │ r13                            │
│ rdi                            │ r14                            │
│ r8                             │ r15                            │
│ r9                             │ rsp（特別：常にスタックトップ）  │
│ r10                            │                                │
│ r11                            │                                │
│ xmm0〜xmm15（上位ビット）        │ xmm保存部分（Windows ABI）     │
└────────────────────────────────┴────────────────────────────────┘
```

### インラインアセンブリでのABI遵守

インラインアセンブリはRustの関数の内部で実行されます。インラインアセンブリ自体が関数を呼び出さない限り、ABIの引数渡し規約を気にする必要はほとんどありません。ただし、以下の点に注意が必要です。

```rust
use std::arch::asm;

/// Callee-savedレジスタを正しく保存・復元する例
fn safe_use_of_callee_saved() -> u64 {
    let result: u64;
    unsafe {
        asm!(
            // rbx は callee-saved なので使うなら保存・復元が必要
            "push rbx",           // rbx を保存
            "mov rbx, {input}",
            "imul rbx, rbx, 7",
            "mov {output}, rbx",
            "pop rbx",            // rbx を復元
            input = in(reg) 6u64,
            output = out(reg) result,
            // out("rbx") _ を使う代わりに手動で保存・復元している
            options(nomem),
        );
    }
    result  // 42
}
```

ただし、Rustのコンパイラに任せる方法がより安全です。クロバーリストを正しく使えば、コンパイラが保存・復元を自動的に行います。

```rust
use std::arch::asm;

/// コンパイラにレジスタ保存を任せる（推奨）
fn let_compiler_save(val: u64) -> u64 {
    let result: u64;
    unsafe {
        asm!(
            "imul {0}, {0}, 7",
            inout(reg) val => result,
            options(pure, nomem, nostack),
            // コンパイラが適切なレジスタを選び、
            // 必要なら保存・復元コードを生成する
        );
    }
    result
}
```

---

## SIMDレジスタ

SSE/AVX命令はXMM/YMM/ZMMレジスタを使います。

```
SIMDレジスタの構造:

  ZMM0 (512-bit)
  ┌──────────────────────────────────────────────────────────────┐
  │                        ZMM0                                   │
  ├──────────────────────────────────────────────────┬───────────┤
  │                  YMM0 (256-bit)                   │上位256bit │
  ├──────────────────────┬───────────────────────────┤           │
  │    XMM0 (128-bit)    │   上位128bit              │           │
  └──────────────────────┴───────────────────────────┴───────────┘

  各レジスタの解釈（XMM0の例）:
  ・4 × f32（単精度浮動小数点）
  ・2 × f64（倍精度浮動小数点）
  ・16 × i8
  ・8 × i16
  ・4 × i32
  ・2 × i64
```

### XMMレジスタを使う例

```rust
use std::arch::asm;

/// SSE2 による2つの f64 の加算
#[target_feature(enable = "sse2")]
unsafe fn sse2_add_f64(a: f64, b: f64) -> f64 {
    let result: f64;
    unsafe {
        asm!(
            "addsd {0}, {1}",          // スカラー倍精度浮動小数点の加算
            inout(xmm_reg) a => result, // xmm_reg クラスを使う
            in(xmm_reg) b,
            options(pure, nomem, nostack),
        );
    }
    result
}

/// SSE による4つの f32 の並列加算
#[target_feature(enable = "sse")]
unsafe fn sse_add_f32x4(
    a: [f32; 4],
    b: [f32; 4],
) -> [f32; 4] {
    let result: [f32; 4];
    // std::arch::x86_64 の組み込み関数を使う方がより安全
    // ここはアセンブリの学習目的での例
    unsafe {
        asm!(
            "movups {out}, [{a_ptr}]",  // a を xmm にロード
            "movups {tmp}, [{b_ptr}]",  // b を xmm にロード
            "addps {out}, {tmp}",       // 4つの f32 を並列加算
            "movups [{result_ptr}], {out}", // 結果をメモリへ
            a_ptr = in(reg) a.as_ptr(),
            b_ptr = in(reg) b.as_ptr(),
            result_ptr = in(reg) result.as_mut_ptr(),  // ← 注: 未初期化
            out = out(xmm_reg) _,
            tmp = out(xmm_reg) _,
            options(nostack),
        );
    }
    result
}
```

> **推奨**: SIMDの多くの操作は `std::arch::x86_64` の組み込み関数（intrinsics）を使う方が安全で移植性があります。インラインアセンブリが必要になるのは、intrinsicsでカバーされない命令を使う場合です。

---

## 実践的なレジスタ操作例

### 符号なし除算（div命令）

`div` 命令は `rdx:rax` を被除数とする128-bit割り算です。

```rust
use std::arch::asm;

/// u128 ÷ u64 → (商: u64, 余り: u64)
/// 注意: 商が u64 に収まらない場合は除算例外（#DE）が発生する
fn div128(high: u64, low: u64, divisor: u64) -> (u64, u64) {
    let quotient: u64;
    let remainder: u64;
    unsafe {
        asm!(
            "div {divisor}",         // rdx:rax ÷ divisor
            divisor = in(reg) divisor,
            inout("rax") low => quotient,  // rax: 被除数の下位 → 商
            inout("rdx") high => remainder, // rdx: 被除数の上位 → 余り
            options(nomem, nostack),
        );
    }
    (quotient, remainder)
}

fn main() {
    // 100 ÷ 7 = 商14, 余り2
    let (q, r) = div128(0, 100, 7);
    println!("100 ÷ 7 = {}, 余り {}", q, r);

    // 大きな数: (2^64 + 0) ÷ 3
    let (q, r) = div128(1, 0, 3);
    println!("2^64 ÷ 3 = {}, 余り {}", q, r);
}
```

### ビット操作命令

```rust
use std::arch::asm;

/// BSF (Bit Scan Forward): 最下位の1のビット位置を返す
fn bsf(val: u64) -> Option<u32> {
    if val == 0 {
        return None;
    }
    let result: u32;
    unsafe {
        asm!(
            "bsf {0:e}, {1:e}",  // 32-bit BSF（64-bit版は bsfq）
            out(reg) result,
            in(reg) val as u32,
            options(pure, nomem, nostack),
        );
    }
    Some(result)
}

/// POPCNT: 1のビット数を数える
#[target_feature(enable = "popcnt")]
unsafe fn popcount(val: u64) -> u32 {
    let result: u32;
    unsafe {
        asm!(
            "popcnt {0:e}, {1:e}",
            out(reg) result,
            in(reg) val as u32,
            options(pure, nomem, nostack),
        );
    }
    result
}

fn main() {
    println!("bsf(0b1010) = {:?}", bsf(0b1010)); // Some(1)
    println!("bsf(0b1000) = {:?}", bsf(0b1000)); // Some(3)
    println!("bsf(0)      = {:?}", bsf(0));       // None

    unsafe {
        println!("popcount(0xFF) = {}", popcount(0xFF)); // 8
        println!("popcount(0x0F) = {}", popcount(0x0F)); // 4
    }
}
```

---

## まとめ

```
レジスタ操作の要点:

  ・汎用レジスタは rax〜r15（各64-bit、サブレジスタ経由でアクセス可）
  ・32-bitへの書き込みは上位32bitをゼロクリアする（重要！）
  ・明示的レジスタ指定: in("rax"), out("rdx") 等
  ・クロバーリスト: out("reg") _ で「使うが値は不要」と宣言
  ・System V AMD64 ABI:
    - 引数: rdi, rsi, rdx, rcx, r8, r9
    - 戻り値: rax（整数）, xmm0（浮動小数点）
    - Callee-saved: rbx, rbp, r12〜r15
  ・SIMDレジスタ: xmm_reg / ymm_reg / zmm_reg クラス
```

次節 [6.3 実践的な最適化テクニック](./03-asm-optimization.md) では、RDTSC・CPUID・PAUSEなどの実用的な命令を紹介します。
