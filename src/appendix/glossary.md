# 附錄A: 詞彙表

嵌入式生態系統中充滿了不同的協議，硬件組件，還有許多與生產商相關的東西，它們都使用自己的縮寫和項目名。這個詞彙表嘗試列出它們以便更好理解它們。

### BSP

板級支持的Crate(Board Support Crate)提供為某個特定板子配置的高級接口。它通常依賴一個[HAL](#hal) crate 。在[存儲映射的寄存器那頁](../start/registers.md)有更多細節的描述或者看[這個視頻](https://youtu.be/vLYit_HHPaY)來獲取一個更廣泛的概述。

### FPU

浮點單元(Floating-Point Unit)。一個只運行在浮點數上的'數學處理器'。

### HAL

硬件抽象層(Hardware Abstraction Layer) crate為微控制器的功能和外設提供一個開發者友好的接口。它通常在[Peripheral Access Crate (PAC)](#pac)之上被實現。它可能也會實現來自[`embedded-hal`](https://crates.io/crates/embedded-hal) crate的traits 。在[存儲映射的寄存器那頁](../start/registers.md)上有更多的細節或者看[這個視頻](https://youtu.be/vLYit_HHPaY)獲取一個更廣泛的概述。

### I2C

有時又被稱為 `I²C` 或者 Intere-IC 。它是一種用於在單個集成電路中進行硬件通信的協議。看[這裡][i2c]來獲取更多細節。

[i2c]: https://en.wikipedia.org/wiki/I2c

### PAC

一個外設訪問 Crate (Peripheral Access Crate)提供了對一個微控制器的外設的訪問。它是一個底層的crates且通常從提供的[SVD](#svd)被直接生成，經常使用[svd2rust](https://github.com/rust-embedded/svd2rust/)。[硬件抽象層](#hal)應該依賴這個crate。在[存儲映射的寄存器那頁](../start/registers.md)有更細節的描述或者看[這個視頻](https://youtu.be/vLYit_HHPaY)獲取一個更廣泛的概述。

### SPI

串行外設接口。看[這裡][spi]獲取更多信息。

[spi]: https://en.wikipedia.org/wiki/Serial_peripheral_interface

### SVD

系統視圖描述文件(System View Description)是一個XML文件格式，以程序員視角來描述一個微控制器設備。你能在[the ARM CMSIS documentation site](https://www.keil.com/pack/doc/CMSIS/SVD/html/index.html)上獲取更多信息。

### UART

通用異步收發器。看[這裡][uart]獲取更多信息。

[uart]: https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter

### USART

通用同步異步收發器。看[這裡][usart]獲取更多信息。

[usart]: https://en.wikipedia.org/wiki/Universal_synchronous_and_asynchronous_receiver-transmitter
