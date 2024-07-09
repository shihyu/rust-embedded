# 存儲映射的寄存器(Memory-Mapped Registers)

嵌入式系統想要繼續執行下去，只有通過執行常規的Rust代碼並在RAM間移動數據才行。如果我們想要獲取或者發出信息(點亮一個LED，發現一個按鈕按下或者在總線上與芯片外設通信)，我們不得不深入瞭解外設和它們的"存儲映射的寄存器"。

你可能會發現，訪問你的微控制器外設所需要的代碼，已經存在於下面的某個抽象層中了。

<p align="center">
<img title="Common crates" src="../assets/crates.png">
</p>

* Micro-architecture Crate(微架構庫) - 這個庫擁有任何對於微控制器的處理器內核來說經常會用到的程序，也包括在這些微控制器中的通用外設。比如 [cortex-m] crate提供給你可以使能和關閉中斷的函數，其對於所有的Cortex-M微控制器都是一樣的。它也提供你訪問'SysTick'外設的能力，在所有的Cortex-M微控制器中都包括了這個外設功能。
* Peripheral Access Crate(PAC)(外設訪問庫) - 這個庫是對各種存儲器封裝的寄存器再進行的一次淺陋封裝，特定於所使用的微控制器的產品號。比如，[tm4c123x]針對TI的Tiva-C TM4C123系列，[stm32f30x]針對ST的STM32F30x系列。這塊，根據微控制器的技術手冊寫的每個外設操作指令，直接和寄存器交互。
* HAL Crate - 這些crates為你的處理器提供了一個更友好的API，通常是通過實現在[embedded-hal]中定義的一些常用的traits來實現的。比如，這個crate可能提供一個`Serial`結構體，它的構造函數需要一組合適的GPIO端口和一個波特率，它為發送數據提供了 `write_byte` 函數。查看 [可移植性] 可以看到更多關於 [embedded-hal] 的信息。
* Board Crate(開發板庫) - 這些Crate通過預配置不同的外設和GPIO管腳再進行了一層抽象以適配你正在使用的特定的開發者工具或者開發板，比如對於STM32F3DISCOVERY開發板來說，是[stm32f3-discovery]

[cortex-m]: https://crates.io/crates/cortex-m
[tm4c123x]: https://crates.io/crates/tm4c123x
[stm32f30x]: https://crates.io/crates/stm32f30x
[embedded-hal]: https://crates.io/crates/embedded-hal
[可移植性]: ../portability/index.md
[stm32f3-discovery]: https://crates.io/crates/stm32f3-discovery
[Discovery]: https://rust-embedded.github.io/discovery/

## 開發板Crate (Board Crate)

如果你是嵌入式Rust新手，board crate是一個完美的開始。它們很好地抽象出了，在開始學習這個項目時，需要耗費心力瞭解的硬件細節，使得標準工作，像是打開或者關閉LED，變得簡單。不同的板子間，它們提供的功能變化很大。因為這本書是不假設我們使用的是何種板子，所以這本書不會提到board crate。

如果你想要用STM32F3DISCOVERY開發板做實驗，強烈建議看一下[stm32f3-discovery]開發板crate，它提供了閃爍LEDs，訪問它的指南針，藍牙和其它的功能。[Discovery]書對於一個board crate的用法提供一個很好的介紹。

但是如果你正在使用一個還沒有提供專用的board crate的系統，或者你需要的一些功能，現存的crates不提供，那我們需要從底層的微架構crates開始。

## Micro-architecture crate

讓我們看一下SysTick外設，SysTick外設存在於所有的Cortex-M微控制器中。我們能在[cortex-m] crate中找到一個相當底層的API，我們能像這樣使用它：

```rust,ignore
#![no_std]
#![no_main]
use cortex_m::peripheral::{syst, Peripherals};
use cortex_m_rt::entry;
use panic_halt as _;

#[entry]
fn main() -> ! {
    let peripherals = Peripherals::take().unwrap();
    let mut systick = peripherals.SYST;
    systick.set_clock_source(syst::SystClkSource::Core);
    systick.set_reload(1_000);
    systick.clear_current();
    systick.enable_counter();
    while !systick.has_wrapped() {
        // Loop
    }

    loop {}
}
```
`SYST`結構體上的功能，相當接近ARM技術手冊為這個外設定義的功能。在這個API中沒有關於 '延遲X毫秒' 的功能 - 我們不得不通過使用一個 `while` 循環來粗略地實現它。注意，我們調用了`Peripherals::take()`才能訪問我們的`SYST`結構體 - 這是一個特別的程序，保障了在我們的整個程序中只存在一個`SYST`結構體實例，更多的信息可以看[外設]部分。

