# Rust 进阶：build.rs、内联汇编与智能指针/锁实战补充

> 本章不再讲全新的大块语法，而是**补几块在内核工程里经常会遇到、但之前只零散出现过的知识**：  
> 1）`build.rs` 的本质、能做什么、在内核项目里怎么用；  
> 2）以 RISC‑V 64 为例的 `asm!` 内联汇编写法；  
> 3）Rust 宏（`macro_rules!`）的写法与 `?` 运算符；  
> 4）TOML 语法及在 Cargo.toml、.cargo/config.toml 中的用法；  
> 5）智能指针和锁在“内核/系统代码”里的几个典型应用模式。  
> 读完之后，组员应该能看懂现有工程里的这几类代码，也能在需要时照着模板自己写。

---

## 1. build.rs：构建脚本的本质与能力边界

### 1.1 build.rs 本质是什么？

- 在 Cargo 里，**`build.rs` 是“构建脚本”**：  
  - 由 Cargo 在编译你的 crate 之前或过程中调用；  
  - 它本身是一个**独立的小 Rust 程序**，编译后在宿主机上运行；  
  - 它通过向标准输出打印以 `cargo:` 开头的指令，来“告诉 Cargo 和 rustc 一些额外的信息”。
- **关键点**：  
  - `build.rs` 运行在 **构建机上（host）**，而你的内核代码是跑在 **目标机（target）** 上的；  
  - 所以 `build.rs` 适合做“**构建期的逻辑**”，比如选择链接脚本、生成代码、探测环境等；
  - 但它**不能直接影响运行时逻辑**（不能直接改内核行为，只能生成/配置编译输入）。

可以简单理解为：  
> **`build.rs` = “编译前跑一小段 Rust 脚本，帮你把编译这件事准备好”。**

### 1.2 Cargo 什么时候调用 build.rs？

Cargo 检测到你的 crate 根目录里有 `build.rs` 时，会：

1. 先用宿主机 Rust 编译 `build.rs` 本身；  
2. 在**每次需要重新构建该 crate** 时，运行一次 `build.rs`；  
3. `build.rs` 运行时可以：
   - 读/写文件（生成代码、拷贝资源、输出头文件等）；
   - 打印 `cargo:` 前缀的行，告诉 Cargo：
     - 链接参数 / 额外库；
     - 头文件/库搜索路径；
     - 什么时候需要重新运行 `build.rs`；
     - 通过环境变量把信息传给真正的 crate 代码。

Cargo 在运行 `build.rs` 时，会设置一系列环境变量，例如：

- `OUT_DIR`：这次构建时分配给你的**输出目录**路径，适合往里面写生成文件；
- `CARGO_MANIFEST_DIR`：当前 crate 的根目录；
- 还有 target triple、profile 等等（可在 `build.rs` 里用 `env::var` 查看）。

### 1.3 build.rs 能做什么？典型场景一览

**1）为链接器传递自定义脚本/参数**

- 内核/裸机项目里最常见的用途：  
  - 指定自定义链接脚本 `link.ld`；  
  - 追加某些 `-Wl,...` 风格的参数；
  - 根据目标板子选择不同的脚本。
- 通过向 stdout 打印：

```rust
println!("cargo:rustc-link-arg=-Tlink.ld");
println!("cargo:rustc-link-arg=-nostartfiles");
```

**2）生成代码或常量**

- 在构建时根据某些输入文件生成 `.rs` 源码或二进制数据：
  - 例如：根据寄存器描述 JSON 生成 `csr.rs`；  
  - 根据 `memory.x` 之类文件生成地址常量；  
  - 生成 `build_info.rs`，写入 git 提交号、构建时间等。
- 通常写到 `OUT_DIR`，再在库里：

```rust
include!(concat!(env!("OUT_DIR"), "/generated.rs"));
```

**3）探测编译环境**

- 类似 C 里的 `./configure`：  
  - 探测当前是否为某个 OS / 架构；  
  - 是否存在某个库/头文件；  
  - 决定是否启用某些功能（再通过 `cargo:` 通知依赖）。

**4）控制“什么时候需要重新运行”**

- 默认情况下，`build.rs` 在很多情况下会重新跑一遍；你可以精细控制：

```rust
println!("cargo:rerun-if-changed=path/to/file");
println!("cargo:rerun-if-env-changed=MY_ENV");
```

这样可以避免每次都跑，减少不必要的重编译。

### 1.4 一个最小 build.rs 示例（内核相关）

假设你的内核 crate 结构大致是：

```text
kernel/
  Cargo.toml
  src/lib.rs
  link.ld
  build.rs
```

你希望：

- 编译时使用 `link.ld` 链接脚本；
- 如果将来要支持多个板子，可根据 env 决定不同脚本；
- 当 `link.ld` 改变时重新运行 `build.rs`。

一个简单的 `build.rs` 可以是：

