# 一個 `no_std` Rust環境

嵌入式編程這個詞被廣泛用於許多不同的編程場景中。小到RAM和ROM只有KB的8位機(像是[ST72325xx](https://www.st.com/resource/en/datasheet/st72325j6.pdf))，大到一個具有32/64位4核Cortex-A53和1GB RAM的系統，比如樹莓派([Model B 3+](https://en.wikipedia.org/wiki/Raspberry_Pi#Specifications))。當編寫代碼時，取決於你的目標環境和用例，將會有不同的限制和侷限。<br>
通常嵌入式編程有兩類:

## 主機環境下

這類環境與一個常見的PC環境類似。意味著向你提供了一個系統接口[比如 POSIX](https://en.wikipedia.org/wiki/POSIX)，使你能和不同的系統進行交互，比如文件系統，網絡，內存管理，進程，等等。標準庫相應地依賴這些接口去實現了它們的功能。可能有某種sysroot並限制了對RAM/ROM的使用，可能還有一些特別的硬件或者I/O。總之感覺像是在專用的PC環境上編程一樣。

## 裸機環境下

在一個裸機環境中，程序被加載前，環境中不存在代碼。沒有系統提供的軟件，我們不能加載標準庫。相反地，程序和它使用的crates只能使用硬件(裸機)去運行。使用`no-std`可以防止rust讀取標準庫。標準庫中與平臺無關的部分在[libcore](https://doc.rust-lang.org/core/)中。libcore剔除了那些在一個嵌入式環境中非必要的東西。比如用於動態分配的內存分配器。如果你需要這些或者其它的某些功能，通常會有提供這些功能的crates。

### libstd運行時

就像之前提到的，使用[libstd](https://doc.rust-lang.org/std/)需要一些系統集成，這不僅僅是因為[libstd](https://doc.rust-lang.org/std/)使用了一個公共的方法訪問操作系統，它也提供了一個運行時環境。這個運行時環境，負責設置堆棧溢出保護，處理命令行參數，並在一個程序的主函數被激活前啟動一個主線程。在一個`no_std`環境中，這個運行時環境也是不可用的。

## 總結
`#![no_std]`是一個crate層級的屬性，它說明crate將連接至core-crate而不是std-crate。[libcore](https://doc.rust-lang.org/core/) crate是std crate的一個的子集，其與平臺無關，它對程序將要運行的系統沒有做要求。比如，它提供了像是floats，strings和切片的APIs，暴露了像是與原子操作和SIMD指令相關的處理器功能的APIs。然而，它缺少涉及到平臺集成的那些APIs。由於這些特性，no_std和[libcore](https://doc.rust-lang.org/core/)代碼可以用於任何引導程序(stage 0)像是bootloaders，固件或者內核。

### 概述

| 特性                                                      | no\_std | std |
|-----------------------------------------------------------|--------|-----|
| 堆 (dynamic memory)                                     |   *    |  ✓  |
| 容器 (Vec, BTreeMap, etc)                          |  **    |  ✓  |
| 棧溢出保護                                 |   ✘    |  ✓  |
| 在進入main之前運行的初始化代碼                                |   ✘    |  ✓  |
| libstd available                                          |   ✘    |  ✓  |
| libcore available                                         |   ✓    |  ✓  |
| 編寫固件，內核，或者引導程序              |   ✓    |  ✘  |

\* 只有在你使用了 `alloc` crate 並設置了一個適合的分配器後，比如[alloc-cortex-m]後可用．

\** 只有在你使用了 `collections` crate 並配置了一個全局默認的分配器後可用．

\** 由於缺少安全的隨機數產生器，所以無法使用HashMap和HashSet．

[alloc-cortex-m]: https://github.com/rust-embedded/alloc-cortex-m

## 參見
* [RFC-1184](https://github.com/rust-lang/rfcs/blob/master/text/1184-stabilize-no_std.md)
