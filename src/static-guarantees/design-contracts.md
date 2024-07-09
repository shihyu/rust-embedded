# 設計約定(design contracts)

在我們的上個章節中，我們寫了一個接口，但沒有強制遵守設計約定。讓我們再看下我們假想的GPIO配置寄存器：


| 名字          | 位數(s) | 值 | 含義   | 註釋 |
| ---:         | ------------: | ----: | ------:   | ----: |
| 使能       | 0             | 0     | 關閉  | 關閉GPIO |
|              |               | 1     | 使能   | 使能GPIO |
| 方向    | 1             | 0     | 輸入     | 方向設置成輸入 |
|              |               | 1     | 輸出    | 方向設置成輸出 |
| 輸入模式   | 2..3          | 00    | 高阻態      | 輸入設置為高阻態 |
|              |               | 01    | 下拉  | 下拉輸入管腳 |
|              |               | 10    | 上拉 | 上拉輸入管腳 |
|              |               | 11    | n/a       | 無效狀態。不要設置 |
| 輸出模式  | 4             | 0     | 拉低   | 把管腳設置成低電平 |
|              |               | 1     | 拉高  | 把管腳設置成高電平 |
| 輸入狀態 | 5             | x     | 輸入電平    | 如果輸入 < 1.5v 為0，如果輸入 >= 1.5v 為1 |


如果在使用底層硬件之前檢查硬件的狀態，在運行時強制用戶遵守設計約定，代碼可能像這一樣:

```rust,ignore
/// GPIO接口
struct GpioConfig {
    /// 由svd2rust生成的GPIO配製結構體
    periph: GPIO_CONFIG,
}

impl GpioConfig {
    pub fn set_enable(&mut self, is_enabled: bool) {
        self.periph.modify(|_r, w| {
            w.enable().set_bit(is_enabled)
        });
    }

    pub fn set_direction(&mut self, is_output: bool) -> Result<(), ()> {
        if self.periph.read().enable().bit_is_clear() {
            // 必須被使能配置方向
            return Err(());
        }

        self.periph.modify(|r, w| {
            w.direction().set_bit(is_output)
        });

        Ok(())
    }

    pub fn set_input_mode(&mut self, variant: InputMode) -> Result<(), ()> {
        if self.periph.read().enable().bit_is_clear() {
            // 必須被使能配置輸入模式
            return Err(());
        }

        if self.periph.read().direction().bit_is_set() {
            // 方向必須被設置成輸入
            return Err(());
        }

        self.periph.modify(|_r, w| {
            w.input_mode().variant(variant)
        });

        Ok(())
    }

    pub fn set_output_status(&mut self, is_high: bool) -> Result<(), ()> {
        if self.periph.read().enable().bit_is_clear() {
            // 設置輸出狀態必須被使能
            return Err(());
        }

        if self.periph.read().direction().bit_is_clear() {
            // 方向必須是輸出
            return Err(());
        }

        self.periph.modify(|_r, w| {
            w.output_mode.set_bit(is_high)
        });

        Ok(())
    }

    pub fn get_input_status(&self) -> Result<bool, ()> {
        if self.periph.read().enable().bit_is_clear() {
            // 獲取狀態必須被使能
            return Err(());
        }

        if self.periph.read().direction().bit_is_set() {
            // 方向必須是輸入
            return Err(());
        }

        Ok(self.periph.read().input_status().bit_is_set())
    }
}
```

因為需要強制遵守硬件上的限制，所以最後做了很多運行時檢查，它浪費了我們很多時間和資源，對於開發者來說，這個代碼用起來就沒那麼愉快了。

## 類型狀態(Type states)

但是，如果我們讓Rust的類型系統去強制遵守狀態轉換的規則會怎樣？看下這個例子:

