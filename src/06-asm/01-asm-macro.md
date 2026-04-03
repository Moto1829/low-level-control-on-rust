# 6.1 `asm!` マクロの基礎

## はじめに

`asm!` マクロはRustのインラインアセンブリ機能の中核です。アセンブリのテンプレート文字列にRustの変数を結び付けることで、型安全とは言えないながらも構造化された方法でアセンブリを書けます。

まずシンプルな例から始めましょう。

```rust
use std::arch::asm;

fn main() {
    unsafe {
        asm!("nop"); // 何もしない命令（no operation）
    }
}
```

---

## 基本構文

`asm!` マクロの構文は以下の形式です。

```
asm!(
    テンプレート文字列,
    オペランド1,
    オペランド2,
    ...,
    options(オプション1, オプション2, ...)
)
```

テンプレート文字列はアセンブリ命令をRustの文字列フォーマット記法で表したものです。`{0}`、`{1}` という形式でオペランドを参照します。

### テンプレート文字列の構造

```
asm!(
    "命令 {出力オペランド}, {入力オペランド}",
    //   ↑ インデックス 0        ↑ インデックス 1
    out(reg) 出力変数,
    in(reg) 入力変数,
)
```

複数の命令を書く場合は、文字列を並べるか改行文字 `\n` で区切ります。

```rust
use std::arch::asm;

fn multi_instruction() -> u64 {
    let result: u64;
    unsafe {
        asm!(
            "mov {0}, 42",    // result = 42
            "add {0}, 8",     // result += 8
            out(reg) result,
        );
    }
    result // 50
}
```

複数の文字列を並べる記法（推奨）:

```rust
use std::arch::asm;

fn multi_instruction_preferred() -> u64 {
    let result: u64;
    unsafe {
        asm!(
            "mov {0}, 42",
            "add {0}, 8",
            out(reg) result,
        );
    }
    result
}
```

---

## オペランドの指定

オペランドはRustの変数とアセンブリのレジスタ（またはメモリ）を結び付けます。

### `in` — 入力オペランド

```rust
use std::arch::asm;

fn use_input() {
    let value: u64 = 100;
    unsafe {
        asm!(
            "mov rax, {val}",  // value の値を rax に移す（実際の動作はコンパイラ次第）
            val = in(reg) value,
        );
    }
}
```

名前付きオペランドを使うと可読性が上がります。

```rust
use std::arch::asm;

fn named_operand(a: u64, b: u64) -> u64 {
    let result: u64;
    unsafe {
        asm!(
            "mov {result}, {a}",
            "add {result}, {b}",
            a = in(reg) a,
            b = in(reg) b,
            result = out(reg) result,
        );
    }
    result
}
```

### `out` — 出力オペランド

出力オペランドは関数の戻り値のように、アセンブリが書き込んだ値をRustの変数に受け渡します。

```rust
use std::arch::asm;

fn get_value() -> u32 {
    let x: u32;
    unsafe {
        asm!(
            "mov {0:e}, 0xDEAD",  // :e は 32-bit (dword) サイズ修飾子
            out(reg) x,
        );
    }
    x
}
```

> **初期化前の変数**: `out(reg)` を使う場合、Rustの変数は未初期化でも構いません。
> アセンブリが必ず値を書き込む契約をしているためです。

### `inout` — 入出力オペランド

入出力オペランドは、同じオペランドが入力と出力を兼ねます。

```rust
use std::arch::asm;

fn increment_by_asm(mut val: u64) -> u64 {
    unsafe {
        asm!(
            "add {0}, 1",
            inout(reg) val,  // val を読み、1を加算して val に書き戻す
        );
    }
    val
}

fn main() {
    println!("{}", increment_by_asm(41)); // 42
}
```

`inout` には入力と出力で異なる変数を使う書き方もあります。

```rust
use std::arch::asm;

fn add_by_asm(a: u64, b: u64) -> u64 {
    let result: u64;
    unsafe {
        asm!(
            "add {0}, {1}",
            inout(reg) a => result,  // a を入力とし、演算結果を result に出す
            in(reg) b,
        );
    }
    result
}
```

### `lateout` — 遅延出力オペランド

`lateout` は入力オペランドと同じレジスタを再利用できます。入力がすべて読まれた後で出力が書かれることをコンパイラに伝えます。

```rust
use std::arch::asm;

fn late_output_example(a: u64, b: u64) -> u64 {
    let result: u64;
    unsafe {
        asm!(
            "imul {result}, {a}, {b}",  // result = a * b (即値形式)
            a = in(reg) a,
            b = in(reg) b,
            result = lateout(reg) result,  // 入力と同じレジスタを使える
        );
    }
    result
}
```

---

## オペランドのレジスタクラス

`reg` はコンパイラが適切なレジスタを選ぶ汎用クラスです。特定のレジスタを指定することもできます。

