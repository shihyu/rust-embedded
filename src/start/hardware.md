# 硬件

現在你應該有點熟悉工具和開發過程了。在這部分我們將切換到真正的硬件上；步驟非常相似。讓我們深入下去。

## 認識你的硬件

在我們開始之前，你需要了解下你的目標設備的一些特性，因為你將用它們來配置項目:
- ARM 內核。比如 Cortex-M3 。
- ARM 內核包括一個FPU嗎?Cortex-M4**F**和Cortex-M7**F**有。
- 目標設備有多少Flash和RAM？比如 256KiB的Flash和32KiB的RAM。
- Flash和RAM映射在地址空間的什麼位置?比如 RAM通常位於 `0x2000_0000` 地址處。

你可以在你的設備的數據手冊和參考手冊上找到這些信息。

這部分，要使用我們的參考硬件，STM32F3DISCOVERY。這個板子包含一個STM32F303VCT6微控制器。這個微控制器擁有:
- 一個Cortex-M4F核心，它包含一個單精度FPU。
- 位於 0x0800_0000 地址的256KiB的Flash。
- 位於 0x2000_0000 地址的40KiB的RAM。(這裡還有其它的RAM區域，但是為了方便起見，我們將忽略它)。

## 配置

我們將使用一個新的模板實例從零開始。對於新手，請參考[先前的QEMU]章節，瞭解如何在沒有`cargo-generate`的情況下完成配置。

[先前的QEMU]: qemu.md

``` text
$ cargo generate --git https://github.com/rust-embedded/cortex-m-quickstart
 Project Name: app
 Creating project called `app`...
 Done! New project created /tmp/app

$ cd app
```

第一步是在`.cargo/config.toml`中設置一個默認編譯目標。

``` console
tail -n5 .cargo/config.toml
```

``` toml
# Pick ONE of these compilation targets
# target = "thumbv6m-none-eabi"    # Cortex-M0 and Cortex-M0+
# target = "thumbv7m-none-eabi"    # Cortex-M3
# target = "thumbv7em-none-eabi"   # Cortex-M4 and Cortex-M7 (no FPU)
target = "thumbv7em-none-eabihf" # Cortex-M4F and Cortex-M7F (with FPU)
```

我們將使用 `thumbv7em-none-eabihf`，因為它包括了Cortex-M4F內核．
> **注意**：你可能還記得先前的章節，我們必須要安裝所有的目標平臺，這個平臺是一個新的．
> 所以，不要忘了為這個平臺運行安裝步驟 `rustup target add thumbv7em-none-eabihf` ．

第二步是將存儲區域信息(memory region information)輸入`memory.x`。

``` text
$ cat memory.x
/* Linker script for the STM32F303VCT6 */
MEMORY
{
  /* NOTE 1 K = 1 KiBi = 1024 bytes */
  FLASH : ORIGIN = 0x08000000, LENGTH = 256K
  RAM : ORIGIN = 0x20000000, LENGTH = 40K
}
```
> **注意**：如果你因為某些理由，在對某個編譯目標首次編譯後，改變了`memory.x`文件，需要在`cargo build`之前執行`cargo clean`。因為`cargo build`可能不會跟蹤`memory.x`的更新。

我們將再次使用hello示例作為開始，但是首先我們必須做一個小改變。

在`examples/hello.rs`中，確保`debug::exit()`調用被註釋掉了或者移除掉了。它只能用於在QEMU中運行的情況。

```rust,ignore
#[entry]
fn main() -> ! {
    hprintln!("Hello, world!").unwrap();

    // 退出 QEMU
    // 注意 不要在硬件上運行這個；它會打破OpenOCD的狀態
    // debug::exit(debug::EXIT_SUCCESS);

    loop {}
}
```

你可以像你之前做的一樣，使用`cargo build`檢查編譯程序，使用`cargo-binutils`觀察二進制項。`cortex-m-rt`庫可以處理所有讓芯片運行起來所需的魔法，幾乎所有的Cortex-M CPUs都按同樣的方式啟動。

``` console
cargo build --example hello
```

## 調試

