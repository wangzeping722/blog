---
title: "使用 GDB 对 rCore 进行 debug"
date: 2021-04-18T20:35:05+08:00
draft: true
---

- 将 QEMU的运行参数 `qemu-system-riscv64 -machine virt -nographic -bios ../bootloader/rustsbi-qemu.bin -device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000` 加上 `-s -S` ，它将在 1234 端口等待调试器接入。
- 运行 `riscv64-unknown-elf-gdb`
- 在 GDB 中执行 `file target/riscv64imac-unknown-none-elf/debug/os` 来加载未被 `strip` 过的内核文件中的各种符号
- 在 GDB 中执行 `target remote localhost:1234` 来连接 QEMU，开始调试
- `set disassemble-next-line on` si 时可以自动反汇编

## GDB 简单使用方法

### 控制流

- `b <函数名>` 在函数进入时设置断点，例如 `b rust_main` 或 `b os::memory::heap::init`
- `c` 继续执行
- `n` 执行下一行代码，不进入函数
- `ni` 执行下一条指令（跳转指令则执行至返回）
- `s` 执行下一行代码，进入函数
- `si` 执行下一条指令，包括跳转指令

### 查看状态

- 如果没有安装 `gdb-dashboard`，可以通过 `layout` 指令来呈现寄存器等信息，具体查看 `help layout`
- 使用 `x/<格式> <地址>` 来查看内存，例如 `x/8i 0x80200000` 表示查看 `0x80200000` 起始的 8 条指令。具体格式查看 `help x`

