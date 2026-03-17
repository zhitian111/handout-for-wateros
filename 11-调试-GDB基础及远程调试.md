# 调试：GDB 与内核调试（QEMU + 开发板）

> 本章目标：掌握 **GDB 基本用法**和**远程调试原理**，重点学会 **在 QEMU 里用 GDB 调试自己写的内核** 的完整流程；并了解 **内核放到开发板上运行时** 如何通过 JTAG/OpenOCD 或 gdbserver 进行调试。

---

## 1. 为什么内核开发离不开 GDB？

在写普通应用程序时，我们可以：

- 大量使用 `println!` / `println` 打日志；
- 在 IDE 里点点鼠标，用图形界面的调试器。

但在内核开发场景里：

- 还没初始化好串口前，**你连日志都打不出来**；
- 一旦出了错，往往就是黑屏 / 死机；
- 很多 bug 只在特定时序或中断场景下出现，日志不一定能覆盖。

这时候，一个可以：

- **随时暂停程序执行**；
- **查看/修改寄存器、内存、栈回溯**；
- **单步执行下一条指令 / 下一句源码**；

的工具就非常关键——这就是 **GDB（GNU Debugger）** 的角色。

在内核场景中，我们一般不会用它来做“全程带着断点的 step-by-step 调试”，而是：

- 在关键路径上设置几个断点，观察状态是否如预期；
- 遇到难以解释的崩溃时，用它来“定点勘察现场”。

---

## 2. GDB 基本命令速览（建议先在用户态练手）

在调内核之前，建议在一个普通用户态程序上把 GDB 用熟。例如：

```c
// demo.c
#include <stdio.h>
int add(int a, int b) { return a + b; }
int main(void) {
    int x = 1, y = 2, z = add(x, y);
    printf("z = %d\n", z);
    return 0;
}
```

用 `-g` 编译后：`gdb ./demo`，常用命令：

| 命令 | 含义 |
|------|------|
| `break main` | 在 `main` 入口设断点 |
| `run` | 运行到断点或结束 |
| `next` | 下一行源码（不进入函数） |
| `step` | 下一行源码（会进入函数） |
| `continue` | 继续跑到下一个断点 |
| `print x` | 打印变量 `x` |
| `info locals` | 当前栈帧局部变量 |
| `backtrace` | 调用栈 |
| `quit` | 退出 |

用户态下把这些命令用顺手后，调内核时才能把注意力放在“分析状态”而不是“怎么敲命令”上。

---

## 3. 远程调试原理：GDB 与“被调试目标”分离

在用户态，GDB 通常自己启动并附着到进程；**内核**则跑在另一侧（QEMU 模拟机或真实开发板），所以需要 **远程调试**：

- 被调试端（QEMU 或 OpenOCD/gdbserver）在某一端口提供 **GDB Remote Protocol** 服务；
- 宿主机上的 GDB 用 **`target remote :端口`** 连上去；
- 之后“读寄存器、单步、下断点”等请求通过协议发给对方，由对方在真实/模拟 CPU 上执行并返回结果。

GDB 只需懂协议，不必懂具体硬件；硬件细节由 QEMU 或 OpenOCD 负责。

---

## 4. 在 QEMU 里用 GDB 调试自己写的内核（重点）

本节把「从零连上 QEMU → 下断点 → 单步跟踪到 Rust」的流程讲清楚，按顺序做即可。

### 4.1 前提：两样东西要对上

- **带调试符号的内核 ELF**（如 `kernel.elf`）：用 `cargo build --target xxx` 生成，**不要 strip**，GDB 靠它做“源码/符号 ↔ 地址”的对应。调试时加载的是 **ELF**，不是 .bin。
- **以“可调试”方式运行的 QEMU**：启动时加 **`-s -S`**：
  - **`-S`**：启动后 CPU **不自动运行**，停在第一条指令，等 GDB 发 `continue` 再跑；
  - **`-s`**：等价于 `-gdb tcp::1234`，在本机 **1234 端口** 开 GDB server。

项目里通常有 `make debug`，会构建 `kernel.bin` 并用上述参数启动 QEMU；GDB 要加载**同一次构建**产生的 `kernel.elf`，这样地址和符号才能一致。

### 4.2 完整操作流程（两步：先 QEMU，再 GDB）

**第一步：启动 QEMU（GDB server 模式）**

在项目根目录：

```bash
make debug
```

或等价地手动：

