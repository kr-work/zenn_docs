---
title: "RustによるESP32とのシリアル通信"
emoji: "👻"
type: "tech"
topics: ["Rust", "Esp32", "Tech"]
published: false
---

## はじめに
研究でRustを使用して、ESP32とシリアル通信しようと調べたところ、"serialport"というクレートを用いた手法が出てきました。しかし、それではESP32とシリアル通信はできなかったので、解決方法を本記事にまとめようと思います。

## 必要なクレート
[esp_idf_svc](https://docs.esp-rs.org/esp-idf-svc/esp_idf_svc/hal/index.html)クレートに、シリアル通信する際に必要な規格が揃ってました。
本記事では、uartを使用していきたいと思います。
```sh
cargo add esp_idf_svc
```

### esp_idf_svc
esp_idf_svcクレートは、ESP-IDFをRustで利用するためのラッパーです。ESP-IDFは、ESP32シリーズのマイコン向けの開発フレームワークであり、FreeRTOSがベースとなっています。このFreeRTOSを使って、デュアルコアにタスクを割り当ててマルチタスクを実装するとかいったこともできるのですが、それはまた別の記事に。


## UartDriver
実際にプログラムを書いていくのですが、用いるのがesp_idf_svc::hal::uartモジュール内の[UartDriver](https://docs.esp-rs.org/esp-idf-hal/esp_idf_hal/uart/struct.UartDriver.html)という構造体です。

```Rust
let port: UartDriver<'_> = UartDriver::new(
    peripherals.uart2,
    tx_pin,
    rx_pin,
    Option::<AnyIOPin>::None,
    Option::<AnyIOPin>::None,
    &uart_config,
)?;
```
今回、私はESP32とPC間でuart0を使用しているので、uart2を選択しました。そのため、txピンは17、rxピンは16です。
cts、rtsについては、今回の通信では必要としなかったためNoneにしました。
最後にconfigですが、esp_idf_hal::uart::config内のConfigという構造体の型エイリアスである、esp_idf_hal::uart::UartConfigを使用しました。
```Rust
let uart_config = UartConfig {
    baudrate: Hertz(115_200),
    rx_fifo_size: (SOC_UART_FIFO_LEN * 8) as usize,
    ..Default::default()
};
```
baudrateを115200にしました。rx_fifo_sizeは受信データのバッファであり、今回は[SOC_UART_FIFO_LEN](https://docs.esp-rs.org/esp-idf-sys/esp_idf_sys/constant.SOC_UART_FIFO_LEN.html)を8倍したサイズを採用しました(正直、適当に決めました)。他にも引数はあるのですが、特に詳細な設定は必要なかったので、デフォルトに設定しました。

### プログラム
実際に書いてみたプログラムも掲載しておきます。
```rust
use esp_idf_sys as _;
use anyhow::Result;
use esp_idf_svc::hal::uart::*;
use esp_idf_svc::hal::{
    prelude::*,
    peripherals::Peripherals,
    gpio::*,
    delay::FreeRtos,
};

const GPS_READ_BUFFER_SIZE: usize = 1024;

fn main() -> anyhow::Result<()> {
    link_patches();
    let tx_pin = peripherals.pins.gpio17;
    let rx_pin = peripherals.pins.gpio16;

    let mut read_buf = [0u8; GPS_READ_BUFFER_SIZE];

        let uart_gps_config = UartConfig {
        baudrate: Hertz(115_200),
        rx_fifo_size: (SOC_UART_FIFO_LEN * 8) as usize,
        ..Default::default()
    };

    let port: UartDriver<'_> = UartDriver::new(
        peripherals.uart2,
        tx_pin,
        rx_pin,
        Option::<AnyIOPin>::None,
        Option::<AnyIOPin>::None,
        &uart_gps_config,
    )?;

    loop {
    // log::info!("Reading from UART...");
    match serial_data.read_serial(&port, &mut read_buf) {
        Ok(data) => {
        log::info!("Data: {:?}", data);
        },
        Err(e) => {
        log::info!("Error: {}", e);
        }
    }
    FreeRtos::delay_ms(50);
    log::info!("Writing to UART...");
    match serial_data.write_serial(&port) {
        Ok(_) => (),
        Err(e) => log::error!("Failed to write to UART: {:?}", e)
    }
    }
}

```