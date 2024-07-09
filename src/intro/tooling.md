# 工具
與微控制器打交道需要使用幾種不同的工具，因為我們要處理的架構與筆記本電腦不同，我們必須在 *遠程* 設備上運行和調試程序。我們將使用下面列舉出來的工具。當沒有指定一個最小版本時，最新的版本應該也可以用，但是我們還是列出了我們已經測過的那些版本。
- Rust 1.31, 1.31-beta, 或者一個更新的，支持ARM Cortex-M編譯的工具鏈。
- [`cargo-binutils`](https://github.com/rust-embedded/cargo-binutils) ~0.1.4
- [`qemu-system-arm`](https://www.qemu.org/). 測試的版本: 3.0.0
- OpenOCD >=0.8. 測試的版本: v0.9.0 and v0.10.0
- 有ARM支持的GDB。強烈建議7.12或者更新的版本。測試版本: 7.10, 7.11 和 8.1
- [`cargo-generate`](https://github.com/ashleygwilliams/cargo-generate) 或者 `git`。這些工具都是可選的，但是跟著這本書來使用它們，會更容易。

下面的文檔將解釋我們為什麼使用這些工具。安裝指令可以在下一頁找到。

## `cargo-generate` 或者 `git`
裸機編程是非標準Rust編程，為了得到正確的程序的內存佈局，需要對鏈接過程進行一些調整，這要求添加一些額外的文件(比如linker scripts)和配置(比如linker flags)。我們已經為你把這些打包進了一個模板裡了，你只需要補充缺失的信息(比如項目名和目標硬件的特性)。<br>
我們的模板兼容`cargo-generate`:一個用來從模板生成新的Cargo項目的Cargo子命令。你也能使用`git`,`curl`,`wget`,或者你的網頁瀏覽器下載模板。

## `cargo-binutils`
`cargo-binutils`是一個Cargo命令的子集，它讓我們能輕鬆使用Rust工具鏈帶來的LLVM工具。這些工具包括LLVM版本的`objdump`，`nm`和`size`，用來查看二進制文件。<br>
在GNU binutils之上使用這些工具的好處是，(a)無論你的操作系統是什麼，安裝這些LLVM工具都可以用同一條命令(`rustup component add llvm-tools-preview`)。(b)像是`objdump`這樣的工具，支持所有`rustc`支持的架構--從ARM到x86_64--因為它們都有一樣的LLVM後端。

## `qemu-system-arm`

QEMU是一個仿真器。在這個例子裡，我們使用能完全仿真ARM系統的改良版QEMU。我們使用QEMU在主機上運行嵌入式程序。多虧了它，你可以在沒有任何硬件的情況下，嘗試這本書的部分示例。

# 用於調試嵌入式Rust的工具

## 概述

在Rust中調試嵌入式系統需要用到專業的工具，這包括用於管理調試進程的軟件，用於觀察和控制程序執行的調試器，和用於便捷主機和嵌入式設備之間進行交互的硬件探測器．這個文檔會介紹像是Probe-rs和OpenOCD這樣的基礎軟件，以及像是GDB和Probe-rs Visual Studio Code擴展這樣常見的調試器．另外，該文檔會覆蓋像是Rusty-probe，ST-Link，J-Link，和MCU-Link這樣的硬件探測器，它們整合在一起可以高效地對嵌入式設備進行調試和編程．

## 驅動調試工具的軟件

### Probe-rs

Probe-rs是一個現代化的，以Rust開發的軟件，被設計用來配合嵌入式系統中的調試器一起工作．不像OpenOCD，Probe-rs設計的時候就考慮到了簡單性，目標是減少在其它調試解決方案中常見的配置重擔．
它支持不同的探測器和目標架構，提供一個用於與嵌入式硬件交互的高層接口．Probe-rs直接集成了Rust工具鏈，並且通過擴展集成進了Visual Studio Code中，允許開發者精簡它們的調試工作流程．


### OpenOCD (Open On-Chip Debugger)

OpenOCD是一個用於調試，測試，和編程嵌入式系統的開源軟件工具．它提供了一個主機系統和嵌入式硬件之間的接口，支持不同的傳輸層，比如JTAG和SWD（Serial Wire Debug）．OpenOCD集成了GDB，其是一個調試器．OpenOCD受到了廣泛的支持，擁有大量的文檔和一個龐大的社區，但是配置可能會很複雜，特別是對於自定義的嵌入式設置．

## Debuggers

調試器允許開發者觀察和控制一個程序的執行，以辨別和糾正錯誤或者bugs．它提供像是設置斷點，一行一行地步進代碼，和研究變量的值以及內存的狀態等功能．調試器本質上是為了通過軟件開發和維護，使得開發者可以確保他們的代碼的行為在不同環境下就像他們預期的那樣運行．

調試器可以知道如何：
 * 與映射到存儲上的寄存器交互．
 * 設置斷點．
 * 讀取和寫入映射到存儲上的寄存器．
 * 檢測什麼時候MCU因為一個調試時間被掛了起來．
 * 在遇到一個調試事件後繼續MCU的執行．
 * 擦出和寫入微控制器的FLASH．

### Probe-rs Visual Studio Code Extension

Probe-rs有一個Visual Studio Code的擴展，提供了不需要額外設置的無縫的調試體驗．通過它的幫助，開發者可以使用Rust特定的特性，像是漂亮的打印和詳細的錯誤信息，確保它們的調試過程可以與Rust的生態對齊． 

### GDB (GNU Debugger) 

GDB是一個多用途的調試工具，其允許開發者研究程序的狀態，無論其正在運行中還是程序崩潰後．對於嵌入式Rust，GDB通過OpenOCD或者其它的調試服務器鏈接到目標系統上去和嵌入式代碼交互．GDB是高度可配置的，並且支持像是遠程調試，變量檢測，和條件斷點．它可以被用於多個平臺，並對Rust特定的調試需求有廣泛的支持，比如好看的打印和與IDEs集成．


## 探測器

硬件探頭是一個被用於嵌入式系統的開發和調試的設備，其可以使得主機和目標嵌入式設備間的通信變得簡單．它通常支持像是JTAG或者SWD這樣的協議，可以編程，調試和分析嵌入式系統上的微控制器或者微處理器．硬件探頭對於要設置斷點，步進代碼，和觀察內存與處理器的寄存器的開發者來說很重要，可以讓開發者們高效地實時地分析和修復問題．

### Rusty-probe

Rusty-probe是一個開源的基於USB的硬件調試探測器，被設計用來輔助probe-rs一起工作．Rusy-Probe和probe-rs的結合為嵌入式Rust應用的開發者提供了一個易用的，成本高效的解決方案．

### ST-Link

ST-Link是一個由STMicroelectronics開發的常見的調試和編程探測器，其主要用於它們的STM32和STM8微控制器系列．它支持通過JTAG或者SWD接口進行調試和編程．因為STMicroelectronics的大量的開發板對其直接支持並且它集成進了主流的IDEs中，所以使得它成為使用STM微控制器的開發者的首選．

### J-Link

J-Link是由SEGGER微控制器開發的，它是一個魯棒和功能豐富的調試器，其支持大量的CPU內核和設備，不僅僅是ARM，比如RISC-V．因其高性能和可讀性而聞名，J-Link支持不同的通信接口，包括JTAG，SWD，和fine-pitch JTAG接口．它因其高級的特性而受到歡迎，比如在flash存儲中的無限的斷點和它與多種開發環境的兼容性．

### MCU-Link

MCU-Link是一個調試探測器，也可以作為編程器使用，由NXP Semiconductors提供．它支持不同的ARM Cortex微控制器且可以與像是MCUXpresso IDE這樣的開發工具進行無縫地交互．MCU-Link因其豐富的功能和易使用而聞名，使它成為像是愛好者，教育者，和專業的開發者們的可行的選項．
