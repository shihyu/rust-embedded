# 使用C的Rust

要在一個Rust項目中使用C或者C++，主要有兩個步驟:

+ 用Rust封裝要暴露出來使用的C API
+ 編譯要和Rust代碼集成在一起的C或者C++代碼

因為對於Rust編譯器來說，C++沒有一個穩定的ABI，當要將Rust和C或者C++結合時，建議優先選擇`C`。

## 定義接口

在Rust消費C或者C++代碼之前，必須定義(在Rust中定義)，在要被鏈接的代碼中存在什麼數據類型和函數簽名。在C或者C++中，你要包含一個頭文件(`.h`或者`.hpp`)，其定義了這個數據。而在Rust中，必須手動地將這些定義翻譯成Rust，或者使用一個工具去生成這些定義。

首先，我們將介紹如何將這些定義從C/C++手動地轉換為Rust。

### 封裝C函數和數據類型

通常，用C或者C++寫的庫會提供一個頭文件，頭文件定義了所有的類型和用於公共接口的函數。如下是一個示例文件:

```C
/* 文件: cool.h */
typedef struct CoolStruct {
    int x;
    int y;
} CoolStruct;

void cool_function(int i, char c, CoolStruct* cs);
```

當翻譯成Rust時，這個接口將看起來像是:

```rust,ignore
/* File: cool_bindings.rs */
#[repr(C)]
pub struct CoolStruct {
    pub x: cty::c_int,
    pub y: cty::c_int,
}

extern "C" {
    pub fn cool_function(
        i: cty::c_int,
        c: cty::c_char,
        cs: *mut CoolStruct
    );
}
```

讓我們一次看一個語句，來解釋每個部分。

```rust,ignore
#[repr(C)]
pub struct CoolStruct { ... }
```

默認，Rust不會保證包含在`struct`中的數據的大小，填充，或者順序。為了保證與C代碼兼容，我們使用`#[repr(C)]`屬性，它指示Rust編譯器總是使用和C一樣的規則去組織一個結構體中的數據。

```rust,ignore
pub x: cty::c_int,
pub y: cty::c_int,
```

由於C或者C++定義一個`int`或者`char`的方式很靈活，所以建議使用在`cty`中定義的基礎類型，它將類型從C映射到Rust中的類型。

```rust,ignore
extern "C" { pub fn cool_function( ... ); }
```

這個語句定義了一個使用C ABI的函數的簽名，叫做`cool_function`。因為只定義了簽名而沒有定義函數的主體，所以這個函數的定義將需要在其它地方定義，或者從一個靜態庫鏈接進最終的庫或者一個二進制文件中。

```rust,ignore
    i: cty::c_int,
    c: cty::c_char,
    cs: *mut CoolStruct
```

與我們上面的數據類型一樣，我們使用C兼容的定義去定義函數參數的數據類型。為了清晰可見，我們還保留了相同的參數名。

這裡我們有了一個新類型，`*mut CoolStruct` 。因為C沒有Rust中像 `&mut CoolStruct` 這樣的引用，替代的是一個裸指針。所以解引用這個指針是`unsafe`的，因為這個指針實際上可能是一個`null`指針，因此當與C或者C++代碼交互時必須要小心對待那些Rust做出的安全保證。

### 自動產生接口

