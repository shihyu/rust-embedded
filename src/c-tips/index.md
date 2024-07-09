# 給嵌入式C開發者的貼士

這個章節收集了可能對於剛開始編寫Rust的，有經驗的嵌入式C開發者來說，有用的各種各樣的貼士。它將解釋你在C中可能已經用到的那些東西與Rust中的有何不同。

## 預處理器

在嵌入式C中，為了各種各樣的目的使用預處理器是很常見的，比如:

* 使用`#ifdef`編譯時選擇代碼塊
* 編譯時的數組大小和計算
* 用來簡化常見的模式的宏(避免調用函數的開銷)

在Rust中沒有預處理器，所以許多案例有不同的處理方法。本章節剩下的部分，我們將介紹各種替代預處理器的方法。

### 編譯時的代碼選擇

Rust中最接近`#ifdef ... #endif`的是[Cargo features]。這些比C預處理器更正式一點: 每個crate顯式列舉的，所有可能的features只能是關了的或者打開了的。當你把一個crate列為依賴項時，Features被打開，且是可添加的：如果你依賴樹中的任何crate為另一個crate打開了一個feature，那麼這個feature將為所有使用那個crate的用戶而打開。

[Cargo features]: https://doc.rust-lang.org/cargo/reference/manifest.html#the-features-section

比如，你可能有一個crate，其提供一個信號處理的基本類型庫(library of signal processing primitives)。每個基本類型可能帶來一些額外的時間去編譯大量的常量，你想要避開這些常量。你可以為你的`Cargo.toml`中每個組件聲明一個Cargo feature。

```toml
[features]
FIR = []
IIR = []
```

然後，在你的代碼中，使用`#[cfg(feature="FIR")]`去控制要包含什麼東西。

```rust
/// 在你的頂層的lib.rs中
#[cfg(feature="FIR")]
pub mod fir;

#[cfg(feature="IIR")]
pub mod iir;
```

同樣地，你可以控制，只有當某個feature _沒有_ 被打開時，包含代碼塊，或者某些features的組合被打開或者被關閉時。 

另外，Rust提供了許多可以使用的自動配置了的條件，比如`target_arch`用來選擇不同的代碼所基於的架構。對於條件編譯的全部細節，可以參看the Rust reference的[conditional compilation]章節。

[conditional compilation]: https://doc.rust-lang.org/reference/conditional-compilation.html

條件編譯將只應用於下一條語句或者塊。如果一個塊不能在現在的作用域中被使用，那麼`cfg`屬性將需要被多次使用。值得注意的是大多數時間，僅是包含所有的代碼而讓編譯器在優化時去刪除死代碼(dead code)更好，通常，在移除不使用的代碼方面的工作，編譯器做得很好。

### 編譯時大小和計算

Rust支持`const fn`，`const fn`是在編譯時可以被計算的函數，因此可以被用在需要常量的地方，比如在數組的大小中。這個能與上述的features一起使用，比如:

```rust
const fn array_size() -> usize {
    #[cfg(feature="use_more_ram")]
    { 1024 }
    #[cfg(not(feature="use_more_ram"))]
    { 128 }
}

static BUF: [u32; array_size()] = [0u32; array_size()];
```

這些對於stable版本的Rust來說是新的特性，從1.31開始引入，因此文檔依然很少。在寫這篇文章的時候`const fn`可用的功能也非常有限; 在未來的Rust release版本中，我們可以期望`const fn`將帶來更多的功能。

### 宏

Rust提供一個極度強大的[宏系統]。雖然C預處理器幾乎直接在你的源代碼之上進行操作，但是Rust宏系統可以在一個更高的級別上操作。存在兩種Rust宏: _聲明宏_ 和 _過程宏_ 。前者更簡單也最常見; 它們看起來像是函數調用，且能擴展成一個完整的表達式，語句，項，或者模式。過程宏更復雜但是卻能讓Rust更強大: 它們可以把任一條Rust語法變成一個新的Rust語法。

