# 交叉编译、QEMU 及工具链

> 本章目标：在大家已熟悉的「预处理 → 编译 → 汇编 → 链接」流水线基础上，搞清楚 **在每一步我们能介入什么、为什么要介入、如何做、以及如何验证介入是否起效**。本课程内核为 **Rust 项目**，因此「如何做」会具体到 **Cargo、.cargo/config.toml、build.rs、global_asm!** 等在 Rust 里的操作方式。学会用 **hex、objdump、readelf、nm、size** 等工具在**各阶段产物**（尤其是 `target/` 下的 ELF）上观察和验证；掌握 **链接器脚本** 和 **汇编** 在构建中的角色，从而能够通过「多个目标文件 + 链接器脚本」描述布局、把符号放到指定段或地址，得到符合预期的内核镜像。了解 **工具链环境配置**（以 Linux 下 **RISC-V 64** 与 **LoongArch 64** 为例）：Rust target 与系统交叉工具链的安装与验证。理解 **QEMU** 在内核开发中的角色：安装、`-kernel`/`-nographic`/`-s -S` 等常用参数；**如何指定引导程序**（`-bios default`/`none`/自定义固件）与 `-kernel` 的启动顺序；**如何查看串口、设备地址与内存布局**（QEMU 文档、设备树 DTB、`-machine dumpdtb=`、参考已有内核）；以及 RISC-V / LoongArch 下运行与调试裸机内核的典型用法。

---

## 1. 为什么内核开发必须用“交叉编译 + 模拟器”

写普通应用程序时，我们往往是「本机编译、本机运行」。而内核开发时：

- **宿主（host）**：你写代码、敲命令的机器（如 x86_64-pc-windows-msvc）；
- **目标（target）**：内核要跑的平台（如 riscv64imac-unknown-none-elf），常常是**裸机、无操作系统**，甚至是你电脑上不存在的架构。

因此需要：

- **交叉编译**：在宿主机上，用面向目标架构的工具链，生成目标平台的机器码；
- **QEMU**：在宿主机上模拟目标硬件，把编译出的内核跑起来。

下面先讲清楚「我们到底在生成什么样的文件」，再讲交叉编译和 QEMU 怎么用。

---

## 2. 目标文件与可执行文件：它们具体是什么

程序从源代码到能在机器上跑，会经过「**目标文件（object file）**」和「**可执行文件（executable）**」两种产物。只有搞清它们各自长什么样，后面用 objdump、readelf、链接脚本时才不会懵。

### 2.1 目标文件（.o / .obj）

**目标文件**是「**单个编译单元**」的产物：一个 `.c`、一个 `.rs` 或一个 `.s`（汇编）经过编译/汇编后，得到的一个二进制文件。典型后缀：`.o`（Unix/Linux）、`.obj`（Windows）。

它里面大致包含：

| 内容 | 说明 |
|------|------|
| **机器码** | 该编译单元里函数、代码段翻译成的目标架构指令 |
| **数据段** | 该单元里的全局变量、静态变量等（已初始化的在 `.data`，未初始化的在 `.bss` 占位） |
| **符号表** | 符号名 ↔ 在本文件内的地址（或段内偏移）的对应关系，供链接器用 |
| **重定位信息** | 哪些指令/数据里还有“未定地址”（例如调用了别的文件的函数），需要链接时填进去 |
| **调试信息**（可选） | 源码行号、变量名与机器码的对应，供 GDB 等使用 |

重要的一点：**目标文件里的地址很多还是“相对的”或“待定”的**——因为编译器不知道这个 `.o` 最终会被放在可执行文件的哪一段、哪个地址，所以跨文件的函数调用、全局变量引用都要留到**链接阶段**再填。

编写程序的本质之一就是在定义「符号」；符号表就是「名字 → 在本文件中的位置」的清单，链接器靠它把多个目标文件拼在一起。

### 2.2 可执行文件（ELF 等）

**可执行文件**是**链接器**把多个目标文件（以及库）拼在一起、并按照一定规则排布后的结果。在 Linux/嵌入式/RISC-V 世界里，常见格式是 **ELF（Executable and Linkable Format）**。

可执行文件与目标文件的主要区别：

- **地址已经确定**：链接器根据我们给的**链接脚本**（或默认规则）决定了每个段、每个符号的**虚拟地址**（VMA）；
- **描述“怎么被加载、从哪执行”**：例如入口点（entry point）、各段在内存中的布局、权限（可读/可写/可执行）等；
- **可以被加载器执行**：操作系统或 QEMU 会按 ELF 头里的信息，把相应段加载到内存，然后从入口点开始执行。

所以：**可执行文件的本质，就是用一种约定好的格式，描述“程序在运行时的内存布局和入口点”**。谁加载它（OS 或 QEMU），谁就按这个描述去布置内存并跳转。

### 2.3 从源文件到可执行文件：整条流水线与我们能干预什么

整体流程可以概括为：

```
源文件(.c/.rs/.s) → 预处理(可选) → 编译 → 汇编 → 目标文件(.o)
                                                          ↓
可执行文件(ELF) ← 链接(ld) ← 多个目标文件 + 库 + 链接脚本
```

- **预处理**：宏展开、条件编译等；我们可干预（例如加 `-D`、改宏定义）。
- **编译**：高级语言 → 汇编或直接 → 目标文件；我们可干预（例如优化等级、`-g` 调试信息、`-T link.ld` 指定链接脚本）。
- **汇编**：汇编源码 → 目标文件；我们可干预（手写 `.s`、在 Rust 里用 `global_asm!` 等）。
- **链接**：多个 `.o` + 链接脚本 → 一个 ELF；**我们最能干预的一步**——通过**链接器脚本**指定每段放在哪、入口是谁、符号在什么地址。

内核开发里，我们经常要做的事就是：**写好多个目标文件（Rust 编译出的、手写汇编的），再用链接器脚本描述“代码段、数据段、栈、堆、入口点”的布局，甚至人为把某些符号放在某段或某地址**，从而得到符合预期地址和布局的二进制，再交给 objcopy 转成裸镜像给 QEMU 或 bootloader。

