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




# qemu 参数
# 没有图像
QEMU_ARGS := -machine virt \
			 -nographic \
			 -bios $(BOOTLOADER) \
			 -device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA)

# 开启 qemu
run-inner:
	@kitty -e qemu-system-riscv64 --nographic -bios $(BOOTLOADER) &