```rust
use std::{env, path::PathBuf};

fn main() {
    // 1. 告诉 Cargo：当 link.ld 改变时，需要重新运行 build.rs
    println!("cargo:rerun-if-changed=link.ld");

    // 2. 找到当前 crate 根目录
    let manifest_dir = PathBuf::from(env::var("CARGO_MANIFEST_DIR").unwrap());
    let link_script = manifest_dir.join("link.ld");

    // 3. 把链接脚本路径传给 rustc（-T）
    println!(
        "cargo:rustc-link-arg=-T{}",
        link_script.to_string_lossy()
    );

    // 4. （可选）传递一些信息给代码，比如构建时间或 git 提交号
    let build_time = chrono::Utc::now().to_rfc3339();
    println!("cargo:rustc-env=KERNEL_BUILD_TIME={}", build_time);
}
```

在内核代码里就可以：

```rust
// somewhere in lib.rs
const BUILD_TIME: &str = env!("KERNEL_BUILD_TIME");
```

> 小结：build.rs **不是**在内核运行时执行的，而是在“编译内核前”执行，用来**配置链接、生成代码、传入常量信息**等。

### 1.5 内核项目中常见的 build.rs 用法清单

在内核/裸机项目里，经常看到的 `build.rs` 用法可以归类为：

- **链接相关**：
  - 指定链接脚本（`-Tlink.ld`）；  
  - 为不同 target/board 选择不同脚本；
  - 添加 `-nostartfiles`、`-static` 等链接参数。
- **生成代码/数据**：
  - 按寄存器/CSR 描述生成访问代码；  
  - 生成 trap 向量表或中断号枚举；  
  - 嵌入二进制镜像（如内核自身镜像、initrd）。
- **环境探测**：
  - 判断当前是否交叉编译、是否开启某些 feature；
  - 输出 `cargo:rustc-cfg=...` 来添加 `#[cfg(...)]` 条件。
- **构建信息注入**：
  - git 提交号、构建时间、构建 profile（debug/release）等。

掌握这些之后，看到项目根目录的 `build.rs`，就能大致猜到它是在“帮这个内核完成哪些编译期工作”。

---

## 2. RISC‑V 64 上的 `asm!` 内联汇编写法

### 2.1 `asm!` 的基本形态

在内核里需要直接访问 CSR、关/开中断、发 `ecall`/`wfi` 等时，就需要写内联汇编。Rust 提供：

- `core::arch::asm!`：在函数内部插入少量汇编；
- `core::arch::global_asm!`：插入整段汇编（比如 `_start`、trap 向量）。

先看 `asm!` 的基本形式（RISC‑V 示例）：

```rust
use core::arch::asm;

fn read_sstatus() -> usize {
    let value: usize;
    unsafe {
        asm!(
            "csrr {0}, sstatus",   // 汇编模板，{0} 对应下面 out(reg) value
            out(reg) value,        // 输出操作数，占一个通用寄存器
        );
    }
    value
}
```

**总体结构：**

```rust
asm!(
    "指令模板 1",
    "指令模板 2",
    in("x10") val,        // 显式指定寄存器
    in(reg) val2,         // 让编译器选寄存器
    out(reg) out_var,     // 输出
    inout(reg) in_out,    // 同时做输入和输出
    options(nostack, preserves_flags),
);
```

- **模板字符串**中可以使用 `{0}`、`{name}` 来引用后面的操作数；
- `in` / `out` / `inout` 指定输入/输出操作数；
- `options(...)` 告诉编译器汇编对栈/flags 的影响，便于优化和安全检查。

下面结合 RISC‑V 64 的几个常见场景。

### 2.2 读写 CSR（控制与状态寄存器）

**例：通用读 CSR 宏（RISC‑V）**

```rust
use core::arch::asm;

#[inline]
fn read_csr(csr: &str) -> usize {
    let value: usize;
    unsafe {
        // 这里演示“固定 CSR 名”的写法，更常见的是为每个 CSR 写一个函数
        asm!(
            "csrr {0}, {1}",
            out(reg) value,
            const csr, // 编译期常量，csr 必须是字面量字符串
        );
    }
    value
}

fn read_sie() -> usize {
    let value: usize;
    unsafe {
        asm!(
            "csrr {0}, sie",
            out(reg) value,
        );
    }
    value
}
```

RISC‑V 里更常见的写法是每个 CSR 对应一个函数，例如：

```rust
#[inline]
fn read_sstatus() -> usize {
    let value: usize;
    unsafe {
        asm!("csrr {0}, sstatus", out(reg) value);
    }
    value
}

#[inline]
fn write_sstatus(val: usize) {
    unsafe {
        asm!("csrw sstatus, {0}", in(reg) val);
    }
}
```

**注意：**

- `out(reg) value`：表示“用一个通用寄存器输出到 `value`”；  
- `in(reg) val`：表示“用一个通用寄存器传入 `val`”；  
- 对 CSR 这样的固定寄存器名，不需要指定 `"xN"`，直接写指令里的 CSR 名即可。

### 2.3 关/开中断：示例封装

在内核里，常见需求是：

- **关中断进入临界区**；
- 做一些操作；
- 恢复原来的中断状态。

RISC‑V 下可以通过操作 `sstatus.SIE` 位；下面是一个典型封装示例（示意用）：

