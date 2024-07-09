# 運行時恐慌(Panicking)

運行時恐慌是Rust語言的一個核心部分。像是索引這樣的內建的操作為了存儲安全性是運行時檢查的。當嘗試越界索引時，這會導致運行時恐慌(panic)。

在標準庫中，運行時恐慌的行為被定義成：展開(unwinds)恐慌的線程的棧，除非用戶自己選擇在恐慌時終止程序。

然而在沒有標準庫的程序中，運行時恐慌的行為是未被定義了的。通過聲明一個 `#[painc_handler]` 函數可以選擇一個運行時恐慌的行為。

這個函數在一個程序的依賴圖中必須只出現一次，且必須有這樣的簽名: `fn(&PanicInfo) -> !`，`PanicInfo`是一個包含關於發生運行時恐慌的位置信息的結構體。

[`PanicInfo`]: https://doc.rust-lang.org/core/panic/struct.PanicInfo.html

鑑於嵌入式系統的範圍從面向用戶的系統到安全關鍵系統，沒有一個運行時恐慌行為能滿足所有場景，但是有許多常用的行為。這些常用的行為已經被打包進了一些crates中，這些crates中定義了 `#[panic_handler]`函數。比如:

- [`panic-abort`]. 這個運行時恐慌會導致終止指令被執行。
- [`panic-halt`]. 這個運行時恐慌會導致程序，或者現在的線程，通過進入一個無限循環中而掛起。
- [`panic-itm`]. 運行時恐慌的信息會被ITM記錄，ITM是一個ARM Cortex-M的特殊的外設。
- [`panic-semihosting`]. 使用半主機技術，運行時恐慌的信息被記錄到主機上。

[`panic-abort`]: https://crates.io/crates/panic-abort
[`panic-halt`]: https://crates.io/crates/panic-halt
[`panic-itm`]: https://crates.io/crates/panic-itm
[`panic-semihosting`]: https://crates.io/crates/panic-semihosting

在crates.io上搜索 [`panic-handler`]，你甚至可以找到更多的crates。

[`panic-handler`]: https://crates.io/keywords/panic-handler

僅僅通過鏈接到相關的crate中，一個程序就可以簡單地從這些行為中選擇一個運行時恐慌行為。將運行時恐慌的行為作為一行代碼放進一個應用的源碼中，不僅僅是因為可以作為文檔使用，而且能根據編譯配置改變運行時恐慌的行為。比如:

``` rust,ignore
#![no_main]
#![no_std]

// dev配置: 更容易調試運行時恐慌; 可以在 `rust_begin_unwind` 上放一個斷點
#[cfg(debug_assertions)]
use panic_halt as _;

// release配置: 最小化應用的二進制文件的大小
#[cfg(not(debug_assertions))]
use panic_abort as _;

// ..
```

在這個例子裡，當使用dev配置編譯的時候(`cargo build`)，crate鏈接到 `panic-halt` crate上，但是當使用release配置編譯時(`cargo build --release`)，crate鏈接到`panic-abort` crate上。

> `use panic_abort as _` 形式的 `use` 語句，被用來確保 `panic_abort` 運行時恐慌函數被包含進我們最終的可執行程序裡，同時讓編譯器清楚地知道我們不會從這個crate顯式地使用任何東西。沒有 `_` 重命名，編譯器將會警告我們有一個未使用的導入。有時候你可能會看到 `extern crate panic_abort`，這是Rust 2018之前的版本使用的更舊的寫法，現在應該只被用於 "sysroot" crates (與Rust一起發佈的crates)，比如 `proc_macro`，`alloc`，`std` 和 `test` 。

## 一個例子

這裡有一個嘗試越界訪問數組的例子。操作的結果導致了一個運行時恐慌(panic)。

```rust,ignore
#![no_main]
#![no_std]

use panic_semihosting as _;

use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
    let xs = [0, 1, 2];
    let i = xs.len();
    let _y = xs[i]; // out of bounds access

    loop {}
}
```

這個例子選擇了`panic-semihosting`行為，運行時恐慌的信息會被打印至使用了半主機模式的主機控制檯上。

``` text
$ cargo run
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb (..)
panicked at 'index out of bounds: the len is 3 but the index is 4', src/main.rs:12:13
```

你可以嘗試將行為改成`panic-halt`，確保在這個案例裡沒有信息被打印。
