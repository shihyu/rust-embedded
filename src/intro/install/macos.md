# macOS

所有的工具都可以使用[Homebrew]或者[MacPorts]來安裝：

[Homebrew]: http://brew.sh/
[MacPorts]: https://www.macports.org/

## 使用[Homebrew]安裝工具

``` text
$ # GDB
$ brew install armmbed/formulae/arm-none-eabi-gcc

$ # OpenOCD
$ brew install openocd

$ # QEMU
$ brew install qemu
```

> **注意** 如果OpenOCD崩潰了，你可能需要用以下方法安裝最新版本: 

```text
$ brew install --HEAD openocd
```

## 使用[MacPorts]安裝工具

``` text
$ # GDB
$ sudo port install arm-none-eabi-gcc

$ # OpenOCD
$ sudo port install openocd

$ # QEMU
$ sudo port install qemu
```


這是全部內容，請轉入[下個章節]．

[下個章節]: verify.md