```rust
bitflags::bitflags! {
    pub struct SstatusFlags: usize {
        const SIE = 1 << 1; // Supervisor Interrupt Enable
        // ... 其他位
    }
}

fn read_sstatus() -> SstatusFlags {
    let bits: usize;
    unsafe {
        asm!("csrr {0}, sstatus", out(reg) bits);
    }
    SstatusFlags::from_bits_truncate(bits)
}

fn write_sstatus(flags: SstatusFlags) {
    unsafe {
        asm!("csrw sstatus, {0}", in(reg) flags.bits());
    }
}

/// 关中断，返回之前的 sstatus，用于之后恢复
fn disable_interrupt() -> SstatusFlags {
    let old = read_sstatus();
    let mut new = old;
    new.remove(SstatusFlags::SIE);
    write_sstatus(new);
    old
}

/// 恢复之前的中断状态
fn restore_interrupt(flags: SstatusFlags) {
    write_sstatus(flags);
}
```

这里内联汇编只做最低层的 `csrr/csrw`，其余逻辑（位操作、不变式）都放在 safe Rust 里，和第 12 讲的“unsafe 收口”原则是一致的。

### 2.4 地址相关：`sfence.vma` 等

页表修改后，需要执行 `sfence.vma` 刷新 TLB。典型封装：

```rust
#[inline]
fn sfence_vma_all() {
    unsafe {
        asm!("sfence.vma", options(nostack, preserves_flags));
    }
}

#[inline]
fn sfence_vma_asid(asid: usize) {
    unsafe {
        asm!(
            "sfence.vma x0, {0}",
            in(reg) asid,
            options(nostack, preserves_flags),
        );
    }
}
```

这里：

- `options(nostack)` 表示这段汇编不使用栈，便于编译器做一些分析；  
- `options(preserves_flags)` 表示不修改“标志寄存器”（对 RISC‑V 影响相对较小，但保持声明清晰是好习惯）。

### 2.5 `asm!` 小结与常见坑

1. **总是在最小封装处写 asm!**，然后暴露一个 safe 函数给外部：  
   - 把 CSR 名、寄存器操作包在一个私有模块里；  
   - 用 `pub fn` / `pub(crate) fn` 提供安全 API。
2. **明确输入/输出与 clobber**：  
   - 对使用了但没有通过 `in/out` 声明的寄存器/内存，要通过 `options` 或 `nomem` 等声明；  
   - RISC‑V 场景下，多数简单指令只用通用寄存器，不会太复杂。
3. **记得 `unsafe`**：  
   - `asm!` 本身是 `unsafe` 的，调用它时要有清晰的不变式说明；
   - 可以在文档注释里写明调用前后的约束（例如“必须在关中断状态下调用”）。

---

## 3. Rust 宏（macro）在内核工程中的用法补充

前面的讲义里多次用到了 `println!`、`vec!`、`bitflags!` 这类宏，本节从**语法细节到工程用法**一步步说清楚，重点是：

- `macro_rules!` 里参数模式的写法（`$name:ident`、`$e:expr`、`$($arg:tt)*` 这类）到底是什么意思；
- 展开时 `$()`、`*`、`,` 这些符号分别起什么作用；
- 给几个可以在内核项目中直接照抄的模板。

### 3.1 宏解决的问题是什么？

和函数相比，宏有几个关键差异：

- **在编译期展开**：宏在编译阶段“生成代码”，不是运行时调用；
- **可以匹配各种语法片段**：参数可以是标识符、类型、表达式、整个语句块等；
- **可以“重复展开”**：通过 `$()*`/`$()+` 匹配和展开一串模式。

在内核里，宏尤其适合：

- **重复的寄存器/CSR 访问模式**；
- **大段结构类似的枚举/表格（错误码、系统调用表等）**；
- **日志/断言/调试辅助**（在 debug/release 下展开成不同实现）；
- `bitflags!` 这种“生成一整套位操作 API”的场景。

简单记忆：

> 能用函数/泛型/trait 解决的优先用函数；  
> 只有在“**需要在语法级别做变形或重复生成代码**”时再考虑宏。

### 3.2 `macro_rules!` 的基本结构

声明式宏的基本骨架是：

```rust
macro_rules! 宏名 {
    ( 模式1 ) => { 展开代码1 };
    ( 模式2 ) => { 展开代码2 };
}
```

其中核心有三层：

- **1）模式里的“元变量”**：形如 `$x:fragment`；
- **2）`$()` 重复**：形如 `$( ... )*` 或 `$( ... ),*`；
- **3）展开体里用 `$变量` 把匹配到的东西插回去。

#### 3.2.1 fragment 说明：`$name:ident` / `$e:expr` / `$arg:tt` 等

常用的 fragment 类型（只列你实际会用到的几种）：

- **`ident`**：标识符（函数名、变量名等），如 `$name:ident`；
- **`expr`**：表达式，如 `$e:expr`，可以是 `1 + 2`、`foo(bar)` 等；
- **`ty`**：类型，如 `$t:ty` 对应 `u32`、`Option<T>` 等；
- **`path`**：路径，如 `$p:path` 对应 `core::mem::size_of` 这类带 `::` 的名字；
- **`tt`**（token tree）：最宽泛的单位，“任意一棵 token 树”。`$arg:tt` 常用来接收“随便什么参数”，比如 `println!` 的参数列表；
- **`pat` / `pat_param`**：模式（match/let 里的模式），用得较少。

