# Rust 进阶：并发、异步与操作系统编程

> 本章简要介绍 **并发**（多线程与通道）、**异步**（async/await 与 Future）、以及 **与操作系统/内核编程相关** 的要点。篇幅控制在可一次课收尾或分两次课讲完的范围内。

---

## 1. 并发基础：线程与 `thread::spawn`

Rust 标准库通过 **`std::thread`** 提供 OS 线程。**`thread::spawn(闭包)`** 会启动一个新线程执行闭包；闭包需要 **`Send`**，且若捕获数据则通常用 **`move`** 把所有权移入闭包。

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        thread::sleep(Duration::from_millis(100));
        println!("子线程");
    });
    println!("主线程");
    handle.join().unwrap();  // 等待子线程结束
}
```

- **`handle.join()`**：阻塞当前线程直到子线程结束，返回 **`Result`**（子线程若 panic 则为 `Err`）。
- 若不 `join`，主线程先结束时子线程可能被强制结束。

---

## 2. 共享数据：`Arc` 与 `Mutex`

多线程下要**共享可变数据**时，单用 **`Rc`** 不行（非线程安全），需用 **`Arc<T>`**（线程安全的引用计数）；若还要**修改**，再配合 **`Mutex<T>`** 或 **`RwLock<T>`**（第 7 章已讲）。

常见组合：**`Arc<Mutex<T>>`** —— 多线程共享且可改。

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let c = Arc::clone(&counter);
        let h = thread::spawn(move || {
            let mut guard = c.lock().unwrap();
            *guard += 1;
        });
        handles.push(h);
    }
    for h in handles {
        h.join().unwrap();
    }
    println!("{}", *counter.lock().unwrap());  // 10
}
```

要点：**`Arc::clone`** 只增加计数，不深拷贝；每个线程通过 **`lock()`** 获取锁再修改；**`Send`** 与 **`Sync`** 由 `Arc`、`Mutex` 保证，编译器会检查。

---

## 3. 线程间通信：通道（channel）

标准库提供 **多生产者单消费者** 通道 **`std::sync::mpsc`**：一端 **`send`** 发送，另一端 **`recv`** 接收；发送端可克隆多个，接收端只有一个。

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        tx.send(42).unwrap();
    });

    println!("收到: {}", rx.recv().unwrap());  // 收到: 42
}
```

- **`send(T)`**：发送一个值，若接收端已 drop 则返回 `Err`。
- **`recv()`**：阻塞直到收到值，返回 **`Result<T, E>`**；发送端全关闭时得到 `Err`。
- **`try_recv()`**：非阻塞，无数据时立即返回 `Err`。

通道**转移所有权**：发送后发送方不再拥有该值，避免数据竞争。

---

## 4. `Send` 与 `Sync`

- **`Send`**：类型可以安全地**跨线程转移所有权**（移到另一线程后只在那边使用）。
- **`Sync`**：类型的**引用**可以安全地在线程间共享（`&T` 是 `Send` 即表示 `T: Sync`）。

绝大多数类型自动实现 `Send + Sync`；若类型里包含裸指针、`Rc` 等非线程安全成分，可能只实现其一或都不实现。**`thread::spawn`** 要求闭包为 **`Send`**；在多个线程里共享 **`Arc<T>`** 时，要求 **`T: Send + Sync`**。编译器会据此报错，保证不会把“不能跨线程”的数据误传到线程里。

---

## 5. 异步入门：`async` / `await` 与 `Future`

Rust 的异步模型基于 **`Future`**：表示“将来会产出一个值”的计算，**需要被轮询（poll）** 才会推进。**`async fn`** 和 **`async { }`** 块会生成实现 **`Future`** 的类型；在异步函数内部用 **`.await`** 等待另一个 Future 完成，而不阻塞当前线程。

```rust
async fn do_something() -> i32 {
    let x = some_async_op().await;
    x + 1
}
```

- **`.await`** 只能在 **`async`** 函数或块内使用。
- 光写 **`async`** 不会执行：必须由 **运行时（executor）** 去 **poll** 这个 Future（如 **tokio**、**async-std** 提供的运行时）。标准库只提供 **`Future`** trait，不提供运行时。

因此实际写异步程序时，通常依赖 **tokio** 等库提供 **`#[tokio::main]`**、**`runtime.block_on(...)`** 或 **`spawn`**，把 Future 放进运行时里跑。内核或裸机场景下，可能自建简单 executor 或与事件/中断配合。

