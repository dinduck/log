这里将会省略众多细节， 直接开干


**图片圆形应为 `RustSBI`**
<div style="text-align: center;"> <img src="https://s2.loli.net/2023/10/19/DxblnzFgeys589o.png" alt="" style="display: block; margin: 0 auto;"> </div>

- 上电初始化我们不必关系4, 这一般厂商约定好的
- 第二阶段加载 RustSBI, 不是BIOS神似BIOS, OpenSBI提供了 `syscall` 给我们调用， 所以我们并不需要自己实现， RustSBI 做完工作后就会跳转
- 第三阶段跳转到 `0x80200000` 地址是 `32` 位的， 再次将运行我们的代码

我们不必要学习汇编， 学习汇编只是为了加深计算机原理的了解， 这是我的汇编笔记, [[汇编程序设计]]

**工欲善其事，必先利其器**

演示环境
<div style="text-align: center;"> <img src="https://s2.loli.net/2023/10/19/3IEeUcKGBCpZAtd.png" alt="" style="display: block; margin: 0 auto;"> </div>

## 环境配置

### Rust

安装
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
# source .cargo/env
```

安装工具链和一些配置包

```
rustup target add riscv64gc-unknown-none-elf
cargo install cargo-binutils
rustup component add llvm-tools-preview
rustup component add rust-src
```

夜间 Channel

```Bash
rustup install nightly
rustup default nightly
rustup update
```

### GDB

由于调试

```Bash
sudo pacman -S riscv64-elf-gdb\
# 全套工具链一把锉
# sudo pacman -S riscv64-elf-binutils
```

### QEMU

由于仿真模拟

```
sudo pacman -S qemu-system-riscv
```

`qemu-system-riscv64` 用于模拟系统环境
`qemu-riscv64` 可以直接运行 `App`


本人版本
<div style="text-align: center;"> <img src="https://s2.loli.net/2023/10/19/ivqVjUSh6QPn9uA.png" alt="" style="display: block; margin: 0 auto;"> </div>



## `no_std`

由于裸机开发我们不需要 `no_std`, 我们只有 `core`,  这是 Rust 的核心库， 没有上游， 没有系统库， 没有 `libc`

创建一个项目
```
cargo new os
```

编辑 `.cargo/config`
```
[build]
target = "riscv64gc-unknown-none-elf"
```


`src/main.rs`
```Rust
#![no_std]
#![no_main]


fn main() {
    
}
```
由于我们不需要标准库， 那么标准的入口也是不可用的。
[[C语言是怎么进入 `main` 的]]

运行 `cargo build`
```Bash
error: `#[panic_handler]` function required, but not found

error: could not compile `os` (bin "os") due to previous error
```

`panic_handler` 作为 `panic` 的处理函数， 去除 `std` 也一并去除了， 这里我们先添加一个死循环的 `panic_handler` 后面我们学到 `syscall` 就能相应做 `panic` 处理了

`src/lang_items.rs`
```Rust
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```
这里先忽略， 我们只需要让我们的代码编译通过


相应的在 `main` 引入 `lang_items`
`src/main.rs`
```Rust
#![no_std]
#![no_main]

mod lang_items;

fn main() {}
```

现在运行 `cargo build`
```Bash
   Compiling os v0.1.0 (/home/dingduck/Cell/GitCell/log/task/os)
warning: function `main` is never used
 --> src/main.rs:6:4
  |
6 | fn main() {}
  |    ^^^^
  |
  = note: `#[warn(dead_code)]` on by default

warning: `os` (bin "os") generated 1 warning
    Finished dev [unoptimized + debuginfo] target(s) in 0.03s
```
爆了一个没有死代码的警告， 原因是我们没有使用 `main()`, 而且目前 `main` 并不是我们的入口

相比 `rcore`   我这里直接忽略 `start` 的问题， 你可以把 `#![no_main]` 注释后编译看看

## 准备 `RustSBI`