写法举例：

```rust
macro_rules! make_getter {
    ($fn_name:ident, $field:ident, $ty:ty) => {
        fn $fn_name(&self) -> $ty {
            self.$field
        }
    };
}
```

- 这里 `$fn_name:ident` 匹配一个函数名；
- `$field:ident` 匹配结构体字段名；
- `$ty:ty` 匹配返回值类型。

调用：

```rust
struct Foo { x: u32 }

impl Foo {
    make_getter!(get_x, x, u32);
}
```

**编译器在展开宏之后**，等价于你手写了：

```rust
impl Foo {
    fn get_x(&self) -> u32 {
        self.x
    }
}
```

#### 3.2.2 `($($arg:tt)*)` 具体含义逐个拆开

这是你经常看到的模式，比如 `println!` 风格的宏：

```rust
macro_rules! kprintln {
    ($($arg:tt)*) => {{
        /* 展开体 */
    }};
}
```

逐个解释：

- 最外层括号：`( ... )` 表示**整个宏模式**；
- `$arg`：是一个**元变量的名字**，叫 `$arg`；
- `:tt`：fragment 类型是 **token tree**，即“任意语法单位”；
- `$arg:tt`：所以表示“匹配一个 token tree，名字叫 `$arg`”；
- 外层再包一层 `$()`：`$($arg:tt)` 表示“重复匹配 `arg:tt` 这个模式”；
- 后缀的 `*`：表示“重复 0 次或多次”。还有：
  - `+`：重复 1 次或多次；
  - `?`：重复 0 次或 1 次。

所以 `($($arg:tt)*)` 的**整体意思**是：

> “匹配 0 个或多个 token tree，把每个匹配到的 token tree 依次绑定到 `$arg`，形成一个列表。”

展开时写的 `$($arg)*` 则表示：

> “把刚才那批 `$arg` 按原样再展开出来（中间不加分隔符）。”

如果你想在重复项之间**自动加逗号**，可以这样写：

```rust
($($arg:tt),*)  // 匹配：arg, arg, arg 这种形式，逗号是模式的一部分
```

展开时：

```rust
$($arg),*      // 展开时在每两个展开之间自动加逗号
```

完整示例（固定打印前缀，再转发参数）：

```rust
macro_rules! kprintln {
    ($($arg:tt)*) => {{
        use core::fmt::Write;
        // 假设你有一个全局 UART writer
        let _ = write!(crate::uart::KERNEL_STDOUT, "[KERNEL] ");
        let _ = write!(crate::uart::KERNEL_STDOUT, $($arg)*);
        let _ = write!(crate::uart::KERNEL_STDOUT, "\n");
    }};
}
```

调用：

```rust
kprintln!("hello {}", 42);
```

大致展开后等价于：

```rust
{
    use core::fmt::Write;
    let _ = write!(crate::uart::KERNEL_STDOUT, "[KERNEL] ");
    let _ = write!(crate::uart::KERNEL_STDOUT, "hello {}", 42);
    let _ = write!(crate::uart::KERNEL_STDOUT, "\n");
}
```

### 3.3 一个“寄存器访问宏”的完整示例

内核里一个非常典型的模式是：**多个 CSR/寄存器有几乎一样的读函数，只是名字和 CSR 字符串不同**。用宏可以把模式提出来：

```rust
macro_rules! define_csr_read {
    ($fn_name:ident, $csr_name:literal) => {
        #[inline]
        pub fn $fn_name() -> usize {
            let value: usize;
            unsafe {
                core::arch::asm!(
                    "csrr {0}, {1}",
                    out(reg) value,
                    const $csr_name,
                );
            }
            value
        }
    };
}

// 展开成两个具体函数
define_csr_read!(read_sstatus, "sstatus");
define_csr_read!(read_sie, "sie");
```

这里每个参数的含义：

- `$fn_name:ident`：匹配一个标识符，用作函数名；
- `$csr_name:literal`：匹配一个字面量（这里是字符串字面量），用作 asm 模板里的常量 CSR 名；
- 展开体里的 `$fn_name` / `$csr_name` 分别替换成调用时传入的实参。

上面两次调用在编译时会展开为：

```rust
#[inline]
pub fn read_sstatus() -> usize {
    let value: usize;
    unsafe {
        core::arch::asm!(
            "csrr {0}, {1}",
            out(reg) value,
            const "sstatus",
        );
    }
    value
}

#[inline]
pub fn read_sie() -> usize {
    let value: usize;
    unsafe {
        core::arch::asm!(
            "csrr {0}, {1}",
            out(reg) value,
            const "sie",
        );
    }
    value
}
```

这样：

- 逻辑仍然清晰可读；  
- 如果将来你要在读 CSR 前后加日志/检查，只需要改宏定义一处；  
- 调用点简洁，出错率低。

