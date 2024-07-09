# 安裝工具
這一頁包含的工具安裝指令與操作系統無關：

### Rust 工具鏈
跟著[https://rustup.rs](https://rustup.rs)的指令安裝rustup。

**注意** 確保你的編譯器版本等於或者大於`1.31`版本。`rustc -V`應該返回一個比下列日期更新的日期。

``` text
$ rustc -V
rustc 1.31.1 (b6c32da9b 2018-12-18)
```
考慮到帶寬和磁盤的使用量，默認的安裝只支持主機環境的編譯。為了添加對ARM Cortex-M架構交叉編譯的支持，從下列編譯目標中選擇一個。對於這本書裡使用的STM32F3DISCOVERY板子，使用`thumbv7em-none-eabihf`作為目標。

Cortex-M0, M0+, 和 M1 (ARMv6-M 架構):
``` console
rustup target add thumbv6m-none-eabi
```

Cortex-M3 (ARMv7-M 架構):
``` console
rustup target add thumbv7m-none-eabi
```

沒有硬件浮點單元的Cortex-M4和M7 (ARMv7E-M架構)
``` console
rustup target add thumbv7em-none-eabi
```

具有硬件浮點單元的Cortex-M4F和M7F (ARMv7E-M架構)
``` console
rustup target add thumbv7em-none-eabihf
```

Cortex-M23 (ARMv8-M架構):
``` console
rustup target add thumbv8m.base-none-eabi
```

Cortex-M33和M35P (ARMv8-M架構):
``` console
rustup target add thumbv8m.main-none-eabi
```

具有硬件浮點單元的Cortex-M33F和M35PF (ARMv8-M架構):
``` console
rustup target add thumbv8m.main-none-eabihf
```


### `cargo-binutils`

``` text
cargo install cargo-binutils

rustup component add llvm-tools-preview
```
WINDOWS: 需要預先安裝 C++ Build Tools for Visual Studio 2019。https://visualstudio.microsoft.com/thank-you-downloading-visual-studio/?sku=BuildTools&rel=16

### `cargo-generate`
我們隨後將使用這個來從模板生成一個項目。

``` console
cargo install cargo-generate
```
注意:在某些Linux發行版上(e.g. Ubuntu) 在安裝cargo-generate之前，你可能需要安裝`libssl-dev`和`pkg-config`

### 特定於操作系統的指令

現在根據你使用的操作系統，來執行對應的指令:

- [Linux](install/linux.md)
- [Windows](install/windows.md)
- [macOS](install/macos.md)
