# 可移植性

在嵌入式環境中，可移植性是一個非常重要的主題: 每個供應商甚至同個製造商的不同系列間，都提供了不同的外設和功能。同樣地，與外設交互的方式也將會不一樣。

通過一個被叫做硬件抽象層或者**HAL**的層去均等化這種差異是一種常見的方法。

> 在軟件中硬件抽象是一組函數，其模仿了一些平臺特定的細節，讓程序可以直接訪問硬件資源。
> 通過向硬件提供標準的操作系統(OS)調用，它可以讓程序員編寫獨立於設備的高性能應用。
>
> *Wikipedia: [Hardware Abstraction Layer]*

[Hardware Abstraction Layer]: https://en.wikipedia.org/wiki/Hardware_abstraction

在這方面，嵌入式系統有點特別，因為通常沒有操作系統和用戶可安裝的軟件，而只有固件鏡像，其作為一個整體被編譯且伴著許多約束。因此雖然維基百科定義的傳統方法可能有用，但是它不是確保可移植性最有效的方法。

在Rust中我們要怎麼實現這個目標呢?讓我們進入**embedded-hal**...

## 什麼是embedded-hal？

簡而言之，它是一組traits，其定義了**HAL implementations**，**驅動**，**應用(或者固件)** 之間的實現約定(implementation contracts)。這些約定包括功能(即約定，如果為某個類型實現了某個trait，**HAL implementation**就提供了某個功能)和方法(即，如果構造一個實現了某個trait的類型，約定保障類型肯定有在trait中指定的方法)。


典型的分層可能如下所示:

![](../assets/rust_layers.svg)

一些在**embedded-hal**中被定義的traits:
* GPIO (input and output pins)
* Serial communication
* I2C
* SPI
* Timers/Countdowns
* Analog Digital Conversion

使用**embedded-hal** traits和依賴**embedded-hal**的crates的主要原因是為了控制複雜性。如果發現一個應用可能必須要實現對硬件外設的使用，以及需要實現應用程序和其它硬件組件間潛在的驅動，那麼其應該很容易被看作是可複用性有限的。用數學語言來說就是，如果**M**是外設HAL implementations的數量，**N**是驅動的數量，那麼如果我們要為每個應用重新發明輪子我們最終會有**M*N**個實現，然而通過使用**embedded-hal**的traits提供的 *API* 將會使實現複雜性變成**M+N** 。當然還有其它好處，比如由於API定義良好，開箱即用，導致試錯減少。


## embedded-hal的用戶

像上面所說的，HAL有三個主要用戶:

### HAL implementation

HAL implentation提供硬件和HAL traits的用戶之間的接口。典型的實現由三部分組成:

* 一個或者多個硬件特定的類型
* 生成和初始化這個類型的函數，函數經常提供不同的配置選項(速度，操作模式，使用的管腳，etc 。)
* 與這個類型有關的一個或者多個 **embedded-hal** traits 的 `trait` `impl`

這樣的一個 **HAL implementation** 可以有多個方法來實現:
* 通過低級硬件訪問，比如通過寄存器。
* 通過操作系統，比如通過使用Linux下的 `sysfs`
* 通過適配器，比如一個與單元測試有關的類型的仿真
* 通過相關硬件適配器的驅動，e.g. I2C多路複用器或者GPIO擴展器(I2C multiplexer or GPIO expander)

### 驅動

驅動為一個外部或者內部組件實現了一組自定義的功能，被連接到一個實現了embedded-hal traits的外設上。這種驅動的典型的例子包括多種傳感器(溫度計，磁力計，加速度計，光照計)，顯示設備(LED陣列，LCD顯示屏)和執行器(電機，發送器)。

必須使用實現了embedded-hal的某個`trait`的類型的實例來初始化驅動，這是通過trait bound來確保的，驅動也提供了它自己的類型實例，這個實例具有一組自定義的方法，這些方法允許與被驅動的設備交互。

### 應用

應用把多個部分結合在一起並確保需要的功能被實現。當在不同的系統間移植時，這部分的適配是花費最多精力的地方，因為應用需要通過HAL implementation正確地初始化真實的硬件，而且不同硬件的初始化也不相同，甚至有時候差別非常大。用戶的選擇也在其中扮演了非常重大的角色，因為組件能被物理連接到不同的端口，硬件總線有時候需要外部硬件去匹配配置，或者用戶在內部外設的使用上有不同的考量。