### 3.4 属性宏与 derive 宏：你经常看到的几种

**属性宏 / derive 宏** 是基于“过程宏”（proc-macro）实现的，但在当前项目中，你主要是**使用者**，不需要自己写 proc-macro。

常见例子：

- `#[derive(Debug, Clone, Copy, PartialEq, Eq)]`：自动为结构体/枚举生成常用 trait 的实现；
- `bitflags::bitflags! { ... }`：生成带位操作方法的结构体和相关 impl；
- 日志宏：`log` crate 的 `info!` / `warn!` / `error!` 等；
- 条件编译：`#[cfg(...)]`（编译器内建）。

使用建议：

- **多用 derive 减少样板代码**：  
  - 特别是各种 ID/newtype、状态枚举，`Debug`/`Copy`/`Eq` 几个 derive 几乎是标配；
- **慎用“看不出展开结果”的重魔法宏**：  
  - 内核代码更需要能在 bug 时“顺着展开结果读下去”，不建议堆过多复杂 proc-macro。

### 3.5 什么时候自己写宏、什么时候停手

**适合写宏的场景：**

- 有一大片**重复的语法结构**（函数/类型/impl 模式高度相似），比如几十个 CSR 访问函数、几十个系统调用包装；
- 手写容易出错（少写/写错某一位）、后期改动需要“一处改、处处变”；
- 宏定义本身不长，团队成员都能看懂其展开意图。

**不适合（至少在这个项目里）：**

- 只是为了“少写几行”，但宏体逻辑比原始代码还难懂；
- 宏里塞了复杂控制流，展开结果难以调试；
- 需要写大型 proc-macro 框架，而团队里大多数人不会读。

在现在这个教学内核项目里，**推荐你们只在两类地方自己写 `macro_rules!`**：

1. **寄存器/CSR/bitflags** 等结构高度重复的内容；
2. **日志/调试辅助**（如 `kprintln!`、`trace_irq!`），方便统一前缀、控制开关。

---

## 4. `?` 运算符与错误传播

在内核和系统代码里，很多操作会返回 `Result<T, E>` 或 `Option<T>`。如果每一层都手写 `match` 或 `if let`，代码会变得很啰嗦。**`?` 运算符**就是用来“遇到错误/空值就提前返回”的语法糖，写 API 和封装时几乎天天会用。

### 4.1 `?` 在 `Result` 上的含义

对 **`Result<T, E>`** 来说：

- **`expr?`** 的含义是：
  - 若 `expr` 是 **`Ok(v)`**，则整个 `expr?` 的求值结果是 **`v`**，程序继续往下执行；
  - 若 `expr` 是 **`Err(e)`**，则**当前函数提前返回**，返回值为 **`Err(e)`**（或经过 `From` 转换后的错误类型）。

因此，**只有在返回类型为 `Result<_, _>`（或 `Option<_>`，见下）的函数里才能使用 `?`**。

**示例：不用 `?` 时**

```rust
fn parse_and_validate(s: &str) -> Result<u32, ParseError> {
    let n: u32 = match s.parse() {
        Ok(x) => x,
        Err(e) => return Err(ParseError::InvalidNumber(e)),
    };
    if n > 100 {
        return Err(ParseError::OutOfRange);
    }
    Ok(n)
}
```

**改用 `?` 后**

```rust
fn parse_and_validate(s: &str) -> Result<u32, ParseError> {
    let n: u32 = s.parse().map_err(ParseError::InvalidNumber)?;
    if n > 100 {
        return Err(ParseError::OutOfRange);
    }
    Ok(n)
}
```

这里 `s.parse()` 返回 `Result<u32, _>`，`.map_err(...)` 把错误转成 `ParseError`，然后 `?` 表示：若是 `Err` 就立刻返回该 `Err`，若是 `Ok(v)` 就把 `v` 赋给 `n`。

### 4.2 链式调用里连续用 `?`

多个可能失败的操作连在一起时，用 `?` 可以保持“成功路径”一条线写下来，错误路径统一由 `?` 提前返回：

```rust
fn load_config_and_init(path: &str) -> Result<Driver, InitError> {
    let contents = read_file(path)?;           // 读失败就返回 Err
    let config = parse_config(&contents)?;     // 解析失败就返回 Err
    let driver = Driver::new(&config)?;        // 初始化失败就返回 Err
    Ok(driver)
}
```

等价于手写（示意）：

```rust
fn load_config_and_init(path: &str) -> Result<Driver, InitError> {
    let contents = match read_file(path) {
        Ok(c) => c,
        Err(e) => return Err(e.into()),
    };
    let config = match parse_config(&contents) {
        Ok(c) => c,
        Err(e) => return Err(e.into()),
    };
    let driver = match Driver::new(&config) {
        Ok(d) => d,
        Err(e) => return Err(e.into()),
    };
    Ok(driver)
}
```

只要每个子函数的错误类型能通过 `From` 转成当前函数的错误类型，就可以一路 `?` 下去。

### 4.3 `?` 在 `Option` 上的含义

对 **`Option<T>`** 来说：