---

## 6. 与操作系统/内核编程相关的点

- **无标准库（`no_std`）**：内核或嵌入式往往不链接 `std`，只用 **`core`**，此时没有 `thread`、`Mutex`、`mpsc` 等，需用平台提供的同步原语或自实现。
- **`alloc`**：若允许堆分配，可启用 **`extern crate alloc`**，使用 **`Vec`**、**`String`**、**`Box`** 等；纯 `no_std` 无堆时则不能用。
- **`Send` / `Sync`**：在内核里写驱动或多核同步时，同样要保证跨核传递或共享的数据满足 `Send`/`Sync`，否则编译不通过。
- **Unsafe**：直接操作硬件、MMIO、关闭中断等，需要在 **`unsafe`** 块中写，并自行保证安全不变式（后续会有专门章节）。

下面在 **no_std** 环境下，针对**内核开发我们会用到的知识**做补充，不删前面内容，只做添加。

---

## 7. no_std 与内核开发：我们会用到的知识

内核或裸机程序通常**不链接标准库 `std`**，只使用 **`core`**（以及可选 **`alloc`**）。这样就没有 OS 提供的线程、文件、网络等抽象，但 Rust 的**所有权、类型、trait、泛型**等仍然可用；并发与同步要由我们自己用**自旋锁、原子、关中断**等方式实现。

### 7.1 库的层次：`core` / `alloc` / `std`

- **`core`**：与平台无关的核心类型与 trait（如 `Option`、`Result`、`Iterator`、`Copy`、`Send`、`Sync`），以及基础类型、切片、字符串切片等。**不依赖堆、不依赖 OS**，`#![no_std]` 时始终可用。
- **`alloc`**：依赖**堆分配**的集合与字符串（`Vec`、`String`、`Box`、`BTreeMap` 等）。需要我们自己提供**全局分配器**；内核里若实现了堆，可 `extern crate alloc` 后使用。
- **`std`**：在 `core` + `alloc` 之上再增加**线程、文件、网络、Mutex、channel** 等依赖 OS 的 API。内核不能链接 `std`。

因此 **no_std 内核**里：一定有 **`core`**；若做了堆和分配器，可以有 **`alloc`**；**没有 `std`**，所以前面讲的 `thread::spawn`、`std::sync::Mutex`、`mpsc` 在内核里**不能直接用**，要用下面说的替代手段。

### 7.2 启用 no_std：属性与必须提供的内容

在 crate 根（如 `lib.rs` 或 `main.rs`）写：

```rust
#![no_std]
// 若用堆：
// extern crate alloc;
```

- **`#![no_main]`**：内核或裸机通常没有标准 C 的 `main` 入口，用 `no_main` 后自己通过 **`_start`** 或链接脚本指定入口。
- **panic 处理**：没有 `std` 时没有默认的 panic 实现，必须自己写 **`#[panic_handler]`**，否则链接失败。例如：

```rust
#[panic_handler]
fn panic(_info: &core::panic::PanicInfo) -> ! {
    loop {}
}
```

- **OOM / 分配失败**：若用了 **`alloc`**，还需提供 **`#[alloc_error_handler]`**，在堆分配失败时被调用。

### 7.3 内核里没有 std 时：缺什么、用什么替代

| std 里的东西 | no_std 内核里 | 常见替代 |
| --- | --- | --- |
| `thread::spawn`、`Mutex`、`RwLock` | 没有 | **自旋锁**（spinlock）、**关中断**、**原子操作**（`core::sync::atomic`） |
| `mpsc::channel` | 没有 | 自实现**无锁队列**或基于自旋锁的**有界/无界队列** |
| `println!` | 默认没有 | 自己实现 **`_print`** 或通过 **串口/日志** 输出 |
| 堆分配（Vec、Box 等） | 可选 | 实现 **`GlobalAlloc`**，再 `use alloc::vec::Vec` 等 |
| `std::sync::Arc` | 没有 | 若需引用计数，可用 **`alloc::sync::Arc`**（需 alloc）；或自己用原子实现 |

