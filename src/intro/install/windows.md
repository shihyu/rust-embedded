# Windows

## `arm-none-eabi-gdb`

ARM提供了用於Windows的`.exe`安裝程序。從[這裡][gcc]獲取, 然後按照說明操作。
在完成安裝之前，勾選/選擇"Add path to environment variable"選項。
然後驗證環境變量是否添加到 `%PATH%`中:

``` text
$ arm-none-eabi-gdb -v
GNU gdb (GNU Tools for Arm Embedded Processors 7-2018-q2-update) 8.1.0.20180315-git
(..)
```

[gcc]: https://developer.arm.com/open-source/gnu-toolchain/gnu-rm/downloads

## OpenOCD

OpenOCD 官方沒有提供Windows的二進制版本， 若你沒有心情去折騰編譯，[這裡][openocd]有xPack提供的一個二進制發佈.。按照說明進行安裝。然後更新你的`%PATH%` 環境變量，將安裝目錄包括進去。 (`C:\Users\USERNAME\AppData\Roaming\xPacks\@xpack-dev-tools\openocd\0.10.0-13.1\.content\bin\`,
如果使用簡易安裝) 

[openocd]: https://xpack.github.io/openocd/

使用以下命令驗證OpenOCD是否在你的`%PATH%`環境變量中 :

``` text
$ openocd -v
Open On-Chip Debugger 0.10.0
(..)
```

## QEMU

從[官網][qemu]獲取QEMU。

[qemu]: https://www.qemu.org/download/#windows

## ST-LINK USB driver

你還需要安裝這個 [USB驅動] 否則OpenOCD將無法工作。按照安裝程序的說明，確保你安裝了正確版本（32位或64位）的驅動程序。

[USB驅動]: http://www.st.com/en/embedded-software/stsw-link009.html

以上是全部內容！轉到 [下個章節]。

[下個章節]: verify.md