- **`expr?`** 的含义是：
  - 若 `expr` 是 **`Some(v)`**，则整个 `expr?` 的求值结果是 **`v`**；
  - 若 `expr` 是 **`None`**，则**当前函数提前返回**，返回值为 **`None`**。

因此，**只有在返回类型为 `Option<_>` 的函数里才能对 `Option` 使用 `?`**。

**示例**

```rust
fn get_nested_value(map: &HashMap<&str, HashMap<&str, i32>>, k1: &str, k2: &str) -> Option<i32> {
    let inner = map.get(k1)?;   // 没有 k1 就返回 None
    let v = inner.get(k2)?;     // 没有 k2 就返回 None
    Some(*v)
}
```

等价于：

```rust
fn get_nested_value(...) -> Option<i32> {
    let inner = match map.get(k1) {
        Some(m) => m,
        None => return None,
    };
    let v = match inner.get(k2) {
        Some(x) => x,
        None => return None,
    };
    Some(*v)
}
```

### 4.4 在内核封装里的典型用法

内核里很多“可能失败”的 API 都返回 `Result`，调用方用 `?` 把错误往上层传，在合适的一层再统一处理（打日志、转成错误码、panic 等）。例如：

```rust
pub fn map_page(
    pt: &mut PageTable,
    vaddr: VirtAddr,
    paddr: PhysAddr,
    flags: PageTableFlags,
) -> Result<(), MapError> {
    let pte = pt.get_pte_mut(vaddr).ok_or(MapError::InvalidVaddr)?;
    if pte.is_valid() {
        return Err(MapError::AlreadyMapped);
    }
    pte.set(paddr, flags);
    Ok(())
}
```

这里 `get_pte_mut` 返回 `Option<&mut Pte>`，用 `.ok_or(...)?` 把 `None` 转成 `Err` 并提前返回；后面若已映射则返回自定义错误，成功则 `Ok(())`。

**小结**：  
- **`Result`**：`expr?` → 成功取里面的值，失败则把 `Err` 从当前函数返回；  
- **`Option`**：`expr?` → 有值取里面的值，`None` 则从当前函数返回 `None`；  
- 函数返回类型必须是 `Result<_, _>` 或 `Option<_>` 才能用对应的 `?`。

---

## 5. TOML 语法与在 Rust 项目中的用法

Rust 生态里几乎所有的配置文件都是 **TOML**（Tom's Obvious, Minimal Language）：`Cargo.toml`、`.cargo/config.toml`、很多工具的配置等。本节把 TOML 的语法和在这些文件里的常见写法讲清楚，方便组员自己改配置、查文档。

### 5.1 TOML 是什么、用在哪儿

- **TOML** 是一种“键值 + 表（section）”的配置文件格式，目标是人类易读、易写，且便于程序解析。
- 在 Rust 项目里最常见的有：
  - **`Cargo.toml`**：包名、版本、依赖、features、构建配置等；
  - **`.cargo/config.toml`**：Cargo 行为、target、`rustflags`、链接器等；
  - 其他工具（如 clippy、rustfmt、某些 CI 配置）也常用 `.toml`。

下面先讲语法，再结合 Cargo 的用法。

### 5.2 基本语法：键值对与类型

**键值对** 用 `key = value`，等号两边可以加空格。值可以是：

| 类型  | 写法示例                                                    |
| --- | ------------------------------------------------------- |
| 字符串 | `name = "hello"` 或 `path = 'C:\Users\foo'`（单引号为字面量，不转义） |
| 整数  | `count = 42`、`hex = 0x10`                               |
| 浮点数 | `ratio = 3.14`                                          |
| 布尔  | `enabled = true`、`debug = false`                        |
| 数组  | `list = [1, 2, 3]`、`tags = ["a", "b"]`                  |
| 内联表 | `point = { x = 1, y = 2 }`                              |

**字符串** 两种常见形式：

- **双引号 `"..."`**：支持转义，如 `\n`、`\"`、`\t`、`\\`；路径里有反斜杠时需写 `\\` 或用单引号。
- **单引号 `'...'`**：字面量字符串，内部不转义（适合 Windows 路径等）。

多行字符串可以用 `"""..."""` 或 `'''...'''`，按需选用。

### 5.3 表（section）与嵌套

用 **`[section]`** 表示一个“表”，下面的键值对都属于这个表，直到下一个 `[section]` 或文件结束。

```toml
[package]
name = "my_kernel"
version = "0.1.0"
edition = "2021"

[dependencies]
```

**嵌套表** 用 **`[a.b]`** 表示“表 `a` 下的子表 `b`”：

```toml
[target.riscv64gc-unknown-none-elf]
rustflags = ["-C", "link-arg=-Tlink.ld"]

[target.'cfg(all())']
rustflags = ["-C", "link-arg=-Tlink.ld"]
```

等价于“先有表 `target`，再在下面有键 `riscv64gc-unknown-none-elf` 或 `cfg(...)`，其值又是一个表”。  
键名若包含特殊字符（如点、空格）或以数字开头，可以用引号：**`[target.'riscv64gc-unknown-none-elf']`**。

