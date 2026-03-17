# 最小可运行内核：在 QEMU 里跑起来

> 本章目标：在学完「交叉编译、QEMU 及工具链」(09) 之后，**亲手把一个小到不能再小的内核在 QEMU 里跑起来**，看到“我们的代码真的在执行”。不追求功能完整，只求：**从加电到执行到 Rust 入口** 这条链路打通，为下一讲「用 GDB 调试」(11) 和「启动流程、内存布局」(12) 打基础。

---

## 1. 为什么需要这一讲？

第 09 讲已经把「目标文件、ELF、链接脚本、QEMU 参数」都讲过了，但**知道工具怎么用**和**真的跑起来一个内核**之间还有一步：把入口汇编、链接脚本、Rust 入口串成一条能跑的链路。很多同学第一次跑的时候会卡在「黑屏没反应」「根本停不住」「不知道到底执行到哪了」——这一讲就是专门把这一步拆开，让你按顺序做一遍，并知道出问题时该查哪里。

跑起来之后，下一讲就可以用 GDB 在这条链路上单步、下断点；再下一讲再展开讲「启动流程、内存布局」时，你已经有直观印象了。

---

## 2. 最小内核需要哪几块？

一个“最小可运行”的内核，至少需要这几部分配合：

| 部分 | 作用 |
|------|------|
| **入口汇编** | CPU 上电后第一条执行的代码：设栈指针、清 BSS（可选）、跳转到 Rust 入口。通常放在 `.text.entry`，由链接脚本放在最前。 |
| **链接脚本** | 规定内核加载地址（如 0x80200000）、各段布局、入口符号 `_start`、以及 `_stack_top` / `_bss_start` / `_bss_end` 等，供汇编和 Rust 使用。 |
| **Rust 入口** | `#![no_std]`、`#![no_main]`，提供一个 `rust_main()`，由汇编在准备好栈和 BSS 后跳转过来。这里做“最少的事”即可。 |
| **一点“可见”输出** | 让组员确信内核在跑：例如通过 QEMU 的串口输出、或 RISC-V 上让 QEMU 以特定退出码退出（semihosting）等，视项目选一种。 |

下面按「先能跑通」的顺序说：链接脚本与加载地址 → 入口汇编 → Rust 侧 → 构建与运行 → 常见问题。

---

## 3. 链接脚本：加载地址与入口

QEMU 的 `-kernel kernel.bin` 会把 **kernel.bin** 按约定加载到固定地址（RISC-V virt 机器常见是 **0x80200000**）。链接脚本必须让内核的**入口段**和**所有要执行的代码、数据**的 VMA（虚拟地址）落在这个加载范围内，并且 **ENTRY** 指向入口符号（如 `_start`）。

要点：

- **MEMORY**：定义一块从加载地址开始的内存区间，把可执行段、数据段、BSS、栈都放在里面。
- **SECTIONS**：把 `.text.entry` 放在最前（这样入口地址就是加载地址），再放 `.text`、`.rodata`、`.data`、`.bss`，最后放栈（并导出 `_stack_top`，供入口汇编给 `sp` 赋值）。
- **ENTRY(_start)**：告诉链接器可执行文件的入口是 `_start`；`readelf -h kernel.elf` 里的 Entry point 就会是这个地址。

具体语法和你们项目里的 `link.ld` 一致即可；09 讲里已经讲过如何用 `rust-readelf -h`、`-S`、`-s` 验证入口和段布局，这里不再重复。**关键**：加载地址（如 0x80200000）要和 QEMU 的约定一致，否则 CPU 会从错误地址取指，表现为“跑飞”或黑屏。

---

## 4. 入口汇编：设栈、清 BSS、跳 rust_main

入口汇编通常只做三件事（顺序可微调）：

1. **设栈**：`la sp, _stack_top`（或等价写法），让后续 C/Rust 代码有栈可用。
2. **清 BSS**（可选但推荐）：用 `_bss_start`、`_bss_end` 把 BSS 段清零，避免未初始化全局变量带随机值。
3. **跳 Rust**：`call rust_main` 或 `jal rust_main`，之后就不再返回。