要点：**并发与同步**在内核里仍然需要，只是不依赖 OS 的线程和锁，而是用 **自旋锁、原子变量、关中断** 等；**Send / Sync** 依然会被编译器检查，跨核或与中断共享数据时要满足。

### 7.4 alloc 与全局分配器

若内核要使用 **`Vec`**、**`String`**、**`Box`** 等，必须提供**全局堆分配器**：实现 **`GlobalAlloc`** trait（通常再包一层，处理对齐与线程/中断安全），并在某处 **`#[global_allocator]`** 指定。

```rust
use core::alloc::{GlobalAlloc, Layout};

struct MyAllocator;

unsafe impl GlobalAlloc for MyAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // 从自己的堆（如静态数组或页分配器）里分配
        todo!()
    }
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        todo!()
    }
}

#[global_allocator]
static A: MyAllocator = MyAllocator;
```

分配器实现里通常要**关中断或加自旋锁**，避免在中断与线程间竞争同一堆。很多教学内核会先用**固定大小数组 + 简单 bump 或链表**实现一个最小可用的分配器。

### 7.5 内核中的并发与同步：自旋锁、原子、关中断

- **自旋锁（spinlock）**：在内核里代替 `Mutex` 的常见做法——获取锁时**自旋等待**，不睡眠（因为内核里往往没有“睡眠/调度”的通用抽象）。数据用 **自旋锁包一层**，多核或中断与主逻辑共享时，先加锁再访问。Rust 里可用 **`spin::Mutex`**（spin crate）或自己用原子实现。
- **原子操作**：**`core::sync::atomic`** 提供 **`AtomicUsize`**、**`AtomicBool`**、**`Ordering`** 等，在 no_std 下可用，适合简单计数、标志位、无锁结构。
- **关中断**：在单核或需要“临界区不被中断打断”时，在访问共享数据前**关中断**，访问完再开。具体 API 依赖目标架构（如 RISC-V 的 `sstatus`、x86 的 `cli`/`sti`），一般在 **`unsafe`** 里封装成 **`critical_section`** 或类似函数。

**Send / Sync**：自旋锁、原子类型会实现 `Send`、`Sync`，用它们包住的数据在满足规则的前提下可以跨核或与中断共享；编译器会检查，写内核时同样要遵守。

### 7.6 unsafe 与直接操作硬件

访问 **MMIO**、**端口 I/O**、**关闭/开启中断**、**读写特殊寄存器** 等，在 Rust 里都要放在 **`unsafe`** 块中，由我们自行保证**安全不变式**（如“关中断期间不调用可能睡眠或依赖当前中断状态的操作”）。具体约定和写法会在 **unsafe 与内核安全** 的专门章节里讲；这里只建立印象：内核开发会大量和 **`unsafe`** 打交道，需要明确**不变式**和**使用边界**。
### 7.7 操作系统编程常用补充

下面几项在内核/裸机编程里经常用到，与 no_std、链接、asm 配合使用。

**1. volatile 读写**

访问 **MMIO**（内存映射 I/O）或硬件寄存器时，每次读/写都**必须真正发生**，不能被子优化掉（编译器可能认为“连续两次读同一地址结果相同”而只保留一次）。应使用 **`core::ptr::read_volatile(ptr)`** 和 **`core::ptr::write_volatile(ptr, value)`**，保证按程序顺序、每次都会产生访问。普通 `*ptr` 的读写不保证这一点。

**2. 内存布局：`repr(C)` 与 `repr(transparent)`**

与 **C 代码**、**硬件结构体**（如寄存器组、描述符）或 **ABI** 交互时，结构体布局必须可控：

- **`#[repr(C)]`**：字段顺序与 C 一致，无 Rust 的重排与填充优化，便于与 C 结构体一一对应、或通过指针强转访问硬件。
- **`#[repr(transparent)]`**：用于“单字段包装类型”，保证该类型与内部唯一字段在内存中布局相同、ABI 相同，便于在不改布局的前提下加类型安全包装（如 `struct VirtAddr(u64)`）。

**3. panic = abort**

no_std 或内核通常**不提供 unwinding**（栈展开），因此常在 **Cargo.toml** 里配置 **`panic = "abort"`**，panic 时直接终止而不展开。这样无需链接 unwinding 运行时，减小体积、简化链接。

