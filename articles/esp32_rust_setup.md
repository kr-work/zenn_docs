---
title: "ESP32をRustで使用するための環境構築"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "ESP32"]
published: false
---

## はじめに
自分がESP32をRustで動かす際に、環境構築で困ったところがあったため、備忘録として置いておきます。なお、下記のコマンドの実行環境はUbuntu22.04です。
(WindowsだとC関係の環境構築も必要となり、面倒なのが分かったのでやってないです)

## 環境構築
### Rustのインストール
最初にRustのインストールを行います。
```shell
sudo apt install git python3 python3-pip gcc build-essential curl pkg-config libudev-dev libtinfo5
```

### ツールチェイン等のインストール
自分が環境構築を行った際は、"cargo install cargo-espflash"と叩くと、エラーが起きてどうやってもインストールできませんでした。そのため、バイナリでインストールする方法をとって解決する方法を見つけたので、下記にまとめておきます(この備忘録のメインです)。
```shell
pip install esptool
cargo install ldproxy
cargo install espup
cargo install espflash
cargo install cargo-binstall
cargo binstall cargo-espflash
cargo install cargo-generate
```

## ESP32用のプロジェクトの作成
プロジェクト作成の際は

[no_std]
```shell
cargo generate --git https://github.com/esp-rs/esp-template
```
[std]
```shell
cargo generate --git https://github.com/esp-rs/esp-idf-template cargo
```
の二つがありますが、基本的にはstd推奨ですかね
