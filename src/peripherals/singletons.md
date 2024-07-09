# 單例

> 在軟件工程中，單例模式是一個軟件設計模式，其限制了一個類到一個對象的實例化。
>
> *Wikipedia: [Singleton Pattern]*

[Singleton Pattern]: https://en.wikipedia.org/wiki/Singleton_pattern


## 為什麼不可以使用全局變量？

可以像這樣，我們可以使每個東西都變成公共靜態的(public static):

```rust,ignore
static mut THE_SERIAL_PORT: SerialPort = SerialPort;

fn main() {
    let _ = unsafe {
        THE_SERIAL_PORT.read_speed();
    };
}
```

但是這個帶來了一些問題。它是一個可變的全局變量，在Rust，與這些變量交互總是unsafe的。這些變量在你所有的程序間也是可見的，意味著借用檢查器不能幫你跟蹤這些變量的引用和所有權。

## 在Rust中要怎麼做?

與其只是讓我們的外設變成一個全局變量，我們不如創造一個結構體，在這個例子裡其被叫做 `PERIPHERALS`，這個全局變量對於我們的每個外設，它都有一個與之對應的 `Option<T>` ．

```rust,ignore
struct Peripherals {
    serial: Option<SerialPort>,
}
impl Peripherals {
    fn take_serial(&mut self) -> SerialPort {
        let p = replace(&mut self.serial, None);
        p.unwrap()
    }
}
static mut PERIPHERALS: Peripherals = Peripherals {
    serial: Some(SerialPort),
};
```

這個結構體允許我們獲得一個外設的實例。如果我們嘗試調用`take_serial()`獲得多個實例，我們的代碼將會拋出運行時恐慌(panic)！

```rust,ignore
fn main() {
    let serial_1 = unsafe { PERIPHERALS.take_serial() };
    // 這裡造成運行時恐慌！
    // let serial_2 = unsafe { PERIPHERALS.take_serial() };
}
```

雖然與這個結構體交互是`unsafe`，然而一旦我們獲得了它包含的 `SerialPort`，我們將不再需要使用`unsafe`，或者`PERIPHERALS`結構體。

這個帶來了少量的運行時開銷，因為我們必須打包 `SerialPort` 結構體進一個option中，且我們將需要調用一次 `take_serial()`，但是這種少量的前期成本，能使我們在接下來的程序中使用借用檢查器(borrow checker) 。

## 已存在的庫支持

雖然我們在上面生成了我們自己的 `Peripherals` 結構體，但這並不是必須的。`cortex_m` crate 包含一個被叫做 `singleton!()` 的宏，它可以為你完成這個任務。

```rust,ignore
use cortex_m::singleton;

fn main() {
    // OK 如果 `main` 只被執行一次
    let x: &'static mut bool =
        singleton!(: bool = false).unwrap();
}
```

[cortex_m docs](https://docs.rs/cortex-m/latest/cortex_m/macro.singleton.html)

另外，如果你使用 [`cortex-m-rtic`](https://github.com/rtic-rs/cortex-m-rtic)，它將獲取和定義這些外設的整個過程抽象了出來，你將獲得一個`Peripherals`結構體，其包含了所有你定義了的項的一個非 `Option<T>` 的版本。

```rust,ignore
// cortex-m-rtic v0.5.x
#[rtic::app(device = lm3s6965, peripherals = true)]
const APP: () = {
    #[init]
    fn init(cx: init::Context) {
        static mut X: u32 = 0;
         
        // Cortex-M外設
        let core: cortex_m::Peripherals = cx.core;
        
        // 設備特定的外設
        let device: lm3s6965::Peripherals = cx.device;
    }
}
```

## 為什麼？

但是這些單例模式是如何使我們的Rust代碼在工作方式上產生很大不同的?

```rust,ignore
impl SerialPort {
    const SER_PORT_SPEED_REG: *mut u32 = 0x4000_1000 as _;

    fn read_speed(
        &self // <------ 這個真的真的很重要
    ) -> u32 {
        unsafe {
            ptr::read_volatile(Self::SER_PORT_SPEED_REG)
        }
    }
}
```


這裡有兩個重要因素:

* 因為我們正在使用一個單例模式，所以我們只有一種方法或者地方去獲得一個 `SerialPort` 結構體。
* 為了調用 `read_speed()` 方法，我們必須擁有一個 `SerialPort` 結構體的所有權或者一個引用。

這兩個因素放在一起意味著，只有當我們滿足了借用檢查器的條件時，我們才有可能訪問硬件，也意味著在任何時候不可能存在多個對同一個硬件的可變引用(&mut)！

```rust,ignore
fn main() {
    // 缺少對`self`的引用！將不會工作。
    // SerialPort::read_speed();

    let serial_1 = unsafe { PERIPHERALS.take_serial() };

    // 你只能讀取你有權訪問的內容
    let _ = serial_1.read_speed();
}
```

## 像對待數據一樣對待硬件

另外，因為一些引用是可變的，一些是不可變的，就可以知道一個函數或者方法是否有能力修改硬件的狀態。比如，

這個函數可以改變硬件的配置:

```rust,ignore
fn setup_spi_port(
    spi: &mut SpiPort,
    cs_pin: &mut GpioPin
) -> Result<()> {
    // ...
}
```

這個不行:

```rust,ignore
fn read_button(gpio: &GpioPin) -> bool {
    // ...
}
```

這允許我們在**編譯時**而不是運行時強制代碼是否應該或者不應該對硬件進行修改。要注意，這通常在只有一個應用的情況下起作用，但是對於裸機系統來說，我們的軟件將被編譯進一個單一應用中，因此這通常不是一個限制。

