---
title: "RustによるESP32のデュアルコアの使用法"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "ESP32"]
published: false
---

## はじめに
なんとなく、RustでESP32のデュアルコアを触りたいなと思って、色々と調べながらやってみようとしたのですが、記事が全くと言ってよいほどなかったのでまとめました。

一応、私の作成した[リポジトリ](https://github.com/kr-work/dual_core)を載せておきます。


## FreeRTOS
今回、デュアルコアでプログラムを作成するにあたって、ESP32のFreeRTOSを使います。

FreeRTOSについては、[Lang-ship](https://lang-ship.com/blog/work/esp32-freertos-l01-about/)さんなどに詳しく記載されているので、気になった方はそちらを参照してください。

## タスク作成
デュアルコアを扱うには、それぞれにタスクを渡して実行する形をとります。タスクの作成には、C言語から関数を呼び出して使用します。

```Rust
unsafe extern "C" fn task1(_: *mut core::ffi::c_void) {
    loop {
        for i in 0..10 {
            log::info!("Task 1 Entered {}", i);
            FreeRtos::delay_ms(1000);
        }
    }
}
```
もう一つのコアには同様の内容で、task2という関数名を定義しておきます。

## タスクの割り当て
```Rust
static TASK1_NAME: &str = "Task 1";
let task1_name = CString::new(TASK1_NAME)?;

```
上記で、Rustの文字列スライスをCString::newを使用して、C言語の文字列に変換したものをtask1_nameという変数に代入します。

```Rust
    unsafe {
        xTaskCreatePinnedToCore(
            Some(task1),
            task1_name.as_ptr(),
            4096,
            std::ptr::null_mut(),
            10,
            std::ptr::null_mut(),
            0,
        );
    }
```
ここでは、タスクを特定のコアに割り当てています。第一引数には、上で定義したtask1関数を渡します。第二引数では先程のタスク名を、第三引数にはタスクのスタックサイズ、第四引数にはタスクに渡す引数をここで書きます。第五引数はタスクの優先度を、第六引数にはタスクハンドル(タスクの管理や操作等)を格納し、第七引数で使用するCPUのコアを定義します。

もうひとつのタスクの割り当てでは、CPUのコア1に割り当てて二つのコアを動かしています。

### プログラム全体
ソースコードもここに載せておきます。
ふたつのタスクでdelayの時間を変えて、両方のコアが起動していることが確認できます。
```Rust
use esp_idf_hal::delay::FreeRtos;
use esp_idf_sys::{self as _};
use esp_idf_sys::xTaskCreatePinnedToCore;
use std::ffi::CString;
use esp_idf_svc::sys::link_patches;
static TASK1_NAME: &str = "Task 1";
static TASK2_NAME: &str = "Task 2";


unsafe extern "C" fn task1(_: *mut core::ffi::c_void) {
    loop {
        for i in 0..10 {
            log::info!("Task 1 Entered {}", i);
            FreeRtos::delay_ms(1000);
        }
    }
}
unsafe extern "C" fn task2(_: *mut core::ffi::c_void) {
    loop {
        for i in 0..10 {
            log::info!("Task 2 Entered {}", i);
            FreeRtos::delay_ms(3000);
        }
    }
}

fn main() -> anyhow::Result<()> {
    link_patches();

    // Bind the log crate to the ESP Logging facilities
    esp_idf_svc::log::EspLogger::initialize_default();

    // ログレベルを警告レベルに設定
    unsafe {
        esp_idf_sys::esp_log_level_set(
            b"uart\0".as_ptr() as *const _,
            esp_idf_sys::esp_log_level_t_ESP_LOG_WARN,
        );
    }
    
    let task1_name = CString::new(TASK1_NAME)?;
    unsafe {
        xTaskCreatePinnedToCore(
            Some(task1),
            task1_name.as_ptr(),
            4096,
            std::ptr::null_mut(),
            10,
            std::ptr::null_mut(),
            0,
        );
    }
    let task2_name = CString::new(TASK2_NAME)?;
    unsafe {
        xTaskCreatePinnedToCore(
            Some(task2),
            task2_name.as_ptr(),
            4096,
            std::ptr::null_mut(),
            9,
            std::ptr::null_mut(),
            1,
        );
    }
    for i in 0..10 {
        log::info!("Main Entered {}", i);
        FreeRtos::delay_ms(5000);
    }
    Ok(())
}

```

## 余談
本当はupsafeを使いたくないのですが、呼び出しているのがC言語ということもあって、Rustの安全性の担保ができないのでは？と思ってます。
分かり次第、追記します...

