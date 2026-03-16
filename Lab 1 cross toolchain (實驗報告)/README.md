# Raspberry Pi ARM Cross Compiler Toolchain Build

# 實驗名稱

建置 Raspberry Pi 嵌入式系統交叉編譯工具鏈  
(Cross Compiler Toolchain Build)

---

# 實驗目的

本實驗的目的是在 **x86_64 Linux 環境** 中建置一套 **ARM 架構 (Raspberry Pi)** 的交叉編譯工具鏈，使得開發者可以在 PC 上編譯程式並在 Raspberry Pi 上執行。

透過本實驗可以學習：

1. ARM 交叉編譯工具鏈建置流程  
2. 工具鏈各元件的角色  
3. Bootstrapping (自我提升) 的編譯流程  
4. ARM 程式的編譯與測試  

---

# Toolchain 組成元件

ARM 交叉編譯工具鏈主要包含以下元件：

| 元件 | 功能 |
|-----|-----|
| Binutils | 提供 assembler、linker 等二進位工具 |
| GCC | GNU Compiler Collection，負責程式編譯 |
| Linux Kernel Headers | Linux 系統呼叫標頭檔 |
| GLIBC | GNU C Standard Library |

---

# 實驗環境

- Host System：x86_64 Linux
- Target Architecture：ARM (Raspberry Pi)
- Toolchain Prefix：arm-linux-gnueabihf

---

# 工作目錄

建立工作資料夾：

```bash
mkdir -p ~/WORK
cd ~/WORK
```

目錄結構：

```
WORK
├── binutils-2.40
├── gcc-12.3.0
├── glibc-2.38
├── linux
├── crossgcc1
├── crossgcc2
└── sysroot
```

---

# 下載原始碼

```bash
cd ~/WORK

wget https://ftp.gnu.org/gnu/binutils/binutils-2.40.tar.gz
wget https://ftp.gnu.org/gnu/gcc/gcc-12.3.0/gcc-12.3.0.tar.gz
wget https://ftp.gnu.org/gnu/libc/glibc-2.38.tar.gz
wget https://github.com/raspberrypi/linux/archive/refs/heads/rpi-6.1.y.tar.gz
```

解壓縮：

```bash
tar xf binutils-2.40.tar.gz
tar xf gcc-12.3.0.tar.gz
tar xf glibc-2.38.tar.gz
tar xf rpi-6.1.y.tar.gz
```

---

# 第一階段：建置 Cross Binutils

建立 build 目錄：

```bash
mkdir build-binutils
cd build-binutils
```

configure：

```bash
../binutils-2.40/configure \
--target=arm-linux-gnueabihf \
--prefix=$HOME/WORK/crossgcc1 \
--with-sysroot=$HOME/WORK/sysroot \
--disable-nls \
--disable-werror
```

編譯與安裝：

```bash
make -j$(nproc)
make install
```

完成後會產生 ARM 的 assembler 與 linker：

```
arm-linux-gnueabihf-as
arm-linux-gnueabihf-ld
```

---

# 第二階段：建置基礎 GCC (Bare-metal GCC)

安裝依賴套件：

```bash
sudo apt install libgmp-dev libmpfr-dev libmpc-dev build-essential
```

建立 build 目錄：

```bash
mkdir build-gcc
cd build-gcc
```

configure：

```bash
../gcc-12.3.0/configure \
--target=arm-linux-gnueabihf \
--prefix=$HOME/WORK/crossgcc1 \
--without-headers \
--enable-languages=c \
--disable-nls
```

編譯：

```bash
make all-gcc -j$(nproc)
make install-gcc
```

此階段建立最基本的 cross compiler。

---

# 第三階段：安裝 Kernel Headers

進入 Linux Kernel 原始碼：

```bash
cd linux-rpi-6.1.y
```

安裝 headers：

```bash
make ARCH=arm INSTALL_HDR_PATH=$HOME/WORK/sysroot/usr headers_install
```

安裝完成後 headers 位於：

```
sysroot/usr/include
```

---

# 第四階段：建置 GLIBC

建立 build 目錄：

```bash
mkdir build-glibc
cd build-glibc
```

configure：

```bash
../glibc-2.38/configure \
--prefix=/usr \
--host=arm-linux-gnueabihf \
--build=$(gcc -dumpmachine) \
--with-headers=$HOME/WORK/sysroot/usr/include
```

編譯：

```bash
make -j$(nproc)
make install_root=$HOME/WORK/sysroot install
```

GLIBC 會安裝到：

```
sysroot/usr/lib
```

---

# 第五階段：建置最終 GCC

回到 GCC build 目錄：

```bash
cd ~/WORK/build-gcc
```

重新 configure：

```bash
../gcc-12.3.0/configure \
--target=arm-linux-gnueabihf \
--prefix=$HOME/WORK/crossgcc2 \
--with-sysroot=$HOME/WORK/sysroot \
--enable-languages=c,c++ \
--disable-nls
```

編譯：

```bash
make -j$(nproc)
make install
```

完成後得到完整交叉編譯器：

```
arm-linux-gnueabihf-gcc
```

---

# 測試交叉編譯器

建立測試程式：

```c
#include <stdio.h>

int main() {
    printf("Hello ARM!\n");
    return 0;
}
```

存成：

```
test.c
```

編譯：

```bash
arm-linux-gnueabihf-gcc test.c -o test
```

產生組合語言：

```bash
arm-linux-gnueabihf-gcc -S test.c
```

若成功會產生：

```
test.s
```

其中會包含 ARM 指令。

---

# 問題與討論

## 1. `--with-sysroot` 的用途

`--with-sysroot` 用於指定交叉編譯時搜尋標頭檔與函式庫的根目錄。

例如：

```
--with-sysroot=/home/wayne/WORK/sysroot
```

原因是：

交叉編譯時不能使用主機系統的：

```
/usr/include
/usr/lib
```

這些都是 x86_64 架構。

因此需要使用：

```
sysroot/usr/include
sysroot/usr/lib
```

來確保連結 ARM 函式庫。

---

## 2. `--with-arch` 的用途

`--with-arch` 用於指定目標 CPU 架構，例如：

```
--with-arch=armv6
```

不同 ARM CPU 支援不同指令集。

設定此參數可以：

- 產生適合 CPU 的機器碼
- 提升效能
- 確保相容性

---

## 3. Raspberry Pi 3B+ 是否可以設定為 armv8

答案：可以。

Raspberry Pi 3B+ 使用：

```
CPU: Cortex-A53
Architecture: ARMv8-A
```

因此可以設定：

```
--with-arch=armv8
```

優點：

- 支援 64-bit 指令
- 效能更好

但缺點是：

舊 Raspberry Pi (Pi 1 / Zero) 使用 ARMv6，因此無法執行。

---

# 結論

本實驗成功建立了一套完整的 **ARM Cross Compiler Toolchain**。

最終得到的編譯器：

```
arm-linux-gnueabihf-gcc
```

此編譯器可以在 **x86 Linux 系統上編譯 ARM 程式**，並在 **Raspberry Pi 上執行**。

透過本實驗可以理解：

- Cross Compilation
- Toolchain Architecture
- Bootstrapping Build Process
- ARM Embedded Development