**数组 of 表**：用 **`[[section]]`** 表示“同名的多个表”，常用于多元素配置，例如：

```toml
[[bin]]
name = "kernel"
path = "src/main.rs"

[[bin]]
name = "tools"
path = "src/bin/tools.rs"
```

### 5.4 在 Cargo.toml 里的常见块

**`[package]`**：当前 crate 的元信息。

```toml
[package]
name = "kernel"
version = "0.1.0"
edition = "2021"
authors = ["you"]
description = "My OS kernel"
```

**`[dependencies]`**：依赖列表。键是 crate 名，值可以是版本字符串或表。

```toml
[dependencies]
bitflags = { version = "2", default-features = false }
log = { version = "0.4", default-features = false }
spin = "0.9"
my_hal = { path = "crates/hal" }
```

**`[dev-dependencies]`**：仅测试/示例用的依赖，不会打进主产物。

**`[features]`**：可选功能，供 `cfg(feature = "xxx")` 和 `--features` 使用。

```toml
[features]
default = []
verbose = []
alloc = []
qemu = []
```

**`[profile.dev]` / `[profile.release]`**：编译配置，如优化级别、panic 策略。

```toml
[profile.dev]
panic = "abort"
opt-level = 0

[profile.release]
panic = "abort"
opt-level = 3
lto = true
```

**`[build-dependencies]`**：仅给 `build.rs` 用的依赖（如 `cc`、`walkdir`），不参与主代码编译。

### 5.5 在 .cargo/config.toml 里的常见块

**`[build]`**：构建相关，例如默认 target。

```toml
[build]
target = "riscv64gc-unknown-none-elf"
```

**`[target.xxx]`**：针对某个 target 的配置，最常见的是 **`rustflags`**（传给 rustc 的参数）。

```toml
[target.riscv64gc-unknown-none-elf]
rustflags = [
    "-C", "link-arg=-Tlink.ld",
    "-C", "link-arg=-nostartfiles",
]
```

**`[alias]`**：给 `cargo xxx` 起别名。

```toml
[alias]
run-qemu = "run --target riscv64gc-unknown-none-elf"
```

### 5.6 小结与查阅建议

- **键值**：`key = value`，值可以是字符串、数字、布尔、数组、内联表。
- **表**：`[section]` 划分块；`[a.b]` 表示嵌套；`[[section]]` 表示“多个同名表”的数组。
- **Cargo.toml**：重点熟悉 `[package]`、`[dependencies]`（含 `version`/`path`/`default-features`/`features`）、`[features]`、`[profile.*]`、`[build-dependencies]`。
- **.cargo/config.toml**：重点熟悉 `[build].target`、`[target.xxx].rustflags`。