### 2.4 每一步我们能介入什么、为何介入、如何做、如何验证

大家已经学过体系结构，对「预处理 → 编译 → 汇编 → 链接」这条流水线不陌生。下面按**每一步**说明：我们在这一步**能介入什么**、**为什么要介入**、**具体怎么做**、以及**如何验证介入是否起效**。这样在调链接脚本、改启动代码时，可以有的放矢地停在某一阶段的产物上做检查。

**说明**：本课程的内核是 **Rust 项目**，构建由 Cargo 驱动。下面的「如何做」会以 **Rust/Cargo 为主**：在哪些配置文件里改、在 `build.rs` 里怎么介入、产物在 `target/` 下的什么路径等。这样大家在自己的 Rust 内核仓库里可以直接对照操作。

| 阶段  | 能介入什么               | 为何要介入                         | 如何做（Rust 项目）                                                                             | 如何验证                                                                     |
| --- | ------------------- | ----------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| 预处理 | 条件编译、自定义 cfg        | 按 ARCH/板级/特性切代码路径             | `#[cfg]`、`cfg!()`、`build.rs` 里 `cargo:rustc-cfg=...`                                     | `cargo build -v` 看 `--cfg`；或代码里确认 cfg 生效                                 |
| 编译  | 优化、调试信息、目标、裸机选项     | 裸机 no_std；要 `-g` 给 GDB 用      | `.cargo/config.toml` 的 `target` 与 `rustflags`；`Cargo.toml` 的 `[profile]`；`build.rs` 按需注入 | 对 `target/.../debug/kernel` 用 `rust-objdump`/`rust-readelf` 看段与符号        |
| 汇编  | 手写段名、指令、符号          | 入口、trap、特殊指令必须我们控制            | `global_asm!(...)` 或 `include_str!("entry.S")`；或 `build.rs` 编 entry.S 再 link             | `rust-objdump -d .../kernel -j .text.entry`；`rust-readelf -s` 看 `_start` |
| 链接  | 段布局、入口、符号地址         | 内核加载地址、入口、栈/BSS 由我们定          | `.cargo/config.toml` 里 `rustflags = ["-C", "link-arg=-Tlink.ld"]`；改 `link.ld`            | `rust-readelf -h` 看入口；`-l`/`-S` 看段；`rust-nm` 看符号地址                       |
| 后处理 | 从 ELF 提纯 .bin、strip | QEMU/bootloader 要 flat binary | 手动 `rust-objcopy -O binary .../kernel kernel.bin`；或在 `build.rs`/Makefile 里自动化            | `xxd`/hex 看 bin 前字节与 .text 一致；`ls -l` 看大小                                |

下面逐段展开。

**预处理阶段**

- **能介入**：宏（如 `CONFIG_DEBUG`、`ARCH_riscv64`）、条件编译分支。
- **为何**：同一份源码在不同板子、是否打开调试时，要编译出不同逻辑；内核里大量用条件编译切代码路径。
- **如何做（Rust 项目）**：  
  - **`#[cfg(...)]` / `cfg!(...)`**：用 `target_os`、`target_arch`、`target_env` 等（如 `#[cfg(target_arch = "riscv64")]`），或自定义条件。  
  - **自定义 cfg**：在 `build.rs` 里根据环境或配置输出 `println!("cargo:rustc-cfg=my_feature")`，源码里写 `#[cfg(my_feature)]`；这样可以在编译前根据脚本逻辑决定打开哪些代码路径。  
  - Rust 没有单独的“预处理产物文件”，条件编译在编译阶段由编译器根据 cfg 裁掉分支。  
- **验证**：Rust 没有 `.i` 文件，可（1）用 `cargo build -v` 看 rustc 收到的 `--cfg` 参数；（2）在源码里通过 `#[cfg(my_feature)]` 包含一段仅在该条件下编译的代码（如某条 `panic!` 或符号），看编译/链接是否按预期生效，从而确认配置。

**编译阶段**

- **能介入**：优化等级、是否生成调试信息、裸机相关选项（无标准库、无内置函数等）、目标三元组。
- **为何**：裸机没有 libc，不能依赖编译器内置的 memset/printf 等；调试时需要行号与符号。
- **如何做（Rust 项目）**：  
  - **目标与通用选项**：在项目根目录 `.cargo/config.toml` 里设置默认 target 和 rustflags，例如：  
    `[build] target = "riscv64imac-unknown-none-elf"`  
    `[target.riscv64imac-unknown-none-elf] rustflags = ["-C", "link-arg=-Tlink.ld", "-C", "debuginfo=2"]`  
    这样每次 `cargo build` 都会用该 target 和这些参数。  
  - **优化与调试**：在 `Cargo.toml` 里用 `[profile.dev]`（如 `opt-level = 0`、`debug = 2`）和 `[profile.release]` 控制；或继续在 `rustflags` 里写 `-C opt-level=0`。  
  - **按 target 区分**：若只想对内核 target 加参数，就只写 `[target.riscv64imac-unknown-none-elf]` 下的 `rustflags`；`build.rs` 里也可用 `println!("cargo:rustc-link-arg=...")` 等按条件注入。  
  - 内核 crate 通常加 `#![no_std]`、`#![no_main]`，入口和布局由我们自己的汇编 + 链接脚本控制。  
- **验证**：Rust 编译出的目标文件在 `target/riscv64imac-unknown-none-elf/debug/deps/`（或 `release/deps/`）下，名字形如 `*.rlib` 或参与链接的 `.o` 在中间目录；可直接对**最终链接前的某个 .o** 或对**整个 ELF** 检查：  
  - `rust-objdump -t target/.../debug/kernel` 看是否有 `rust_main` 等符号；  
  - `rust-objdump -d ...` 看反汇编；  
  - `rust-readelf -S target/.../debug/kernel` 看段，确认有 `.text`、`.data` 等且地址已由链接器确定。

**汇编阶段**