```bash
qemu-system-riscv64 -machine virt -nographic -kernel kernel.bin -s -S
```

此时 QEMU 会停在“第一条指令”之前，终端几乎无输出，**等待 GDB 连接**。

**第二步：另开一个终端，启动 GDB 并连接**

```bash
gdb path/to/kernel.elf
```

例如（按你项目路径改）：

```bash
gdb target/riscv64imac-unknown-none-elf/debug/kernel
```

进入 GDB 后：

```gdb
(gdb) target remote :1234
```

看到 “Remote debugging using :1234” 即表示已连上。若启动 GDB 时没带 ELF，可连上后再加载符号：

```gdb
(gdb) symbol-file path/to/kernel.elf
```

之后就可以下断点、单步、查看寄存器与内存。

### 4.3 从入口到 rust_main：断点与单步

- **入口**：由链接脚本和汇编决定，常见符号名如 `_start`、`_entry` 等；也可用 `readelf -h kernel.elf` 看 **Entry point** 地址（如 `0x80200000`）。
- 在入口下断点并跑过去：

  ```gdb
  (gdb) break _start
  (gdb) continue
  ```

  若用地址：`break *0x80200000` 再 `continue`，会停在该条指令。

- **单步**：
  - **指令级**：`stepi` / `nexti`（一条汇编）；
  - **源码级**：`step` / `next`（一行 C/Rust，需符号和源码正确）。  
  入口附近建议用 `stepi`，进入 Rust 后用 `step`/`next` 配合 `print` 看变量。

- 跟踪到 Rust：入口汇编一般会“设栈、清 BSS、跳转 `rust_main`”。在 Rust 入口下断点即可：

  ```gdb
  (gdb) break rust_main
  (gdb) continue
  ```

  停住后即可按源码行单步、`print 变量`、`backtrace` 等。

### 4.4 常用观察命令（内核场景）

断点停住或单步时常用：

| 命令                 | 作用                                   |
| ------------------ | ------------------------------------ |
| `info registers`   | 所有通用寄存器；可 `info registers pc`、`sp` 等 |
| `x/10i $pc`        | 当前 PC 附近 10 条指令                      |
| `x/16x 0x80200000` | 以 16 进制显示 16 个单位内存                   |
| `x/s addr`         | 当该地址是字符串时按字符串显示                      |
| `backtrace`        | 调用栈（栈和符号正确时能看到从入口到当前的链）              |
| `list`             | 显示当前源码片段                             |

可结合 `rust-objdump -d kernel.elf` 查某地址处的指令，与 GDB 的 `x/i $pc` 对照。

### 4.5 常见问题与排查

- **断点打不中 / 停不到预期位置**  
  - 确认 GDB 用的 `kernel.elf` 与 QEMU 用的 `kernel.bin` 是**同一次构建**；  
  - 用 `readelf -h kernel.elf` 看 Entry point，用 `break *入口地址` 试一次；  
  - 若入口是汇编符号，确认未被 strip，且名字与链接脚本、汇编一致。

- **符号对不上 / 没有源码**  
  - 用 **debug 构建**（不 strip、带调试信息）；  
  - 连上后用 `symbol-file kernel.elf` 再加载一次；  
  - Rust 若 LTO/内联多，行号可能对不齐，可多用 `stepi` 和 `x/10i $pc`。

- **连接被拒绝**  
  - 确认先启动 QEMU（带 `-s -S`），再启动 GDB；  
  - Windows/WSL/远程时注意端口是否为本机 `127.0.0.1:1234` 或对应网卡。

- **单步后“跑飞”**  
  - 可能是栈未设好或 BSS 未清，用 `info registers sp` 检查，对照链接脚本里 `_stack_top` 和入口汇编对 `sp` 的初始化。

### 4.6 固化流程：.gdbinit 与固定断点

流程跑通后，可固化一套“观察路线”：

- 在项目里写 **`.gdbinit`**（或脚本）：连上后自动 `break _start`、`break rust_main`，再 `continue` 一次就停在内核入口或 Rust 入口。
- 固定几个关键断点：如 trap 入口、第一次开中断、第一次调度等，配合 `backtrace` 和 `info registers` 做最小复现。
- 在 README 或 Makefile 注释里写清：“`make debug` 后，在另一终端执行 `gdb -x .gdbinit target/.../kernel`”，方便团队上手。

---

## 5. 内核在开发板上运行时如何调试

