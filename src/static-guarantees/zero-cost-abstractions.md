# 零成本抽象

類型狀態是一個零成本抽象的傑出案例 - 把某些行為移到編譯時執行或者分析的能力。這些類型狀態不包含真實的數據，只用來作為標記。因為它們不包含數據，在運行時它們在內存中不存在實際的表示。

```rust,ignore
use core::mem::size_of;

let _ = size_of::<Enabled>();    // == 0
let _ = size_of::<Input>();      // == 0
let _ = size_of::<PulledHigh>(); // == 0
let _ = size_of::<GpioConfig<Enabled, Input, PulledHigh>>(); // == 0
```

## 零大小的類型(Zero Sized Types)

```rust,ignore
struct Enabled;
```

像這樣定義的結構體被稱為零大小的類型，因為它們不包含實際數據。雖然這些類型在編譯時像是"真實的"(real) - 你可以拷貝它們，移動它們，引用它們，等等，然而優化器將會完全跳過它們。

在這個代碼片段裡:

```rust,ignore
pub fn into_input_high_z(self) -> GpioConfig<Enabled, Input, HighZ> {
    self.periph.modify(|_r, w| w.input_mode().high_z());
    GpioConfig {
        periph: self.periph,
        enabled: Enabled,
        direction: Input,
        mode: HighZ,
    }
}
```

我們返回的GpioConfig在運行時並不存在。對這個函數的調用通常會被歸納為一條彙編指令 - 把一個常量寄存器值存進一個寄存器裡。這意味著我們開發的類型狀態接口是一個零成本抽象 - 它不會用更多的CPU，RAM，或者代碼空間去跟蹤`GpioConfig`的狀態，會被渲染成和直接訪問寄存器一樣的機器碼。

## 嵌套

通常，這些抽象可能會被深深地嵌套起來。一旦結構體使用的所有的組件是零大小類型的，整個結構體將不會在運行時存在。

對於複雜或者深度嵌套的結構體，定義所有可能的狀態組合可能很乏味。在這些例子中，宏可能可以被用來生成所有的實現。