- **能介入**：手写汇编的段名（如 `.text.entry`）、指令序列、`.global` 符号、对齐与标签。
- **为何**：入口第一条指令、trap 向量、`sfence.vma` 等必须由我们写死，编译器不会自动生成。
- **如何做（Rust 项目）**：  
  - **方式一（推荐）**：在 Rust 里用 **`core::arch::global_asm!`** 直接嵌入汇编。例如把入口放在单独段里：  
    `global_asm!(r#" .section .text.entry ... "#);`  
    或把汇编写在单独文件里用 **`global_asm!(include_str!("entry.S"))`**，在 `entry.S` 里写 `.section .text.entry`、`.global _start` 等。这样由 rustc 调后端汇编器生成 .o，并和 crate 一起参与链接，无需单独编 `entry.o`。  
  - **方式二**：单独写 `entry.S`，在 **`build.rs`** 里用 `cc` crate（或调用系统汇编器）把 `entry.S` 编译成 `entry.o`，再用 `cargo:rustc-link-search=native=...` 和 `cargo:rustc-link-lib=...` 把该 .o 链进内核；或在一个“链接用”的目录里生成 .o，通过 `-C link-arg=entry.o` 传给链接器。  
  - 无论哪种方式，**链接脚本**里都要用 `*(.text.entry)` 等把该段放在最前，保证入口地址对应我们写的第一条指令。  
- **验证**：  
  - 若用 `global_asm!`，最终入口在 ELF 里，可直接 `rust-objdump -d target/.../debug/kernel -j .text.entry` 看第一条指令（如 RISC-V 的 `la sp, _stack_top`）；  
  - `rust-readelf -s .../kernel` 确认有 `_start` 且绑定正确；  
  - `rust-readelf -S .../kernel` 确认段名与链接脚本里写的一致。

**链接阶段**

- **能介入**：链接脚本（MEMORY、SECTIONS、ENTRY、符号赋值）、参与链接的 .o 列表与顺序、是否链接库。
- **为何**：内核的加载地址（如 0x80200000）、入口点、.text/.data/.bss/栈的布局、以及 `_stack_top`/`_bss_start` 等符号都必须由我们规定。
- **如何做（Rust 项目）**：  
  - **指定链接脚本**：在 `.cargo/config.toml` 的 `[target.riscv64imac-unknown-none-elf]` 下设置  
    `rustflags = ["-C", "link-arg=-Tlink.ld"]`  
    若脚本不在项目根目录，可用 `-C link-arg=-T./src/boot/link.ld` 等相对路径。这样 `cargo build` 时 rustc 会把 `-Tlink.ld` 传给链接器。  
  - **脚本位置**：一般把 `link.ld` 放在项目根或 `src/` 下，并可在 `build.rs` 里加 `println!("cargo:rerun-if-changed=link.ld");`，这样改链接脚本会触发重新链接。  
  - **自定义链接器（可选）**：若要用 LLD 等，在 config.toml 里设 `[target.xxx] linker = "rust-lld"`（或 `lld` 的路径），再在 rustflags 里继续传 `-T`。  
  - 最终可执行 ELF 在 **`target/riscv64imac-unknown-none-elf/debug/<包名>`** 或 **`.../release/<包名>`**（无后缀），这就是我们的 `kernel.elf`。  
- **验证**（这里最容易出问题，建议养成习惯）：  
  - **入口**：`rust-readelf -h target/riscv64imac-unknown-none-elf/debug/kernel`（或你的包名）→ 看 **Entry point address**，应与链接脚本里 `ENTRY(_start)` 及 `_start` 所在段地址一致；  
  - **段布局**：`rust-readelf -l` / `-S` 看 LOAD 段与各节地址、大小是否与脚本一致；  
  - **关键符号**：`rust-readelf -s` 或 `rust-nm`，查 `_start`、`_stack_top`、`_bss_start`、`_bss_end` 等是否在预期地址；  
  - **反汇编**：`rust-objdump -d .../kernel` 看入口处几条指令是否来自你的入口汇编。

**后处理阶段（objcopy / strip）**

- **能介入**：从 ELF 中“切出”纯二进制（去掉文件头、只保留要加载的段）、或 strip 掉符号/调试信息减小体积。
- **为何**：QEMU 的 `-kernel`、很多 bootloader 期望的是 flat binary，而不是 ELF；有时发布时不需要符号表。
- **如何做（Rust 项目）**：  
  - **手动**：先 `cargo build --target riscv64imac-unknown-none-elf`，得到 `target/.../debug/kernel`（即 kernel.elf），再执行  
    `rust-objcopy -O binary target/.../debug/kernel kernel.bin`  
    把 `kernel.bin` 放在项目根或某目录，供 QEMU 或脚本使用。  
  - **自动化**：在 **`build.rs`** 里在 build 完成后用 `std::process::Command` 调用 `rust-objcopy -O binary <elf> <bin>`，把 bin 输出到 `target/.../` 或 `out/`；这样每次 `cargo build` 后都会得到最新的 .bin。也可在 **Makefile** 或 **justfile** 里写 `cargo build && rust-objcopy -O binary ... kernel.bin`，由上层脚本统一驱动。  
  - 需要时加 `-j .text -j .rodata -j .data` 只保留指定段；strip 用 `rust-strip` 处理 ELF（通常对 .bin 不需要）。  
- **验证**：  
  - 用 `xxd kernel.bin | head` 或 PowerShell `Format-Hex kernel.bin` 看前若干字节，应与 ELF 里 .text 段开头的机器码一致（可先 `rust-readelf -S .../kernel` 看 .text 在文件中的偏移，再在 bin 里对比）；  
  - `ls -l kernel.bin` 看大小是否合理（约等于各加载段长度之和）；  
  - 若做了 strip，用 `rust-readelf -s .../kernel` 确认符号表已去掉或只剩需要的。

把「可介入点 → 为何 → 如何做 → 验证」串成习惯后，改链接脚本或启动代码时就可以快速定位是**哪一阶段的产物**不符合预期，再用对应工具检查。

---

## 3. 用工具观察二进制：hex、objdump、readelf、nm、size

理解“目标文件/可执行文件里有什么”之后，就需要会用工具**打开看**，用于调试、验证布局、查符号。这些工具不关心“是 C 还是 Rust 编译的”，只认 ELF/二进制格式。

### 3.1 看“原始字节”：hex 与 xxd

