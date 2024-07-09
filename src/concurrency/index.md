# 併發

當程序的不同部分有可能會在不同的時刻被執行或者不按順序地被執行時，那併發就出現了。在一個嵌入式環境中，這包括:

* 中斷處理函數，一旦相關的中斷髮生時，中斷處理函數就會運行，
* 不同的多線程形式，在這塊，微處理器通常會在程序的不同部分間進行切換，
* 在一些多核微處理器系統中，每個核可以同時獨立地運行程序的不同部分。

因為許多嵌入式程序需要處理中斷，因此併發遲早會出現，這也是許多微妙和困難的bugs會出現的地方。幸運地是，Rust提供了許多抽象和安全保障去幫助我們寫正確的代碼。

## 沒有併發

對於一個嵌入式程序來說最簡單的併發是沒有併發: 軟件由一個保持運行的main循環組成，一點中斷也沒有。有時候這非常適合手邊的問題! 通常你的循環將會讀取一些輸入，執行一些處理，且寫入一些輸出。

```rust,ignore
#[entry]
fn main() {
    let peripherals = setup_peripherals();
    loop {
        let inputs = read_inputs(&peripherals);
        let outputs = process(inputs);
        write_outputs(&peripherals, outputs);
    }
}
```

因為這裡沒有併發，因此不需要擔心程序不同部分間的共享數據或者同步對外設的訪問。如果可以使用一個簡單的方法來解決問題，這種方法是個不錯的選擇。

## 全局可變數據

不像非嵌入式Rust，我們通常不會奢侈地在堆上分配數據，並將對該數據的引用傳遞到新創建的線程中。相反，我們的中斷處理函數隨時可能被調用，且必須知道如何訪問我們正在使用的共享內存。從最底層看來，這意味著我們必須有 _靜態分配的_ 可變的內存，中斷處理函數和main代碼都可以引用這塊內存。

在Rust中，[`static mut`]這樣的變量讀取或者寫入總是unsafe的，因為不特別關注它們的話，可能會觸發一個競態條件，對變量的訪問在中途就被一個也訪問那個變量的中斷打斷了。

[`static mut`]: https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#accessing-or-modifying-a-mutable-static-variable

為了舉例這種行為如何在代碼中導致了微妙的錯誤，思考一個嵌入式程序，這個程序在每一秒的週期內計數一些輸入信號的上升沿(一個頻率計數器):

```rust,ignore
static mut COUNTER: u32 = 0;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // 危險 - 實際不安全! 可能導致數據競爭。
            unsafe { COUNTER += 1 };
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    unsafe { COUNTER = 0; }
}
```

每秒計時器中斷會把計數器設置回0。這期間，main循環連續地測量信號，且當看到從低電平到高電平的變化時，增加計數器的值。因為它是`static mut`的，我們不得不使用`unsafe`去訪問`COUNTER`，意思是我們向編譯器保證我們的操作不會導致任何未定義的行為。你能發現競態條件嗎？`COUNTER`上的增加並不一定是原子的 - 事實上，在大多數嵌入式平臺上，它將被分開成一個讀取操作，然後是增加，然後是寫回。如果中斷在計數器被讀取之後但是在被寫回之前被激活，在中斷返回後，重置回0的操作會被忽略掉 - 那期間，我們會算出兩倍的轉換次數。

## 臨界區(Critical Sections)