[宏系統]: https://doc.rust-lang.org/book/ch19-06-macros.html

通常，你可能想知道在那些使用一個C預處理器宏的地方，能否使用一個聲明宏做同樣的工作。你可以在crate中定義它們，且在你的crate中輕鬆使用它們或者導出給其他人用。但是請注意，因為它們必須擴展成完整的表達式，語句，項或者模式，因此C預處理器宏的某些用例沒法用，比如可以擴展成一個變量名的一部分的宏或者可以把列表中的項擴展成不完整的集合的宏。

和Cargo features一樣，值得考慮下你是否真的需要宏。在一些例子中一個常規的函數更容易被理解，它也能被內聯成和一個和宏一樣的代碼。`#[inline]`和`#[inline(always)]` [attributes] 能讓你更深入控制這個過程，這裡也要小心 - 編譯器會從同一個crate的恰當的地方自動地內聯函數，因此不恰當地強迫它內聯函數實際可能會導致性能下降。

[attributes]: https://doc.rust-lang.org/reference/attributes.html#inline-attribute

研究完整的Rust宏系統超出了本節內容，因此我們鼓勵你去查閱Rust文檔瞭解完整的細節。

## 編譯系統

大多數Rust crates使用Cargo編譯 (即使這不是必須的)。這解決了傳統編譯系統帶來的許多難題。然而，你可能希望自定義編譯過程。為了實現這個目的，Cargo提供了[`build.rs`腳本]。它們是可以根據需要與Cargo編譯系統進行交互的Rust腳本。

[`build.rs`腳本]: https://doc.rust-lang.org/cargo/reference/build-scripts.html

與編譯腳本有關的常見用例包括:

* 提供編譯時信息，比如靜態嵌入編譯日期或者Git commit hash進你的可執行文件中
* 根據被選擇的features或者其它邏輯在編譯時生成鏈接腳本
* 改變Cargo的編譯配置
* 添加額外的靜態鏈接庫以進行鏈接

現在還不支持post-build腳本，通常將它用於像是從編譯的對象自動生生成二進制文件或者打印編譯信息這類任務中。

### 交叉編譯

為你的編譯系統使用Cargo也能簡化交叉編譯。在大多數例子裡，告訴Cargo `--target thumbv6m-none-eabi`就行了，可以在`target/thumbv6m-none-eabi/debug/myapp`中找到一個合適的可執行文件。

對於那些並不是Rust原生支持的平臺，將需要自己為那個目標平臺編譯`libcore`。遇到這樣的平臺，[Xargo]可以作為Cargo的替代來使用，它可以自動地為你編譯`libcore`。

[Xargo]: https://github.com/japaric/xargo

## 迭代器與數組訪問

在C中，你可能習慣於通過索引直接訪問數組:

```c
int16_t arr[16];
int i;
for(i=0; i<sizeof(arr)/sizeof(arr[0]); i++) {
    process(arr[i]);
}
```

在Rust中，這是一個反模式(anti-pattern)：索引訪問可能會更慢(因為它可能需要做邊界檢查)且可能會阻止編譯器的各種優化。這是一個重要的區別，值得再重複一遍: Rust會在手動的數組索引上進行越界檢查以保障內存安全性，而C允許索引數組外的內容。

可以使用迭代器來替代:

```rust,ignore
let arr = [0u16; 16];
for element in arr.iter() {
    process(*element);
}
```

迭代器提供了一個有強大功能的數組，在C中你不得不手動實現它，比如chaining，zipping，enumerating，找到最小或最大值，summing，等等。迭代器方法也能被鏈式調用，提供了可讀性非常高的數據處理代碼。

閱讀[Iterators in the Book]和[Iterator documentation]獲取更多細節。

[Iterators in the Book]: https://doc.rust-lang.org/book/ch13-02-iterators.html
[Iterator documentation]: https://doc.rust-lang.org/core/iter/trait.Iterator.html

## 引用和指針

