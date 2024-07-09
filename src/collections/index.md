# 集合

最後，還希望在程序裡使用動態數據結構(也稱為集合)。`std` 提供了一組常見的集合: [`Vec`]，[`String`]，[`HashMap`]，等等。所有這些在`std`中被實現的集合都使用一個全局動態分配器(也稱為堆)。
 
[`Vec`]: https://doc.rust-lang.org/std/vec/struct.Vec.html
[`String`]: https://doc.rust-lang.org/std/string/struct.String.html
[`HashMap`]: https://doc.rust-lang.org/std/collections/struct.HashMap.html

因為`core`的定義中是沒有內存分配的，所以這些實現在`core`中是沒有的，但是我們可以在編譯器附帶的`alloc` crate中找到。

如果需要集合，一個基於堆分配的實現不是唯一的選擇。也可以使用 *fixed capacity* 集合; 其實現可以在 [`heapless`] crate中被找到。

[`heapless`]: https://crates.io/crates/heapless

在這部分，我們將研究和比較這兩個實現。

## 使用 `alloc`

`alloc` crate與標準的Rust發行版在一起。你可以直接 `use` 導入這個crate，而不需要在`Cargo.toml`文件中把它聲明為一個依賴。

``` rust,ignore
#![feature(alloc)]

extern crate alloc;

use alloc::vec::Vec;
```

為了能使用集合，首先需要使用`global_allocator`屬性去聲明程序將使用的全局分配器。它要求選擇的分配器實現了[`GlobalAlloc`] trait 。

[`GlobalAlloc`]: https://doc.rust-lang.org/core/alloc/trait.GlobalAlloc.html

為了完整性和儘可能保持本節的自包含性，我們將實現一個簡單線性指針分配器且用它作為全局分配器。然而，我們 *強烈地* 建議你在你的程序中使用一個來自crates.io的久經戰鬥測試的分配器而不是這個分配器。

``` rust,ignore
// 線性指針分配器實現

use core::alloc::{GlobalAlloc, Layout};
use core::cell::UnsafeCell;
use core::ptr;

use cortex_m::interrupt;

// 用於單核系統的線性指針分配器
struct BumpPointerAlloc {
    head: UnsafeCell<usize>,
    end: usize,
}

unsafe impl Sync for BumpPointerAlloc {}

unsafe impl GlobalAlloc for BumpPointerAlloc {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // `interrupt::free`是一個臨界區，臨界區讓我們的分配器在中斷中用起來安全
        interrupt::free(|_| {
            let head = self.head.get();
            let size = layout.size();
            let align = layout.align();
            let align_mask = !(align - 1);

            // 將start移至下一個對齊邊界。
            let start = (*head + align - 1) & align_mask;

            if start + size > self.end {
                // 一個空指針通知內存不足
                ptr::null_mut()
            } else {
                *head = start + size;
                start as *mut u8
            }
        })
    }

    unsafe fn dealloc(&self, _: *mut u8, _: Layout) {
        // 這個分配器從不釋放內存
    }
}

// 全局內存分配器的聲明
// 注意 用戶必須確保`[0x2000_0100, 0x2000_0200]`內存區域
// 沒有被程序的其它部分使用
#[global_allocator]
static HEAP: BumpPointerAlloc = BumpPointerAlloc {
    head: UnsafeCell::new(0x2000_0100),
    end: 0x2000_0200,
};
```

除了選擇一個全局分配器，用戶也必須要定義如何使用*不穩定的*`alloc_error_handler`屬性來處理內存溢出錯誤。

``` rust,ignore
#![feature(alloc_error_handler)]

use cortex_m::asm;

#[alloc_error_handler]
fn on_oom(_layout: Layout) -> ! {
    asm::bkpt();

    loop {}
}
```

一旦一切都完成了，用戶最後就可以在`alloc`中使用集合。

```rust,ignore
#[entry]
fn main() -> ! {
    let mut xs = Vec::new();

    xs.push(42);
    assert!(xs.pop(), Some(42));

    loop {
        // ..
    }
}
```

如果你已經使用了`std` crate中的集合，那麼這些對你來說將非常熟悉，因為他們的實現一樣。

## 使用 `heapless`

`heapless`無需設置，因為它的集合不依賴一個全局內存分配器。只是`use`它的集合然後實例化它們:

```rust,ignore
// heapless version: v0.4.x
use heapless::Vec;
use heapless::consts::*;

#[entry]
fn main() -> ! {
    let mut xs: Vec<_, U8> = Vec::new();

    xs.push(42).unwrap();
    assert_eq!(xs.pop(), Some(42));
    loop {}
}
```