有时需要看文件开头的魔数、某一段的原始内容，最直接的方式就是看十六进制。

- **Linux/macOS**：`xxd kernel.elf` 或 `xxd -l 256 kernel.elf`（只看前 256 字节）。
- **PowerShell**：可用 `Format-Hex kernel.elf` 或先装 `xxd`/用其他 hex 工具。

ELF 文件开头几个字节一般是 `7f 45 4c 46`（`0x7f 'E' 'L' 'F'`），用来标识这是 ELF。用 hex 可以快速确认“这是不是 ELF”“某偏移处是什么值”。

### 3.2 objdump：反汇编、看段、看符号

`objdump` 用来把目标文件/可执行文件“拆开看”：

- **反汇编**：把机器码还原成汇编助记符，便于对照源码和链接结果。

  ```bash
  rust-objdump -d kernel.elf          # 反汇编所有可执行段
  rust-objdump -d kernel.elf -j .text # 只反汇编 .text 段
  ```

- **看段（section）布局**：哪些段、多大、在文件中的偏移。

  ```bash
  rust-objdump -h kernel.elf
  ```

- **看符号表**：符号名和地址的对应，便于确认入口、某函数是否被正确放置。

  ```bash
  rust-objdump -t kernel.elf
  ```

交叉编译时，要用**目标架构**对应的 objdump（如 `rust-objdump` 或 `riscv64-unknown-elf-objdump`），否则可能无法正确反汇编。

调试时常用思路：**怀疑某段没放对** → 用 `-h` 看段地址和大小；**怀疑某函数地址不对** → 用 `-t` 查符号；**怀疑某条指令不对** → 用 `-d` 看反汇编。

### 3.3 readelf：ELF 头、段表、节表、符号表

`readelf` 专门针对 ELF 格式，比 objdump 更细致地展示“ELF 这个容器”里的结构：

- **ELF 头**：入口点、机器类型、段表/节表偏移等。

  ```bash
  rust-readelf -h kernel.elf
  ```

  这里能看到 **Entry point address**，也就是“第一条指令”的地址，可以和链接脚本里设的 `ENTRY(...)` 对照。

- **段表（Program Headers）**：描述“加载时”的段（哪些要加载到内存、权限、VMA/LMA）。

  ```bash
  rust-readelf -l kernel.elf
  ```

- **节表（Section Headers）**：描述“链接时”的节（.text、.data、.bss、符号表、调试信息等）。

  ```bash
  rust-readelf -S kernel.elf
  ```

- **符号表**：和 objdump -t 类似，但有时更清晰。

  ```bash
  rust-readelf -s kernel.elf
  ```

综合使用方式：**先 `readelf -h` 看入口和整体；再用 `readelf -l` / `-S` 看布局是否和链接脚本一致；用 `readelf -s` 或 `objdump -t` 查关键符号的地址**。这样就能在“不运行程序”的情况下，验证我们通过链接脚本和汇编/Rust 得到的二进制是否符合预期。

### 3.4 nm：快速查符号与地址

`nm` 专门用来列目标文件或可执行文件里的**符号名和地址**，比 readelf -s 输出更紧凑，适合快速查“某个符号在不在、地址多少”。

```bash
rust-nm kernel.elf              # 列出所有符号及地址
rust-nm -n kernel.elf           # 按地址排序，便于看布局
rust-nm kernel.elf | grep _start # 只查入口
```

符号类型（如 `T` 表示在 .text、`D` 在 .data、`B` 在 .bss）能帮助确认符号是否被放到预期段。**验证链接脚本**时：改完脚本重新链接，用 `nm` 查 `_stack_top`、`_bss_start`、`_start` 的地址是否与脚本里设计的一致。

### 3.5 size：看各段占多少空间

`size` 汇总可执行文件里 .text、.data、.bss 等段的大小，一眼看出代码和数据各占多少，便于评估内核体积和判断是否某段异常大。

```bash
rust-size kernel.elf
```

输出示例（具体列名因工具链而异）：

```
   text    data     bss     dec     hex filename
  12345     256    4096   16697    4139 kernel.elf
```

**验证用途**：若你改了链接脚本里某段的对齐或合并方式，用 `size` 看 dec/hex 总和是否合理；若 bss 为 0 却期望有未初始化数据，说明 .bss 可能没被正确收集或链接脚本没包含 `*(.bss*)`。

### 3.6 按阶段用的验证清单（小结）

| 阶段产物 | 建议检查命令 | 重点看什么 |
|----------|--------------|------------|
| 预处理结果 (.i / -E) | 直接打开文件 | 宏、`#if` 分支是否按预期 |
| 目标文件 (.o) | `readelf -S`、`objdump -t`、`objdump -d` | 段存在、符号存在、反汇编合理、地址多为 0 |
| 汇编得到的 .o | `objdump -d -j <段名>`、`readelf -s` | 入口指令、`_start` 等符号、段名与链接脚本一致 |
| 链接后的 ELF | `readelf -h`、`readelf -l`、`readelf -S`、`readelf -s` 或 `nm`、`objdump -d` | 入口地址、LOAD 段 VMA、各节地址与大小、关键符号地址、入口处指令 |
| 裸镜像 (.bin) | `xxd`/`Format-Hex`、`ls -l` | 前若干字节与 .text 一致、文件大小合理 |

养成「改哪一阶段就用哪一阶段的产物 + 上表对应命令验证」的习惯，可以很快确认自己的介入有没有起效。

---

## 4. 链接器脚本：描述布局，把符号放到指定段或地址

链接器（如 `ld`）的职责是：把多个目标文件里的段按某种规则合并，并给它们分配**最终地址**。**链接器脚本（linker script）** 就是用来告诉链接器“怎么排布”的：哪些段放在哪块内存、顺序如何、入口点是谁、哪些符号要放在哪个地址。

### 4.1 为什么需要链接器脚本

默认情况下，链接器会使用内置的默认规则（例如“所有 .text 放一起、所有 .data 放一起”），适合普通用户态程序。但内核/裸机程序往往有特殊需求：