在Rust中，存在指針(被叫做 [_裸指針_])但是隻能在特殊的環境中被使用，因為解引用裸指針總是被認為是`unsafe`的 -- Rust通常不能保障指針背後有什麼。

[_裸指針_]: https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#dereferencing-a-raw-pointer

在大多數例子裡，我們使用 _引用_ 來替代，由`&`符號指出，或者 _可變引用_，由`&mut`指出。引用與指針相似，因為它能被解引用來訪問底層的數據，但是它們是Rust的所有權系統的一個關鍵部分: Rust將嚴格強迫你在任何給定時間只有一個可變引用 _或者_ 對相同數據的多個不變引用。

在實踐中，這意味著你必須要更加小心你是否需要對數據的可變訪問：在C中默認是可變的，你必須顯式地使用`const`，在Rust中正好相反。

某個情況下，你可能仍然要使用裸指針直接與硬件進行交互(比如，寫入一個指向DMA外設寄存器中的緩存的指針)，它們也被所有的外設訪問crates在底層使用，讓你可以讀取和寫入存儲映射寄存器。

## Volatile訪問

在C中，某個變量可能被標記成`volatile`，向編譯器指出，變量中的值在訪問間可能改變。Volatile變量通常用於一個與存儲映射的寄存器有關的嵌入式上下文中。

在Rsut中，並不使用`volatile`標記變量，我們使用特定的方法去執行volatile訪問: [`core::ptr::read_volatile`] 和 [`core::ptr::write_volatile`]。這些方法使用一個 `*const T` 或者一個 `*mut T` (上面說的 _裸指針_ )，執行一個volatile讀取或者寫入。

[`core::ptr::read_volatile`]: https://doc.rust-lang.org/core/ptr/fn.read_volatile.html
[`core::ptr::write_volatile`]: https://doc.rust-lang.org/core/ptr/fn.write_volatile.html

比如，在C中你可能這樣寫:

```c
volatile bool signalled = false;

void ISR() {
    // 提醒中斷已經發生了
    signalled = true;
}

void driver() {
    while(true) {
        // 睡眠直到信號來了
        while(!signalled) { WFI(); }
        // 重置信號提示符
        signalled = false;
        // 執行一些正在等待這個中斷的任務
        run_task();
    }
}
```

在Rust中對每個訪問使用volatile方法能達到相同的效果:

```rust,ignore
static mut SIGNALLED: bool = false;

#[interrupt]
fn ISR() {
    // 提醒中斷已經發生
    // (在正在的代碼中，你應該考慮一個更高級的基本類型,
    // 比如一個原子類型)
    unsafe { core::ptr::write_volatile(&mut SIGNALLED, true) };
}

fn driver() {
    loop {
        // 睡眠直到信號來了
        while unsafe { !core::ptr::read_volatile(&SIGNALLED) } {}
        // 重置信號指示符
        unsafe { core::ptr::write_volatile(&mut SIGNALLED, false) };
        // 執行一些正在等待中斷的任務
        run_task();
    }
}
```

在示例代碼中有些事情值得注意:
  * 我們可以把`&mut SIGNALLED`傳遞給要求`*mut T`的函數中，因為`&mut T`會自動轉換成一個`*mut T` (對於`*const T`來說是一樣的)
  * 我們需要為`read_volatile`/`write_volatile`方法使用`unsafe`塊，因為它們是`unsafe`的函數。確保操作安全變成了程序員的責任：看方法的文檔獲得更多細節。

在你的代碼中直接使用這些函數是很少見的，因為它們通常由更高級的庫封裝起來為你提供服務。對於存儲映射的外設，提供外設訪問的crates將自動實現volatile訪問，而對於併發的基本類型，存在更好的抽象可用。(看[併發章節])

[併發章節]: ../concurrency/index.md

## 填充和對齊類型

在嵌入式C中，告訴編譯器一個變量必須遵守某個對齊或者一個結構體必須被填充而不是對齊，是很常見的行為，通常是為了滿足特定的硬件或者協議要求。

