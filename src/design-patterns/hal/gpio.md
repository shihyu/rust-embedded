# 關於GPIO接口的建議

<a id="c-zst-pin"></a>
## Pin類型默認是零大小的(C-ZST-PIN)

由HAL暴露的GPIO接口應該為所有接口或者端口上的每一個管腳提供一個專用的零大小類型，從而當所有的管腳分配靜態已知時，提供一個零開銷抽象。

每個GPIO接口或者端口應該實現一個`split`方法，它返回一個擁有所有管腳的結構體。

案例:

```rust
pub struct PA0;
pub struct PA1;
// ...

pub struct PortA;

impl PortA {
    pub fn split(self) -> PortAPins {
        PortAPins {
            pa0: PA0,
            pa1: PA1,
            // ...
        }
    }
}

pub struct PortAPins {
    pub pa0: PA0,
    pub pa1: PA1,
    // ...
}
```

<a id="c-erased-pin"></a>
## 管腳類型提供方法去擦除管腳和端口(C-ERASED-PIN)

從編譯時到運行時，管腳都應該提供可以改變屬性的類型擦出方法，允許在應用中有更多的靈活性。

案例:

```rust
/// 端口 A, 管腳 0。
pub struct PA0;

impl PA0 {
    pub fn erase_pin(self) -> PA {
        PA { pin: 0 }
    }
}

/// 端口A上的A管腳。
pub struct PA {
    /// 管腳號。
    pin: u8,
}

impl PA {
    pub fn erase_port(self) -> Pin {
        Pin {
            port: Port::A,
            pin: self.pin,
        }
    }
}

pub struct Pin {
    port: Port,
    pin: u8,
    // (這些字段)
    // (這些字段可以打包以減少內存佔用)
}

enum Port {
    A,
    B,
    C,
    D,
}
```

<a id="c-pin-state"></a>
## 管腳狀態應該被編碼成類型參數 (C-PIN-STATE)

取決於芯片或者芯片系列，管腳可能被配置為具有不同特性的輸出或者輸入。這個狀態應該編碼進類型系統中以避免在錯誤的狀態中使用管腳。

另外，也可以用這個方法使用額外的類型參數編碼芯片特定的狀態(eg. 驅動強度)。

用來改變管腳狀態的方法應該被實現成`into_input`和`into_output`方法。

另外，應該提供`with_{input,output}_state`方法，在一個不同的狀態中臨時配置一個管腳而不是移動它。

應該為每個的管腳類型提供下列的方法(也就是說，已擦除和未擦除的管腳類型應該提供一樣的API):

* `pub fn into_input<N: InputState>(self, input: N) -> Pin<N>`
* `pub fn into_output<N: OutputState>(self, output: N) -> Pin<N>`
* ```ignore
  pub fn with_input_state<N: InputState, R>(
      &mut self,
      input: N,
      f: impl FnOnce(&mut PA1<N>) -> R,
  ) -> R
  ```
* ```ignore
  pub fn with_output_state<N: OutputState, R>(
      &mut self,
      output: N,
      f: impl FnOnce(&mut PA1<N>) -> R,
  ) -> R
  ```

管腳狀態應該用sealed traits來綁定。HAL的用戶不必添加他們自己的狀態。這個traits能提供HAL特定的方法，實現管腳狀態API需要這些方法。

案例:

```rust
# use std::marker::PhantomData;
mod sealed {
    pub trait Sealed {}
}

pub trait PinState: sealed::Sealed {}
pub trait OutputState: sealed::Sealed {}
pub trait InputState: sealed::Sealed {
    // ...
}

pub struct Output<S: OutputState> {
    _p: PhantomData<S>,
}

impl<S: OutputState> PinState for Output<S> {}
impl<S: OutputState> sealed::Sealed for Output<S> {}

pub struct PushPull;
pub struct OpenDrain;

impl OutputState for PushPull {}
impl OutputState for OpenDrain {}
impl sealed::Sealed for PushPull {}
impl sealed::Sealed for OpenDrain {}

pub struct Input<S: InputState> {
    _p: PhantomData<S>,
}

impl<S: InputState> PinState for Input<S> {}
impl<S: InputState> sealed::Sealed for Input<S> {}

pub struct Floating;
pub struct PullUp;
pub struct PullDown;

impl InputState for Floating {}
impl InputState for PullUp {}
impl InputState for PullDown {}
impl sealed::Sealed for Floating {}
impl sealed::Sealed for PullUp {}
impl sealed::Sealed for PullDown {}

pub struct PA1<S: PinState> {
    _p: PhantomData<S>,
}

impl<S: PinState> PA1<S> {
    pub fn into_input<N: InputState>(self, input: N) -> PA1<Input<N>> {
        todo!()
    }

    pub fn into_output<N: OutputState>(self, output: N) -> PA1<Output<N>> {
        todo!()
    }

    pub fn with_input_state<N: InputState, R>(
        &mut self,
        input: N,
        f: impl FnOnce(&mut PA1<N>) -> R,
    ) -> R {
        todo!()
    }

    pub fn with_output_state<N: OutputState, R>(
        &mut self,
        output: N,
        f: impl FnOnce(&mut PA1<N>) -> R,
    ) -> R {
        todo!()
    }
}

// 對於`PA`和`Pin`一樣的，對於其它管腳類型來說也是。
```
