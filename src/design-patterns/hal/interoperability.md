# 互用性


<a id="c-free"></a>
## 封裝類型提供一個析構方法 (C-FREE)

任何由HAL提供的非`Copy`封裝類型應該提供一個`free`方法，這個方法消費封裝類且返回最初生成它的外設(可能是其它對象)。

如果有必要，方法應該關閉和重置外設。使用由`free`返回的原始外設去調用`new`不應該由於設備的意外狀態而失敗，

如果HAL類型要求構造其它的非`Copy`對象(比如 I/O 管腳)，任何這樣的對象應該也由`free`返回和釋放。在這種情況下`free`應該返回一個元組。

比如:

```rust
# pub struct TIMER0;
pub struct Timer(TIMER0);

impl Timer {
    pub fn new(periph: TIMER0) -> Self {
        Self(periph)
    }

    pub fn free(self) -> TIMER0 {
        self.0
    }
}
```

<a id="c-reexport-pac"></a>
## HALs重新導出它們的寄存器訪問crate(C-REEXPORT-PAC)

可以在[svd2rust]生成的PACs之上，或在其它純寄存器訪問的crates之上編寫HALs。HALs需要在crate root中重新導出它們所基於的寄存器訪問crate

一個PAC應該被重新導出在`pac`名下，無論這個crate實際的名字是什麼，因為HAL的名字應該已經明確了正被訪問的是什麼PAC 。

[svd2rust]: https://github.com/rust-embedded/svd2rust

<a id="c-hal-traits"></a>
## 類型實現`embedded-hal` traits (C-HAL-TRAITS)

HAL提供的類型應該實現所有的由[`embedded-hal`] crate提供的能用的traits。

同個類型可能實現多個traits。

[`embedded-hal`]: https://github.com/rust-embedded/embedded-hal