[外設]: ../peripherals/index.md

## 使用一個外設訪問Crate (PAC)

如果我們把自己只侷限於每個Cortex-M擁有的基本外設，那我們的嵌入式軟件開發將不會走得太遠。我們準備需要寫一些特定於我們正在使用的微控制器的代碼。在這個例子裡，讓我們假設我們有一個TI的TM4C123 - 一個有256KiB Flash的中等規模的80MHz的Cortex-M4。我們用[tm4c123x] crate去使用這個芯片。

```rust,ignore
#![no_std]
#![no_main]

use panic_halt as _; // panic handler

use cortex_m_rt::entry;
use tm4c123x;

#[entry]
pub fn init() -> (Delay, Leds) {
    let cp = cortex_m::Peripherals::take().unwrap();
    let p = tm4c123x::Peripherals::take().unwrap();

    let pwm = p.PWM0;
    pwm.ctl.write(|w| w.globalsync0().clear_bit());
    // Mode = 1 => Count up/down mode
    pwm._2_ctl.write(|w| w.enable().set_bit().mode().set_bit());
    pwm._2_gena.write(|w| w.actcmpau().zero().actcmpad().one());
    // 528 cycles (264 up and down) = 4 loops per video line (2112 cycles)
    pwm._2_load.write(|w| unsafe { w.load().bits(263) });
    pwm._2_cmpa.write(|w| unsafe { w.compa().bits(64) });
    pwm.enable.write(|w| w.pwm4en().set_bit());
}

```

我們訪問 `PWM0` 外設的方法和我們之前訪問 `SYST` 的方法一樣，除了我們調用的是 `tm4c123x::Peripherals::take()` 之外。因為這個crate是使用[svd2rust]自動生成的，訪問我們寄存器位段的函數的參數是一個閉包，而不是一個數值參數。雖然這看起來像是有了更多的代碼，但是Rust編譯器能使用這個閉包為我們執行一系列檢查，且產生的機器碼十分接近手寫的彙編碼！如果自動生成的代碼不能確保某個訪問函數其所有可能的參數都能發揮作用(比如，如果寄存器被SVD定義為32位，但是沒有說明某些32位值是否有特殊作用)，那麼該函數需要被標記為 `unsafe` 。我們能在上面看到這樣的例子，我們使用 `bits()` 函數設置 `load` 和 `compa` 子域。

### Reading

`read()` 函數返回一個對象，這個對象提供了對這個寄存器中不同子域的只讀訪問，由廠商提供的這個芯片的SVD文件定義。在 [tm4c123x documentation][tm4c123x documentation R] 中你能找到在這個特別的返回類型 `R` 上所有可用的函數，其與特定芯片中的特定外設的特定寄存器有關。

```rust,ignore
if pwm.ctl.read().globalsync0().is_set() {
    // Do a thing
}
```

### Writing

`write()`函數使用一個只有一個參數的閉包。通常我們把這個參數叫做 `w`。然後這個參數提供對這個寄存器中不同的子域的讀寫訪問，由廠商關於這個芯片的SVD文件提供。再一次，在 [tm4c123x documentation][tm4c123x documentation W] 中你能找到 `W` 所有可用的函數，其與特定芯片中的特定外設的特定寄存器有關。注意,所有我們沒有設置的子域將會被設置成一個默認值 - 將會丟失任何在這個寄存器中的現存的內容。


```rust,ignore
pwm.ctl.write(|w| w.globalsync0().clear_bit());
```

### Modifying

如果我們希望只改變這個寄存器中某個特定的子域而讓其它子域不變，我們能使用`modify`函數。這個函數使用一個具有兩個參數的閉包 - 一個用來讀取，一個用來寫入。通常我們分別稱它們為 `r` 和 `w` 。 `r` 參數能被用來查看這個寄存器現在的內容，`w` 參數能被用來修改寄存器的內容。

