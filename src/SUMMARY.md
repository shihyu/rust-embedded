# Summary

<!--

Definition of the organization of this book is still a work in process.

Refer to https://github.com/rust-embedded/book/issues for
more information and coordination

-->

- [引言](./intro/index.md)
    - [硬件](./intro/hardware.md)
    - [`no_std`](./intro/no-std.md)
    - [工具](./intro/tooling.md)
    - [安裝](./intro/install.md)
        - [Linux](./intro/install/linux.md)
        - [MacOS](./intro/install/macos.md)
        - [Windows](./intro/install/windows.md)
        - [驗證工具鏈的安裝](./intro/install/verify.md)
- [開始](./start/index.md)
  - [QEMU](./start/qemu.md)
  - [硬件](./start/hardware.md)
  - [存儲映射的寄存器](./start/registers.md)
  - [半主機模式](./start/semihosting.md)
  - [運行時恐慌(Panicking)](./start/panicking.md)
  - [異常](./start/exceptions.md)
  - [中斷](./start/interrupts.md)
  - [IO](./start/io.md)
- [外設](./peripherals/index.md)
    - [Rust嚐鮮](./peripherals/a-first-attempt.md)
    - [借用檢查器](./peripherals/borrowck.md)
    - [單例](./peripherals/singletons.md)
- [靜態保障(static guarantees)](./static-guarantees/index.md)
    - [類型狀態編程](./static-guarantees/typestate-programming.md)
    - [把外設當作狀態機](./static-guarantees/state-machines.md)
    - [設計約定](./static-guarantees/design-contracts.md)
    - [零成本抽象](./static-guarantees/zero-cost-abstractions.md)
- [可移植性](./portability/index.md)
- [併發](./concurrency/index.md)
- [容器](./collections/index.md)
- [設計模式](./design-patterns/index.md)
    - [HALs](./design-patterns/hal/index.md)
        - [列表](./design-patterns/hal/checklist.md)
        - [命名](./design-patterns/hal/naming.md)
        - [互操性](./design-patterns/hal/interoperability.md)
        - [可預見性](./design-patterns/hal/predictability.md)
        - [GPIO](./design-patterns/hal/gpio.md)
- [給嵌入式C開發者的貼士](./c-tips/index.md)
    <!-- TODO: Define Sections -->
- [互操性](./interoperability/index.md)
    - [使用C的Rust](./interoperability/c-with-rust.md)
    - [使用Rust的C](./interoperability/rust-with-c.md)
- [沒有排序的主題](./unsorted/index.md)
  - [優化: 速度與大小間的博弈](./unsorted/speed-vs-size.md)
  - [執行數學運算](./unsorted/math.md)

---

[附錄A: 詞彙表](./appendix/glossary.md)