```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

**4. extern "C" 与 ABI**

内核入口、**汇编调 Rust** 或 **Rust 调汇编** 时，需要约定**调用约定**（如参数与返回值如何传递）。**`extern "C"`** 表示按 C 的 ABI 生成函数，便于与汇编或 C 互调。例如供汇编调用的 Rust 入口：

```rust
#[no_mangle]
pub extern "C" fn rust_main() -> ! {
    // ...
}
```

**`#[no_mangle]`** 保证符号名不被修饰，链接脚本或汇编里可直接用 `rust_main` 这个名字。不同架构的 C ABI 细节（如哪些寄存器传参）由目标平台约定，写 asm 或链接时要一致。

### 7.8 零大小类型（ZST）

**零大小类型（Zero-Sized Type，ZST）** 是**不占任何内存**的类型：`size_of::<T>() == 0`。编译器不会为它们分配空间，多个 ZST 可以“共享”同一地址；创建或复制 ZST 没有运行时开销。在内核或驱动里常用来做**类型级标记**或**类型参数**，在不增加布局的前提下区分语义。

**常见 ZST：**

- **`()`（unit type）**：Rust 的“空”类型，表达式无有意义返回值时就是 `()`。
- **空枚举**：如 `enum Never {}`，没有变体，无法构造，可用于表示“不会返回”等（与 `!` 类似）。
- **仅包含 ZST 的结构体**：例如只放“标记”类型做类型参数，不携带数据。

**典型用法：标记类型（marker type）**

用不同的 ZST 区分“同一形状、不同语义”的类型，便于类型系统在编译期区分用途，而不增加运行时大小。例如区分“物理地址”与“虚拟地址”：

```rust
#[repr(transparent)]
struct PhysAddr(u64);

#[repr(transparent)]
struct VirtAddr(u64);

// 标记“当前在用户态/内核态”等，仅作类型参数，不占空间
struct KernelMode;
struct UserMode;
```

再如泛型里用 ZST 选实现：`struct Driver<M: Mode>`，`M` 为 `KernelMode` 或 `UserMode` 时走不同逻辑，但运行时没有额外字段。

**PhantomData\<T\>**

标准库提供的 ZST **`PhantomData<T>`** 用来“在类型上挂一个 `T`，但不实际持有 `T` 的值”。常用于：

- 让**未使用的类型参数**参与类型检查（如结构体持有 `*const T` 但不直接包含 `T` 时，加 `PhantomData<T>` 以表达与 `T` 的关联、满足 trait 约束）；
- 表达**协变/逆变**或与生命周期的关系。

ZST 和 `PhantomData` 在 no_std 下也可用（`core::marker::PhantomData`），内核里写抽象或驱动时经常会遇到。

---

## 8. 链接器脚本与内嵌汇编

内核或裸机程序在**连接阶段**往往需要自定义**链接器脚本**以指定入口、段布局和内存映射；在代码里则常用**内嵌汇编**访问特权指令、关中断、或与硬件直接交互。二者都是内核开发中会实际用到的内容。

### 8.1 连接阶段使用链接器脚本

**链接器脚本**（如 `linker.ld`）告诉链接器如何把各 **section**（如 `.text`、`.data`、`.bss`、`.stack`）拼成最终的可执行映像，以及**入口点**、**各段加载/运行地址**等。内核通常需要：

- **指定入口**：例如 `ENTRY(_start)`，与 `#![no_main]` 配合，由我们提供 `_start` 符号。
- **段布局与地址**：把 `.text` 放在指定物理/虚拟地址（如 `0x80200000`），`.data`、`.bss`、栈顶等按顺序排布。
- **符号导出**：在脚本里定义 `__stack_top`、`__bss_start`、`__bss_end` 等，供 Rust/C 代码引用，用于初始化 BSS、设置栈指针等。

在 **Rust** 里通过 **Cargo** 把脚本传给链接器，例如在 `.cargo/config.toml` 中（按目标 triple 配置）：

```toml
[target.riscv64gc-unknown-none-elf]
rustflags = ["-C", "link-arg=-Tlinker.ld"]
```