遇到不认识的键，直接查 [The Cargo Book](https://doc.rust-lang.org/cargo/reference/config.html) 和 [Cargo.toml 参考](https://doc.rust-lang.org/cargo/reference/manifest.html) 即可；TOML 语法本身记住“键值 + 表 + 嵌套”就够用。

---

## 6. 智能指针与锁在内核/系统代码中的应用模式

前面《生命周期与智能指针》《并发、异步与操作系统编程》已经系统讲过各种智能指针和锁的语义，本节只补充**在“内核/系统代码”里高频用到的几种组合模式**，方便组员照着用。

### 6.1 全局可变状态：`spin::Mutex` / 自旋锁

在内核里，很多“全局唯一”的东西需要：

- 可变（会被修改）；  
- 在中断/多核场景下安全访问；  
但不能用 `std::sync::Mutex`（no_std 环境），也不能睡眠。

典型做法：使用 **自旋锁**，如 `spin::Mutex`（来自 `spin` crate），或自己的自旋锁封装。

示例：全局 UART 日志器（简化版）：

```rust
use spin::Mutex;

pub struct Uart {
    base: usize,
    // ...
}

impl Uart {
    pub fn write_byte(&self, b: u8) {
        // 使用前面讲过的 write_volatile / asm! 写寄存器
    }
}

// 全局串口实例，用自旋锁保护
static UART: Mutex<Option<Uart>> = Mutex::new(None);

pub fn init_uart(base: usize) {
    let mut uart = UART.lock();
    *uart = Some(Uart { base });
}

pub fn uart_write_str(s: &str) {
    if let Some(uart) = UART.lock().as_ref() {
        for b in s.bytes() {
            uart.write_byte(b);
        }
    }
}
```

要点：

- `static UART: Mutex<Option<Uart>>`：  
  - `static` + `Mutex` 表达“全局共享”；  
  - `Option` 表达“可能还没初始化完”，避免用裸指针；
- 调用方只见到 safe API：`init_uart` / `uart_write_str`。

在多核内核中，这种“`static` + 自旋锁包一层”的模式非常常见：  
比如全局调度器、全局页分配器、设备表等。

### 6.2 多所有者共享读/写：`Arc<Mutex<T>>`（带 alloc 时）

在支持堆分配的内核里（已实现全局分配器、启用 `alloc`），有时需要在多个子系统之间共享某个结构体的可变所有权，例如：

- 一个挂在多条链路上的进程表；  
- 多个子系统都需要访问的缓存结构。

此时可以用 `alloc::sync::Arc` 搭配自旋锁：

```rust
extern crate alloc;
use alloc::sync::Arc;
use spin::Mutex;

struct ProcessTable {
    // ...
}

type SharedProcTable = Arc<Mutex<ProcessTable>>;

fn new_proc_table() -> SharedProcTable {
    Arc::new(Mutex::new(ProcessTable { /* ... */ }))
}

fn use_table(table: &SharedProcTable) {
    let mut guard = table.lock();
    // 在这里安全地修改进程表
}
```

要点：

- `Arc` 解决“多处持有同一个表”的问题（引用计数）；  
- `Mutex`（自旋锁）解决“并发可变访问”的问题；  
- 在内核里通常用 **`Arc<自旋锁<T>>`** 替代用户态常见的 `Arc<std::sync::Mutex<T>>`。

### 6.3 单线程内部可变：`Rc<RefCell<T>>` 的场景

尽管内核整体是并发的，但在某些**严格单线程上下文**（比如：

- 仅在 boot 阶段、单核、单线程；
- 某个数据结构只在某条初始化路径上操作；

可以用 `Rc<RefCell<T>>` 提供：

- 多处共享同一数据结构（`Rc`）；  
- 在只持有不可变引用的前提下做可变修改（`RefCell` 内部可变性）。

这种模式更常见于：

- 配置解析、构建复杂树状结构时（如 VFS 树在初始化阶段）；  
- 之后如果需要跨核共享，再换成 `Arc<Mutex<T>>` 或专门的锁。

示意代码（不特定于内核）：

```rust
use std::cell::RefCell;
use std::rc::Rc;

type NodeRef = Rc<RefCell<Node>>;

struct Node {
    name: String,
    children: Vec<NodeRef>,
}

fn add_child(parent: &NodeRef, child: NodeRef) {
    parent.borrow_mut().children.push(child);
}
```

在 no_std 内核里，通常不会直接用 `Rc`/`RefCell`，而是用类似思想自己实现“树结构 + 内部可变性”的封装；这里主要是帮助理解已经存在的类似模式。

### 6.4 锁的 RAII 与生命周期：典型模式回顾

无论是 `spin::Mutex<T>` 还是 `std::sync::Mutex<T>`，都遵循同一模式：

- `lock()` 返回一个 guard 类型（如 `MutexGuard<'a, T>`）；  
- guard 实现 `Deref<Target = T>` / `DerefMut`；  
- guard 在 `Drop` 时自动释放锁。

这意味着：

```rust
fn example(m: &spin::Mutex<u32>) {
    {
        let mut guard = m.lock(); // 加锁
        *guard += 1;
    } // guard 离开作用域，自动解锁
}
```

在内核里写锁封装时，也推荐沿用这一模式：  

- 让“**锁的生命周期 = guard 变量的作用域**”；  
- 避免显式 `lock()` / `unlock()` 搭配不当导致死锁。

---

## 7. 本讲小结

| 主题 | 要点 |
|------|------|
| **build.rs 本质** | 构建脚本，是在编译 crate 之前/过程中在宿主机上执行的小 Rust 程序，通过打印 `cargo:` 指令控制链接参数、生成代码、注入环境变量等。 |
| **build.rs 能做什么** | 为链接器传递脚本和参数、生成 `.rs`/二进制文件、探测环境并设置 `cfg`/feature、控制何时重新运行、注入构建信息。 |
| **RISC‑V 上的 asm!** | 用 `core::arch::asm!` 写少量内联汇编，典型场景包括读写 CSR、关/开中断、执行 `sfence.vma` 等；始终把 asm 封装在最小范围，向外暴露 safe API。 |
| **宏** | 宏在编译期展开，适合处理寄存器/CSR 访问、重复的表结构、日志辅助等“语法级重复”；用 `macro_rules!` 写声明式宏，善用 `#[derive(...)]` 和常见库宏，避免过度魔法。 |
| **? 运算符** | 在返回 `Result` 或 `Option` 的函数里，`expr?` 表示：成功则取内部值继续执行，失败/空则从当前函数提前返回该错误或 `None`；用于链式错误传播，减少手写 `match`。 |
| **TOML** | Rust 项目配置多用 TOML：键值对、`[section]` 与 `[a.b]` 嵌套、`[[section]]` 数组 of 表；`Cargo.toml` 里熟悉 package/dependencies/features/profile，`.cargo/config.toml` 里熟悉 target 与 rustflags。 |
| **智能指针与锁模式** | 内核中常用 `static + 自旋锁` 管理全局状态，`Arc<Mutex<T>>` 共享可变数据（在有堆时），在严格单线程初始化路径中可用 `Rc<RefCell<T>>` 这类内部可变模式。 |

这些内容都是前面几讲中零散出现过的“工程化碎片”，本讲集中整理之后，希望组员在面对 `build.rs`、内联汇编、以及各种锁与智能指针的组合时，都能**有一套可参考的模板**，而不是从零猜测。下一讲将把这些技术点放进完整的“项目结构与工作流程”中，形成一套可持续协作的开发规范。

