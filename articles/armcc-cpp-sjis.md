---
title: "armccでShift-JISのC++ファイルをコンパイルする"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cpp", "Arm", "embedded"]
published: true
---

`armcc`でShift-JISのC++ファイルをコンパイルする際に必要な設定を解説する。

## モチベーション

`armcc`で何も考えずにShift-JISエンコーディングの日本語を含むコードをコンパイルすると、[ダメ文字](https://ja.wikipedia.org/wiki/Shift_JIS#2%E3%83%90%E3%82%A4%E3%83%88%E7%9B%AE%E3%81%8C5C%E7%AD%89%E3%81%AB%E3%81%AA%E3%82%8A%E3%81%86%E3%82%8B%E3%81%93%E3%81%A8%E3%81%AB%E3%82%88%E3%82%8B%E5%95%8F%E9%A1%8C)が適切に処理されず、文字のバイト列から`0x5C`が取り除かれてしまう。

この問題を解決するための`armcc`のコンパイルオプションの設定方法と、その際に発生しうるエラーの対処法を解説する。

## `armcc`？

ARM社製の、ARMプロセッサ向けC/C++コンパイラ。

少し古めのArm Compiler 5や更に古いRVCT(RealView Compilation Tools)で使用するコンパイラがこれ。  
現行バージョン(2024年現在)のArm Compiler 6はLLVMベースの`armclang`というコンパイラが使われている。

## コンパイルオプションの設定

WindowsとLinuxで微妙に異なる。

### Windows

`--multibyte_chars --locale=japanese`オプションを指定する。  
また、コンパイル時に表示されるメッセージを日本語にしたい場合、`--message_locale=ja_JP`を指定する。

```bash
armcc --multibyte_chars --locale=japanese --message_locale=ja_JP -c main.cpp
```

### Linux

`--multibyte_chars --locale=ja_JP.sjis`オプションを指定する。  
また、コンパイル時に表示されるメッセージを日本語にしたい場合、OSのエンコードに応じて`--message_locale=ja_JP.utf8`や`--message_locale=ja_JP.sjis`を指定する。

```bash
armcc --multibyte_chars --locale=ja_JP.sjis --message_locale=ja_JP.utf8 -c main.cpp
```

### `could not set locale "ja_JP" to allow processing of multibyte characters`が出たら

上記のオプションを設定すると、`could not set locale "ja_JP" to allow processing of multibyte characters`のコンパイルエラーが出る場合がある。
これは、OSの言語設定として日本語が存在していないときに発生する。わざわざ日本語をインストールしないLinuxで発生しがち。

以下、Ubuntuでの対処方法を解説する。

1. 必要なパッケージをインストールする。

    ```bash
    # apt install locales language-pack-ja -y
    ```

1. システムのロケール設定ファイルに日本語のShift-JISを追加する。

    ```bash
    # echo "ja_JP.SJIS SHIFT_JIS" >> /etc/locale.gen
    ```

1. ロケールを再生成する。

    ```bash
    # locale-gen
    ```

1. `ja_JP.SJIS`が追加されたことを確認する。

    ```bash
    $ locale -a
    C
    C.utf8
    POSIX
    ja_JP.sjis
    ja_JP.utf8
    ```

## 参考

- [ARM Compiler armcc User Guide Version 5.06](https://developer.arm.com/documentation/dui0472/m/Compiler-Command-line-Options/--locale-lang-country?lang=en)
  - `--locale`オプションについて解説されているが、エンコーディングごとの細かい設定は記載されておらず微妙に不親切。
