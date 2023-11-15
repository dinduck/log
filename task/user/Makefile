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

