# 作為狀態機的外設

一個微控制器的外設可以被想成是一組狀態機。比如，一個簡化的[GPIO管腳]的配置可以被表達成下列的狀態樹:

[GPIO管腳]: https://en.wikipedia.org/wiki/General-purpose_input/output

* 關閉
* 使能
    * 配置成輸出
        * 輸出: 高
        * 輸出: 低
    * 配置成輸入
        * 輸入: 高阻態
        * 輸入: 下拉
        * 輸入: 上拉

如果外設開始於`關閉`模式，切換到`輸入: 高阻態`模式，我們必須執行下面的步驟:

1. 關閉
2. 使能
3. 配置成輸入
4. 輸入: 高阻態

如果我們想要從`輸入: 高阻態`切換到`輸入: 下拉`，我們必須執行下列的步驟:

1. 輸入: 高阻抗
2. 輸入: 下拉

同樣地，如果我們想要把一個GPIO管腳從`輸入: 下拉`切換到`輸出: 高`，我們必須執行下列的步驟:
1. 輸入: 下拉
2. 配置成輸入
3. 配置成輸出
4. 輸出: 高

## 硬件表徵(Hardware Representation)

通常，通過向映射到GPIO外設上的指定的寄存器中寫入值可以配置上面列出的狀態。讓我們定義一個假想的GPIO配置寄存器來解釋下它:

| 名字          | 位數(s) | 值 | 含義   | 註釋 |
| ---:         | ------------: | ----: | ------:   | ----: |
| 使能       | 0             | 0     | 關閉  | 關閉GPIO |
|              |               | 1     | 使能   | 使能GPIO |
| 方向    | 1             | 0     | 輸入     | 方向設置成輸入 |
|              |               | 1     | 輸出    | 方向設置成輸出 |
| 輸入模式   | 2..3          | 00    | hi-z      | 輸入設置為高阻態 |
|              |               | 01    | 下拉  | 下拉輸入管腳 |
|              |               | 10    | 上拉 | 上拉輸入管腳 |
|              |               | 11    | n/a       | 無效狀態。不要設置 |
| 輸出模式  | 4             | 0     | 拉低   | 輸出管腳變成地電平 |
|              |               | 1     | 拉高  | 輸出管腳變成高電平 |
| 輸入狀態 | 5             | x     | in-val    | 如果輸入 < 1.5v為0，如果輸入 >= 1.5v為1 |

_可以_ 在Rust中暴露下列的結構體來控制這個GPIO:

```rust,ignore
/// GPIO接口
struct GpioConfig {
    /// 由svd2rust生成的GPIO配置結構體
    periph: GPIO_CONFIG,
}

impl GpioConfig {
    pub fn set_enable(&mut self, is_enabled: bool) {
        self.periph.modify(|_r, w| {
            w.enable().set_bit(is_enabled)
        });
    }

    pub fn set_direction(&mut self, is_output: bool) {
        self.periph.modify(|_r, w| {
            w.direction().set_bit(is_output)
        });
    }

    pub fn set_input_mode(&mut self, variant: InputMode) {
        self.periph.modify(|_r, w| {
            w.input_mode().variant(variant)
        });
    }

    pub fn set_output_mode(&mut self, is_high: bool) {
        self.periph.modify(|_r, w| {
            w.output_mode.set_bit(is_high)
        });
    }

    pub fn get_input_status(&self) -> bool {
        self.periph.read().input_status().bit_is_set()
    }
}
```

然而，這會允許我們修改某些沒有意義的寄存器。比如，如果當我們的GPIO被配置為輸入時我們設置`output_mode`字段，將會發生什麼？

通常使用這個結構體會允許我們訪問到上面的狀態機沒有定義的狀態：比如，一個被上拉的輸出，或者一個被拉高的輸入。對於一些硬件，這並沒有關係。對另外一些硬件來說，這將會導致不可預期或者沒有定義的行為！

雖然這個接口很方便寫入，但是它沒有強制我們遵守硬件實現所設的設計約定。
