# QEMU
我們將開始為[LM3S6965]編寫程序，一個Cortex-M3微控制器。因為它能使用[QEMU仿真](https://wiki.qemu.org/Documentation/Platforms/ARM#Supported_in_qemu-system-arm)，所以我們選擇它作為我們的第一個目標，本節中，不需要使用硬件，我們注意力可以集中在工具和開發過程上。

[LM3S6965]: http://www.ti.com/product/LM3S6965

**重要**
在這個引導裡，我們將使用"app"這個名字來代指項目名。無論何時你看到單詞"app"，你應該用你選擇的項目名來替代"app"。或者你也可以選擇把你的項目命名為"app"，避免要替換掉。

## 生成一個非標準的 Rust program
我們將使用[`cortex-m-quickstart`]項目模板來生成一個新項目。生成的項目將包含一個最基本的應用:對於一個新的嵌入式rust應用來說，是一個很好的開始。另外，項目將包含一個`example`文件夾，文件夾中有許多獨立的應用，突出了一些關鍵的嵌入式rust的功能。

[`cortex-m-quickstart`]: https://github.com/rust-embedded/cortex-m-quickstart

### 使用 `cargo-generate`
首先安裝 cargo-generate
```console
cargo install cargo-generate
```
然後生成一個新項目
```console
cargo generate --git https://github.com/rust-embedded/cortex-m-quickstart
```

```text
 Project Name: app
 Creating project called `app`...
 Done! New project created /tmp/app
```

```console
cd app
```

### 使用 `git`

克隆倉庫

```console
git clone https://github.com/rust-embedded/cortex-m-quickstart app
cd app
```

然後補充`Cargo.toml`文件中的佔位符

```toml
[package]
authors = ["{{authors}}"] # "{{authors}}" -> "John Smith"
edition = "2018"
name = "{{project-name}}" # "{{project-name}}" -> "app"
version = "0.1.0"

# ..

[[bin]]
name = "{{project-name}}" # "{{project-name}}" -> "app"
test = false
bench = false
```

### 要麼使用

抓取最新的 `cortex-m-quickstart` 模板，解壓它。

```console
curl -LO https://github.com/rust-embedded/cortex-m-quickstart/archive/master.zip
unzip master.zip
mv cortex-m-quickstart-master app
cd app
```

或者你可以瀏覽[`cortex-m-quickstart`]，點擊綠色的 "Clone or download" 按鈕，然後點擊 "Download ZIP" 。

然後像在 “使用 `git`” 那裡的第二部分寫的那樣填充 `Cargo.toml` 。

## 項目概覽

這是`src/main.rs`中源碼最重要的部分。

```rust,ignore
#![no_std]
#![no_main]

use panic_halt as _;

use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
    loop {
        // your code goes here
    }
}
```

這個程序與標準Rust程序有一點不同，讓我們走近點看看。

`#![no_std]`指出這個程序將 *不會* 鏈接標準crate`std`。反而它將會鏈接到它的子集: `core` crate。

`#![no_main]`指出這個程序將不會使用標準的且被大多數Rust程序使用的`main`接口。使用`no_main`的主要理由是，在`no_std`上下文中使用`main`接口需要 nightly 版的 Rust。

`use panic_halt as _;`。這個crate提供了一個`panic_handler`，它定義了程序陷入`panic`時的行為。我們將會在這本書的[運行時恐慌(Panicking)](panicking.md)章節中覆蓋更多的細節。

[`#[entry]`][entry] 是一個由[`cortex-m-rt`]提供的屬性，它用來標記程序的入口。當我們不使用標準的`main`接口時，我們需要其它方法來指示程序的入口，那就是`#[entry]`。

[entry]: https://docs.rs/cortex-m-rt-macros/latest/cortex_m_rt_macros/attr.entry.html
[`cortex-m-rt`]: https://crates.io/crates/cortex-m-rt

`fn main() -> !`。我們的程序將會是運行在目標板子上的 *唯一* 的進程，因此我們不想要它結束！我們使用一個[發散函數](https://doc.rust-lang.org/rust-by-example/fn/diverging.html) (函數簽名中的 `-> !` )來確保在編譯時就是這麼回事兒。

## 交叉編譯

下一步是為Cortex-M3架構*交叉*編譯程序。如果你知道編譯目標(`$TRIPLE`)應該是什麼，運行`cargo build --target $TRIPLE`就可以了。幸運地，模板中的`.cargo/config.toml`有這個答案:
```console
tail -n6 .cargo/config.toml
```
```toml
[build]
# Pick ONE of these compilation targets
# target = "thumbv6m-none-eabi"    # Cortex-M0 and Cortex-M0+
target = "thumbv7m-none-eabi"    # Cortex-M3
# target = "thumbv7em-none-eabi"   # Cortex-M4 and Cortex-M7 (no FPU)
# target = "thumbv7em-none-eabihf" # Cortex-M4F and Cortex-M7F (with FPU)
```
為了交叉編譯Cortex-M3架構我們不得不使用`thumbv7m-none-eabi`。當安裝Rust工具時，target不會自動被安裝，如果還沒有添加，現在可以去添加那個target到工具鏈上。
``` console
rustup target add thumbv7m-none-eabi
```
因為`thumbv7m-none-eabi`編譯目標在你的`.cargo/config.toml`中被設置成默認值，下面的兩個命令是一樣的效果:
```console
cargo build --target thumbv7m-none-eabi
cargo build
```

## 檢查

現在在`target/thumbv7m-none-eabi/debug/app`中有一個非主機環境的ELF二進制文件。我們能使用`cargo-binutils`檢查它。

使用`cargo-readobj`我們能打印ELF頭，確認這是一個ARM二進制。

```console
cargo readobj --bin app -- --file-headers
```

注意:
* `--bin app` 是一個用來查看二進制項`target/$TRIPLE/debug/app`的語法糖
* `--bin app` 需要時也會重新編譯二進制項。

``` text
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0x0
  Type:                              EXEC (Executable file)
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x405
  Start of program headers:          52 (bytes into file)
  Start of section headers:          153204 (bytes into file)
  Flags:                             0x5000200
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         2
  Size of section headers:           40 (bytes)
  Number of section headers:         19
  Section header string table index: 18
```

`cargo-size` 能打印二進制項的linker section的大小。

```console
cargo size --bin app --release -- -A
```

我們使用`--release`查看優化後的版本

``` text
app  :
section             size        addr
.vector_table       1024         0x0
.text                 92       0x400
.rodata                0       0x45c
.data                  0  0x20000000
.bss                   0  0x20000000
.debug_str          2958         0x0
.debug_loc            19         0x0
.debug_abbrev        567         0x0
.debug_info         4929         0x0
.debug_ranges         40         0x0
.debug_macinfo         1         0x0
.debug_pubnames     2035         0x0
.debug_pubtypes     1892         0x0
.ARM.attributes       46         0x0
.debug_frame         100         0x0
.debug_line          867         0x0
Total              14570
```

> ELF linker sections的複習
>
> - `.text` 包含程序指令
> - `.rodata` 包含像是字符串這樣的常量
> - `.data` 包含靜態分配的初始值*非*零的變量
> - `.bss` 也包含靜態分配的初始值*是*零的變量
> - `.vector_table` 是一個我們用來存儲向量(中斷)表的*非*標準的section
> - `.ARM.attributes` 和 `.debug_*` sections包含元數據，當燒錄二進制文件時，它們不會被加載到目標上。

**重要**: ELF文件包含像是調試信息這樣的元數據，因此它們在*硬盤上的尺寸*沒有正確地反應處程序被燒錄到設備上時將佔據的空間的大小。要*一直*使用`cargo-size`檢查一個二進制項的大小。

`cargo-objdump` 能用來反編譯二進制項。

```console
cargo objdump --bin app --release -- --disassemble --no-show-raw-insn --print-imm-hex
```

> **注意** 如果上面的命令抱怨 `Unknown command line argument` 看下面的bug報告:https://github.com/rust-embedded/book/issues/269

> **注意** 在你的系統上這個輸出可能不一樣。rustc, LLVM 和庫的新版本能產出不同的彙編。我們截取了一些指令

```text
app:  file format ELF32-arm-little

Disassembly of section .text:
main:
     400: bl  #0x256
     404: b #-0x4 <main+0x4>

Reset:
     406: bl  #0x24e
     40a: movw  r0, #0x0
     < .. 截斷了更多的指令 .. >

DefaultHandler_:
     656: b #-0x4 <DefaultHandler_>

UsageFault:
     657: strb  r7, [r4, #0x3]

DefaultPreInit:
     658: bx  lr

__pre_init:
     659: strb  r7, [r0, #0x1]

__nop:
     65a: bx  lr

HardFaultTrampoline:
     65c: mrs r0, msp
     660: b #-0x2 <HardFault_>

HardFault_:
     662: b #-0x4 <HardFault_>

HardFault:
     663: <unknown>
```

## 運行

接下來，讓我們看一個嵌入式程序是如何在QEMU上運行的！此刻我們將使用 `hello` 示例，來做些真正的事。

為了方便起見，這是`examples/hello.rs`的源碼:

```rust,ignore
//! 使用semihosting在主機調試臺上打印 "Hello, world!"

#![no_main]
#![no_std]

use panic_halt as _;

use cortex_m_rt::entry;
use cortex_m_semihosting::{debug, hprintln};

#[entry]
fn main() -> ! {
    hprintln!("Hello, world!").unwrap();

    // 退出 QEMU
    // NOTE 不要在硬件上運行這個;它會打破OpenOCD的狀態
    debug::exit(debug::EXIT_SUCCESS);

    loop {}
}
```

這個程序使用被叫做semihosting的東西去打印文本到主機調試臺上。當使用的是真實的硬件時，需要一個調試對話這個程序才能工作，但是當使用的是QEMU時這就可以工作了。

讓我們開始編譯示例

```console
cargo build --example hello
```

輸出的二進制項將位於`target/thumbv7m-none-eabi/debug/examples/hello`。

為了在QEMU上運行這個二進制項，執行下列的命令:

```console
qemu-system-arm \
  -cpu cortex-m3 \
  -machine lm3s6965evb \
  -nographic \
  -semihosting-config enable=on,target=native \
  -kernel target/thumbv7m-none-eabi/debug/examples/hello
```

```text
Hello, world!
```

這個命令應該在打印文本之後成功地退出 (exit code = 0)。你可以使用下列的指令檢查下:

```console
echo $?
```

```text
0
```

讓我們看看QEMU命令:

+ `qemu-system-arm`。這是QEMU仿真器。這些QEMU二進制項有一些變體，這個仿真器能做ARM機器的全系統仿真。

+ `-cpu cortex-m3`。這告訴QEMU去仿真一個Cortex-M3 CPU。指定CPU模型會讓我們捕捉到一些誤編譯錯誤:比如，運行一個為Cortex-M4F編譯的程序，它具有一個硬件FPU，在執行時將會使QEMU報錯。
+ `-machine lm3s6965evb`。這告訴QEMU去仿真 LM3S6965EVB，一個包含LM3S6965微控制器的評估板。
+ `-nographic`。這告訴QEMU不要啟動它的GUI。
+ `-semihosting-config (..)`。這告訴QEMU使能半主機模式。半主機模式允許被仿真的設備，使用主機的stdout，stderr，和stdin，並在主機上創建文件。
+ `-kernel $file`。這告訴QEMU在仿真機器上加載和運行哪個二進制項。

輸入這麼長的QEMU命令太費功夫了！我們可以設置一個自定義運行器(runner)簡化步驟。`.cargo/config.toml` 有一個被註釋掉的，可以調用QEMU的運行器。讓我們去掉註釋。
```console
head -n3 .cargo/config.toml
```
```toml
[target.thumbv7m-none-eabi]
# uncomment this to make `cargo run` execute programs on QEMU
runner = "qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel"
```
這個運行器只會應用於 `thumbv7m-none-eabi` 目標，它是我們的默認編譯目標。現在 `cargo run` 將會編譯程序且在QEMU上運行它。
```console
cargo run --example hello --release
```
```text
   Compiling app v0.1.0 (file:///tmp/app)
    Finished release [optimized + debuginfo] target(s) in 0.26s
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel target/thumbv7m-none-eabi/release/examples/hello`
Hello, world!
```

## 調試
對於嵌入式開發來說，調試非常重要。讓我們來看下如何調試它。

因為我們想要調試的程序所運行的機器上並沒有運行一個調試器程序(GDB或者LLDB)，所以調試一個嵌入式設備就涉及到了 *遠程* 調試

遠程調試涉及一個客戶端和一個服務器。在QEMU的情況中，客戶端將是一個GDB(或者LLDM)進程且服務器將會是運行著嵌入式程序的QEMU進程。

在這部分，我們要使用我們已經編譯的 `hello` 示例。

調試的第一步是在調試模式中啟動QEMU：

```console
qemu-system-arm \
  -cpu cortex-m3 \
  -machine lm3s6965evb \
  -nographic \
  -semihosting-config enable=on,target=native \
  -gdb tcp::3333 \
  -S \
  -kernel target/thumbv7m-none-eabi/debug/examples/hello
```

這個命令將不打印任何東西到調試臺上，且將會阻塞住終端。此刻我們還傳遞了兩個額外的標誌。
+ `-gdb tcp::3333`。這告訴QEMU在3333的TCP端口上等待一個GDB連接。
+ `-S`。這告訴QEMU在啟動時，凍結機器。沒有這個，在我們有機會啟動調試器之前，程序有可能已經到達了主程序的底部了!

接下來我們在另一個終端啟動GDB，且告訴它去加載示例的調試符號。
```console
gdb-multiarch -q target/thumbv7m-none-eabi/debug/examples/hello
```

**注意**: 你可能需要另一個gdb版本而不是 `gdb-multiarch`，取決於你在安裝章節中安裝了哪個。這個可能是 `arm-none-eabi-gdb` 或者只是 `gdb`。

然後在GDB shell中，我們連接QEMU，QEMU正在等待一個在3333 TCP端口上的連接。
```console
target remote :3333
```

```text
Remote debugging using :3333
Reset () at $REGISTRY/cortex-m-rt-0.6.1/src/lib.rs:473
473     pub unsafe extern "C" fn Reset() -> ! {
```

你將看到，進程被掛起了，程序計數器正指向一個名為 `Reset` 的函數。那是 reset 句柄：Cortex-M 內核在啟動時執行的中斷函數。

> 注意在一些配置中，可能不會像上面一樣，顯示`Reset() at $REGISTRY/cortex-m-rt-0.6.1/src/lib.rs:473`，gdb可能打印一些警告，比如:
>
>`core::num::bignum::Big32x40::mul_small () at src/libcore/num/bignum.rs:254`
> `    src/libcore/num/bignum.rs: No such file or directory.`
>
> 那是一個已知的小bug，你可以安全地忽略這些警告，你非常大可能已經進入Reset()了。

這個reset句柄最終將調用我們的主函數，讓我們使用一個斷點和`continue`命令跳過所有的步驟。為了設置斷點，讓我們首先看下我們想要在我們代碼哪裡打斷點，使用`list`指令

```console
list main
```
這將顯示從examples/hello.rs文件來的源代碼。
```text
6       use panic_halt as _;
7
8       use cortex_m_rt::entry;
9       use cortex_m_semihosting::{debug, hprintln};
10
11      #[entry]
12      fn main() -> ! {
13          hprintln!("Hello, world!").unwrap();
14
15          // exit QEMU
```
我們想要在"Hello, world!"之前添加一個斷點，在13行那裡。我們可以使用`break`命令

```console
break 13
```

我們現在能使用`continue`命令指示gdb運行到我們的主函數。

```console
continue
```

```text
Continuing.

Breakpoint 1, hello::__cortex_m_rt_main () at examples\hello.rs:13
13          hprintln!("Hello, world!").unwrap();
```

我們現在靠近打印"Hello, world!"的代碼。讓我們使用`next`命令繼續前進。

``` console
next
```

```text
16          debug::exit(debug::EXIT_SUCCESS);
```

在這裡，你應該看到 "Hello, world!" 被打印到正在運行 `qemu-system-arm` 的終端上。

```text
$ qemu-system-arm (..)
Hello, world!
```

再次調用`next`將會終止QEMU進程。

```console
next
```

```text
[Inferior 1 (Remote target) exited normally]
```

你現在能退出GDB的會話了。

``` console
quit
```