- **入口点**必须在某条我们指定的指令（如 `_start` 或 `rust_main` 前的汇编 stub）；
- **某段必须放在固定地址**（例如 RISC-V 上内核期望被加载到 `0x80200000`）；
- **栈、堆的位置和大小**要我们显式留出来；
- **某些符号**需要暴露给 C/Rust 代码使用（如 `_stack_top`、`_bss_start`），用于初始化。

这些都要在链接器脚本里写清楚。

### 4.2 链接器脚本里常写什么

链接器脚本一般包含以下几类内容（语法以 GNU ld 为例）：

- **MEMORY**：定义“内存区域”，例如一块 RAM、一块 ROM，以及它们的起始地址和长度、属性（只读/可执行等）。
- **SECTIONS**：核心部分。定义**输出段**的名字、在内存中的位置（VMA）、由哪些**输入段**组成（例如 `*(.text*)` 表示所有目标文件的 .text）、以及**符号赋值**（如 `_etext = .;` 表示当前地址记作 `.text` 段末尾）。
- **ENTRY(symbol)**：指定程序的入口点，即“第一条要执行的指令”对应的符号。
- **符号赋值**：在脚本里写 `symbol = expression;`，就可以在 C/Rust 里用 `extern` 引用这些符号（例如栈顶 `_stack_top`）。

通过**调整 SECTIONS 里的顺序、地址和对齐**，我们就能**人为地把某些段放在某块内存、把某些符号放在某段或某地址**，从而得到符合预期的可执行文件。

### 4.3 内核里常见用法示例（示意）

下面是一个极简的、示意性质的链接脚本片段，仅用来说明“我们如何控制布局”：

```ld
/* 假设内核被加载到 0x80200000 */
MEMORY
{
  RAM (rwx) : ORIGIN = 0x80200000, LENGTH = 128M
}

ENTRY(_start)

SECTIONS
{
  . = 0x80200000;

  .text : {
    *(.text.entry)   /* 手写汇编的入口，保证在最前 */
    *(.text .text.*)
  }

  .rodata : { *(.rodata .rodata.*) }
  .data   : { *(.data .data.*) }
  .bss    : {
    _bss_start = .;
    *(.bss .bss.*)
    _bss_end = .;
  }

  . = ALIGN(16);
  _stack_top = .;    /* 栈顶，给汇编/Rust 初始化栈用 */
  . += 0x10000;     /* 预留 64KB 栈 */
}
```

含义可以概括为：

- 入口是 `_start`（通常由汇编文件提供）；
- 代码段从 0x80200000 开始，且我们把 `.text.entry` 放在最前面，这样“第一条指令”一定是我们写的入口；
- 在脚本里定义了 `_bss_start`、`_bss_end`、`_stack_top`，这样在 C/Rust 里可以 `extern` 它们，用于清零 BSS 和设置栈指针。

更完整的语法（如 LMA/VMA、AT、KEEP、ALIGN 等）可参考本系列「可执行文件构建流程及相关工具」讲义；这里重点是：**通过构建多个目标文件 + 链接器脚本，我们可以精确描述可执行文件的布局，并把符号放到指定段或地址，从而得到符合预期的二进制**。

---

## 5. 汇编文件在构建中的角色

内核启动、trap 入口、开关中断等，经常需要**精确控制某几条指令**或访问**编译器不会生成的指令**，这时就要手写汇编（或在内联汇编里写）。汇编文件和 C/Rust 一样，先变成**目标文件**，再一起参与链接。

### 5.1 汇编 → 目标文件 → 链接

- 手写汇编文件（如 `entry.s`）经过**汇编器**（如 `as` 或 `rustc` 调用的后端）生成 `entry.o`；
- 这个 `entry.o` 和 Rust/C 编译出的其他 `.o` 一起交给链接器；
- 链接器根据**链接脚本**把 `entry.o` 里的段（例如 `.text.entry`）放到我们指定的位置，于是“第一条指令”就来自这段汇编。

在 Rust 项目里，除了单独的 `.s` 文件外，还可以用 **`global_asm!`** 把一段汇编嵌进当前 crate，编译后同样会生成对应的段，并参与链接；链接脚本里用 `*(.text.entry)` 或类似名字即可把这段放在入口。

### 5.2 何时需要手写汇编

典型场景：

- **入口**：从 bootloader/固件跳过来后的第一条指令（设栈、清 BSS、跳转到 Rust 的 `rust_main`）；
- **trap/异常向量**：保存上下文、调用 Rust 的 trap 处理函数、恢复上下文并返回；
- **原子操作、特殊指令**：例如 RISC-V 的 `sfence.vma`、关中断等，用内联汇编或单独 `.s` 更稳妥。

理解“汇编也是编译单元 → 生成 .o → 和别的 .o 一起链接”之后，就能在 Makefile 或 Cargo 的 build.rs 里正确地把汇编编进去，并在链接脚本里控制它的位置。

---

## 6. 交叉编译的本质：host / target 与三元组

交叉编译指：在**宿主机**上，用面向**目标平台**的编译器/工具链，生成只能在**目标平台**上运行的程序。编译器/工具链用「三元组」描述平台，例如：

- `x86_64-pc-windows-msvc`
- `riscv64imac-unknown-none-elf`

对内核来说，host 是你当前开发机，target 是裸机（如 RISC-V）。构建时使用：

```bash
cargo build --target riscv64imac-unknown-none-elf
```

并配合链接脚本和（必要时）汇编，得到的是“给目标架构用的” ELF；再用 objcopy 等转成 QEMU 或 bootloader 能加载的镜像。

---

## 7. 工具链环境配置（以 Linux 为例）

在 Linux 上做内核开发，需要为**目标架构**准备好编译器、链接器、binutils（objdump、readelf、objcopy 等）以及 QEMU。下面以 **RISC-V 64 位**和 **LoongArch 64 位**为例，说明如何安装和验证环境；其他架构思路类似。

### 7.1 RISC-V 64（riscv64）工具链

**Rust 目标**

- 内核常用 target：`riscv64imac-unknown-none-elf`（裸机、无 OS、ELF）。
- 安装：`rustup target add riscv64imac-unknown-none-elf`。
- 验证：`rustc --print target-list | grep riscv64` 能看到该 target；`cargo build --target riscv64imac-unknown-none-elf` 能通过即说明 Rust 侧已就绪。