```rust,ignore
/// GPIO接口
struct GpioConfig<ENABLED, DIRECTION, MODE> {
    /// 由svd2rust產生的GPIO配置結構體
    periph: GPIO_CONFIG,
    enabled: ENABLED,
    direction: DIRECTION,
    mode: MODE,
}

// GpioConfig中MODE的類型狀態
struct Disabled;
struct Enabled;
struct Output;
struct Input;
struct PulledLow;
struct PulledHigh;
struct HighZ;
struct DontCare;

/// 這些函數可能被用於所有的GPIO管腳
impl<EN, DIR, IN_MODE> GpioConfig<EN, DIR, IN_MODE> {
    pub fn into_disabled(self) -> GpioConfig<Disabled, DontCare, DontCare> {
        self.periph.modify(|_r, w| w.enable.disabled());
        GpioConfig {
            periph: self.periph,
            enabled: Disabled,
            direction: DontCare,
            mode: DontCare,
        }
    }

    pub fn into_enabled_input(self) -> GpioConfig<Enabled, Input, HighZ> {
        self.periph.modify(|_r, w| {
            w.enable.enabled()
             .direction.input()
             .input_mode.high_z()
        });
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Input,
            mode: HighZ,
        }
    }

    pub fn into_enabled_output(self) -> GpioConfig<Enabled, Output, DontCare> {
        self.periph.modify(|_r, w| {
            w.enable.enabled()
             .direction.output()
             .input_mode.set_high()
        });
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Output,
            mode: DontCare,
        }
    }
}

/// 這個函數可能被用於一個輸出管腳
impl GpioConfig<Enabled, Output, DontCare> {
    pub fn set_bit(&mut self, set_high: bool) {
        self.periph.modify(|_r, w| w.output_mode.set_bit(set_high));
    }
}

/// 這些方法可能被用於任意一個使能的輸入GPIO
impl<IN_MODE> GpioConfig<Enabled, Input, IN_MODE> {
    pub fn bit_is_set(&self) -> bool {
        self.periph.read().input_status.bit_is_set()
    }

    pub fn into_input_high_z(self) -> GpioConfig<Enabled, Input, HighZ> {
        self.periph.modify(|_r, w| w.input_mode().high_z());
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Input,
            mode: HighZ,
        }
    }

    pub fn into_input_pull_down(self) -> GpioConfig<Enabled, Input, PulledLow> {
        self.periph.modify(|_r, w| w.input_mode().pull_low());
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Input,
            mode: PulledLow,
        }
    }

    pub fn into_input_pull_up(self) -> GpioConfig<Enabled, Input, PulledHigh> {
        self.periph.modify(|_r, w| w.input_mode().pull_high());
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Input,
            mode: PulledHigh,
        }
    }
}
```

現在讓我們看下代碼如何用這個API:

```rust,ignore
/*
 * 案例 1: 從未配置到高阻輸入
 */
let pin: GpioConfig<Disabled, _, _> = get_gpio();

// 不能這麼做，pin沒有被使能
// pin.into_input_pull_down();

// 現在把管腳從未配置變為高阻輸入
let input_pin = pin.into_enabled_input();

// 從管腳讀取
let pin_state = input_pin.bit_is_set();

// 不能這麼做，輸入管腳沒有這個接口
// input_pin.set_bit(true);

/*
 * 案例 2: 高阻輸入到下拉輸入
 */
let pulled_low = input_pin.into_input_pull_down();
let pin_state = pulled_low.bit_is_set();

/*
 * 案例 3: 下拉輸入到輸出, 拉高
 */
let output_pin = pulled_low.into_enabled_output();
output_pin.set_bit(true);

// 不能這麼做，輸出管腳沒有這個接口
// output_pin.into_input_pull_down();
```

這絕對是存儲管腳狀態的便捷方法，但是為什麼這麼做?為什麼這比把狀態當成一個`enum`存在我們的`GpioConfig`結構體中更好？

## 編譯時功能安全(Functional Safety)

因為我們在編譯時完全強制遵守設計約定，這造成了沒有運行時開銷。當管腳處於輸入模式時時，是不可能設置輸出模式的。必須先把它設置成一個輸出管腳，然後再設置輸出模式。因為在執行一個函數前會檢查現在的狀態，因此沒有運行時消耗。

也因為這些狀態被類型系統強制遵守，因此沒有為這個接口的使用者留太多的犯錯餘地。如果它們嘗試執行一個非法的狀態轉換，代碼將不會編譯成功！