```
┌────────────────┬──────────────────────────────────────────────┐
│ レジスタクラス  │ 意味                                          │
├────────────────┼──────────────────────────────────────────────┤
│ reg            │ 汎用整数レジスタ（コンパイラが選ぶ）           │
│ reg_byte       │ 8-bitレジスタ（al, bl, cl, dl 等）            │
│ reg_abcd       │ al/ah使用可能なレジスタ（rax/rbx/rcx/rdx）    │
│ xmm_reg        │ XMMレジスタ（SSE）                             │
│ ymm_reg        │ YMMレジスタ（AVX）                             │
│ zmm_reg        │ ZMMレジスタ（AVX-512）                         │
│ mmx_reg        │ MMXレジスタ（非推奨）                          │
│ x87_reg        │ x87 浮動小数点スタック                         │
│ kreg           │ AVX-512 マスクレジスタ（k0〜k7）               │
└────────────────┴──────────────────────────────────────────────┘
```

明示的なレジスタ指定（詳細は [6.2節](./02-registers.md) で解説）:

```rust
use std::arch::asm;

fn explicit_reg() -> u64 {
    let result: u64;
    unsafe {
        asm!(
            "mov rax, 0x1234",
            out("rax") result,  // 明示的に rax を指定
        );
    }
    result
}
```

---

## サイズ修飾子

テンプレート文字列内の `{0}` にコロンでサイズ修飾子を付けられます。

```
┌──────────┬───────────────────────────────────────┐
│ 修飾子    │ 意味                                   │
├──────────┼───────────────────────────────────────┤
│ {0}      │ デフォルト（変数の型によって決まる）     │
│ {0:r}    │ 64-bit (rax, rbx, ...)                 │
│ {0:e}    │ 32-bit (eax, ebx, ...)                 │
│ {0:x}    │ 16-bit (ax, bx, ...)                   │
│ {0:l}    │ 8-bit low  (al, bl, ...)               │
│ {0:h}    │ 8-bit high (ah, bh, ...) ※ reg_abcd   │
└──────────┴───────────────────────────────────────┘
```

```rust
use std::arch::asm;

fn size_modifier_example() {
    let mut val: u64 = 0;
    unsafe {
        asm!(
            "mov {0:e}, 0xFF",   // 32-bit で 0xFF を書き込む（上位32bitはゼロ拡張）
            inout(reg) val,
        );
    }
    println!("val = {:#x}", val); // 0xff
}
```

---

## 整数加算の実践例

シンプルな整数加算をインラインアセンブリで実装し、Rustの通常の加算と比べます。

```rust
use std::arch::asm;

/// アセンブリによる u64 加算
fn asm_add(a: u64, b: u64) -> u64 {
    let result: u64;
    unsafe {
        asm!(
            "add {result}, {b}",
            result = inout(reg) a => result,
            b = in(reg) b,
            options(pure, nomem, nostack),
        );
    }
    result
}

/// 通常のRust加算（比較用）
fn rust_add(a: u64, b: u64) -> u64 {
    a + b
}

fn main() {
    let a: u64 = 12345;
    let b: u64 = 67890;

    let r1 = asm_add(a, b);
    let r2 = rust_add(a, b);

    println!("asm_add({}, {}) = {}", a, b, r1);   // 80235
    println!("rust_add({}, {}) = {}", a, b, r2);  // 80235
    assert_eq!(r1, r2);
    println!("結果は一致しました！");
}
```

> **実用上の注意**: 単純な加算にインラインアセンブリを使う意味はほとんどありません。
> コンパイラは同等かそれ以上のコードを生成します。この例はあくまで学習目的です。

---

## `options` の指定

`options()` はコンパイラにアセンブリの副作用を伝えるためのヒントです。正しく指定することで最適化が改善されます。

### `nostack`

アセンブリがスタックポインタ（RSP）を操作しないことをコンパイラに伝えます。

```rust
unsafe {
    asm!(
        "add {0}, 1",
        inout(reg) val,
        options(nostack),  // RSP を変更しない
    );
}
```

### `pure`

アセンブリが副作用を持たず、同じ入力に対して常に同じ出力を返すことを示します。これによりコンパイラはCSE（共通部分式の除去）などの最適化を適用できます。

```rust
use std::arch::asm;

fn pure_add(a: u64, b: u64) -> u64 {
    let result: u64;
    unsafe {
        asm!(
            "lea {result}, [{a} + {b}]",  // lea はメモリアクセスなしで加算できる
            a = in(reg) a,
            b = in(reg) b,
            result = out(reg) result,
            options(pure, nostack, nomem),
        );
    }
    result
}
```

### `nomem`

アセンブリがメモリを読み書きしないことを示します。これによりコンパイラはメモリ操作の順序を変更できます。

```rust
unsafe {
    asm!(
        "xor {0}, {0}",  // レジスタを 0 クリア
        out(reg) val,
        options(nomem, nostack),
    );
}
```

### `readonly`

アセンブリがメモリを読むが書かないことを示します（`nomem` より緩い制約）。

```rust
unsafe {
    asm!(
        "mov {0}, [{1}]",  // メモリを読むが書かない
        out(reg) val,
        in(reg) ptr,
        options(readonly, nostack),
    );
}
```

### `preserves_flags`

