# 半主機模式

半主機模式是一種可以讓嵌入式設備在主機上進行I/O操作的的機制，主要被用來記錄信息到主機控制檯上。半主機模式需要一個debug會話，除此之外幾乎沒有其它要求了，因此它非常易於使用。缺點是它非常慢：每個寫操作需要幾毫秒的時間，其取決於你的硬件調試器(e.g. ST-LINK)。

[`cortex-m-semihosting`] crate 提供了一個API去在Cortex-M設備上執行半主機操作。下面的程序是"Hello, world!"的半主機版本。

[`cortex-m-semihosting`]: https://crates.io/crates/cortex-m-semihosting

```rust,ignore
#![no_main]
#![no_std]

use panic_halt as _;

use cortex_m_rt::entry;
use cortex_m_semihosting::hprintln;

#[entry]
fn main() -> ! {
    hprintln!("Hello, world!").unwrap();

    loop {}
}
```

如果你在硬件上運行這個程序，你將會在OpenOCD的logs中看到"Hello, world!"信息。

``` text
$ openocd
(..)
Hello, world!
(..)
```

你首先需要從GDB使能OpenOCD中的半主機模式。
``` console
(gdb) monitor arm semihosting enable
semihosting is enabled
```

QEMU理解半主機操作，因此上面的程序不需要啟動一個debug會話，也能在`qemu-system-arm`中工作。注意你需要傳遞`-semihosting-config`標誌給QEMU去使能支持半主機模式；這些標識已經被包括在模板的`.cargo/config.toml`文件中了。

``` text
$ # this program will block the terminal
$ cargo run
     Running `qemu-system-arm (..)
Hello, world!
```

`exit`半主機操作也能被用於終止QEMU進程。重要：**不要**在硬件上使用`debug::exit`；這個函數會關閉你的OpenOCD對話，這樣你就不能執行其它的程序調試操作了，除了重啟它。

```rust,ignore
#![no_main]
#![no_std]

use panic_halt as _;

use cortex_m_rt::entry;
use cortex_m_semihosting::debug;

#[entry]
fn main() -> ! {
    let roses = "blue";

    if roses == "red" {
        debug::exit(debug::EXIT_SUCCESS);
    } else {
        debug::exit(debug::EXIT_FAILURE);
    }

    loop {}
}
```

``` text
$ cargo run
     Running `qemu-system-arm (..)

$ echo $?
1
```

最後一個提示：你可以將運行時恐慌(panicking)的行為設置成 `exit(EXIT_FAILURE)`。這會允許你編寫可以在QEMU上運行通過的 `no_std` 測試。

為了方便，`panic-semihosting` crate有一個 "exit" 特性。當它使能的時候，在主機stderr上打印恐慌(painc)信息後會調用 `exit(EXIT_FAILURE)` 。

```rust,ignore
#![no_main]
#![no_std]

use panic_semihosting as _; // features = ["exit"]

use cortex_m_rt::entry;
use cortex_m_semihosting::debug;

#[entry]
fn main() -> ! {
    let roses = "blue";

    assert_eq!(roses, "red");

    loop {}
}
```

``` text
$ cargo run
     Running `qemu-system-arm (..)
panicked at 'assertion failed: `(left == right)`
  left: `"blue"`,
 right: `"red"`', examples/hello.rs:15:5

$ echo $?
1
```

**注意**: 為了在`panic-semihosting`上使能這個特性，編輯你的`Cargo.toml`依賴，`panic-semihosting`改寫成:

``` toml
panic-semihosting = { version = "VERSION", features = ["exit"] }
```

`VERSION`是想要的版本。關於依賴features的更多信息查看Cargo book的[`specifying dependencies`]部分。

[`specifying dependencies`]:
https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html