当内核烧录到**真实开发板**（如 RISC-V 或 ARM 板子）上运行时，无法再依赖 QEMU 的 `-s -S`，需要用“宿主机 GDB + 板子上的调试服务”的方式。常见有两种：**JTAG/SWD + OpenOCD**，或 **板载 gdbserver**（若板子跑的是带网络和 gdbserver 的系统）。下面以**裸机/RTOS 内核 + JTAG + OpenOCD** 为主说明（最常见于自己写的 OS 内核）。

### 5.1 思路：OpenOCD 当“中间人”

- 开发板通过 **JTAG 或 SWD** 与主机相连（USB 转 JTAG 调试器，如 DAP-Link、J-Link、FT2232 等）。
- 主机上运行 **OpenOCD**，连接调试器，并同时在本机开一个 **GDB server**（默认 TCP 3333）。
- 主机上运行 **GDB**，加载带符号的 `kernel.elf`，用 **`target remote :3333`** 连接 OpenOCD。
- 此后与 QEMU 类似：GDB 发“暂停、单步、读寄存器、下断点”等，OpenOCD 通过 JTAG 在板子 CPU 上执行并返回结果。

也就是说：**把 QEMU 换成“OpenOCD + 开发板”**，GDB 的用法几乎一样，只是连接端口和“何时停住”略有不同。

### 5.2 接线与 OpenOCD 配置

- **接线**：用调试器把主机 USB 和板子的 JTAG/SWD 口连起来（VCC、GND、SWDIO、SWCLK 等，按板子手册接）。
- **OpenOCD 配置**：需要一份配置文件（`.cfg`），指定调试器类型和芯片/板子型号。例如 RISC-V 某开发板可能类似（具体以你板子为准）：

  ```bash
  # 示例：openocd -f interface/cmsis-dap.cfg -f target/xxx.cfg
  openocd -f interface/cmsis-dap.cfg -f target/riscv.cfg
  ```

  启动后 OpenOCD 会占住调试器，并在本机打开 GDB 端口（默认 **3333**）。

### 5.3 在宿主机上用 GDB 连接

在**另一个终端**：

```bash
gdb path/to/kernel.elf
```

然后：

```gdb
(gdb) target remote :3333
```

若连接成功，GDB 就附着到板子上的 CPU。注意：

- 板子可能已经在跑内核，**不会像 QEMU 的 `-S` 那样一开始就停住**；需要你在 GDB 里先 `break rust_main`（或其它位置）再 `continue`，或直接 `Ctrl+C` 中断当前执行再下断点。
- 若之前已经烧录过内核，当前运行的二进制应与 `kernel.elf` 一致（同一次构建），否则断点/符号会错位。

之后单步、查看寄存器、内存、`backtrace` 等与 QEMU 下相同。

### 5.4 与 QEMU 调试的差异与注意点

| 项目 | QEMU | 开发板（OpenOCD） |
|------|------|-------------------|
| 谁当 GDB server | QEMU（`-s -S`） | OpenOCD（默认 :3333） |
| 启动时是否暂停 | `-S` 会停在第一指令 | 一般不自动停，需 GDB 打断或下断点 |
| 端口 | 常用 :1234 | 常用 :3333 |
| 内核从哪来 | QEMU 用 `-kernel kernel.bin` 加载 | 需事先烧录到 Flash/内存，GDB 只带符号 |

- 开发板调试前要**先烧录**当前要调的内核，再让 GDB 加载**同一次构建**的 `kernel.elf`。  
- 若板子支持“复位后停在复位向量”，可在 OpenOCD 里配置或使用 `monitor reset halt` 等，实现类似 QEMU `-S` 的“上电即停”；具体查 OpenOCD 和板子文档。  
- 若你的环境是“板子跑 Linux + 上面跑 gdbserver、调试的是用户程序”，则用 `target remote 板子IP:端口` 即可，不涉及 OpenOCD；本节针对的是**裸机/自己写的内核**。

---

## 小结

- **用户态**先练熟 GDB 基本命令和远程概念；  
- **QEMU**：`-s -S` 启动 QEMU，GDB 加载 `kernel.elf` 并 `target remote :1234`，从入口或 `rust_main` 下断点、单步、观察；  
- **开发板**：JTAG/SWD + OpenOCD 在本机开 GDB server（:3333），GDB 同样加载 `kernel.elf` 并 `target remote :3333`，烧录与符号一致即可按同样方式调试。  
把“在 QEMU 里用 GDB 调内核”走通后，开发板调试只是换一个连接对象和端口，核心思路一致。