或在 **build.rs** 里根据环境变量设置 **`RUSTFLAGS`**，加上 **`-C link-arg=-T<脚本路径>`**。这样在运行 **`cargo build --target=riscv64gc-unknown-none-elf`** 时，链接阶段会使用指定的 **`linker.ld`**。

脚本里常用语法包括：`MEMORY { ... }` 定义内存区域，`SECTIONS { ... }` 里写 `. = ALIGN(...);`、`*(.text)`、`*(.rodata)` 等，以及 `PROVIDE(__stack_top = .);` 这类符号。具体写法依赖目标架构和 bootloader 约定，会结合项目在实验/课内给出完整示例。

### 8.2 内嵌汇编（inline asm）

Rust 中可用 **`core::arch::asm!`**（Rust 1.59+ 稳定）在函数内插入目标平台的汇编指令；整段汇编用 **`core::arch::global_asm!`**。内核里常用于：

- **关/开中断**：如 RISC-V 的 `csrci sstatus, 2`、x86 的 `cli`/`sti`。
- **读写 CSR**：RISC-V 的 `csrr`/`csrw` 等。
- **与 ABI 约定一致的调用**：例如用 `ecall` 触发生态调用、或封装几条指令为一个小函数。

**基本用法**（示例：读 RISC-V 的 `sstatus`）：

```rust
use core::arch::asm;

fn read_sstatus() -> usize {
    let x: usize;
    unsafe {
        asm!("csrr {}, sstatus", out(reg) x, options(nostack, preserves_flags));
    }
    x
}
```

- **`out(reg) x`**：表示用通用寄存器把结果写入变量 `x`。
- **`in(reg) val`**：表示把变量 `val` 的值放进某个寄存器作为输入。
- **`options(nostack, preserves_flags)`**：可减少对栈和标志位的影响，按需使用。
- 内嵌汇编是 **`unsafe`** 的，需要自己保证指令合法和副作用。

**`global_asm!`**：用于在 crate 内嵌入**整段汇编**（如 `_start` 入口、trap 向量表），生成独立于 Rust 函数的符号，由链接器按脚本布局放入对应段。

内核或裸机中，**链接器脚本**与**内嵌 asm** 会一起使用：脚本决定“代码放哪、栈设哪”，asm 实现“第一步跳转到 Rust 前如何初始化栈、清零 BSS、再跳转到 `main` 或 `rust_main`”等。后续实验会按目标架构给出完整链路。

---

## 9. cfg 条件编译与 Cargo features

内核或跨平台库经常需要“按目标平台、按功能开关”编译不同代码。Rust 提供 **`cfg` 条件编译**，以及 **Cargo.toml 的 `[features]`**，二者配合可实现“按 feature 或目标做条件编译”。

### 9.1 `cfg` 条件编译

**`#[cfg(...)]`** 是**属性**，加在项（item）上，表示“仅当条件成立时编译该项”；**`cfg!(...)`** 是**宏**，在编译期求值，得到 `true`/`false`，可参与 `if` 等逻辑。

**常见条件：**

| 条件示例                                              | 含义                            |
| ------------------------------------------------- | ----------------------------- |
| **`#[cfg(target_os = "linux")]`**                 | 目标系统是 Linux 时编译。              |
| **`#[cfg(target_arch = "riscv64")]`**             | 目标架构是 riscv64 时编译。            |
| **`#[cfg(feature = "alloc")]`**                   | 启用 Cargo feature `alloc` 时编译。 |
| **`#[cfg(not(feature = "std"))]`**                | 未启用 `std` 时编译（常与 no_std 配合）。  |
| **`#[cfg(all(unix, not(target_os = "macos")))]`** | 组合：Unix 且非 macOS。             |
| **`#[cfg(any(feature = "a", feature = "b"))]`**   | feature `a` 或 `b` 任一启用时编译。    |

**属性用法**（整项参与条件编译）：

```rust
#[cfg(feature = "verbose")]
fn debug_log(s: &str) {
    println!("[DEBUG] {}", s);
}

#[cfg(target_arch = "riscv64")]
use core::arch::asm;
```

**`cfg(debug_assertions)`**：开启 debug 构建（未加 `--release`）时为真，可用于只在调试时执行的代码。

### 9.2 Cargo.toml 中的 `[features]`

在 **Cargo.toml** 里用 **`[features]`** 定义可选功能，供用户或当前 crate 自己按需启用。