有一個叫做[bindgen]的工具，它可以自動執行這些轉換，而不用手動生成這些接口，手動進行這樣的操作非常繁瑣且容易出錯。關於[bindgen]的使用指令，可以參考[bindgen user's manual]，常用的步驟大致如下:

1. 收集所有定義了你可能在Rust中會用到的數據類型或者接口的C或者C++頭文件。
2. 寫一個`bindings.h`文件，其`#include "..."`每一個你在步驟一中收集的文件。
3. 將這個`bindings.h`文件和任何用來編譯你代碼的編譯標識發給`bindgen`。貼士: 使用`Builder.ctypes_prefix("cty")` / `--ctypes-prefix=cty` 和 `Builder.use_core()` / `--use-core` 去使生成的代碼兼容`#![no_std]`
4. `bindgen`將會在終端窗口輸出生成的Rust代碼。這個文件可能會被通過管道發送給你項目中的一個文件，比如`bindings.rs` 。你可能要在你的Rust項目中使用這個文件來與被編譯和鏈接成一個外部庫的C/C++代碼交互。貼士: 如果你的類型在生成的綁定中被前綴了`cty`，不要忘記使用[`cty`](https://crates.io/crates/cty) crate 。

[bindgen]: https://github.com/rust-lang/rust-bindgen
[bindgen user's manual]: https://rust-lang.github.io/rust-bindgen/

## 編譯你的 C/C++ 代碼

因為Rust編譯器並不直接知道如何編譯C或者C++代碼(或者從其它語言來的代碼，其提供了一個C接口)，所以必須要靜態編譯你的非Rust代碼。

對於嵌入式項目，這通常意味著把C/C++代碼編譯成一個靜態庫文檔(比如 `cool-library.a`)，然後其能在最後鏈接階段與你的Rust代碼組合起來。

如果你要使用的庫已經作為一個靜態庫文檔被髮布，那就沒必要重新編譯你的代碼。只需按照上面所述轉換提供的接口頭文件，且在編譯/鏈接時包含靜態庫文檔。

如果你的代碼作為一個源項目(source project)存在，將你的C/C++代碼編譯成一個靜態庫將是必須的，要麼通過使用你現存的編譯系統(比如 `make`，`CMake`，等等)，要麼通過使用一個被叫做`cc` crate的工具移植必要的編譯步驟。關於這兩個，都必須使用一個`build.rs`腳本。

### Rust的 `build.rs` 編譯腳本

一個 `build.rs` 腳本是一個用Rust語法編寫的文件，它被運行在你的編譯機器上，發生在你項目的依賴項被編譯**之後**，但是在你的項目被編譯**之前** 。

可能能在[這裡](https://doc.rust-lang.org/cargo/reference/build-scripts.html)發現完整的參考。`build.rs` 腳本能用來生成代碼(比如通過[bindgen])，調用外部編譯系統，比如`Make`，或者直接通過使用`cc` crate來直接編譯C/C++ 。

### 使用外部編譯系統

對於有複雜的外部項或者編譯系統的項目，使用[`std::process::Command`]通過遍歷相對路徑來向其它編譯系統"輸出"，調用一個固定的命令(比如 `make library`)，然後拷貝最終的靜態庫到`target`編譯文件夾中恰當的位置，可能是最簡單的方法。

雖然你的crate目標可能是一個`no_std`嵌入式平臺，但你的`build.rs`只運行在負責編譯你的crate的機器上。這意味著你能使用任何Rust crates，其將運行在你的編譯主機上。

[`std::process::Command`]: https://doc.rust-lang.org/std/process/struct.Command.html

### 使用`cc` crate構建C/C++代碼

對於具有有限的依賴項或者複雜度的項目，或者對於那些難以修改編譯系統去生成一個靜態庫(而不是一個二進制文件或者可執行文件)的項目，使用[`cc` crate]可能更容易，它提供了一個符合Rust語法的接口，這個接口是關於主機提供的編譯器的。

[`cc` crate]: https://github.com/alexcrichton/cc-rs

在把一個C文件編譯成一個靜態庫的依賴項的最簡單的場景下，可以使用[`cc` crate]，示例`build.rs`腳本看起來像這樣:

```rust,ignore
fn main() {
    cc::Build::new()
        .file("src/foo.c")
        .compile("foo");
}
```

要把`build.rs`放在包的根目錄下．然後`cargo build`會在構建包之前編譯和執行它．一個靜態的名為`libfoo.a`的歸檔文件會生成並被放在`target`文件夾中．
