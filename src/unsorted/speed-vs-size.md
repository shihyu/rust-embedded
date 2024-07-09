# 優化: 速度與大小之間的博弈

每個人都想要程序變得即快又小，但是同時滿足這兩個條件是不可能的。這部分討論`rustc`提供的不同的優化等級，和它們是如何影響執行時間和一個程序的二進制項的大小。

## 無優化

這是默認的。當你調用`cargo build`時，你使用的是development(又叫`dev`)配置。這個配置優化的目的是為了調試，因此它使能了調試信息且*關閉*了所有優化，i.e. 它使用 `-C opt-level = 0` 。

至少對於裸機開發來說，調試信息不會佔用Flash/ROM中的空間，意味著在這種情況下，調試信息是零開銷的，因此實際上我們推薦你在release配置中使能調試信息 -- 默認它被關閉了。那會讓你調試release版本的固件時可以使用斷點。

``` toml
[profile.release]
# 調試符號很好且它們不會增加Flash上的大小
debug = true
```

無優化對於調試來說是最好的選擇，因為單步調試代碼感覺像是你正在逐條語句地執行程序，且你能在GDB中`print`棧變量和函數參數。當代碼被優化了，嘗試打印變量會導致`$0 = <value optimized out>`被打印出來。

`dev`配置最大的缺點就是最終的二進制項將會變得巨大且緩慢。大小通常是一個更大的問題，因為未優化的二進制項會佔據大量KiB的Flash，你的目標設備可能沒這麼多Flash -- 結果: 你未優化的二進制項無法燒錄進你的設備中！

我們可以有更小的，調試友好的二進制項嗎?是的，這裡有一個技巧。

### 優化依賴

這裡有個名為[`profile-overrides`]的Cargo feature，其可以讓你覆蓋依賴項的優化等級。你能使用這個feature去優化所有依賴的大小，而保持頂層的crate沒有被優化以致調試起來友好。

需要知道，泛型代碼有時是在它被實例化的庫中被優化的，而不是它被定義的地方．如果你在你的應用中生成了一個泛型結構體的實例，
並且發現它讓代碼體積變得更大，那可能是因為相關的依賴的優化等級的增加沒有造成影響．

[`profile-overrides`]: https://doc.rust-lang.org/cargo/reference/profiles.html#overrides

這是一個示例:

``` toml
# Cargo.toml
[package]
name = "app"
# ..

[profile.dev.package."*"] # +
opt-level = "z" # +
```

沒有覆蓋:

``` text
$ cargo size --bin app -- -A
app  :
section               size        addr
.vector_table         1024   0x8000000
.text                 9060   0x8000400
.rodata               1708   0x8002780
.data                    0  0x20000000
.bss                     4  0x20000000
```

有覆蓋:

``` text
$ cargo size --bin app -- -A
app  :
section               size        addr
.vector_table         1024   0x8000000
.text                 3490   0x8000400
.rodata               1100   0x80011c0
.data                    0  0x20000000
.bss                     4  0x20000000
```

在Flash的使用上減少了6KiB，而不會損害頂層crate的可調試性。如果你步進一個依賴項，然後你將開始再次看到那些`<value optimized out>`信息，但是通常的情況下你只想調試頂層的crate而不是依賴項。如果你 *需要* 調試一個依賴項，那麼你可以使用`profile-overrides` feature去防止一個特定的依賴項被優化。看下面的例子:

``` toml
# ..

# 不要優化`cortex-m-rt` crate
[profile.dev.package.cortex-m-rt] # +
opt-level = 0 # +

# 但是優化所有其它依賴項
[profile.dev.package."*"]
codegen-units = 1 # better optimizations
opt-level = "z"
```

現在頂層的crate和`cortex-m-rt`對調試器很友好！

## 優化速度

自2018-09-18開始 `rustc` 支持三個 "優化速度" 的等級: `opt-level = 1`, `2` 和 `3` 。當你運行 `cargo build --release` 時，你正在使用的是release配置，其默認是 `opt-level = 3` 。

`opt-level = 2` 和 `3` 都以二進制項大小為代價優化速度，但是等級`3`比等級`2`做了更多的向量化和內聯。特別是，你將看到在`opt-level`等於或者大於`2`時LLVM將展開循環。循環展開在 Flash / ROM 方面的成本相當高(e.g. from 26 bytes to 194 for a zero this array loop)但是如果條件合適(迭代次數足夠大)，也可以將執行時間減半。

現在還沒有辦法在`opt-level = 2`和`3`的情況下關閉循環展開，因此如果你不能接受它的開銷，你應該選擇優化你的程序的大小。

## 優化大小

自2018-09-18開始`rustc`支持兩個"優化大小"的等級: `opt-level = "s"` 和 `"z"` 。這些名字傳承自 clang / LLVM 且不具有描述性，但是`"z"`意味著它產生的二進制文件比`"s"`更小。

如果你想要發佈一個優化了大小的二進制項，那麼改變下面展示的`Cargo.toml`中的`profile.release.opt-level`配置。

``` toml
[profile.release]
# or "z"
opt-level = "s"
```

這兩個優化等級極大地減小了LLVM的內聯閾值，一個用來決定是否內聯或者不內聯一個函數的度量。Rust其中一個概念是零成本抽象；這些抽象趨向於去使用許多新類型和小函數去保持不變量(e.g. 像是`deref`，`as_ref`這樣借用內部值的函數)因此一個低內聯閾值會使LLVM失去優化的機會(e.g. 去掉死分支(dead branches)，內聯對閉包的調用)。

當優化大小時，你可能想要嘗試增加內聯閾值去觀察是否會對你的二進制項的大小有影響。推薦的改變內聯閾值的方法是在`.cargo/config.toml`中往其它rustflags後插入`-C inline-threshold` 。

``` toml
# .cargo/config.toml
# 這裡假設你正在使用cortex-m-quickstart模板
[target.'cfg(all(target_arch = "arm", target_os = "none"))']
rustflags = [
  # ..
  "-C", "inline-threshold=123", # +
]
```

用什麼值?[從1.29.0開始，這些是不同優化級別使用的內聯閾值][inline-threshold]:

[inline-threshold]: https://github.com/rust-lang/rust/blob/1.29.0/src/librustc_codegen_llvm/back/write.rs#L2105-L2122

- `opt-level = 3` 使用 275
- `opt-level = 2` 使用 225
- `opt-level = "s"` 使用 75
- `opt-level = "z"` 使用 25

當優化大小時，你應該嘗試`225`和`275` 。
