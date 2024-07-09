# 互操性

Rust和C代碼之間的互操性始終依賴於數據在兩個語言間的轉換．為了互操性，在`stdlib`中有一個專用的模塊，叫作
[`std::ffi`](https://doc.rust-lang.org/std/ffi/index.html).

`std::ffi`提供了與C基礎類型對應的類型定義，比如`char`， `int`，和`long`．
它也提供了一些工具用於更復雜的類型之間的轉換，比如字符串，可以把`&str`和`String`映射成更容易和安全處理的C類型．

從Rust 1.30以來，`std::ffi`的功能也出現在`core::ffi`或者`alloc::ffi`中，取決於是否涉及到內存分配．
[`cty`]庫和[`cstr_core`]庫也提供了相同的功能．

[`cstr_core`]: https://crates.io/crates/cstr_core
[`cty`]: https://crates.io/crates/cty

| Rust類型      | 間接 | C類型        |
|----------------|--------------|----------------|
| `String`       | `CString`    | `char *`       |
| `&str`         | `CStr`       | `const char *` |
| `()`           | `c_void`     | `void`         |
| `u32` or `u64` | `c_uint`     | `unsigned int` |
| etc            | ...          | ...            |

一個C基本類型的值可以被用來作為相關的Rust類型的值，反之亦然，因此前者僅僅是後者的一個類型偽名．
比如，下列的代碼可以在`unsigned int`是32位寬的平臺上編譯．

```rust,ignore
fn foo(num: u32) {
    let c_num: c_uint = num;
    let r_num: u32 = c_num;
}
```

## 與其它編譯系統的互用性

在嵌入式項目中引入Rust的一個常見需求是，把Cargo結合進你現存的編譯系統中，比如make或者cmake。

在[issue #61]的issue tracker上，我們正在為這個需求收集例子和用例。

[issue #61]: https://github.com/rust-embedded/book/issues/61


## 與RTOSs的互操性

將Rust和一個RTOS集成在一起，比如FreeRTOS或者ChibiOS仍然在進行中; 尤其是從Rust調用RTOS函數可能很棘手。

在[issue #62]的issue tracker上，我們正為這件事收集例子和用例。

[issue #62]: https://github.com/rust-embedded/book/issues/62
