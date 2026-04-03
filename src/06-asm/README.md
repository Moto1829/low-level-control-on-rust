# 第6章: インラインアセンブリ

## 概要

Rustは安全性と高性能を両立する言語ですが、ハードウェアを直接操作したい場面では、Rustのコンパイラが生成するコードだけでは不十分なことがあります。そのような場面で威力を発揮するのが **インラインアセンブリ（inline assembly）** です。

Rust 1.59（2022年2月リリース）から `asm!` マクロが安定版（stable）に昇格しました。それ以前はnightlyのみで利用可能だったため、現在では安定版Rustを使う多くのプロジェクトでインラインアセンブリを活用できます。

```
Rust バージョン対応状況

  < 1.59  ──── nightly のみ（llvm_asm! / asm! 不安定）
  ≥ 1.59  ──── stable で asm! が利用可能
  ≥ 1.59  ──── global_asm! も stable
```

---

## インラインアセンブリが必要な場面

Rustの通常のコードで実現できないことがあります。代表的なユースケースを整理します。

```
┌──────────────────────────────────────────────────────────────────┐
│                インラインアセンブリが必要な場面                     │
├──────────────────────────┬───────────────────────────────────────┤
│ ユースケース              │ 具体例                                 │
├──────────────────────────┼───────────────────────────────────────┤
│ 特定CPU命令の直接利用     │ RDTSC, CPUID, PAUSE, PREFETCH          │
│ レジスタの直接読み書き    │ CR3, MSR, フラグレジスタ               │
│ 割り込み・例外ハンドラ    │ idt, iret, cli/sti                     │
│ SIMD命令（組み込み関数    │ AVX-512 の細かい制御                   │
│ 非対応のもの）            │                                        │
│ アトミック操作の低水準    │ lock xchg, cmpxchg16b                  │
│ 制御                      │                                        │
│ 高精度タイマー            │ RDTSC / RDTSCP による計測              │
│ OS・カーネル開発          │ コンテキストスイッチ、TLBフラッシュ     │
└──────────────────────────┴───────────────────────────────────────┘
```

### コンパイラに任せる vs 手書きアセンブリ

多くの場合、Rustコンパイラ（LLVM）は優秀な最適化を行います。インラインアセンブリの使用は **最後の手段** です。

```
最適化の優先順位:

  1. Rust標準のコード（コンパイラに任せる）
  2. std::arch の組み込み関数（SIMD等）
  3. #[target_feature] によるCPU機能の有効化
  4. インラインアセンブリ（asm!）← 本章の対象
```

---

## 安定版Rustでの利用

`asm!` マクロを使うために特別なフラグは不要です。安定版ツールチェーンで動作します。

```rust
// Cargo.toml に特別な設定は不要
// rustup toolchain が stable (≥ 1.59) であれば動作する

use std::arch::asm;

fn main() {
    let x: u64 = 5;
    let y: u64;
    unsafe {
        asm!(
            "mov {0}, {1}",
            out(reg) y,
            in(reg) x,
        );
    }
    println!("y = {}", y); // y = 5
}
```

> **注意**: `asm!` の中身は `unsafe` ブロックで囲む必要があります。
> アセンブリはRustの安全保証の外側にあるため、プログラマが正しさを保証する責任があります。

---

## 対応アーキテクチャ

本章のサンプルコードは主に **x86-64** を対象としています。

```
asm! がサポートするアーキテクチャ（Rust 1.59+）:

  ✓ x86 / x86-64
  ✓ ARM (32-bit)
  ✓ AArch64 (ARM 64-bit)
  ✓ RISC-V (32/64-bit)
  ✓ LoongArch
  ✓ MIPS / MIPS64 (nightly)
  ✓ PowerPC (nightly)
  ✓ s390x (nightly)
```

---

## 学習目標

この章を学ぶと、以下のことができるようになります。

- `asm!` マクロの基本的な構文を理解し、使用できる
- レジスタの種類と役割を理解し、入出力オペランドを正しく指定できる
- クロバーリストを正確に記述し、コンパイラとの契約を守れる
- RDTSC・CPUID・PAUSE などの実用的な命令を実装できる
- インラインアセンブリを用いた最適化の効果を検証できる

---

## 各節の内容

| 節 | タイトル | 内容 |
|---|---------|------|
| [6.1](./01-asm-macro.md) | `asm!` マクロの基礎 | 構文・テンプレート・オペランド・options |
| [6.2](./02-registers.md) | レジスタと呼び出し規約 | x86-64レジスタ・クロバーリスト・ABI |
| [6.3](./03-asm-optimization.md) | 実践的な最適化テクニック | RDTSC・CPUID・PAUSE・PREFETCH |

---

## 事前知識

この章を読む前に、以下の知識があると理解が深まります。

- Rustの基本的な型システム（特に整数型）
- `unsafe` ブロックの概念（[第2章: unsafe Rust](../02-unsafe/README.md) 参照）
- x86-64アセンブリの基礎（mov, add, ret等）
- コンピュータアーキテクチャの基礎（レジスタ、スタック、メモリ）

---

## 参考資料

- [Rust Reference: Inline Assembly](https://doc.rust-lang.org/reference/inline-assembly.html)
- [Rust By Example: Inline assembly](https://doc.rust-lang.org/rust-by-example/unsafe/asm.html)
- [Intel 64 and IA-32 Architectures Software Developer Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [System V AMD64 ABI](https://www.intel.com/content/dam/develop/external/us/en/documents/mpx-linux64-abi.pdf)
