# 異常

異常和中斷，是處理器用來處理異步事件和致命錯誤(e.g. 執行一個無效的指令)的一種硬件機制。異常意味著搶佔並涉及到異常處理程序，即響應觸發事件的信號的子程序。

`cortex-m-rt` crate提供了一個 [`exception`] 屬性去聲明異常處理程序。

[`exception`]: https://docs.rs/cortex-m-rt-macros/latest/cortex_m_rt_macros/attr.exception.html

``` rust,ignore
// SysTick (System計時器)異常的異常處理函數
#[exception]
fn SysTick() {
    // ..
}
```

除了 `exception` 屬性，異常處理函數看起來和普通函數一樣，但是有一個很大的不同: `exception` 處理函數 *不能* 被軟件調用。在先前的例子中，語句 `SysTick();` 將會導致一個編譯錯誤。

這麼做是有目的的，因為異常處理函數必須具有一個特性: 在異常處理函數中被聲明為`static mut`的變量能被安全(safe)地使用。

``` rust,ignore
#[exception]
fn SysTick() {
    static mut COUNT: u32 = 0;

    // `COUNT` 被轉換到了 `&mut u32` 類型且它用起來是安全的
    *COUNT += 1;
}
```

就像你可能已經知道的那樣，在一個函數里使用`static mut`變量，會讓函數變成[*非可重入函數(non-reentrancy)*](https://en.wikipedia.org/wiki/Reentrancy_(computing))。從多個異常/中斷處理函數，或者從`main`函數和多個異常/中斷處理函數中，直接或者間接地調用一個非可重入(non-reentrancy)函數是未定義的行為。

安全的Rust不能導致未定義的行為出現，所以非可重入函數必須被標記為 `unsafe`。然而，我剛說了`exception`處理函數能安全地使用`static mut`變量。這怎麼可能？因為`exception`處理函數 *不* 能被軟件調用因此重入(reentrancy)不會發生，所以這才變得可能。

> 注意，`exception`屬性，通過將靜態變量封裝進`unsafe`塊中併為我們提供了名字相同的，類型為 `&mut` 的，合適的新變量，轉換了函數中靜態變量的定義。因此我們可以通過 `*` 解引用訪問變量的值而不需要將它們打包進一個 `unsafe` 塊中。

## 一個完整的例子

這裡有個例子，使用系統計時器大概每秒拋出一個 `SysTick` 異常。異常處理函數使用 `COUNT` 變量追蹤它自己被調用了多少次，然後使用半主機模式(semihosting)打印 `COUNT` 的值到主機控制檯上。

> **注意**: 你能在任何Cortex-M設備上運行這個例子;你也能在QEMU運行它。

```rust,ignore
#![deny(unsafe_code)]
#![no_main]
#![no_std]

use panic_halt as _;

use core::fmt::Write;

use cortex_m::peripheral::syst::SystClkSource;
use cortex_m_rt::{entry, exception};
use cortex_m_semihosting::{
    debug,
    hio::{self, HStdout},
};

#[entry]
fn main() -> ! {
    let p = cortex_m::Peripherals::take().unwrap();
    let mut syst = p.SYST;

    // 配置系統的計時器每秒去觸發一個SysTick異常
    syst.set_clock_source(SystClkSource::Core);
    // 這是關於LM3S6965的配置，其有一個12MHz的默認CPU時鐘
    syst.set_reload(12_000_000);
    syst.clear_current();
    syst.enable_counter();
    syst.enable_interrupt();

    loop {}
}

#[exception]
fn SysTick() {
    static mut COUNT: u32 = 0;
    static mut STDOUT: Option<HStdout> = None;

    *COUNT += 1;

    // 惰性初始化(Lazy initialization)
    if STDOUT.is_none() {
        *STDOUT = hio::hstdout().ok();
    }

    if let Some(hstdout) = STDOUT.as_mut() {
        write!(hstdout, "{}", *COUNT).ok();
    }

    // 重要信息 如果運行在真正的硬件上，去掉這個 `if` 塊，
    // 否則你的調試器將會以一種不一致的狀態結束
    if *COUNT == 9 {
        // 這將終結QEMU進程
        debug::exit(debug::EXIT_SUCCESS);
    }
}
```

``` console
tail -n5 Cargo.toml
```

``` toml
[dependencies]
cortex-m = "0.5.7"
cortex-m-rt = "0.6.3"
panic-halt = "0.2.0"
cortex-m-semihosting = "0.3.1"
```

``` text
$ cargo run --release
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb (..)
123456789
```

如果你在Discovery開發板上運行這個例子，你將會在OpenOCD控制檯上看到輸出。還有，當計數到達9的時候，程序將 *會* 停止。

## 默認異常處理函數

`exception` 屬性真正做的是，*覆蓋* 了一個特定異常的默認異常處理函數。如果你不覆蓋一個特定異常的處理函數，它將會被 `DefaultHandler` 函數處理，其默認的是:

``` rust,ignore
fn DefaultHandler() {
    loop {}
}
```

這個函數是 `cortex-m-rt` crate提供的，且被標記為 `#[no_mangle]` 因此你能在 "DefaultHandler" 上放置一個斷點並捕獲 *unhandled* 異常。 

可以使用 `exception` 屬性覆蓋這個 `DefaultHandler`:

``` rust,ignore
#[exception]
fn DefaultHandler(irqn: i16) {
    // 自定義默認處理函數
}
```

`irqn` 參數指出了被服務的是哪個異常。一個負數值指出了被服務的是一個Cortex-M異常;0或者一個正數值指出了被服務的是一個設備特定的異常，也就是中斷。

## 硬錯誤(Hard Fault)處理函數

`HardFault`異常有點特別。當程序進入一個無法工作的狀態時，這個異常被觸發，因此它的處理函數 *不能* 返回，因為這麼做可能導致一個未定義的行為。在用戶定義的 `HardFault` 處理函數被調用之前，運行時crate還做了一些工作以改進調試功能。

結果是，`HardFault`處理函數必須有下列的簽名: `fn(&ExceptionFrame) -> !` 。處理函數的參數是一個指針，它指向被異常推入棧中的寄存器。這些寄存器是異常被觸發那刻，處理器狀態的一個記錄，能被用來分析一個硬錯誤。

這裡有個執行不合法操作的案例: 讀取一個不存在的存儲位置。

> **注意**: 這個程序在QEMU上不能起作用，i.e. 它不會崩潰，因為 `qemu-system-arm -machine lm3s6965evb` 不對讀取存儲的操作進行檢查，且讀取無效存儲時將會開心地返回 `0`。

```rust,ignore
#![no_main]
#![no_std]

use panic_halt as _;

use core::fmt::Write;
use core::ptr;

use cortex_m_rt::{entry, exception, ExceptionFrame};
use cortex_m_semihosting::hio;

#[entry]
fn main() -> ! {
    // 讀取一個無效的存儲位置
    unsafe {
        ptr::read_volatile(0x3FFF_FFFE as *const u32);
    }

    loop {}
}

#[exception]
fn HardFault(ef: &ExceptionFrame) -> ! {
    if let Ok(mut hstdout) = hio::hstdout() {
        writeln!(hstdout, "{:#?}", ef).ok();
    }

    loop {}
}
```

`HardFault`處理函數打印了`ExceptionFrame`值。如果你運行這個，你將會看到下面的東西打印到OpenOCD控制檯上。

``` text
$ openocd
(..)
ExceptionFrame {
    r0: 0x3ffffffe,
    r1: 0x00f00000,
    r2: 0x20000000,
    r3: 0x00000000,
    r12: 0x00000000,
    lr: 0x080008f7,
    pc: 0x0800094a,
    xpsr: 0x61000000
}
```

`pc`值是異常時程序計數器(Program Counter)的值，它指向觸發了異常的指令。

如果你看向程序的反彙編:

``` text
$ cargo objdump --bin app --release -- -d --no-show-raw-insn --print-imm-hex
(..)
ResetTrampoline:
 8000942:       movw    r0, #0xfffe
 8000946:       movt    r0, #0x3fff
 800094a:       ldr     r0, [r0]
 800094c:       b       #-0x4 <ResetTrampoline+0xa>
```

你可以在反彙編中搜索程序計數器`0x0800094a`的值。你將會看到一個讀取操作(`ldr r0, [r0]`)導致了異常。`ExceptionFrame`的`r0`字段將告訴你，那時寄存器`r0`的值是`0x3fff_fffe` 。
