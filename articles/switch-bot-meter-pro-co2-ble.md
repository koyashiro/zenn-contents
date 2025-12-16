---
title: "SwitchBot CO2センサー (温湿度計)のBLEを解析して温湿度とCO2濃度を取得する"
emoji: "🌡 "
type: "tech"
topics: ["switchbot", "ble", "raspberrypi", "rust"]
published: true
published_at: 2025-12-17 09:00
publication_name: team_soda
---

:::message
スニダンを開発しているSODA inc.の [Advent Calendar 2025](https://qiita.com/advent-calendar/2025/soda-inc) 17日目の記事です。
:::

## はじめに

最近、SwitchBotのCO2センサー (温湿度計)を手に入れました。

@[card](https://www.switchbot.jp/products/switchbot-co2-meter)

この製品は温度、湿度、CO2のセンサーが内蔵されており、部屋のコンディションを手軽に測定することができます。

測定値をSwitchBotが提供するクラウド上にアップロードし、過去2年分の記録をスマートフォンの専用アプリから確認することもできます[^hub]。

[^hub]: [SwitchBot ハブ3](https://www.switchbot.jp/products/switchbot-hub3)等の製品と連携させる必要があります。

また、SwitchBotは[Web API](https://github.com/OpenWonderLabs/SwitchBotAPI)を公開しているため、プログラムから測定値を取得することもできます。

しかし、Web APIを使うためにはインターネット接続が必要であり、ローカル環境のみで完結させたい場合には不向きです。

そこで今回はBLE (Bluetooth Low Energy) を使い、ローカル環境で直接測定値を取得してみることにしました。

## 動作環境

Raspberry Pi 4 Model B 4GBを使って検証を実施します。

```shellsession
$ cat /etc/os-release
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
NAME="Debian GNU/Linux"
VERSION_ID="12"
VERSION="12 (bookworm)"
VERSION_CODENAME=bookworm
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
```

```shellsession
$ cargo -V
cargo 1.92.0 (344c4567c 2025-10-21)
$ rustc -V
rustc 1.92.0 (ded5c06cf 2025-12-08)
```

## btleplug クレートでBLEを受信してみる

btleplug クレートを使ってBLEを受信してみます。

@[card](https://github.com/deviceplug/btleplug)

btleplug の他、非同期ランタイムの tokio とServiceDataで使う uuid クレートもインストールします。

```toml:Cargo.toml
[package]
name = "switchbot-meter-pro-co2-ble"
version = "0.1.0"
edition = "2024"

[dependencies]
btleplug = "0.11.8"
tokio = { version = "1.48.0", features = ["full"] }
uuid = "1.19.0"
```

まずは受信したBLEをそのまま出力してみます。

:::message
本記事で使用しているMACアドレス `00:00:5E:00:53:00` は例示用のアドレスです。実際に使用する際は、ご自身のデバイスのMACアドレスに置き換えてください。
:::

```rust:src/main.rs
use std::{collections::HashMap, fmt::Display, time::Duration};

use btleplug::{
    api::{Central as _, Manager as _, Peripheral as _, ScanFilter},
    platform::Manager,
};
use tokio::time::sleep;
use uuid::Uuid;

// 00:00:5E:00:53:00
const TARGET_MAC_ADDRESS: [u8; 6] = [0x00, 0x00, 0x5e, 0x00, 0x53, 0x00];

struct MacAddress<'a>(&'a [u8; 6]);

impl Display for MacAddress<'_> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        for (i, byte) in self.0.iter().enumerate() {
            if i != 0 {
                write!(f, ":")?;
            }
            write!(f, "{:02X}", byte)?;
        }
        Ok(())
    }
}

struct ManufacturerData<'a>(&'a HashMap<u16, Vec<u8>>);

impl Display for ManufacturerData<'_> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{{ ")?;
        for (i, (k, v)) in self.0.iter().enumerate() {
            if i != 0 {
                write!(f, ", ")?;
            }
            write!(f, "0x{:04X}: 0x", k)?;
            for byte in v {
                write!(f, "{:02X}", byte)?;
            }
        }
        write!(f, " }}")?;
        Ok(())
    }
}

struct ServiceData<'a>(&'a HashMap<Uuid, Vec<u8>>);

impl Display for ServiceData<'_> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{{ ")?;
        for (i, (k, v)) in self.0.iter().enumerate() {
            if i != 0 {
                write!(f, ", ")?;
            }
            write!(f, "{}: 0x", k)?;
            for byte in v {
                write!(f, "{:02X}", byte)?;
            }
        }
        write!(f, " }}")?;
        Ok(())
    }
}

#[tokio::main]
async fn main() {
    println!("Creating BLE manager...");
    let manager = Manager::new().await.expect("failed to create BLE manager");

    println!("Getting adapters...");
    let adapters = manager.adapters().await.expect("failed to get adapters");

    println!("Starting BLE scan...");
    let adapter = adapters.first().expect("no BLE adapter found");
    adapter
        .start_scan(ScanFilter::default())
        .await
        .expect("failed to start BLE scan");

    loop {
        sleep(Duration::from_secs(2)).await;

        let peripherals = adapter
            .peripherals()
            .await
            .expect("failed to get peripherals");

        for peripheral in peripherals.iter() {
            if let Ok(Some(properties)) = peripheral.properties().await {
                let mac_address = properties.address.into_inner();

                if mac_address != TARGET_MAC_ADDRESS {
                    continue;
                }

                println!(
                    "\
MacAddress:         {}
ManufacturerData:   {}
ServiceData:        {}
",
                    MacAddress(&mac_address),
                    ManufacturerData(&properties.manufacturer_data),
                    ServiceData(&properties.service_data),
                );
            }
        }
    }
}
```

```shellsession
$ cargo run
   Compiling switch-bot-meter-pro-co2-ble v0.1.0 (/home/koyashiro/github.com/koyashiro/switch-bot-meter-pro-co2-ble)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.16s
     Running `target/debug/switch-bot-meter-pro-co2-ble`
Creating BLE manager...
Getting adapters...
Starting BLE scan...
MacAddress:         00:00:5E:00:53:00
ManufacturerData:   { 0x0969: 0x00005E00530069E402982C0031038700 }
ServiceData:        { 0000fd3d-0000-1000-8000-00805f9b34fb: 0x350064 }
```

ManufacturerData が16バイト、 ServiceData が 3バイトあるようです。

| データ           | キー                                                  | バイト数 | 値                                   |
| ---------------- | ----------------------------------------------------- | -------- | ------------------------------------ |
| ManufacturerData | `0x0969`[^company-id]                                 | 16       | `0x00005E00530069E402982C0031038700` |
| ServiceData      | `0000fd3d-0000-1000-8000-00805f9b34fb`[^service-uuid] | 3        | `0x350064`                           |

[^company-id]: BLE の Manufacturer Specific Data における Company ID です。[Bluetooth SIG の Company Identifiers](https://www.bluetooth.com/specifications/assigned-numbers/) によると、`0x0969` は Qingdao Yeelink Information Technology Co., Ltd. に割り当てられた Company ID であり、SwitchBot デバイスはこの Company ID を使用しています。

[^service-uuid]: Bluetooth Base UUID (`00000000-0000-1000-8000-00805f9b34fb`) に16ビットUUID `0xFD3D` を埋め込んだ形式です。`0xFD3D` は [Bluetooth SIG によって SwitchBot に割り当てられた16ビットUUID](https://www.bluetooth.com/specifications/assigned-numbers/) です。

なお、このときCO2センサー (温湿度計)の液晶には以下の値が表示されていました。

| 項目    | 値     |
| ------- | ------ |
| 温度    | 24.2℃  |
| 湿度    | 44%    |
| CO2濃度 | 903ppm |

ここからは SwitchBot 公式のドキュメントを参考にしながら解析していきます。

@[card](https://github.com/OpenWonderLabs/SwitchBotAPI-BLE)

このリポジトリはしばらく更新が止まっているようで、CO2センサー (温湿度計)に関する直接的な情報はありませんでした。

しかし、一部で共通の仕様もあるようだったので、これをベースに解析していきます。

## ServiceData の解析

まずは3バイトしかない ServiceData から調べていきます。

ドキュメントを見てみると

```
ServiceData:        { 0000fd3d-0000-1000-8000-00805f9b34fb: 0x350064 }
```

> The Byte: 0, Byte: 1 and Byte: 2 are for every Device Type.

<https://github.com/OpenWonderLabs/SwitchBotAPI-BLE/blob/2bd727ecf7c0898b25ac2df58a4886b5930c9138/devicetypes/meter.md#new-broadcast-message>

と記載があり、0バイト目がデバイス識別のための Device type 、2バイト目がバッテリー残量となっているようです。

今回の計測結果では Device type にあたる0バイト目が `0x35`、 バッテリー残量にあたる2バイト目が `0x64` でした。
SwitchBotアプリで確認したバッテリー残量は100%であったため、 BLEに含まれているバッテリー残量 `0x64 = 100%`と一致しています。

## ManufacturerData の解析

次に16バイトある ManufacturerData を調べます。

```
ManufacturerData:   { 0x0969: 0x00005E00530069E402982C0031038700 }
```

ドキュメントの[Outdoor Temperature/Humidity Sensor の記述](https://github.com/OpenWonderLabs/SwitchBotAPI-BLE/blob/2bd727ecf7c0898b25ac2df58a4886b5930c9138/devicetypes/meter.md#outdoor-temperaturehumidity-sensor)が参考になりそうなので、こちらをもとに調べます。

> ```
> # Data from Type: 0xFF (Manufacturer Specific Data)
> temp = ((data[10] & 0x0F) * 0.1 + (data[11] & 0x7F)) * (((data[11] & 0x80) > 0 : 1 : -1);
> humidity = data[12] & 0x7F;
> ```

10~11バイト目から温度を、12バイト目から湿度を計算できるようです。

### 湿度を探す

湿度の計算が比較的シンプルだったので、まずは湿度が何バイト目にあるかを探してみます。
CO2センサー (温湿度計)の液晶に表示されていた湿度は44%だったため、これを元に探してみます。

```
Byte:   0   1   2   3   4   5   6   7   8   9  10  11  12  13  14  15
Value: 00  00  5E  00  53  00  69  E4  02  98  2C  00  31  03  87  00
                                                   ^^
                                                   湿度 (0x2C = 44%)
```

10バイト目に `0x2C` がありました。

10バイト目を湿度と仮定して、次に温度を探します。

### 温度を探す

10バイト目(湿度)の直前、8-9バイト目を使って温度を計算してみます。

```
Byte:   0   1   2   3   4   5   6   7   8   9  10  11  12  13  14  15
Value: 00  00  5E  00  53  00  69  E4  02  98  2C  00  31  03  87  00
                                       ^^  ^^
                                       |   整数部+符号 (0x98)
                                       小数部 (0x02)
```

小数部: `0x02 & 0x0F = 2`
整数部: `0x98 & 0x7F = 24`
符号 : `0x98 & 0x80 = 0x80 (+)`

これらをあわせると、温度は `+24.2` ℃と計算できました。

CO2センサー (温湿度計)の液晶に表示されていた温度24.2℃とも一致しています。

以上から、8-9バイト目が温度と仮定できました。

### CO2濃度を探す

最後にCO2濃度が何バイト目かを探します。

CO2センサー (温湿度計)の液晶に表示されていたCO2濃度は903ppmでした。

903をそのまま16進数にした `0x0387` を探してみると、13~14バイト目に見つかりました。

```
Byte:   0   1   2   3   4   5   6   7   8   9  10  11  12  13  14  15
Value: 00  00  5E  00  53  00  69  E4  02  98  2C  00  31  03  87  00
                                                       ^^^^^^
                                                       CO2濃度 (0x0387 = 903ppm)
```

13-14バイト目がCO2濃度と仮定できました。

## BLEを使って温度、湿度、CO2濃度を計算してみる

ここまでの解析結果をもとに、BLEから温度、湿度、CO2濃度を計算するコードを書いてみます。

```rust:src/main.rs
use std::{fmt::Display, time::Duration};

use btleplug::{
    api::{Central as _, Manager as _, Peripheral as _, ScanFilter},
    platform::Manager,
};
use tokio::time::sleep;

// 00:00:5E:00:53:00
const TARGET_MAC_ADDRESS: [u8; 6] = [0x00, 0x00, 0x5e, 0x00, 0x53, 0x00];

struct MacAddress<'a>(&'a [u8; 6]);

impl Display for MacAddress<'_> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        for (i, byte) in self.0.iter().enumerate() {
            if i != 0 {
                write!(f, ":")?;
            }
            write!(f, "{:02X}", byte)?;
        }
        Ok(())
    }
}

#[tokio::main]
async fn main() {
    println!("Creating BLE manager...");
    let manager = Manager::new().await.expect("failed to create BLE manager");

    println!("Getting adapters...");
    let adapters = manager.adapters().await.expect("failed to get adapters");

    println!("Starting BLE scan...");
    let adapter = adapters.first().expect("no BLE adapter found");
    adapter
        .start_scan(ScanFilter::default())
        .await
        .expect("failed to start BLE scan");

    loop {
        sleep(Duration::from_secs(2)).await;

        let peripherals = adapter
            .peripherals()
            .await
            .expect("failed to get peripherals");

        for peripheral in peripherals.iter() {
            if let Ok(Some(properties)) = peripheral.properties().await {
                let mac_address = properties.address.into_inner();

                if mac_address != TARGET_MAC_ADDRESS {
                    continue;
                }

                let data = match properties.manufacturer_data.get(&0x0969) {
                    Some(d) => d,
                    None => continue,
                };

                let temp = ((data[8] & 0x0f) as f32 * 0.1f32 + (data[9] & 0x7f) as f32)
                    * (if data[9] & 0x80 > 0 { 1f32 } else { -1f32 });
                let humidity = data[10] & 0x7f;
                let co2 = u16::from_be_bytes([data[13], data[14]]);

                println!(
                    "\
MacAddress:         {}
Temperature:        {}℃
Humidity:           {}%
CO2:                {}ppm
",
                    MacAddress(&mac_address),
                    temp,
                    humidity,
                    co2,
                );
            }
        }
    }
}
```

```shellsession
$ cargo run
   Compiling switch-bot-meter-pro-co2-ble v0.1.0 (/home/koyashiro/github.com/koyashiro/switch-bot-meter-pro-co2-ble)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 4.69s
     Running `target/debug/switch-bot-meter-pro-co2-ble`
Creating BLE manager...
Getting adapters...
Starting BLE scan...
MacAddress:         00:00:5E:00:53:00
Temperature:        22.5℃
Humidity:           46%
CO2:                704ppm
```

## おわりに

SwitchBot CO2センサー (温湿度計) のBLEを解析し、温度、湿度、CO2濃度を取得することができました。

BLEを使うことでインターネット接続やクラウドサービスに依存せず、ローカル環境のみで測定値を取得できます。
将来的にはRaspberry Piで定期的にデータを収集してグラフ化したり、CO2濃度が高くなったら換気を促すアラートを上げる、といった活用をしてみたいと考えています。
