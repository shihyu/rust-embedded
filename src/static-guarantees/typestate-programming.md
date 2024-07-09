# 類型狀態編程(Typestate Programming)

[typestates]的概念是指將有關對象當前狀態的信息編碼進該對象的類型中。雖然這聽起來有點神秘，如果你在Rust中用過[建造者模式]，你就已經開始使用類型狀態編程了！

[typestates]: https://en.wikipedia.org/wiki/Typestate_analysis
[建造者模式]: https://doc.rust-lang.org/1.0.0/style/ownership/builders.html

```rust
pub mod foo_module {
    #[derive(Debug)]
    pub struct Foo {
        inner: u32,
    }

    pub struct FooBuilder {
        a: u32,
        b: u32,
    }

    impl FooBuilder {
        pub fn new(starter: u32) -> Self {
            Self {
                a: starter,
                b: starter,
            }
        }

        pub fn double_a(self) -> Self {
            Self {
                a: self.a * 2,
                b: self.b,
            }
        }

        pub fn into_foo(self) -> Foo {
            Foo {
                inner: self.a + self.b,
            }
        }
    }
}

fn main() {
    let x = foo_module::FooBuilder::new(10)
        .double_a()
        .into_foo();

    println!("{:#?}", x);
}
```

在這個例子裡，不能直接生成一個`Foo`對象。必須先生成一個`FooBuilder`，並且恰當地初始化`FooBuilder`後，才能獲取到需要的`Foo`對象。

這個最小的例子編碼了兩個狀態:

* `FooBuilder`，其表示一個"沒有被配置"，或者"正在配置"狀態
* `Foo`，其表示了一個"被配置"，或者"可以使用"狀態。

## 強類型

因為Rust有一個[強類型系統]，沒有什麼簡單的方法可以奇蹟般地生成一個`Foo`實例，也沒有簡單的方法可以不用調用`into_foo()`方法而把一個`FooBuilder`變成一個`Foo`。另外，調用`into_foo()`方法消費了最初的`FooBuilder`結構體，意味著不生成一個新的實例就不能被再次使用它。

[強類型系統]: https://en.wikipedia.org/wiki/Strong_and_weak_typing

這允許我們可以將系統的狀態表示成類型，把狀態轉換必須的動作包括進轉換兩個類型的方法中。通過生成一個 `FooBuilder`，轉換成一個 `Foo` 對象，我們已經使用了一個基本的狀態機。