前往 [Github](https://github.com/rustsbi/rustsbi-qemu)
有两版本 `debug` 和 `release` 
这里我选择为 `release`
在 `os` 同级目录创建 `bootloader` 文件夹
解压 `rustsbi-qemu.bin` 至 `bootloader`
<div style="text-align: center;"> <img src="https://s2.loli.net/2023/10/19/sJPRxWIzVnuaOEK.png" alt="" style="display: block; margin: 0 auto;"> </div>


## 开始第一条指令
我们已经准备好了环境和 `"bios"`

根据开头所示， 代码最终会跳转 `0x80200000`
我们要完成两个任务

- [ ] 怎么设置入口点 `entry`
- [ ] 怎么让代码布局在 `0x8020000` 内存上

### 汇编与链接脚本

某些操作系统实践可能没有汇编代码， 这里使用汇编代码的原因是因为， 后面我们要实现自己的函数栈吗虽然 `C` 和 `Rust`  编译器会在函数调用的时候做栈帧操作， 设置编译器为我们实现的。使用汇编的原因也在某些系统操作需要汇编去处理

接下来， 编写我们的第一个汇编代码

`stc/entry.asm`
```Assembly
    .section  .text.entry
    .globl  _start
_start:
  li x1, 100
```
这段代码非常简单的， 我们定义段 `.text.entry` 具有一个全局符号 `_start`
`_start` 符号表示代码段 `_start`的开始
`li x1, 100` 将立即数 `100` 加载至 `x1` 寄存器

接下来将我们的汇编代码嵌入程序
`src/main.rs`
```Rust
#![no_std]
#![no_main]
mod lang_items;

use core::arch::global_asm;

global_asm!(include_str!("entry.asm"));
```

接下来就会编写我们的链接脚本， 链接脚本是一个内存描述文件， 用于描述段在内存中的布局

`src/linker.ld`
```
OUTPUT_ARCH(riscv)
ENTRY(_start)
BASE_ADDRESS = 0x80200000;

SECTIONS
{
    . = BASE_ADDRESS;
    skernel = .;

    stext = .;
    .text : {
        *(.text.entry)
        *(.text .text.*)
    }

    . = ALIGN(4K);
    etext = .;
    srodata = .;
    .rodata : {
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    }

    . = ALIGN(4K);
    erodata = .;
    sdata = .;
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }

    . = ALIGN(4K);
    edata = .;
    .bss : {
        *(.bss.stack)
        sbss = .;
        *(.bss .bss.*)
        *(.sbss .sbss.*)
    }

    . = ALIGN(4K);
    ebss = .;
    ekernel = .;

    /DISCARD/ : {
        *(.eh_frame)
    }
}
```
这个链接脚本干了什么， 告诉编译器输出架构为 `riscv`,  我们程序的入口点为 `_start`, 基地址 `0x80200000`

`. = BASE_ADDRESS`， 表示当前地址为 `0x80200000`, 内容将由此开始输出

```
output = {input rule}
```
右边表示输入规则， 左边表示输出
```
.text : {
        *(.text.entry)
        *(.text .text.*)
    }
```
这里会输入所有 `.text.entry` ， `.text`  和 `.text.*` 并将输出为 `.text`， 因为每个源文件都有自己的段， 链接过程就是将各个段排列在一起， `Entry` 能保证 `_start` 标签保证 `.text` 最起始的位置, 其他在什么位置我们不用管。程序知道其他标签在哪里。

`ALIGN(4)` 以 4 字节对其边界

关于段的知识， 你需要了解 [[汇编程序设计]] 和 [ELF文件解析](https://gdufs-king.github.io/2020/10/21/ELF%E6%96%87%E4%BB%B6%E7%BB%93%E6%9E%84%E8%A7%A3%E6%9E%90/) 也可选择自行查阅， 这里有一篇比较好的 [PDF](https://paper.seebug.org/papers/Archive/refs/elf/Understanding_ELF.pdf) 关于 ELF
这里引用 `rcore tutorial` 的图
![image.png](https://s2.loli.net/2023/10/19/E2dUfB4YHKvmiGb.png)

将链接脚本传给编译器
`.cargo/config`
```
[build]
target = "riscv64gc-unknown-none-elf"

[target.riscv64gc-unknown-none-elf]
rustflags = [
  "-Clink-arg=-Tsrc/linker.ld", "-Cforce-frame-pointers=yes"
]
```

运行 `cargo build --release` 以发行版构建

运行 `readelf -h target/riscv64gc-unknown-none-elf/release/os` 查看 ELF 信息
![image.png](https://s2.loli.net/2023/10/20/Otvg1iKI9MTxPsX.png)

其中 `ELF` 文件中存在大量元数据， 如果用过 `gcc` 做单片机开发就知道用 `objcopy` 了

运行 `rust-objcopy --strip-all target/riscv64gc-unknown-none-elf/release/os -O binary target/riscv64gc-unknown-none-elf/release/os.bin`

查看两大小
![image.png](https://s2.loli.net/2023/10/20/evM89joRPl3ZbFW.png)

我们只有一条指令,  我们的二进制文件也仅有 4 字节， 也就是 32 位， 非常符合

## 启动 QEMU 调试

启动 ！！！

```
qemu-system-riscv64 -machine virt -nographic -bios ../bootloader/rustsbi-qemu.bin -device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000 -s -S
```
我们的机器名字为 `virt`, 没有图形界面， 选择 `bios`, 加载我们的设备到 `0x80200000`
`-s` 启动 GDB 调试, 端口是 `1234`,  `-S` 冻结虚拟机运行， 我们想让 `GDB` 控制下一步运行

`GDB` 启动
```
riscv64-elf-gdb -ex 'file target/riscv64gc-unknown-none-elf/release/os' -ex 'set arch riscv:rv64' -ex 'target remote localhost:1234'
```
也可选择， 先启动 `GDB`
![image.png](https://s2.loli.net/2023/10/20/X1lQWIu8zaxHs2R.png)

接下来依次输入 `layout next` `layout reg` 可以启动 `TUI`, 方便观察, 输入 `b *0x80200000`, 设置断点,  输入 `c` 继续执行
![image.png](https://s2.loli.net/2023/10/20/skZOfUACoIRK16r.png)
我们观察到代码确实跳转到我们的第一指令 `li x1, 100`

现在， 我们已经跑上了我们的第一代码， 接下来就是编写我们的系统调用功能， 以便我们输出 `Hello, World!`

收工睡觉

## RVVM
这是一个 rv 模拟器

```
rvvm ../bootloader/rustsbi-qemu.bin -m 256M -k target/riscv64gc-unknown-none-elf/release/os.bin
```
你可以添加 `-nogui` 选项， 唯一缺点， 不可调试

![](https://s2.loli.net/2023/10/20/LpXytvA54WOgaEQ.png)


## Simple `Makefile`

现在我们有了常用指令


现在完善 `Makefile`
```Makefile
# 编译模式
MODE := release

# Bootloader
BOOTLOADER := ../bootloader/rustsbi-qemu.bin

# 目标平台
TARGET := riscv64gc-unknown-none-elf

# 内核的可执行文件 但是存在大量元数据， 方便调试
KERNEL_ELF := ./target/riscv64gc-unknown-none-elf/$(MODE)/os

# 内核文件， 去除了元数据
KERNEL_BIN := $(KERNEL_ELF).bin

# 设置编译参数
ifeq ($(MODE), release)
	MODE_ARG := --release
endif

# 内核入口
KERNEL_ENTRY_PA := 0x80200000

# 用于反汇编
OBJDUMP := rust-objdump --arch-name=riscv64
# 用于获取二进制文件
OBJCOPY := rust-objcopy --binary-architecture=riscv64

# 构建
build: $(KERNEL_BIN)


$(KERNEL_BIN): kernel
	@$(OBJCOPY) $(KERNEL_ELF) --strip-all -O binary $@


kernel:
	@cargo build $(MODE_ARG)


clean:
	@cargo clean


# qemu 参数
# 没有图像
QEMU_ARGS := -machine virt \
			-nographic \
			-bios $(BOOTLOADER) \
			-device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA)

# 开启 qemu
run-inner: build
	@kitty -e qemu-system-riscv64 $(QEMU_ARGS) &

# RVVM 运行
rvvm: build
	@rvvm ../bootloader/rustsbi-qemu.bin -m 256M -k target/riscv64gc-unknown-none-elf/release/os.bin

# 调试
debug: build
	@kitty -e qemu-system-riscv64 $(QEMU_ARGS) -s -S &
	@riscv64-elf-gdb -ex 'file $(KERNEL_ELF)' -ex 'set arch riscv:rv64' -ex 'target remote :1234' -ex 'layout next' -ex 'layout reg'

```


使用 `make debug` 可快速开启调试， 注意默认终端为 `kitty`, 
`make rvvm` 可使用 `rvvm` 运行
