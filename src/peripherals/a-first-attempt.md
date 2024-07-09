# Rust嚐鮮

## 寄存器

讓我們看下 'SysTick' 外設 - 一個簡單的計時器，它存在於每個Cortex-M處理器內核中。通常你能在芯片廠商的數據手冊或者*技術參考手冊*中看到它們，但是下面的例子對所有ARM Cortex-M核心都是通用的，讓我們看下[ARM參考手冊]。我們能看到這裡有四個寄存器:

[ARM參考手冊]: http://infocenter.arm.com/help/topic/com.arm.doc.dui0553a/Babieigh.html

| Offset | Name        | Description                 | Width  |
|--------|-------------|-----------------------------|--------|
| 0x00   | SYST_CSR    | 控制和狀態寄存器               | 32 bits|
| 0x04   | SYST_RVR    | 重裝載值寄存器                | 32 bits|
| 0x08   | SYST_CVR    | 當前值寄存器                  | 32 bits|
| 0x0C   | SYST_CALIB  | 校準值寄存器                  | 32 bits|

## C語言風格的方法(The C Approach)

在Rust中，我們可以像C語言一樣，用一個 `struct` 表示一組寄存器。

```rust,ignore
#[repr(C)]
struct SysTick {
    pub csr: u32,
    pub rvr: u32,
    pub cvr: u32,
    pub calib: u32,
}
```
限定符 `#[repr(C)]` 告訴Rust編譯器像C編譯器一樣去佈局這個結構體。這非常重要，因為Rust允許結構體字段被重新排序，而C語言不允許。你可以想象下如果這些字段被編譯器悄悄地重新排了序，在調試時會給我們帶來多大的麻煩！有了這個限定符，我們就有了與上表對應的四個32位的字段。但當然，這個 `struct` 本身沒什麼用處 - 我們需要一個變量。

```rust,ignore
let systick = 0xE000_E010 as *mut SysTick;
let time = unsafe { (*systick).cvr };
```

## volatile訪問(Volatile Accesses)

現在，上面的方法有一堆問題。

1. 每次想要訪問外設，不得不使用unsafe 。
2. 無法指定哪個寄存器是隻讀的或者讀寫的。
3. 程序中任何地方的任何一段代碼都可以通過這個結構體訪問硬件。
4. 最重要的是，實際上它並不能工作。

現在的問題是編譯器很聰明。如果你往RAM同個地方寫兩次，一個接著一個，編譯器會注意到這個行為，且完全跳過第一個寫入操作。在C語言中，我們能標記變量為`volatile`去確保每個讀或寫操作按所想的那樣發生。在Rust中，我們將*訪問*操作標記為易變的(volatile)，而不是將變量標記為volatile。

```rust,ignore
let systick = unsafe { &mut *(0xE000_E010 as *mut SysTick) };
let time = unsafe { core::ptr::read_volatile(&mut systick.cvr) };
```
這樣，我們已經修復了一個問題，但是現在我們有了更多的 `unsafe` 代碼!幸運的是，有個第三方的crate可以幫助到我們 - [`volatile_register`]

[`volatile_register`]: https://crates.io/crates/volatile_register

```rust,ignore
use volatile_register::{RW, RO};

#[repr(C)]
struct SysTick {
    pub csr: RW<u32>,
    pub rvr: RW<u32>,
    pub cvr: RW<u32>,
    pub calib: RO<u32>,
}

fn get_systick() -> &'static mut SysTick {
    unsafe { &mut *(0xE000_E010 as *mut SysTick) }
}

fn get_time() -> u32 {
    let systick = get_systick();
    systick.cvr.read()
}
```

現在通過`read`和`write`方法，volatile accesses可以被自動執行。執行寫操作仍然是 `unsafe` 的，但是公平地講，硬件有一堆可變的狀態，對於編譯器來說沒有辦法知道是否這些寫操作是真正安全的，因此默認就這樣是個不錯的選擇。

## Rust風格的封裝

我們需要把這個`struct`封裝進一個更高抽象的API中，這個API對於用戶來說，可以安全地調用。作為驅動的作者，我們親手驗證不安全的代碼是否正確，然後為我們的用戶提供一個safe的API，因此用戶們不必擔心它(讓他們相信我們不會出錯!)。

有可能可以這樣寫:

```rust,ignore
use volatile_register::{RW, RO};

pub struct SystemTimer {
    p: &'static mut RegisterBlock
}

#[repr(C)]
struct RegisterBlock {
    pub csr: RW<u32>,
    pub rvr: RW<u32>,
    pub cvr: RW<u32>,
    pub calib: RO<u32>,
}

impl SystemTimer {
    pub fn new() -> SystemTimer {
        SystemTimer {
            p: unsafe { &mut *(0xE000_E010 as *mut RegisterBlock) }
        }
    }

    pub fn get_time(&self) -> u32 {
        self.p.cvr.read()
    }

    pub fn set_reload(&mut self, reload_value: u32) {
        unsafe { self.p.rvr.write(reload_value) }
    }
}

pub fn example_usage() -> String {
    let mut st = SystemTimer::new();
    st.set_reload(0x00FF_FFFF);
    format!("Time is now 0x{:08x}", st.get_time())
}
```

現在，這種方法帶來的問題是，下列的代碼完全可以被編譯器接受:

```rust,ignore
fn thread1() {
    let mut st = SystemTimer::new();
    st.set_reload(2000);
}

fn thread2() {
    let mut st = SystemTimer::new();
    st.set_reload(1000);
}
```

雖然 `set_reload` 函數的 `&mut self` 參數保證了沒有引用到其它的`SystemTimer`結構體，但是不能阻止用戶去創造第二個`SystemTimer`，其指向同個外設！如果作者足夠努力的話，他能發現所有這些'重複的'驅動實例，那麼按這種方式寫的代碼就可以工作，但是一旦代碼被散播一段時間，散播給多個模塊，驅動，開發者，它會越來越容易觸發此類錯誤。