**定义与默认：**

```toml
[package]
name = "my_kernel"
version = "0.1.0"

[features]
default = ["std"]   # 默认启用的 feature 列表（可不写 default）
std = ["alloc", "trace"]            # 无额外依赖的“开关”型 feature
alloc = []          # 表示“启用堆”，代码里用 #[cfg(feature = "alloc")] 配合
trace = []          # 例如打开调试追踪
```

- **不带依赖的 feature**：仅作开关，在代码里用 **`#[cfg(feature = "xxx")]`** 或 **`cfg!(feature = "xxx")`** 判断。
- **带依赖的 feature**：可把**可选依赖**和 feature 绑在一起（见下）。

**可选依赖与 feature：**

```toml
[dependencies]
spin = { version = "0.9", optional = true }   # 可选依赖

[features]
default = []
spin = ["dep:spin"]   # 启用 feature "spin" 时才会拉取 spin（dep: 语法要求 Cargo 较新）
```

启用方式：**`cargo build --features "alloc,trace"`** 或 **`cargo build --no-default-features`**（不启用 default 列表）。依赖当前 crate 的其它项目可在其 **Cargo.toml** 里写 **`features = ["alloc"]`** 传递启用。

### 9.3 按 feature 做条件编译

- 在**代码**里用 **`#[cfg(feature = "alloc")]`** 包裹需要堆的模块或函数，用 **`#[cfg(feature = "trace")]`** 包裹追踪逻辑；no_std 内核里常见 **`#[cfg(not(feature = "std"))]`** 与 **`#![no_std]`** 配合。
- 在**依赖**里用 **`optional = true`** + **`[features]` 里 `xxx = ["dep:optional_crate"]`**，实现“只有启用某 feature 才编译、才链接该依赖”。
- 文档与测试也可用 **`#[cfg(feature = "xxx")]`** 控制是否参与 `cargo test` / `cargo doc`。

这样可以在同一份代码里支持“纯 no_std 无堆”“no_std + alloc”“带 trace”等多种构建配置，适合内核或嵌入式项目。

---

## 10. 小结

| 主题 | 要点 |
| --- | --- |
| **线程** | `thread::spawn`、`join`；闭包需 `Send`，常用 `move`。 |
| **共享与同步** | `Arc<Mutex<T>>` 多线程共享可变；通道 `mpsc` 用于线程间传值。 |
| **Send / Sync** | 跨线程转移与共享的标记 trait；编译器据此检查并发安全。 |
| **异步** | `async`/`await` 生成 `Future`；需运行时 poll；实际常用 tokio 等。 |
| **no_std** | `core`/`alloc`/`std` 层次；`#![no_std]`、`panic_handler`、`alloc_error_handler`、`no_main`。 |
| **内核替代** | 无 `std` 线程/锁；用自旋锁、原子、关中断；可选 `alloc` + 自实现 `GlobalAlloc`。 |
| **unsafe** | MMIO、关中断等需 `unsafe`；安全不变式后续章节讲。 |
| **volatile / repr / panic=abort / ABI** | MMIO 用 `read_volatile`/`write_volatile`；与 C/硬件用 `repr(C)`；no_std 常用 `panic = "abort"`；与 asm 互调用用 `extern "C"` + `#[no_mangle]`。 |
| **ZST** | 零大小类型不占内存；用于标记类型、类型参数；`PhantomData<T>` 表示“逻辑上关联 T 但不持有”。 |
| **链接器脚本** | 连接时用 `-C link-arg=-Tlinker.ld` 指定脚本；定义入口、段布局与符号。 |
| **内嵌 asm** | `asm!` / `global_asm!` 插入汇编；关中断、读 CSR 等；需 `unsafe`。 |
| **cfg** | `#[cfg(...)]` 与 `cfg!(...)`；按 target_os、target_arch、feature 等条件编译。 |
| **Cargo features** | `[features]` 定义开关；可选依赖 + feature；`--features`、`--no-default-features`。 |

前面 1～5 节是**通用基础**；6～7 节偏 **no_std 与内核**；第 8 节为**链接与汇编**；第 9 节为**条件编译与 features**，后续课会结合项目再细化（如自旋锁实现、分配器、某架构下的关中断与启动流程等）。
