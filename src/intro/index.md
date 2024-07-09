# 引言
歡迎閱讀嵌入式Rust:一本關於如何在裸機(比如，微處理器)上使用Rust編程語言的入門書籍。

## 嵌入式Rust是為誰準備的
嵌入式Rust是為了那些既想要進行嵌入式編程，又想要使用Rust語言所提供的高級概念和安全保障的人們而準備的(參見[Who Rust Is For](https://doc.rust-lang.org/book/ch00-00-introduction.html))

## 本書範圍
這本書的目的是：
+ 讓開發者快速上手Rust嵌入式開發，比如，如何設置一個開發環境。
+ 分享那些關於使用Rust進行嵌入式開發的，現存的，最好的實踐經驗，比如，如何最大程度上地利用好Rust語言的特性去寫更正確的嵌入式軟件
+ 某種程度下作為工具書，比如，如何在一個項目裡將C和Rust混合在一起使用

雖然儘可能地嘗試讓這本書可以用於大多數場景，但是為了使讀者和作者更容易理解，在所有的示例中，這本書都使用了ARM Cortex-M架構。然而，這本書並不需要讀者熟悉這個架構，書中會在需要時對這個架構的特定細節進行解釋。

## 這本書是為誰準備的

這本書適合那些有一些嵌入式背景或者有Rust背景的人，然而我相信每一個對Rust嵌入式編程好奇的人都能從這本書中獲得某些收穫。對於那些先前沒有任何經驗的人，我們建議你讀一下“要求和預備知識”部分。從其它資料中獲取、補充缺失的知識，這樣能提高你的閱讀體驗。你可以看看“其它資源”部分，以找到你感興趣的那些主題的資源。

### 要求和預備知識

+ 你可以輕鬆地使用Rust編程語言，且在一個桌面環境上寫過，運行過，調試過Rust應用。你應該也要熟悉[2018 edition]的術語，因為這本書是面向Rust 2018的。

[2018 edition]: https://doc.rust-lang.org/edition-guide/
+ 你可以輕鬆地使用其它語言，比如C，C++或者Ada，開發和調試嵌入式系統，且熟悉如下的概念：
  + 交叉編譯
  + 存儲映射的外設（Memory Mapped Peripherals）
  + 中斷
  + I2C，SPI，串口等等常見的接口

### 其它資源

如果你還不熟悉上面提到的東西或者你對這本書中提到的某個特定主題感興趣，你也許能從這些資源中找到有用的信息。

|  主題        |   資源    |     描述     |
|--------------|----------|-------------|
| Rust         | [Rust Book](https://doc.rust-lang.org/book/) | 如果你還不熟悉Rust，我們強烈建議你讀這本書．|
| Rust, Embedded | [Discovery Book](https://docs.rust-embedded.org/discovery/) | 如果你從沒做過嵌入式編程，這本書可能是個更好的開端．|
| Rust, Embedded | [Embedded Rust Bookshelf](https://docs.rust-embedded.org) | 在這裡，你可以找到由Rust的嵌入式工作組提供的許多其它資源．|
| Rust, Embedded | [Embedonomicon](https://docs.rust-embedded.org/embedonomicon/) | 用Rust進行嵌入式編程的細節．|
| Rust, Embedded | [embedded FAQ](https://docs.rust-embedded.org/faq.html) | Rust在嵌入式上下文中遇到的常見問題．|
| Rust, Embedded | [Comprehensive Rust 🦀: Bare Metal](https://google.github.io/comprehensive-rust/bare-metal.html) | 用於一天課時的裸機Rust開發課程的教學資料．|
| Interrupts | [Interrupt](https://en.wikipedia.org/wiki/Interrupt) | - |
| Memory-mapped IO/Peripherals | [Memory-mapped I/O](https://en.wikipedia.org/wiki/Memory-mapped_I/O) | - |
| SPI, UART, RS232, USB, I2C, TTL | [Stack Exchange about SPI, UART, and other interfaces](https://electronics.stackexchange.com/questions/37814/usart-uart-rs232-usb-spi-i2c-ttl-etc-what-are-all-of-these-and-how-do-th) | - |

### 翻譯

這本書是已經被一些慷慨的志願者們翻譯了。如果你想要將你的翻譯列在這裡，請打開一個PR去添加它。

* [日文](https://tomoyuki-nakabayashi.github.io/book/)
  ([repository](https://github.com/tomoyuki-nakabayashi/book))

* [中文](https://xxchang.github.io/book/)
  ([repository](https://github.com/xxchang/book))

## 如何使用這本書
這本書通常假設你是按順序閱讀的。之後的章節是建立在先前的章節中提到的概念之上的，先前章節可能不會深入一個主題的細節，因為在隨後的章節將會再次重溫這個主題。
在大多數示例中這本書將使用[STM32F3DISCOVERY]開發板。這個板子是基於ARM Cortex-M架構的，且基本功能與大多數基於這個架構的CPUs功能相似。微處理器的外設和其它實現細節在不同的廠家之間是不同的，甚至來自同一個廠家，不同處理器系列之間也是不同的。
因此我們建議購買[STM32F3DISCOVERY]開發板來嘗試這本書中的例子。

[STM32F3DISCOVERY]: http://www.st.com/en/evaluation-tools/stm32f3discovery.html

## 貢獻

這本書的工作主要在[這個倉庫]裡管理，且主要由[resouces team]開發。

[這個倉庫]: https://github.com/rust-embedded/book
[resouces team]: https://github.com/rust-embedded/wg#the-resources-team

如果你按著這本書的操作遇到了什麼麻煩，或者這本書的一些部分不夠清楚，或者很難進行下去，那這本書就是有個bug，這個bug應該被報道給這本書的[the issue tracker] 。

[the issue tracker]: https://github.com/rust-embedded/book/issues/

修改拼寫錯誤和添加新內容的Pull requests非常歡迎！

## 二次使用這個材料

這本書根據以下許可證發佈:

* 本書中包含的代碼示例和獨立的Cargo項目均根據[MIT License]和[Apache License v2.0]發放許可的。
* 本書中包含的文檔，圖片和表格均根據[CC-BY-SA v4.0]發放許可的。

[MIT License]: https://opensource.org/licenses/MIT
[Apache License v2.0]: http://www.apache.org/licenses/LICENSE-2.0
[CC-BY-SA v4.0]: https://creativecommons.org/licenses/by-sa/4.0/legalcode

總之：如果你想在你的工作中使用我們的文檔或者圖片，你需要：

+ 提供合適的授信 (i.e. 在你的幻燈片中提到本書，提供相關頁面的連接)
+ 提供[CC-BY-SA v4.0]的許可證的鏈接
+ 指出你是否改變了材料的內容，在同一個許可證下，可以對材料進行任何改變

也請告訴我這本書對你是否有幫助！

