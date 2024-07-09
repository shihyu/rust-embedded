# 使用Rust的C

在C或者C++中使用Rust代碼通常由兩部分組成。

- 用Rust生成一個C友好的API
- 將你的Rust項目嵌入一個外部的編譯系統

除了`cargo`和`meson`，大多數編譯系統沒有原生Rust支持。因此你最好只用`cargo`編譯你的crate和依賴。

## 設置一個項目

像往常一樣創建一個新的`cargo`項目。有一些標誌可以告訴`cargo`去生成一個系統庫，而不是常規的rust目標文件。如果你想要它與crate的其它部分不一樣，你也可以為你的庫設置一個不同的輸出名。

```toml
[lib]
name = "your_crate"
crate-type = ["cdylib"]      # 生成動態鏈接庫
# crate-type = ["staticlib"] # 生成靜態鏈接庫
```

## 構建一個`C` API

因為對於Rust編譯器來說，C++沒有穩定的ABI，因此對於不同語言間的互操性我們使用`C`。在C和C++代碼的內部使用Rust時也不例外。

### `#[no_mangle]`

Rust對符號名的修飾與主機的代碼鏈接器所期望的不同。因此，需要告知任何被Rust導出到Rust外部去使用的函數不要被編譯器修飾。

### `extern "C"`

默認，任何用Rust寫的函數將使用Rust ABI(這也不穩定)。當編譯面向外部的FFI APIs時，我們需要告訴編譯器去使用系統ABI 。 

取決於你的平臺，你可能想要針對一個特定的ABI版本，其記錄在[這裡](https://doc.rust-lang.org/reference/items/external-blocks.html)。

---

把這些部分放在一起，你得到一個函數，其粗略看起來像是這個。

```rust,ignore
#[no_mangle]
pub extern "C" fn rust_function() {

}
```

就像在Rust項目中使用`C`代碼時那樣，現在需要把數據轉換為應用中其它部分可以理解的形式。

## 鏈接和更大的項目上下文

問題只解決了一半。

你現在要如何使用它?

**這很大程度上取決於你的項目或者編譯系統**

`cargo`將生成一個`my_lib.so`/`my_lib.dll`或者`my_lib.a`文件，取決於你的平臺和配置。可以通過編譯系統簡單地鏈接這個庫。

然而，從C調用一個Rust函數要求一個頭文件去聲明函數的簽名。

在Rust-ffi API中的每個函數需要有一個相關的頭文件函數。

```rust,ignore
#[no_mangle]
pub extern "C" fn rust_function() {}
```

將會變成

```C
void rust_function();
```

等等。

這裡有個工具可以自動化這個過程，叫做[cbindgen]，其會分析你的Rust代碼然後為C和C++項目生成頭文件。

[cbindgen]: https://github.com/eqrion/cbindgen

此時從C中使用Rust函數非常簡單，只需包含頭文件和調用它們！

```C
#include "my-rust-project.h"
rust_function();
```