你會注意到這些集合與`alloc`中的集合有兩個不一樣的地方。

第一，你必須預先聲明集合的容量。`heapless`集合從來不會發生重分配且具有固定的容量;這個容量是集合的類型簽名的一部分。在這個例子裡，我們已經聲明瞭`xs`的容量為8個元素，也就是說，這個vector最多隻能有八個元素。這是通過類型簽名中的`U8` (看[`typenum`])來指定的。

[`typenum`]: https://crates.io/crates/typenum

第二，`push`方法和另外一些方法返回的是一個`Result`。因為`heapless`集合有一個固定的容量，所以所有插入的操作都可能會失敗。通過返回一個`Result`，API反應了這個問題，指出操作是否成功還是失敗。相反，`alloc`集合自己將會在堆上重新分配去增加它的容量。

自v0.4.x版本起，所有的`heapless`集合將所有的元素內聯地存儲起來了。這意味著像是`let x = heapless::Vec::new()`這樣的一個操作將會在棧上分配集合，但是它也能夠在一個`static`變量上分配集合，或者甚至在堆上(`Box<Vec<_, _>>`)。

## 取捨

當在堆分配的可重定位的集合和固定容量的集合間進行選擇的時候，記住這些內容。

### 內存溢出和錯誤處理

使用堆分配，內存溢出總是有可能出現的且會發生在任何一個集合需要增長的地方: 比如，所有的 `alloc::Vec.push` 調用會潛在地產生一個OOM(Out of Memory)條件。因此一些操作可能會*隱式地*失敗。一些`alloc`集合暴露了`try_reserve`方法，可以當增加集合時讓你檢查潛在的OOM條件，但是你需要主動地使用它們。

如果你只使用`heapless`集合，而不使用內存分配器，那麼一個OOM條件不可能出現。反而，你必須逐個處理容量不足的集合。也就是必須處理*所有*的`Result`，`Result`由像是`Vec.push`這樣的方法返回的。

與在所有由`heapless::Vec.push`返回的`Result`上調用`unwrap`相比，OOM錯誤更難調試，因為錯誤被發現的位置可能與導致問題的位置*不*一致。比如，甚至如果分配器接近消耗完`vec.reserve(1)`都能觸發一個OOM，因為一些其它的集合正在洩露內存(內存洩露在安全的Rust是會發生的)。

### 內存使用

推理堆分配集合的內存使用是很難的因為長期使用的集合的大小會在運行時改變。一些操作可能隱式地重分配集合，增加了它的內存使用，一些集合暴露的方法，像是`shrink_to_fit`，會潛在地減少集合使用的內存 -- 最終，它由分配器去決定是否確定減小內存的分配或者不。另外，分配器可能不得不處理內存碎片，它會*明顯*增加內存的使用。

另一方面，如果你只使用固定容量的集合，請把大多數的數據保存在`static`變量中，併為調用棧設置一個最大尺寸，隨後如果你嘗試使用大於可用的物理內存的內存大小時，鏈接器會發現它。

另外，在棧上分配的固定容量的集合可以通過[`-Z emit-stack-sizes`]標識來報告，其意味著用來分析棧使用的工具(像是[`stack-sizes`])將會把在棧上分配的集合包含進它們的分析中。

[`-Z emit-stack-sizes`]: https://doc.rust-lang.org/beta/unstable-book/compiler-flags/emit-stack-sizes.html
[`stack-sizes`]: https://crates.io/crates/stack-sizes

然而，固定容量的集合*不*能被減少，與可重定位集合所能達到的負載係數(集合的大小和它的容量之間的比值)相比，它能產生更低的負載係數。

### 最壞執行時間 (WCET)

如果你正在搭建時間敏感型應用或者硬實時應用，那麼你可能更關心你程序的不同部分的最壞執行時間。

`alloc`集合能重分配，所以操作的WCET可能會增加，集合也將包括它用來重分配集合所需的時間，它取決於集合的*運行時*容量。這使得它更難去決定操作，比如`alloc::Vec.push`，的WCET，因為它依賴被使用的分配器和它的運行時容量。

另一方面固定容量的集合不會重分配，因此所有的操作有個可預期的執行時間。比如，`heapless::Vec.push`以固定時間執行。

### 易用性

`alloc`要求配置一個全局分配器而`heapless`不需要。然而，`heapless`要求你去選擇你要實例化的每一個集合的容量。

`alloc` API幾乎為每一個Rust開發者所熟知。`heapless` API嘗試模仿`alloc` API，但是因為`heapless`的顯式錯誤處理，它們不可能會一模一樣 -- 一些開發者可能會覺得顯式的錯誤處理過多或太麻煩。