```rust,ignore
pwm.ctl.modify(|r, w| w.globalsync0().clear_bit());
```

`modify` 函數在這裡真正展示了閉包的能量。在C中，我們經常需要讀取一些臨時值，修改成正確的比特，然後再把值寫回。這意味著出現錯誤的範圍非常大。

```C
uint32_t temp = pwm0.ctl.read();
temp |= PWM0_CTL_GLOBALSYNC0;
pwm0.ctl.write(temp);
uint32_t temp2 = pwm0.enable.read();
temp2 |= PWM0_ENABLE_PWM4EN;
pwm0.enable.write(temp); // 哦 不! 錯誤的變量!
```

[svd2rust]: https://crates.io/crates/svd2rust
[tm4c123x documentation R]: https://docs.rs/tm4c123x/0.7.0/tm4c123x/pwm0/ctl/struct.R.html
[tm4c123x documentation W]: https://docs.rs/tm4c123x/0.7.0/tm4c123x/pwm0/ctl/struct.W.html

## 使用一個HAL crate

一個芯片的HAL crate是通過為PAC暴露的基礎結構體們實現一個自定義Trait來發揮作用的。經常這個trait將會為某個外設定義一個被稱作 `constrain()` 的函數，或者為像是有多個管腳的GPIO端口這類東西定義一個`split()`函數。這個函數將會使用基礎的外設結構體，然後返回一個具有更高抽象的API的新對象。這個API還可以做一些事，比如讓Serial port的 `new` 函數變成需要某個`Clock`結構體的函數，這個結構體只能通過調用配置PLLs並設置所有的時鐘頻率的函數來生成。在這時，生成一個Serial port對象而不先配置時鐘速率是不可能的，對於Serial port對象來說錯誤地將波特率轉換為時鐘滴答數也是不會發生的。一些crates甚至為每個GPIO管腳的狀態定義了特定的 traits，在把管腳傳遞進外設前，要求用戶去把一個管腳設置成正確的狀態(通過選擇Alternate Function模式) 。所有這些都沒有運行時開銷的！

讓我們看一個例子:

```rust,ignore
#![no_std]
#![no_main]

use panic_halt as _; // panic handler

use cortex_m_rt::entry;
use tm4c123x_hal as hal;
use tm4c123x_hal::prelude::*;
use tm4c123x_hal::serial::{NewlineMode, Serial};
use tm4c123x_hal::sysctl;

#[entry]
fn main() -> ! {
    let p = hal::Peripherals::take().unwrap();
    let cp = hal::CorePeripherals::take().unwrap();

    // 將SYSCTL結構體封裝成一個有更高抽象API的對象
    let mut sc = p.SYSCTL.constrain();
    // 選擇我們的晶振配置
    sc.clock_setup.oscillator = sysctl::Oscillator::Main(
        sysctl::CrystalFrequency::_16mhz,
        sysctl::SystemClock::UsePll(sysctl::PllOutputFrequency::_80_00mhz),
    );
    // 設置PLL
    let clocks = sc.clock_setup.freeze();

    // 把GPIO_PORTA結構體封裝成一個有更高抽象API的對象
    // 注意它需要借用 `sc.power_control` 因此它能自動開啟GPIO外設。
    let mut porta = p.GPIO_PORTA.split(&sc.power_control);

    // 激活UART
    let uart = Serial::uart0(
        p.UART0,
        // 傳送管腳
        porta
            .pa1
            .into_af_push_pull::<hal::gpio::AF1>(&mut porta.control),
        // 接收管腳
        porta
            .pa0
            .into_af_push_pull::<hal::gpio::AF1>(&mut porta.control),
        // 不需要RTS或者CTS
        (),
        (),
        // 波特率
        115200_u32.bps(),
        // 輸出處理
        NewlineMode::SwapLFtoCRLF,
        // 我們需要時鐘頻率去計算波特率除法器(divisors)
        &clocks,
        // 我們需要這個去啟動UART外設
        &sc.power_control,
    );

    loop {
        writeln!(uart, "Hello, World!\r\n").unwrap();
    }
}
```
