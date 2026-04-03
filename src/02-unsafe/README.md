# 第2章: unsafe コード

## 概要

Rust の安全性モデルは強力ですが、すべての正しいプログラムをコンパイラが証明できるわけではありません。
オペレーティングシステムのカーネル、デバイスドライバ、FFI バインディング、高性能データ構造など、
低レイヤーのプログラミングでは「コンパイラには証明できないが、プログラマが正しさを保証できる」操作が
どうしても必要になります。そのための脱出ハッチが `unsafe` キーワードです。

本章では `unsafe` の仕組みと、それを**最小限かつ安全に**使うための原則・パターンを学びます。

---

## なぜ unsafe を学ぶのか

```
通常の Rust コード
┌─────────────────────────────────────────────────────────┐
│ 借用チェッカー・型システム・生存期間 ──> 自動的に安全     │
└──────────────────────┬──────────────────────────────────┘
                       │ どうしても越えられない壁
                       ▼
            unsafe ブロック / unsafe fn
┌─────────────────────────────────────────────────────────┐
│ 生ポインタの操作・FFI・可変静的変数・unsafe trait        │
│ 安全性の証明責任が「コンパイラ」から「プログラマ」へ移る  │
└─────────────────────────────────────────────────────────┘
```

`unsafe` を学ぶ理由は大きく3つあります。

1. **標準ライブラリを読み解く力**
   `Vec<T>`、`String`、`Arc<T>` など、Rust の根幹となるデータ構造はすべて `unsafe` で実装されています。
   その仕組みを理解することで、正しい抽象化とは何かを深く学べます。

2. **C/C++ ライブラリとの連携**
   既存の膨大な C エコシステムを活用するには FFI (Foreign Function Interface) が必要です。
   FFI のコードは必然的に `unsafe` を含みます。

3. **パフォーマンスの限界を突く**
   境界チェックの省略・SIMD・インラインアセンブリなど、極限のパフォーマンスが求められる場面では
   `unsafe` が唯一の手段になることがあります。

---

## unsafe で許可される操作

Rust において `unsafe` ブロックまたは `unsafe fn` 内でのみ許可される操作は以下の5種類です。

| 操作 | 説明 |
|------|------|
| 生ポインタのデリファレンス | `*const T` / `*mut T` を `*ptr` で間接参照する |
| `unsafe fn` の呼び出し | unsafe として宣言された関数・メソッドを呼ぶ |
| `unsafe trait` の実装 | `unsafe impl` でトレイトを実装する |
| 可変静的変数へのアクセス | `static mut` 変数の読み書き |
| `union` フィールドへのアクセス | union のフィールドを読む |

これ以外の操作（例：インデックスによる配列アクセス、整数演算）は `unsafe` に入れても
安全性ルールは変わりません。**`unsafe` はマジックワードではなく、責任の宣言です。**

---

## unsafe の黄金律

> **`unsafe` ブロックは可能な限り小さく保ち、安全な抽象に包む。**

悪い例:

```rust
// unsafe ブロックが広すぎる
unsafe {
    let ptr = allocate(1024);
    let slice = std::slice::from_raw_parts_mut(ptr, 256);
    process(slice);
    deallocate(ptr);
}
```

良い例:

```rust
// 最小限の unsafe + 安全なラッパー
fn with_buffer<F: FnOnce(&mut [u8])>(size: usize, f: F) {
    let ptr = allocate(size);
    // SAFETY: allocate は size バイトのアラインされたメモリを返す
    let slice = unsafe { std::slice::from_raw_parts_mut(ptr, size / std::mem::size_of::<u8>()) };
    f(slice);
    // SAFETY: ptr は allocate で確保したもので、まだ解放されていない
    unsafe { deallocate(ptr) };
}
```

`// SAFETY:` コメントは**なぜこの unsafe が安全か**を説明する慣習であり、
コードレビューとメンテナンスに不可欠です。

---

## 学習目標

本章を修了すると、以下ができるようになります。

- [ ] 生ポインタ (`*const T` / `*mut T`) の作成・操作・デリファレンスができる
- [ ] `unsafe fn` を定義し、最小スコープの `unsafe` ブロックで呼び出せる
- [ ] unsafe な内部実装を持つ安全な API を設計・実装できる
- [ ] C ライブラリを FFI 経由で Rust から呼び出せる
- [ ] `bindgen` を使って C ヘッダから Rust バインディングを自動生成できる
- [ ] `// SAFETY:` コメントで invariant を適切に文書化できる

---

## 各節の内容

| 節 | タイトル | 主要トピック |
|----|----------|--------------|
| [01](01-raw-pointers.md) | 生ポインタ | `*const T` / `*mut T`、ダングリングポインタ、アドレス演算 |
| [02](02-unsafe-functions.md) | unsafe 関数とトレイト | `unsafe fn`、`unsafe trait`、スコープ最小化 |
| [03](03-safe-abstraction.md) | 安全な抽象化 | invariant の文書化、`NonNull<T>`、スプリットバッファ実装 |
| [04](04-ffi.md) | FFI | `extern "C"`、型対応表、`bindgen`、所有権の境界管理 |

---

## 前提知識

- 第1章「メモリ管理」の内容（スタック・ヒープ・所有権・借用）
- Rust の型システムの基礎（ジェネリクス・トレイト・生存期間）

---

## 参考資料

- [The Rustonomicon](https://doc.rust-lang.org/nomicon/) — unsafe Rust の公式ガイド
- [Rust Reference: Unsafety](https://doc.rust-lang.org/reference/unsafety.html)
- [unsafe guidelines RFC](https://rust-lang.github.io/unsafe-code-guidelines/)