調試會看起來有點不一樣。事實上，取決於不同的目標設備，第一步可能看起來不一樣。在這個章節裡，我們將展示，調試一個在STM32F3DISCOVERY上運行的程序，所需要的步驟。這作為一個參考。關於調試有關的設備特定的信息，可以看[the Debugonomicon](https://github.com/rust-embedded/debugonomicon)。

像之前一樣，我們將進行遠程調試，客戶端將是一個GDB進程。不同的是，OpenOCD將是服務器。

像是在[安裝驗證]中做的那樣，把你的筆記本/個人電腦和discovery開發板連接起來，檢查ST-LINK的短路帽是否被安裝了。

[安裝驗證]: ../intro/install/verify.md

在一個終端上運行 `openocd` 連接到你的開發板上的 ST-LINK 。從模板的根目錄運行這個命令；`openocd` 將會選擇 `openocd.cfg` 文件，它指出了所使用的接口文件(interface file)和目標文件(target file)。

``` console
cat openocd.cfg
```

``` text
# Sample OpenOCD configuration for the STM32F3DISCOVERY development board

# Depending on the hardware revision you got you'll have to pick ONE of these
# interfaces. At any time only one interface should be commented out.

# Revision C (newer revision)
source [find interface/stlink.cfg]

# Revision A and B (older revisions)
# source [find interface/stlink-v2.cfg]

source [find target/stm32f3x.cfg]
```

> **注意** 如果你在[安裝驗證]章節中，發現你的discovery開發板是一個更舊的版本，那麼你應該修改你的 `openocd.cfg` 文件，註釋掉 `interface/stlink.cfg`，讓它去使用 `interface/stlink-v2.cfg` 。

``` text
$ openocd
Open On-Chip Debugger 0.10.0
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
Info : auto-selecting first available session transport "hla_swd". To override use 'transport select <transport>'.
adapter speed: 1000 kHz
adapter_nsrst_delay: 100
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
none separate
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : clock speed 950 kHz
Info : STLINK v2 JTAG v27 API v2 SWIM v15 VID 0x0483 PID 0x374B
Info : using stlink api v2
Info : Target voltage: 2.913879
Info : stm32f3x.cpu: hardware has 6 breakpoints, 4 watchpoints
```

在另一個終端，也是從模板的根目錄，運行GDB。

``` text
gdb-multiarch -q target/thumbv7em-none-eabihf/debug/examples/hello
```

**注意**: 像之前一樣，你可能需要另一個版本的gdb而不是`gdb-multiarch`，取決於你在之前的章節安裝了什麼工具。這也可能使用的是`arm-none-eabi-gdb`或者只是`gdb` 。

接下來把GDB連接到OpenOCD，它正在等待一個在端口3333上的TCP鏈接。

``` console
(gdb) target remote :3333
Remote debugging using :3333
0x00000000 in ?? ()
```

接下來使用`load`命令，繼續 *flash*(加載) 程序到微控制器上。

``` console
(gdb) load
Loading section .vector_table, size 0x400 lma 0x8000000
Loading section .text, size 0x1518 lma 0x8000400
Loading section .rodata, size 0x414 lma 0x8001918
Start address 0x08000400, load size 7468
Transfer rate: 13 KB/sec, 2489 bytes/write.
```

程序現在被加載了。這個程序使用半主機模式，因此在我們調用半主機模式之前，我們必須告訴OpenOCD使能半主機。你可以使用 `monitor` 命令，發送命令給OpenOCD 。

``` console
(gdb) monitor arm semihosting enable
semihosting is enabled
```

> 通過調用 `monitor help` 命令，你能看到所有的OpenOCD命令。

像我們之前一樣，使用一個斷點和 `continue` 命令我們可以跳過所有的步驟到 `main` 。

``` console
(gdb) break main
Breakpoint 1 at 0x8000490: file examples/hello.rs, line 11.
Note: automatically using hardware breakpoints for read-only addresses.

(gdb) continue
Continuing.

Breakpoint 1, hello::__cortex_m_rt_main_trampoline () at examples/hello.rs:11
11      #[entry]
```

> **注意** 如果在你使用了上面的`continue`命令後，GDB阻塞住了終端而不是停在了斷點處，你可能需要檢查下`memory.x`文件中的存儲分區的信息，對於你的設備來說是否被正確的設置了起始位置**和**大小 。

使用`step`步進main函數里。

``` console
(gdb) step
halted: PC: 0x08000496
hello::__cortex_m_rt_main () at examples/hello.rs:13
13          hprintln!("Hello, world!").unwrap();
```

在使用了`next`讓函數繼續執行之後，你應該看到 "Hello, world!" 被打印到了OpenOCD控制檯上。

``` text
$ openocd
(..)
Info : halted: PC: 0x08000e6c
Hello, world!
Info : halted: PC: 0x08000d62
Info : halted: PC: 0x08000d64
Info : halted: PC: 0x08000d66
Info : halted: PC: 0x08000d6a
Info : halted: PC: 0x08000a0c
Info : halted: PC: 0x08000d70
Info : halted: PC: 0x08000d72
```
消息只打印一次，然後進入定義在19行的無限循環中: `loop {}`

使用 `quit` 命令，你現在可以退出 GDB 了。

``` console
(gdb) quit
A debugging session is active.

        Inferior 1 [Remote target] will be detached.

Quit anyway? (y or n)
```

現在調試比之前多了點步驟，因此我們要把所有步驟打包進一個名為 `openocd.gdb` 的GDB腳本中。這個文件在 `cargo generate` 步驟中被生成，因此不需要任何修改了。讓我們看一下:

``` console
cat openocd.gdb
```

``` text
target extended-remote :3333

# print demangled symbols
set print asm-demangle on

# detect unhandled exceptions, hard faults and panics
break DefaultHandler
break HardFault
break rust_begin_unwind

monitor arm semihosting enable

load

# start the process but immediately halt the processor
stepi
```

現在運行 `<gdb> -x openocd.gdb target/thumbv7em-none-eabihf/debug/examples/hello` 將會立即把GDB和OpenOCD連接起來，使能半主機，加載程序和啟動進程。

另外，你能將 `<gdb> -x openocd.gdb` 放進一個自定義的 runner 中，使 `cargo run` 能編譯程序並啟動一個GDB會話。這個 runner 在 `.cargo/config.toml` 中，但是它被註釋掉了。

``` console
head -n10 .cargo/config.toml
```

``` toml
[target.thumbv7m-none-eabi]
# uncomment this to make `cargo run` execute programs on QEMU
# runner = "qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel"

[target.'cfg(all(target_arch = "arm", target_os = "none"))']
# uncomment ONE of these three option to make `cargo run` start a GDB session
# which option to pick depends on your system
runner = "arm-none-eabi-gdb -x openocd.gdb"
# runner = "gdb-multiarch -x openocd.gdb"
# runner = "gdb -x openocd.gdb"
```

``` text
$ cargo run --example hello
(..)
Loading section .vector_table, size 0x400 lma 0x8000000
Loading section .text, size 0x1e70 lma 0x8000400
Loading section .rodata, size 0x61c lma 0x8002270
Start address 0x800144e, load size 10380
Transfer rate: 17 KB/sec, 3460 bytes/write.
(gdb)
```