**系统交叉工具链（binutils / GCC，可选但推荐）**

- 用于 `objdump`、`readelf`、`objcopy`、`nm` 等查看和加工 ELF；若只用 Rust 自带的 `rust-objcopy` 等则可依赖 rustup 提供的组件，否则建议装系统工具链。
- Debian/Ubuntu 示例：
  ```bash
  sudo apt update
  sudo apt install gcc-riscv64-unknown-elf
  ```
  会提供 `riscv64-unknown-elf-gcc`、`riscv64-unknown-elf-ld`、`riscv64-unknown-elf-objdump`、`riscv64-unknown-elf-readelf`、`riscv64-unknown-elf-objcopy`、`riscv64-unknown-elf-nm` 等（具体包名以发行版为准，有的发行版可能是 `gcc-riscv64-unknown-elf` 或拆成多个包）。
- 若发行版没有，可从 [riscv-collab/riscv-gnu-toolchain](https://github.com/riscv-collab/riscv-gnu-toolchain) 自行配置编译，或使用预编译的 tarball，安装后把 `bin` 目录加入 `PATH`。
- 验证：`riscv64-unknown-elf-readelf --version` 或 `which riscv64-unknown-elf-objcopy`，能执行即表示工具链在 PATH 中可用。

**小结**：Rust 内核至少需要 `rustup target add riscv64imac-unknown-none-elf`；做反汇编、看 ELF、生成 .bin 时用系统提供的 `riscv64-unknown-elf-*` 或 Rust 的 `rust-objdump`/`rust-objcopy` 等（需与目标架构一致）。

### 7.2 LoongArch 64（loongarch64）工具链

**Rust 目标**

- 裸机 target：`loongarch64-unknown-none`（LP64D，硬浮点）；若需软浮点可用 `loongarch64-unknown-none-softfloat`。
- 安装：`rustup target add loongarch64-unknown-none`（及按需 `loongarch64-unknown-none-softfloat`）。
- 说明：该目标为 Tier 2，Rust 官方文档提到链接时可能需要配合 LoongArch 的 C/C++ 交叉工具链；若只用 `rustc` 做链接且不依赖外部 C 库，通常仅 rustup 即可。
- 验证：`rustc --print target-list | grep loongarch64`；`cargo build --target loongarch64-unknown-none` 能成功即说明 Rust 可用。

**系统交叉工具链（binutils / GCC）**

- LoongArch 在多数 Linux 发行版的仓库中不如 RISC-V 常见，可能需要：
  - 从 LoongArch 官方或社区提供的预编译工具链安装（如 Loongnix 等文档中的 gcc/ binutils）；
  - 或从源码构建 [loongarch-dev 等工具链](https://github.com/loongarch/) 并加入 `PATH`。
- 安装后会有 `loongarch64-unknown-elf-objdump`、`readelf`、`objcopy` 等（前缀以实际安装为准），用于查看和加工 LoongArch 的 ELF。
- 验证：`loongarch64-unknown-elf-readelf --version` 或执行对应 `objcopy`，能运行即可。

### 7.3 如何验证环境是否可用（通用）

- **Rust**：`cargo build --target <target>` 不报错；`rustc --print target-list | grep <arch>` 能看到目标。
- **binutils**：对编译出的 ELF 执行一次 `readelf -h <elf>` 或 `objdump -h <elf>`，能正确显示该架构的 ELF 头即可。
- **QEMU**：见下一节；`qemu-system-riscv64 -version`、`qemu-system-loongarch64 -version` 能输出版本即表示已安装。

若团队统一用 Linux，可在文档或脚本里写明上述包名和命令，便于新同学一键配好 RISC-V 与 LoongArch 的开发环境。

---

## 8. Rust 内核项目的交叉编译实践与工具链（小结）

我们的内核是 **Rust 项目**，所有“介入”都围绕 Cargo 和几个配置文件完成，这里做一汇总，方便在仓库里直接对照操作。

- **安装目标**：`rustup target add riscv64imac-unknown-none-elf`。
- **配置构建与链接**：在项目根目录的 **`.cargo/config.toml`** 里设置：  
  - `[build] target = "riscv64imac-unknown-none-elf"`，这样默认就为内核 target 编译；  
  - `[target.riscv64imac-unknown-none-elf] rustflags = ["-C", "link-arg=-Tlink.ld", "-C", "debuginfo=2"]`（或你的 `link.ld` 路径），这样链接阶段会自动使用我们的链接脚本并带上调试信息。  
- **内核 crate**：通常 `#![no_std]`、`#![no_main]`，入口和内存布局由我们自己的 **`global_asm!` 入口 + 链接脚本** 控制；汇编用 `global_asm!(include_str!("entry.S"))` 或写在 `build.rs` 里编 `entry.S` 再链入。  
- **后处理**：ELF 在 `target/riscv64imac-unknown-none-elf/debug/<包名>`（或 `release/` 下）；用 `rust-objcopy -O binary <该 elf> kernel.bin` 得到 QEMU 用的镜像，可放在 `build.rs` 或 Makefile 里自动化。

**工具**：objcopy、objdump、readelf、nm、size 等用目标架构对应的版本（如 `rust-objcopy`、`rust-readelf`），对 **`target/.../debug/<包名>`** 做切裸镜像、反汇编、查符号、看段大小等；验证时都针对这份 ELF 和生成的 .bin 即可。

---

## 9. QEMU 的角色与基本用法

QEMU 在宿主机上**模拟目标架构的整机**（CPU、内存、外设等），用来运行我们交叉编译出的内核镜像，无需真实开发板即可调试。下面说明安装、常用参数、以及 RISC-V / LoongArch 两种架构下的典型用法。

### 9.1 安装与验证（Linux）

- **安装**：多数发行版提供 `qemu-system-misc` 或按架构拆分的包，例如：
  ```bash
  sudo apt install qemu-system-misc   # 常包含 riscv64、loongarch64 等
  ```
  若希望只装某一架构，可查发行版是否提供 `qemu-system-riscv64`、`qemu-system-loongarch64` 等单独包。
- **验证**：  
  - `qemu-system-riscv64 -version`、`qemu-system-loongarch64 -version` 能输出版本即表示已安装；  
  - 用 `-help` 可看该可执行文件支持的机器类型和选项。

### 9.2 典型流程：内核怎么被加载与运行

1. 我们得到的是**裸镜像**（如 `kernel.bin`），由 `objcopy -O binary` 从 ELF 切出，通常从链接脚本约定的地址（如 RISC-V 的 0x80200000）开始放置代码。
2. 使用 **`-kernel kernel.bin`** 时，QEMU 会把该文件读入模拟内存的对应地址，然后从该架构的“上电复位”或约定入口开始执行，相当于我们的 `_start` 成为第一条指令。
3. 无图形界面时加 **`-nographic`**，串口与终端绑定，内核里对“串口”的读写会直接显示在当前终端，便于打印调试。

因此：**构建 → objcopy 得到 .bin → 用 QEMU 的 -kernel 加载 .bin、-nographic 看输出**，是裸机内核开发中最常见的一条链路。

### 9.3 RISC-V 64：常用命令与参数

```bash
qemu-system-riscv64 -machine virt -nographic -kernel kernel.bin
```

- **`-machine virt`**：使用 QEMU 的“virt”虚拟板，内存布局、外设等固定，内核链接脚本里的加载地址（如 0x80200000）需与 virt 的约定一致。  
- **`-nographic`**：不弹图形窗口，串口重定向到当前终端；内核里用 UART 输出会直接打在当前 shell。  
- **`-kernel kernel.bin`**：把 `kernel.bin` 放到 virt 约定地址并从这里启动。  
- **`-m 128M`**：指定模拟内存大小（默认可能较小，可按需调大）。  
- **`-s -S`**：启动时暂停，并打开 GDB server（默认 1234 端口），便于下一章「单步跟踪内核」里用 GDB 连接。

调试时常用：`qemu-system-riscv64 -machine virt -nographic -kernel kernel.bin -s -S`，再在另一终端用 `gdb` 或 `rust-gdb` 连接 `target remote :1234`。

### 9.4 LoongArch 64：常用命令与参数

```bash
qemu-system-loongarch64 -machine virt -cpu la464 -nographic -kernel kernel.bin
```

- **`-machine virt`**：LoongArch 的“virt”虚拟机类型。  
- **`-cpu la464`**：指定模拟的 CPU 型号（la464 为常见选项），LoongArch 下通常需显式指定。  
- **`-kernel kernel.bin`**：加载裸内核镜像；若你的内核是 EFI 或需 BIOS，再按需加 `-bios ...`（裸机课里多数仅 -kernel 即可）。  
- **`-nographic`**：同上，串口到终端。  
- **`-s -S`**：同样用于 GDB 远程调试。

LoongArch 的链接脚本加载地址需与 QEMU virt 机器的约定一致（可查 QEMU 文档或现有内核项目）。

### 9.5 常用参数小结与在 Rust 项目中的用法

| 参数               | 含义                             |
| ---------------- | ------------------------------ |
| `-machine virt`  | 使用“virt”虚拟板，与链接脚本中的加载地址对应      |
| `-kernel <file>` | 加载内核镜像（.bin 或 ELF，视 QEMU 是否支持） |
| `-nographic`     | 无图形，串口到当前终端                    |
| `-serial stdio`  | 显式指定串口为标准输入输出（部分架构默认即如此）       |
| `-m 128M`        | 模拟内存大小                         |
| `-s`             | 开放 GDB server（默认 :1234）        |
| `-S`             | 启动后暂停，等 GDB 连接后再执行             |

在 Rust 内核项目里，通常用 **Makefile** 或 **justfile** 把“编译 → objcopy → 启动 QEMU”写在一起，例如：

- `make run`：`cargo build` → `rust-objcopy -O binary ... kernel.bin` → `qemu-system-riscv64 -machine virt -nographic -kernel kernel.bin`  
- `make debug`：同上，但 QEMU 加 `-s -S`，方便另一终端用 GDB 连接。

这样团队同学只需装好工具链和 QEMU（见第 7 节），即可在同一套命令下跑起 RISC-V 或 LoongArch 内核。

### 9.6 如何指定引导程序（-bios / -kernel 与启动顺序）

裸机内核有时需要“先跑一段固件再跳转到内核”，这段固件就是**引导程序**（如 RISC-V 上的 OpenSBI）。QEMU 用 **`-bios`** 和 **`-kernel`** 控制“谁先执行、谁后执行”。

- **`-kernel <文件>`**：把内核（或 bootloader）镜像加载到机器约定地址（如 RISC-V virt 的 0x80200000），并可由固件或 QEMU 从该地址启动。
- **`-bios <文件>`**：把“固件/BIOS”镜像加载并**先执行**；固件初始化完后再跳转到内核（例如通过约定好的地址或寄存器传递内核入口）。
  - **`-bios default`**：使用 QEMU 内置的默认固件。对 RISC-V virt 而言，通常是 **OpenSBI**；启动时先跑 OpenSBI（M 态），再由它跳转到 `-kernel` 指定的内核（S 态），此时内核期望的加载地址（如 0x80200000）需与 OpenSBI 的约定一致。
  - **`-bios none`**：不加载任何固件；若仍用 `-kernel`，QEMU 可能直接把内核放到约定地址并从这里开始执行（相当于“无固件、内核自己负责最早期的初始化”）。适合做“从零启动”的裸机实验。
  - **`-bios /path/to/opensbi.bin`**：使用自己编译的 OpenSBI 或其它固件，便于固定版本、调试固件本身。

**典型组合**：

- 只要内核、不要固件（课程里最常见）：`-bios none -kernel kernel.bin` 或直接 `-kernel kernel.bin`（部分 QEMU 默认即“无 bios”直接跑 kernel）。
- 先 OpenSBI 再内核（更接近真实板子）：`-bios default -kernel kernel.bin` 或 `-bios opensbi.bin -kernel kernel.bin`。

不同机器类型（virt、sifive_u 等）的默认行为可能不同，拿不准时用 `qemu-system-riscv64 -machine virt -help` 或查 QEMU 官方文档里该机器的 “Boot options”。

### 9.7 如何查看串口、设备地址与内存布局

内核里要访问 UART、PLIC、时钟等设备，必须知道它们的** MMIO 地址**。这些地址由“机器类型”决定，不是 QEMU 命令行参数直接给的，需要通过下面几种方式查。

**（1）看 QEMU 官方文档**

- 每种机器有对应文档，例如 RISC-V 的 virt：  
  [QEMU 文档 → RISC-V → virt 机器](https://www.qemu.org/docs/master/system/riscv/virt.html)  
  里会写该机器支持哪些设备、设备树 (DTB) 如何生成等。部分设备地址会在文档或同站其它说明里给出。
- **RISC-V virt 常用地址**（仅作示例，以你使用的 QEMU 版本和文档为准）：  
  - NS16550 兼容 UART：**0x10000000**（串口输出/输入通常就访问这里）；  
  - PLIC（平台级中断控制器）、CLINT（核本地中断）等地址需查同一份文档或设备树。  
  写裸机驱动时，把这些地址在代码里写成常量或从设备树解析即可。

**（2）设备树 (DTB)**

- QEMU 的 virt 等机器会**自动生成设备树 blob (DTB)** 并传给客户机（例如通过 a1 寄存器或约定内存位置）。DTB 里描述了内存布局、各设备的 reg（地址、长度）、中断号等。
- 在**内核里**解析 DTB，即可在运行时拿到 UART、PLIC 等设备的基地址和范围，而不写死地址（更利于兼容不同机器）。
- 若想“先看一眼” QEMU 生成了什么 DTB，可以：用 `-machine dumpdtb=guest.dtb` 让 QEMU 把当前机器的 DTB 导出到文件，再用 `dtc -I dtb -O dts guest.dtb` 反编译成可读的 .dts，在里边搜 `uart`、`plic` 等即可看到 `reg` 地址。

**（3）参考已有内核项目**

- rcore、xv6-riscv、Linux 等已有 RISC-V / LoongArch 内核，它们的 `virt` 或同款机器的 UART/PLIC 地址通常是按 QEMU 文档写的，直接看其源码里的地址常量和注释即可，例如搜索 “0x10000000”“UART”“NS16550”“PLIC” 等。

**串口与终端对应关系**

- **`-nographic`**：把“第一个串口”重定向到当前终端（stdio），内核往该 UART 写的数据会出现在你跑 QEMU 的 shell 里。
- **`-serial stdio`**：显式指定第一个串口为 stdio，效果同上（部分架构默认就是 stdio）。
- 若有多个串口，可用 **`-chardev` + `-serial chardev:xxx`** 把不同串口接到不同后端（如 `stdio`、`file`、`socket`），便于区分“内核日志”和“用户输入”等。入门阶段一条 `-nographic` 或 `-serial stdio` 即可。

### 9.8 小结：QEMU 还能帮我们做什么（拓展）

- **指定内存大小**：`-m 128M`，避免默认内存过小导致内核以为“没那么多 RAM”。  
- **多核**：`-smp 4`，方便以后做多核、调度实验。  
- **挂载磁盘/镜像**：`-drive file=fs.img,format=raw,id=hd0` 等，做文件系统实验时用。  
- **调试**：`-s -S` 已提过；配合 GDB 可单步、看寄存器、看内存。  
- **不同机器类型**：`-machine ?` 可列出当前可执行文件支持的所有机器，换机器类型会改变设备布局和引导方式，需同时改链接脚本或内核中的设备地址。

遇到“设备地址对不上”“没有输出”“启动顺序不对”时，优先：**确认 -bios/-kernel 组合、确认链接脚本加载地址与 QEMU 文档一致、用文档或 DTB 核对 UART/设备地址**。

---

## 10. 小结与可扩展思路

本章重点：

- **目标文件**：单编译单元的机器码、数据、符号表、重定位信息；**可执行文件**：链接后的 ELF，地址已定，描述加载与入口。
- 在**预处理、编译、汇编、链接、后处理**每一步，我们都能介入：要清楚**能做什么、为何做、怎么做、以及用哪些命令验证**；链接阶段是内核开发里介入最多的一步——通过**链接器脚本**描述布局、把符号放到指定段或地址。
- 用 **hex、objdump、readelf、nm、size** 在**对应阶段的产物**上做验证：.o 看段与符号、ELF 看入口与段布局与关键符号地址、.bin 看开头字节与大小。
- **汇编文件**作为编译单元参与构建，用于入口、trap、特殊指令；与多个目标文件 + 链接脚本配合，得到符合预期的内核二进制。
- **工具链环境**：以 Linux 为例，RISC-V 64 用 `rustup target add riscv64imac-unknown-none-elf`，并按需安装系统交叉工具链（如 `gcc-riscv64-unknown-elf`）以使用 objdump/readelf/objcopy；LoongArch 64 用 `rustup target add loongarch64-unknown-none`，binutils 视发行版或从源码/预编译包安装。安装后用对应架构的 `readelf`/`objdump` 和 `cargo build --target ...` 验证即可。
- **QEMU**：用 `-machine virt -kernel kernel.bin -nographic` 跑裸机内核，用 `-s -S` 配合 GDB 调试；**引导程序**用 `-bios default`/`none` 或 `-bios <文件>` 与 `-kernel` 配合控制启动顺序；**串口、设备地址**通过 QEMU 文档、设备树 (DTB)、`-machine dumpdtb=` 导出 DTB 反编译、或参考已有内核项目获取；RISC-V/LoongArch 的命令和加载地址需与各自链接脚本一致；可在 Makefile/justfile 里封装 `run`/`debug` 目标，统一团队使用方式。

在此基础上，可以扩展：为不同机器类型或内存布局写不同链接脚本、在 Makefile 里加 objdump/readelf/nm 的检查目标、用脚本自动对比“期望的入口地址”和 `readelf -h` 的结果等，让“符合预期的二进制”可重复、可验证。