因此，關於數據競爭可以做些什麼？一個簡單的方法是使用 _臨界區(critical sections）_ ，在臨界區的上下文中中斷被關閉了。通過把對`main`中的`COUNTER`訪問封裝進一個臨界區，我們能確保計時器中斷將不會激活，直到我們完成了增加`COUNTER`的操作:

```rust,ignore
static mut COUNTER: u32 = 0;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // 新的臨界區確保對COUNTER的同步訪問
            cortex_m::interrupt::free(|_| {
                unsafe { COUNTER += 1 };
            });
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    unsafe { COUNTER = 0; }
}
```

在這個例子裡，我們使用 `cortex_m::interrupt::free`，但是其它平臺將會有更簡單的機制在一個臨界區中執行代碼。它們都有一樣的邏輯，關閉中斷，運行一些代碼，然後重新使能中斷。

注意，有兩個理由，不需要把一個臨界區放進計時器中斷中:

  * 向`COUNTER`寫入0不會被一個競爭影響，因為我們不需要讀取它
  * 無論如何，它永遠不會被`main`線程中斷

如果`COUNTER`被多個可能相互 _搶佔_ 的中斷處理函數共享，那麼每一個也需要一個臨界區。

這解決了我們眼前的問題，但是我們仍然要編寫許多unsafe的代碼，我們需要仔細推敲這些代碼，有些我們可能不需要使用臨界區。因為每個臨界區暫時暫停了中斷處理，就會帶來一些相關的成本，一些額外的代碼大小，更高的中斷延遲和抖動(中斷可能花費很長時間去處理，等待被處理的時間變化非常大)。這是否是個問題取決於你的系統，但是通常，我們想要避免它。

值得注意的是，雖然一個臨界區保障了不會發生中斷，但是它在多核系統上不提供一個排他性保證(exclusivity guarantee)！其它核可能很開心地訪問與你的核一樣的內存區域，即使沒有中斷。如果你正在使用多核，你將需要更強的同步原語(synchronisation primitives)。

## 原子訪問

在一些平臺上，可以使用特定的原子指令，它保障了讀取-修改-寫回操作。針對Cortex-M: `thumbv6`(Cortex-M0，Cortex-M0+)只提供原子讀取和存取指令，而`thumv7`(Cortex-M3及以上)提供完整的比較和交換(CAS)指令。這些CAS指令可以替代過重的禁用所有中斷的方法: 我們可以嘗試執行加法操作，它在大多數情況下都會成功，但是如果它被中斷了它將會自動重試完整的加法操作。這些原子操作甚至在多核間也是安全的。

```rust,ignore
use core::sync::atomic::{AtomicUsize, Ordering};

static COUNTER: AtomicUsize = AtomicUsize::new(0);

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // 使用 `fetch_add` 原子性地給 COUNTER 加一
            COUNTER.fetch_add(1, Ordering::Relaxed);
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // 使用 `store` 將 0 直接寫入 COUNTER
    COUNTER.store(0, Ordering::Relaxed)
}
```

這時，`COUNTER`是一個safe的`static`變量。多虧了`AtomicUsize`類型，不需要禁用中斷，`COUNTER`能從中斷處理函數和main線程被安全地修改。當可以這麼做時，這是一個更好的解決方案 - 然而平臺上可能不支持這麼做。

關於[`Ordering`]的提醒: 它可能影響編譯器和硬件如何重新排序指令，也會影響緩存可見性。假設目標是個單核平臺，在這個案例裡`Relaxed`是充足的和最有效的選擇。更嚴格的排序將導致編譯器在原子操作周圍產生內存屏障(Memory Barriers)；取決於你做什麼原子操作，你可能需要或者不需要這個排序！原子模型的精確細節是複雜的，最好寫在其它地方。

關於原子操作和排序的更多細節，可以看這裡[nomicon]。

[`Ordering`]: https://doc.rust-lang.org/core/sync/atomic/enum.Ordering.html
[nomicon]: https://doc.rust-lang.org/nomicon/atomics.html


## 抽象，Send和Sync

上面的解決方案都不是特別令人滿意。它們需要`unsafe`塊，`unsafe`塊必須要被十分小心地檢查且不符合人體工程學。確實，我們在Rust中可以做得更好！

我們可以把我們的計數器抽象進一個安全的接口中，它可以在代碼的其它地方被安全地使用。在這個例子裡，我們將使用臨界區的(cirtical-section)計數器，但是你可以用原子操作做一些非常類似的事情。

```rust,ignore
use core::cell::UnsafeCell;
use cortex_m::interrupt;

// 我們的計數器只是包圍UnsafeCell<u32>的一個封裝，它是Rust中內部可變性
// (interior mutability)的關鍵。通過使用內部可變性，我們能讓COUNTER
// 變成`static`而不是`static mut`，但是仍能改變它的計數器值。
struct CSCounter(UnsafeCell<u32>);

const CS_COUNTER_INIT: CSCounter = CSCounter(UnsafeCell::new(0));

impl CSCounter {
    pub fn reset(&self, _cs: &interrupt::CriticalSection) {
        // 通過要求一個CriticalSection被傳遞進來，我們知道我們肯定正在一個
        // CriticalSection中操作，且因此可以自信地使用這個unsafe塊(調用UnsafeCell::get的前提)。
        unsafe { *self.0.get() = 0 };
    }

    pub fn increment(&self, _cs: &interrupt::CriticalSection) {
        unsafe { *self.0.get() += 1 };
    }
}

// 允許靜態CSCounter的前提。看下面的解釋。
unsafe impl Sync for CSCounter {}

// COUNTER不再是`mut`的因為它使用內部可變性;
// 因此訪問它也不再需要unsafe塊。
static COUNTER: CSCounter = CS_COUNTER_INIT;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // 這裡不用unsafe!
            interrupt::free(|cs| COUNTER.increment(cs));
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // 這裡我們需要進入一個臨界區，只是為了傳遞進一個有效的cs token，儘管我們知道
    // 沒有其它中斷可以搶佔這個中斷。 
    interrupt::free(|cs| COUNTER.reset(cs));

    // 如果我們真的需要，我們可以使用unsafe代碼去生成一個假CriticalSection，
    // 避免開銷:
    // let cs = unsafe { interrupt::CriticalSection::new() };
}
```

我們已經把我們的`unsafe`代碼移進了精心安排的抽象中，現在我們的應用代碼不包含任何`unsafe`塊。

這個設計要求應用傳遞一個`CriticalSection` token進來: 這些tokens僅由`interrupt::free`安全地產生，因此通過要求傳遞進一個`CriticalSection` token，我們確保我們正在一個臨界區中操作，不用自己動手鎖起來。這個保障由編譯器靜態地提供: 這將不會帶來任何與`cs`有關的運行時消耗。如果我們有多個計數器，它們都可以被指定同一個`cs`，而不用要求多個嵌套的臨界區。

這也帶來了Rust中關於併發的一個重要主題: [`Send` and `Sync`] traits。總結一下Rust book，當一個類型能夠安全地被移動到另一個線程，它是Send，當一個類型能被安全地在多個線程間共享的時候，它是Sync。在一個嵌入式上下文中，我們認為中斷是在應用代碼的一個獨立線程中執行的，因此在一箇中斷和main代碼中都能被訪問的變量必須是Sync。

[`Send` and `Sync`]: https://doc.rust-lang.org/nomicon/send-and-sync.html

在Rust中的大多數類型，這兩個traits都會由你的編譯器為你自動地產生。然而，因為`CSCounter`包含了一個[`UnsafeCell`]，它不是Sync，因此我們不能使用一個`static CSCounter`: `static` 變量 _必須_ 是Sync，因此它們能被多個線程訪問。

[`UnsafeCell`]: https://doc.rust-lang.org/core/cell/struct.UnsafeCell.html

為了告訴編譯器我們已經注意到`CSCounter`事實上在線程間共享是安全的，我們顯式地實現了Sync trait。與之前使用的臨界區一樣，這隻在單核平臺上是安全的: 對於多核，你需要做更多的事來確保安全。

## 互斥量(Mutexs)

我們已經為我們的計數器問題創造了一個有用的抽象，但是關於併發這裡還存在許多通用的抽象。

一個互斥量(mutex)，互斥(mutual exclusion)的縮寫，就是這樣的一個 _同步原語_ 。這些構造確保了對一個變量的排他訪問，比如我們的計數器。一個線程會嘗試 _lock_ (或者 _acquire_) 互斥量，或者當互斥量不能被鎖住時返回一個錯誤。當線程持有鎖時，它有權訪問被保護的數據，當線程工作完成了，它 _unlocks_ (或者 _releases_) 互斥量，允許其它線程鎖住它。在Rust中，我們通常使用[`Drop`] trait實現unlock去確保當互斥量超出作用域時它總是被釋放。

[`Drop`]: https://doc.rust-lang.org/core/ops/trait.Drop.html

將中斷處理函數與一個互斥量一起使用可能有點棘手: 阻塞中斷處理函數通常是不可接受的，如果它阻塞等待main線程去釋放一個鎖，那將是一場災難。因為我們會 _死鎖_ (因為執行停留在中斷處理函數中，主線程將永遠不會釋放鎖)。死鎖被認為是不安全的: 即使在安全的Rust中這也是可能發生的。

為了完全避免這個行為，我們可以實現一個要求臨界區的互斥量去鎖住，就像我們的計數器例子一樣。臨界區的存在時間必須和鎖存在的時間一樣長，我們能確保我們對被封裝的變量有排他式訪問，甚至不需要跟蹤互斥量的 lock/unlock 狀態。

實際上我們在 `cortex_m` crate中就是這麼做的！我們可以用它來寫入我們的計數器:

```rust,ignore
use core::cell::Cell;
use cortex_m::interrupt::Mutex;

static COUNTER: Mutex<Cell<u32>> = Mutex::new(Cell::new(0));

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            interrupt::free(|cs|
                COUNTER.borrow(cs).set(COUNTER.borrow(cs).get() + 1));
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // 這裡我們仍然需要進入一個臨界區去滿足互斥量。
    interrupt::free(|cs| COUNTER.borrow(cs).set(0));
}
```

我們現在使用了[`Cell`]，它與它的兄弟`RefCell`一起被用於提供safe的內部可變性。我們已經見過`UnsafeCell`了，在Rust中它是內部可變性的底層: 它允許你去獲得對某個值的多個可變引用，但是隻能與不安全的代碼一起工作。一個`Cell`像一個`UnsafeCell`一樣但是它提供了一個安全的接口: 它只允許拷貝現在的值或者替換它，不允許獲取一個引用，因此它不是Sync，它不能被在線程間共享。這些限制意味著它用起來是safe的，但是我們不能直接將它用於`static`變量因為一個`static`必須是Sync。

[`Cell`]: https://doc.rust-lang.org/core/cell/struct.Cell.html

因此為什麼上面的例子可以工作?`Mutex<T>`對於任何是Send的`T`實現了Sync - 比如一個`Cell`。因為它只能在臨界區對它的內容進行訪問，所以它這麼做是safe的。因此我們可以即使沒有一點unsafe的代碼我們也能獲取一個safe的計數器！

對於我們的簡單類型，像是我們的計數器的`u32`來說是很棒的，但是對於更復雜的不能拷貝的類型呢？在一個嵌入式上下文中一個極度常見的例子是一個外設結構體，通常它們不是Copy。針對那種情況，我們可以使用`RefCell`。

## 共享外設

使用`svd2rust`生成的設備crates和相似的抽象，通過強制要求同時只能存在一個外設結構體的實例，提供了對外設的安全的訪問。這個確保了安全性，但是使得它很難從main線程和一箇中斷處理函數一起訪問一個外設。

為了安全地共享對外設的訪問，我們能使用我們之前看到的`Mutex`。我們也將需要使用[`RefCell`]，它使用一個運行時檢查去確保對一個外設每次只有一個引用被給出。這個比純`Cell`消耗更多，但是因為我們正給出引用而不是拷貝，我們必須確保每次只有一個引用存在。

[`RefCell`]: https://doc.rust-lang.org/core/cell/struct.RefCell.html

最終，我們也必須考慮在main代碼中初始化外設後，如何將外設移到共享變量中。為了做這個，我們使用`Option`類型，初始成`None`，之後設置成外設的實例。

```rust,ignore
use core::cell::RefCell;
use cortex_m::interrupt::{self, Mutex};
use stm32f4::stm32f405;

static MY_GPIO: Mutex<RefCell<Option<stm32f405::GPIOA>>> =
    Mutex::new(RefCell::new(None));

#[entry]
fn main() -> ! {
    // 獲得外設的單例並配置它。這個例子來自一個svd2rust生成的crate，
    // 但是大多數的嵌入式設備crates都相似。
    let dp = stm32f405::Peripherals::take().unwrap();
    let gpioa = &dp.GPIOA;

    // 某個配置函數。假設它把PA0設置成一個輸入和把PA1設置成一個輸出。
    configure_gpio(gpioa);

    // 把GPIOA存進互斥量中，移動它。
    interrupt::free(|cs| MY_GPIO.borrow(cs).replace(Some(dp.GPIOA)));
    // 我可以不再用`gpioa`或者`dp.GPIOA`，反而必須通過互斥量訪問它。

    // 請注意，只有在設置MY_GPIO後才能使能中斷: 要不然當MY_GPIO還是包含None的時候，
    // 中斷可能會發生，然後像上面寫的那樣操作(使用`unwrap()`)，它將發生運行時恐慌。
    set_timer_1hz();
    let mut last_state = false;
    loop {
        // 我們現在將通過互斥量，讀取其作為數字輸入時的狀態。
        let state = interrupt::free(|cs| {
            let gpioa = MY_GPIO.borrow(cs).borrow();
            gpioa.as_ref().unwrap().idr.read().idr0().bit_is_set()
        });

        if state && !last_state {
            // 如果我們在PA0上已經看到了一個上升沿，拉高PA1。
            interrupt::free(|cs| {
                let gpioa = MY_GPIO.borrow(cs).borrow();
                gpioa.as_ref().unwrap().odr.modify(|_, w| w.odr1().set_bit());
            });
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // 這次在中斷中，我們將清除PA0。
    interrupt::free(|cs| {
        // 我們可以使用`unwrap()` 因為我們知道直到MY_GPIO被設置後，中斷都是禁用的；
        // 否則我應該處理會出現一個None值的潛在可能
        let gpioa = MY_GPIO.borrow(cs).borrow();
        gpioa.as_ref().unwrap().odr.modify(|_, w| w.odr1().clear_bit());
    });
}
```

這需要理解的內容很多，所以讓我們把重要的內容分解一下。

```rust,ignore
static MY_GPIO: Mutex<RefCell<Option<stm32f405::GPIOA>>> =
    Mutex::new(RefCell::new(None));
```

我們的共享變量現在是一個包圍了一個`RefCell`的`Mutex`，`RefCell`包含一個`Option`。`Mutex`確保只在一個臨界區中的時候可以訪問，因此使變量變成了Sync，甚至即使一個純`RefCell`不是Sync。`RefCell`賦予了我們引用的內部可變性，我們將需要使用我們的`GPIOA`。`Option`讓我們可以初始化這個變量成空的東西，只在隨後實際移動變量進來。只有在運行時，我們才能靜態地訪問外設單例，因此這是必須的。

```rust,ignore
interrupt::free(|cs| MY_GPIO.borrow(cs).replace(Some(dp.GPIOA)));
```

在一個臨界區中，我們可以在互斥量上調用`borrow()`，其給了我們一個指向`RefCell`的引用。然後我們調用`replace()`去移動我們的新值進來`RefCell`。

```rust,ignore
interrupt::free(|cs| {
    let gpioa = MY_GPIO.borrow(cs).borrow();
    gpioa.as_ref().unwrap().odr.modify(|_, w| w.odr1().set_bit());
});
```

最終，我們用一種安全和併發的方式使用`MY_GPIO`。臨界區禁止了中斷像往常一樣發生，讓我們借用互斥量。`RefCell`然後給了我們一個`&Option<GPIOA>`並追蹤它還要借用多久 - 一旦引用超出作用域，`RefCell`將會被更新去指出引用不再被借用。

因為我不能把`GPIOA`移出`&Option`，我們需要用`as_ref()`將它轉換成一個`&Option<&GPIOA>`，最終我們能使用`unwrap()`獲得`&GPIOA`，其讓我們可以修改外設。

如果我們需要一個共享的資源的可變引用，那麼`borrow_mut`和`deref_mut`應該被使用。下面的代碼展示了一個使用TIM2計時器的例子。

```rust,ignore
use core::cell::RefCell;
use core::ops::DerefMut;
use cortex_m::interrupt::{self, Mutex};
use cortex_m::asm::wfi;
use stm32f4::stm32f405;

static G_TIM: Mutex<RefCell<Option<Timer<stm32::TIM2>>>> =
	Mutex::new(RefCell::new(None));

#[entry]
fn main() -> ! {
    let mut cp = cm::Peripherals::take().unwrap();
    let dp = stm32f405::Peripherals::take().unwrap();

    // 某個計時器配置函數。假設它配置了TIM2計時器和它的NVIC中斷，
    // 最終啟動計時器。
    let tim = configure_timer_interrupt(&mut cp, dp);

    interrupt::free(|cs| {
        G_TIM.borrow(cs).replace(Some(tim));
    });

    loop {
        wfi();
    }
}

#[interrupt]
fn timer() {
    interrupt::free(|cs| {
        if let Some(ref mut tim) =  G_TIM.borrow(cs).borrow_mut().deref_mut() {
            tim.start(1.hz());
        }
    });
}

```

呼！這是安全的，但也有點笨拙。我們還能做些什麼嗎？

## RTIC

另一個方法是使用[RTIC框架]，Real Time Interrupt-driven Concurrency的縮寫。它強制執行靜態優先級並追蹤對`static mut`變量("資源")的訪問去確保共享資源總是能被安全地訪問，而不需要總是進入臨界區和使用引用計數帶來的消耗(如`RefCell`中所示)。這有許多好處，比如保證沒有死鎖且時間和內存的消耗極度低。

[RTIC框架]: https://github.com/rtic-rs/cortex-m-rtic

這個框架也包括了其它的特性，像是消息傳遞(message passing)，消息傳遞減少了對顯式共享狀態的需要，還提供了在一個給定時間調度任務去運行的功能，這功能能被用來實現週期性的任務。看下[文檔]可以知道更多的信息！

[文檔]: https://rtic.rs

## 實時操作系統

與嵌入式併發有關的另一個模型是實時操作系統(RTOS)。雖然現在在Rust中的研究較少，但是它們被廣泛用於傳統的嵌入式開發。開源的例子包括[FreeRTOS]和[ChibiOS](譯者注: 目前有個純Rust實現的[Tock](https://www.tockos.org/))。這些RTOSs提供對運行多個應用線程的支持，CPU在這些線程間進行切換，切換要麼發生在當線程讓出控制權的時候(被稱為非搶佔式多任務)，要麼是基於一個常規計時器或者中斷(搶佔式多任務)。RTOS通常提供互斥量或者其它的同步原語，經常與硬件功能相互使用，比如DMA引擎。

[FreeRTOS]: https://freertos.org/
[ChibiOS]: http://chibios.org/

在撰寫本文時，沒有太多的Rust RTOS示例可供參考，但這是一個有趣的領域，所以請關注這塊！

## 多個核心

在嵌入式處理器中有兩個或者多個核心很正常，其為併發添加了額外一層複雜性。所有使用臨界區的例子(包括`cortex_m::interrupt::Mutex`)都假設了另一個執行的線程僅是中斷線程，但是在一個多核系統中，這不再是正確的假設。反而，我們將需要為多核設計的同步原語(也被叫做SMP，symmetric multi-processing的縮寫)。

我們之前看到的，這些通常使用原子指令，因為處理系統將確保原子性在所有的核中都保持著。

覆蓋這些主題的細節已經超出了本書的範圍，但是常規的模式與單核的相似。
