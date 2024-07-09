# Linux

這部分是在某些Linux發行版下的安裝指令。

## 依賴包

- Ubuntu 18.04 或者更新的版本 / Debian stretch 或者更新的版本

> **注意** `gdb-multiarch` 是你將用來調試你的ARM Cortex-M程序的GDB命令

<!-- Debian stretch -->
<!-- GDB 7.12 -->
<!-- OpenOCD 0.9.0 -->
<!-- QEMU 2.8.1 -->

<!-- Ubuntu 18.04 -->
<!-- GDB 8.1 -->
<!-- OpenOCD 0.10.0 -->
<!-- QEMU 2.11.1 -->


``` console
sudo apt install gdb-multiarch openocd qemu-system-arm
```

- Ubuntu 14.04 and 16.04

> **注意** `arm-none-eabi-gdb` 是你將用來調試你的ARM Cortex-M程序的GDB命令

<!-- Ubuntu 14.04 -->
<!-- GDB 7.6 (!) -->
<!-- OpenOCD 0.7.0 (?) -->
<!-- QEMU 2.0.0 (?) -->

``` console
sudo apt install gdb-arm-none-eabi openocd qemu-system-arm
```

- Fedora 27 或者更新的版本

<!-- Fedora 27 -->
<!-- GDB 7.6 (!) -->
<!-- OpenOCD 0.10.0 -->
<!-- QEMU 2.10.2 -->

``` console
sudo dnf install gdb openocd qemu-system-arm
```

- Arch Linux

> **注意** `arm-none-eabi-gdb` 是你將用來調試你的ARM Cortex-M程序的GDB命令

``` console
sudo pacman -S arm-none-eabi-gdb qemu-system-arm openocd
```

## udev 規則

這個規則可以讓你在不使用超級用戶權限的情況下，使用OpenOCD和Discovery開發板。

生成包含下列內容的 `/etc/udev/rules.d/70-st-link.rules` 文件

``` text
# STM32F3DISCOVERY rev A/B - ST-LINK/V2
ATTRS{idVendor}=="0483", ATTRS{idProduct}=="3748", TAG+="uaccess"

# STM32F3DISCOVERY rev C+ - ST-LINK/V2-1
ATTRS{idVendor}=="0483", ATTRS{idProduct}=="374b", TAG+="uaccess"
```

然後重新加載所有的udev規則

``` console
sudo udevadm control --reload-rules
```

如果你已經把開發板插入到筆記本中了，請拔下它然後再插上它。

你可以通過運行這個命令檢查權限:

``` console
lsusb
```

終端可能有如下顯示

```text
(..)
Bus 001 Device 018: ID 0483:374b STMicroelectronics ST-LINK/V2.1
(..)
```

記住bus和device號，使用這些數字組合成一個像是 `/dev/bus/usb/<bus>/<device>` 這樣的路徑。然後像這樣使用這個路徑:

``` console
ls -l /dev/bus/usb/001/018
```

```text
crw-------+ 1 root root 189, 17 Sep 13 12:34 /dev/bus/usb/001/018
```

```console
getfacl /dev/bus/usb/001/018 | grep user
```

```text
user::rw-
user:you:rw-
```

權限後的 `+` 指出存在一個擴展權限。`getfacl` 命令顯示，`user`也就是`你`，可以使用這個設備。

現在，去往[下個章節].

[下個章節]: verify.md