アセンブリがフラグレジスタ（RFLAGS）を変更しないことを示します。

```rust
unsafe {
    asm!(
        "mov {0}, {1}",  // mov はフラグを変更しない
        out(reg) dst,
        in(reg) src,
        options(nomem, nostack, preserves_flags),
    );
}
```

### `att_syntax`

AT&T 構文（GCCスタイル）でアセンブリを書くことを指定します。デフォルトはIntel構文です。

```rust
unsafe {
    // Intel 構文（デフォルト）: "add rax, rbx"（目的レジスタが先）
    // AT&T 構文: "addq %rbx, %rax"（ソースが先、サイズ接尾辞あり）
    asm!(
        "addq {1}, {0}",
        inout(reg) a,
        in(reg) b,
        options(att_syntax),
    );
}
```

### options のまとめ

```
┌──────────────────┬────────────────────────────────────────────────┐
│ option           │ 意味                                            │
├──────────────────┼────────────────────────────────────────────────┤
│ pure             │ 副作用なし（CSE等の最適化を許可）               │
│ nomem            │ メモリ読み書きなし                              │
│ readonly         │ メモリ読み取りのみ（書き込みなし）              │
│ preserves_flags  │ RFLAGSを変更しない                             │
│ nostack          │ スタックポインタを操作しない                    │
│ att_syntax       │ AT&T構文を使用（デフォルトはIntel構文）         │
│ raw              │ テンプレート文字列を生の文字列として扱う         │
└──────────────────┴────────────────────────────────────────────────┘
```

最も安全なパフォーマンスのための組み合わせ:

```rust
options(pure, nomem, nostack)
// 意味: 「このアセンブリはレジスタのみ操作し、
//        副作用がなく、常に同じ入力から同じ出力を返す」
```

---

## `const` オペランド

定数値をアセンブリのリテラルとして使うことができます。

```rust
use std::arch::asm;

const SHIFT_AMOUNT: u64 = 3;

fn shift_left_const(val: u64) -> u64 {
    let result: u64;
    unsafe {
        asm!(
            "shl {0}, {1}",         // val を SHIFT_AMOUNT ビット左シフト
            inout(reg) val => result,
            const SHIFT_AMOUNT,     // コンパイル時定数をそのまま埋め込む
            options(pure, nomem, nostack),
        );
    }
    result
}

fn main() {
    println!("{}", shift_left_const(1));  // 8（1 << 3）
    println!("{}", shift_left_const(5));  // 40（5 << 3）
}
```

---

## `sym` オペランド

Rustのシンボル（関数や静的変数のアドレス）をアセンブリで参照できます。

```rust
use std::arch::asm;

static MY_VALUE: u64 = 0xDEADBEEF;

fn read_static() -> u64 {
    let result: u64;
    unsafe {
        asm!(
            "mov {0}, qword ptr [rip + {sym}]",  // RIP相対アドレッシング
            out(reg) result,
            sym MY_VALUE,
            options(readonly, nostack),
        );
    }
    result
}
```

---

## よくある間違いと注意点

### 1. クロバーの宣言忘れ

アセンブリが暗黙に変更するレジスタを宣言し忘れると、未定義動作になります。

```rust
// 危険な例（修正前）
unsafe {
    asm!(
        "push rax",  // rsp を変更するが nostack を指定していない
        "pop rax",
        out("rax") _,
        // options(nostack) を書くべきではない！スタックを操作している
    );
}

// 正しい例
unsafe {
    asm!(
        "push rax",
        "pop rax",
        out("rax") _,
        // nostack を書かない = スタック操作ありと認識される
    );
}
```

### 2. 入出力の方向の誤り

Intel構文では「目的←ソース」の順ですが、混乱しやすいです。

```rust
// Intel構文: mov 目的, ソース
// "mov {dst}, {src}" が正しい

unsafe {
    asm!(
        "mov {dst}, {src}",  // OK
        src = in(reg) source_val,
        dst = out(reg) dest_val,
    );
}
```

### 3. `pure` の誤用

システムコールや I/O を伴うアセンブリに `pure` を付けてはいけません。

```rust
// 危険！rdtsc は毎回異なる値を返すため pure ではない
unsafe {
    asm!(
        "rdtsc",
        out("eax") low,
        out("edx") high,
        // options(pure) を付けてはいけない
        options(nomem, nostack),
    );
}
```

---

## まとめ

```
asm! マクロの要点:

  テンプレート文字列  ─── Intel構文（デフォルト）のアセンブリ命令
  in(reg)            ─── 読み取り専用の入力オペランド
  out(reg)           ─── 書き込み専用の出力オペランド
  inout(reg)         ─── 読み書き両用のオペランド
  lateout(reg)       ─── 入力後に書き込む出力オペランド
  const              ─── コンパイル時定数
  sym                ─── Rustシンボルの参照
  options(...)       ─── コンパイラへのヒント（pure/nomem/nostack 等）
```

次節 [6.2 レジスタと呼び出し規約](./02-registers.md) では、x86-64のレジスタ体系と呼び出し規約について詳しく解説します。
