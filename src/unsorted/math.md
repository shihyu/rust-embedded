# 在`#[no_std]`下執行數學運算

如果你想要執行數學相關的函數，像是計算平方根或者一個數的指數並有完整的標準庫支持，代碼可能看起來像這樣:

```rs
//! 可用一些標準支持的數學函數

fn main() {
    let float: f32 = 4.82832;
    let floored_float = float.floor();

    let sqrt_of_four = floored_float.sqrt();

    let sinus_of_four = floored_float.sin();

    let exponential_of_four = floored_float.exp();
    println!("Floored test float {} to {}", float, floored_float);
    println!("The square root of {} is {}", floored_float, sqrt_of_four);
    println!("The sinus of four is {}", sinus_of_four);
    println!(
        "The exponential of four to the base e is {}",
        exponential_of_four
    )
}
```

沒有標準庫支持的時候，這些函數不可用。反而可以使用像是[`libm`](https://crates.io/crates/libm)這樣一個外部庫。示例的代碼將會看起來像這樣:

```rs
#![no_main]
#![no_std]

use panic_halt as _;

use cortex_m_rt::entry;
use cortex_m_semihosting::{debug, hprintln};
use libm::{exp, floorf, sin, sqrtf};

#[entry]
fn main() -> ! {
    let float = 4.82832;
    let floored_float = floorf(float);

    let sqrt_of_four = sqrtf(floored_float);

    let sinus_of_four = sin(floored_float.into());

    let exponential_of_four = exp(floored_float.into());
    hprintln!("Floored test float {} to {}", float, floored_float).unwrap();
    hprintln!("The square root of {} is {}", floored_float, sqrt_of_four).unwrap();
    hprintln!("The sinus of four is {}", sinus_of_four).unwrap();
    hprintln!(
        "The exponential of four to the base e is {}",
        exponential_of_four
    )
    .unwrap();
    // 退出QEMU
    // 注意不要在硬件上使用這個; 它能破壞OpenOCD的狀態
    // debug::exit(debug::EXIT_SUCCESS);

    loop {}
}
```

如果需要在MCU上執行更復雜的操作，像是DSP信號處理或者更高級的線性代數，下列的crates可能可以幫到你

- [CMSIS DSP library binding](https://github.com/jacobrosenthal/cmsis-dsp-sys)
- [`constgebra`](https://crates.io/crates/constgebra)
- [`micromath`](https://github.com/tarcieri/micromath)
- [`microfft`](https://crates.io/crates/microfft)
- [`nalgebra`](https://github.com/dimforge/nalgebra)
