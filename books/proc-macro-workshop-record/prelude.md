---
title: "開始前"
---

# 前章

## この記事を書き始めた状態

* [rustlings](https://github.com/rust-lang/rustlings) を完了している
* Rustを用いて小さい規模の何かを実装したことがある
    * マクロを使用しない範囲で

## この本の目的

proc-macro-workshopに取り組み、実際にやった処理を残す。

## 執筆手順

1. 通常の手段で実装を進める
    * 他の人の実装や解説を見るなど
1. 実装方法を確認したうえで、再度実装をする
    * このとき、なるべく他の人の実装や解説を参照せずに、どのようにして調べるべきだったかを検討し、その手順を残す

## 事前準備

1. git をインストール(任意)
    * proc-macro-workshopのダウンロードと作業記録のために使用
1. [rustup](https://www.rust-lang.org/ja/tools/install) をインストール
1. *nightly版* Rust をインストール
    * `RUSTFLAGS="-Zexternal-macro-backtrace"` を使用できるようにするため
    * 導入したら上記設定で `export` しておくと便利かもしれない。

## proc-macro-workshop をダウンロードする

1. proc-macro-workshopをダウンロード
    * 推奨： GitHub上に記録を残すため、Forkしてからclone
1. Debugging tipsの項目を読む
1. [cargo-expand](https://github.com/dtolnay/cargo-expand) をインストール

# 謝辞・参考文献

以下の記事を参考にしました。

* proc_macro_workshopでRustの手続き的マクロに入門する - CADDi Tech Blog
    * [前編](https://caddi.tech/archives/3555)
    * [後編](https://caddi.tech/archives/3752)
* [Rust の procedural macro を操って黒魔術師になろう〜proc-macro-workshop の紹介](https://zenn.dev/magurotuna/articles/bab4db5999ebfa)
* [proc-macro-workshop/builder に取り組む](https://zenn.dev/hpp/scraps/7b984ca72eefce)
* [magurotuna/proc-macro-workshop](https://github.com/magurotuna/proc-macro-workshop)
* [jonhoo/proc-macro-workshop](https://github.com/jonhoo/proc-macro-workshop)
