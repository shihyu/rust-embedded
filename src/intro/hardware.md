# 熟悉你的硬件

先來熟悉下我們要用的硬件。

## STM32F3DISCOVERY (the "F3")

<p align="center">
<img title="F3" src="../assets/f3.jpg">
</p>

這個板子有什麼？

+ 一個[STM32F303VCT6](https://www.st.com/en/microcontrollers/stm32f303vc.html)微控制器。這個微控制器包含
  + 一個單核的ARM Cortex-M4F 處理器，支持單精度浮點運算，72MHz的最大時鐘頻率。
  + 256 KiB的"Flash"存儲。
  + 48 KiB的RAM
  + 多種多樣的外設，比如計時器，I2C，SPI和USART
  + 通用GPIO和在板子兩側的其它類型的引腳
  + 一個寫著“USB USER”的USB接口
+ 一個位於[LSM303DLHC](https://www.st.com/en/mems-and-sensors/lsm303dlhc.html)芯片上的[加速度計](https://en.wikipedia.org/wiki/Accelerometer)。

+ 一個位於[LSM303DLHC](https://www.st.com/en/mems-and-sensors/lsm303dlhc.html)芯片上的[磁力計](https://en.wikipedia.org/wiki/Magnetometer)。

+ 一個位於[L3GD20](https://www.pololu.com/file/0J563/L3GD20.pdf)芯片上的[陀螺儀](https://en.wikipedia.org/wiki/Gyroscope).

+ 8個擺得像一個指南針形狀的user LEDs。

+ 一個二級微控制器: [STM32F103](https://www.st.com/en/microcontrollers/stm32f103cb.html)。這個微控制器實際上是一個板載編程器/調試器的一部分，與名為“USB ST-LINK”的USB端口相連。

關於所列舉的功能的更多細節和開發板的更多規格請查閱[STMicroelectronics](https://www.st.com/en/evaluation-tools/stm32f3discovery.html)網站。

提醒一句: 如果想要為板子提供外部信號，請小心。微控制器STM32F303VCT6管腳的標稱電壓是3.3伏。更多信息請查看[6.2 Absolute maximum ratings section in the manual](https://www.st.com/resource/en/datasheet/stm32f303vc.pdf)。