段名用 `.section .text.entry`，并在链接脚本里用 `*(.text.entry)` 放在最前面，保证 `_start` 的地址就是 ELF 的 Entry point。09 讲里已经讲过如何用 `global_asm!` 或单独 `entry.S` 配合链接脚本，这里只需保证：**sp 指向链接脚本里定义的栈顶、BSS 区间与链接脚本里一致、跳转目标是 Rust 里用 `#[no_mangle]` 暴露的 `rust_main`**。

---

## 5. Rust 侧：no_std、no_main、rust_main

内核 crate 需要：

- `#![no_std]`：不用标准库（没有操作系统依赖）。
- `#![no_main]`：入口不是 `main`，由我们自己的 `_start` → `rust_main` 控制。
- 提供 **`rust_main()`**：用 `#[no_mangle]` 且不要 name mangling，供汇编 `call rust_main` 调用。

在“最小可运行”阶段，`rust_main` 里做一件事即可：**让运行结果可见**。常见做法：

- **串口输出**：若你们已经封装了往 QEMU 串口写字符的接口，在 `rust_main` 里打一行字符串，QEMU 用 `-nographic` 时会在终端看到。
- **QEMU 退出**：RISC-V 上可用 semihosting 或特定 CSR 写让 QEMU 退出并带退出码，在 Makefile 里根据退出码判断“内核执行到了我们预期的路径”。

先不追求“完整启动流程”，只要能在 QEMU 里看到**我们写的代码产生的效果**，就达到本节目标。

---

## 6. 构建与在 QEMU 里运行

- **构建**：`cargo build --target riscv64imac-unknown-none-elf`（或你们项目的 target），得到 `kernel.elf`；再用 `rust-objcopy -O binary kernel.elf kernel.bin` 得到 QEMU 用的 `kernel.bin`。项目里通常用 Makefile 或 `build.rs` 把这两步自动化。
- **运行**：在项目根目录执行类似：
  ```bash
  qemu-system-riscv64 -machine virt -nographic -kernel kernel.bin
  ```
  若一切正常，应看到串口输出或 QEMU 按预期退出。

建议先**不要**加 `-s -S`，确认能跑起来；再加 `-s -S` 配合下一讲的 GDB 做“启动即暂停、用 GDB 连上来再跑”。

---

## 7. 常见第一次跑不起来的问题

| 现象 | 可能原因 | 排查方向 |
|------|----------|----------|
| 黑屏、无输出、QEMU 不退出 | 入口地址错、或第一条指令就跑飞 | `readelf -h kernel.elf` 看 Entry point 是否与链接脚本、QEMU 加载地址一致；`objdump -d -j .text.entry kernel.elf` 看入口几条指令是否合理。 |
| 有输出但马上乱码/死机 | 栈没设好或 BSS 没清，导致后续代码行为异常 | 在入口汇编里确认 `sp` 指向 `_stack_top`；确认 BSS 起止与链接脚本一致并在入口里清零。 |
| 链接错误：undefined reference to `rust_main` | Rust 里没暴露或名字不对 | 确认 `rust_main` 带 `#[no_mangle]`，且与汇编里 `call` 的符号一致（不要被 name mangling 掉）。 |
| 链接错误：undefined reference to `_stack_top` | 链接脚本里没定义或没在 SECTIONS 里放栈 | 在 link.ld 里用 `. = ALIGN(16); _stack_top = .;` 之类在段末定义，并保证该段被链接进内核。 |

用 09 讲里的 **readelf、objdump、nm** 逐项对照：入口地址、段布局、符号地址，一般能很快定位到是“链接脚本”“汇编”还是“Rust 暴露符号”的问题。

---

## 8. 和后面两讲的关系

- **第 11 讲（GDB 调试）**：会在“能跑起来”的基础上，用 `-s -S` 启动 QEMU，用 GDB 连上去，在 `_start`、`rust_main` 下断点、单步，观察寄存器和内存。本节把最小内核跑通，调试时才有“可见的停靠点”。
- **第 12 讲（启动流程、内存布局）**：会系统讲「从加电到 rust_main」的完整流程和内存布局；本节相当于一个**最小子集**：只把这条链路打通，不展开中断、多核、设备树等。

把本节做完，组员就完成了“从零到在 QEMU 里跑起来自己写的内核”的第一步；再配合 11、12，整个「工具链 → 跑起来 → 调试 → 理解启动与布局」的培训路径就连贯了。