在Rust中，這由一個結構體或者聯合體上的`repr`屬性來控制。默認的表示(representation)不保障佈局，因此不應該被用於與硬件或者C互用的代碼。編譯器可能會對結構體成員重新排序或者插入填充，且這種行為可能在未來的Rust版本中改變。

```rust
struct Foo {
    x: u16,
    y: u8,
    z: u16,
}

fn main() {
    let v = Foo { x: 0, y: 0, z: 0 };
    println!("{:p} {:p} {:p}", &v.x, &v.y, &v.z);
}

// 0x7ffecb3511d0 0x7ffecb3511d4 0x7ffecb3511d2
// 注意為了改進填充，順序已經被變成了x, z, y
```

使用`repr(C)`可以確保佈局可以與C互用。

```rust
#[repr(C)]
struct Foo {
    x: u16,
    y: u8,
    z: u16,
}

fn main() {
    let v = Foo { x: 0, y: 0, z: 0 };
    println!("{:p} {:p} {:p}", &v.x, &v.y, &v.z);
}

// 0x7fffd0d84c60 0x7fffd0d84c62 0x7fffd0d84c64
// 順序被保留了，佈局將不會隨著時間而改變
// `z`是兩個字節對齊，因此在`y`和`z`之間填充了一個字節。
```

使用`repr(packed)`去確保表示(representation)被填充了:

```rust
#[repr(packed)]
struct Foo {
    x: u16,
    y: u8,
    z: u16,
}

fn main() {
    let v = Foo { x: 0, y: 0, z: 0 };
    // 引用必須總是對齊的，因此為了檢查結構體字段的地址，我們使用
    // `std::ptr::addr_of!()`去獲取一個裸指針而不僅是打印`&v.x`
    let px = std::ptr::addr_of!(v.x);
    let py = std::ptr::addr_of!(v.y);
    let pz = std::ptr::addr_of!(v.z);
    println!("{:p} {:p} {:p}", px, py, pz);
}

// 0x7ffd33598490 0x7ffd33598492 0x7ffd33598493
// 在`y`和`z`沒有填充被插入，因此現在`z`沒有被對齊。
```

注意使用`repr(packed)`也會將類型的對齊設置成`1` 。

最後，為了指定一個特定的對齊，可以使用`repr(align(n))`，`n`是要對齊的字節數(必須是2的冪):

```rust
#[repr(C)]
#[repr(align(4096))]
struct Foo {
    x: u16,
    y: u8,
    z: u16,
}

fn main() {
    let v = Foo { x: 0, y: 0, z: 0 };
    let u = Foo { x: 0, y: 0, z: 0 };
    println!("{:p} {:p} {:p}", &v.x, &v.y, &v.z);
    println!("{:p} {:p} {:p}", &u.x, &u.y, &u.z);
}

// 0x7ffec909a000 0x7ffec909a002 0x7ffec909a004
// 0x7ffec909b000 0x7ffec909b002 0x7ffec909b004
// `u`和`v`兩個實例已經被放置在4096字節的對齊上。
// 它們地址結尾處的`000`證明了這件事。
```

注意我們可以結合`repr(C)`和`repr(align(n))`來獲取一個對齊的c兼容的佈局。不允許將`repr(align(n))`和`repr(packed)`一起使用，因為`repr(packed)`將對齊設置為`1`。也不允許一個`repr(packed)`類型包含一個`repr(align(n))`類型。

關於類型佈局更多的細節，參考the Rust Reference的[type layout]章節。

[type layout]: https://doc.rust-lang.org/reference/type-layout.html

## 其它資源

* 這本書中:
    * [使用C的Rust](../interoperability/c-with-rust.md)
    * [使用Rust的C](../interoperability/rust-with-c.md)
* [The Rust Embedded FAQs](https://docs.rust-embedded.org/faq.html)
* [Rust Pointers for C Programmers](http://blahg.josefsipek.net/?p=580)
* [I used to use pointers - now what?](https://github.com/diwic/reffers-rs/blob/master/docs/Pointers.md)
