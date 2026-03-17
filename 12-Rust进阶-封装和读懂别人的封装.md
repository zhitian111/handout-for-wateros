# Rust 进阶：封装和读懂别人的封装

> 本章不再讲新的语法，而是回答两个更“工程化”的问题：  
> **1）如何在 Rust 里做好“封装”；2）如何快速读懂别人已经封装好的东西。**  
> 目标是：看完之后，组员能**少造轮子，多复用**；在内核项目里看到各种 `mod xxx`、`trait Xxx`、`unsafe {}` 也不会慌，知道怎么顺藤摸瓜。

---

## 1. 为什么要专门讲“封装”和“读别人的封装”

- **内核项目特点**：
  - 底层细节多：寄存器、页表、trap、锁、调度器……
  - 但绝大部分人**不需要、也不应该**每次都直接操作这些底层细节。
  - 需要有一层层合理的封装：从“接近硬件”的 unsafe，到对外暴露的 safe API。
- **Rust 的优势**恰好就在这里：
  - 强类型 + 所有权 + trait + 模块系统，可以表达非常丰富的“不变式”和“安全边界”。
  - 做好封装之后，调用方只用安全 API，错误直接在编译期暴露。

本节会围绕两个问题展开：

1. **怎么“写”封装**：模块、可见性、trait、newtype、错误处理、unsafe 包裹等。
2. **怎么“读”封装**：拿到一个 crate / 模块 / trait，如何从外到内、一层层看进去。

---

## 2. “封装”在 Rust / 内核里的具体含义

### 2.1 封装不只是“写很多模块”

在 Rust / 内核语境下，**封装 = 明确边界 + 隐藏细节 + 保证不变式**：

- **明确边界**：清楚“外部能做什么、不能做什么”。
  - 比如：只能通过 `Mutex` 的 API 来访问内部数据，不能直接拿到底层的裸指针。
- **隐藏细节**：实现可以随时重构，但对外 API 尽量稳定。
  - 比如：页表内部是用数组、链表还是哈希表，调用方不关心。
- **保证不变式**：模块内部负责维护“永远成立的条件”。
  - 比如：某个状态只可能是 Running / Ready / Blocked，而不会出现“半初始化”。

### 2.2 内核中常见的几种“封装边界”

| 层次          | 示例             | 对外暴露                             | 内部细节                |
| ----------- | -------------- | -------------------------------- | ------------------- |
| **硬件抽象**    | 寄存器访问、CSR、MMIO | 安全的读写函数、类型化寄存器                   | `unsafe` 裸指针、`asm!` |
| **内存管理**    | 页表、分配器         | `alloc_page()`、`map()` 等接口       | 伙伴算法、slab、位图等       |
| **同步原语**    | Mutex、SpinLock | `lock()` / `unlock()`、RAII guard | 原子指令、关中断策略          |
| **调度/进程管理** | Task / Process | `spawn()`、`yield_now()`          | 就绪队列、时间片、优先级策略      |

思考：**你希望组员“一眼就能看懂和用起来”的，是哪一层？**  
— 那一层就是你要设计的“封装 API 层”；再往下是“实现细节层”。

---

## 3. Rust 里做封装的几把“硬工具”

这里不重复基础语法，只从“封装”的角度再看一次这些工具。

### 3.1 模块与可见性：`mod` / `pub` / `pub(crate)` / `pub(super)`

- **模块 (`mod`)**：划分代码物理和逻辑边界。
- **可见性 (`pub` 修饰)**：控制谁能访问什么。

常用几种可见性：

- `pub`：向**所有 crate** 暴露（库的公共 API）。
- `pub(crate)`：只在**当前 crate 内**可见（对外隐藏，实现细节）。
- `pub(super)`：对**父模块**可见，常用于分层封装。
- 默认不写 `pub`：仅在**当前模块**内可见。

典型模式：

```rust
// src/mem/mod.rs
mod frame;          // 物理帧分配
mod page_table;     // 页表操作

pub use frame::FrameAllocator;       // 对外暴露高层接口
pub use page_table::PageTable;

// 具体实现细节只在本 crate 内可见
pub(crate) use frame::PhysFrame;
```

**建议**：  
把“别人需要知道的名字”集中在 `mod.rs` 或 `lib.rs` 里用 `pub use` 暴露，调用者只需要 `use crate::mem::PageTable`，不用关心内部目录结构。

### 3.2 结构体字段私有 + `impl` 提供 API

把字段设为私有，只通过方法访问，是最常见的“封装不变式”的手段：

```rust
pub struct Task {
    id: usize,
    state: TaskState,
}

impl Task {
    pub fn id(&self) -> usize {
        self.id
    }

    pub fn is_runnable(&self) -> bool {
        matches!(self.state, TaskState::Ready)
    }

    pub fn set_state(&mut self, state: TaskState) {
        // 这里可以做各种检查，不允许非法状态转换
        self.state = state;
    }
}
```

- **外部不能随便改 `state`**，只能通过 `set_state`，这样你有机会在里面检查、打 log、统计指标。
- 以后要改 `TaskState` 的实现细节，外部 API 不一定需要变。

### 3.3 trait 作为“接口”：约定能力，而非类型

第 5 讲已经详细讲过 trait 的语法，这里只看“封装”视角：

- **trait = 某一类东西能做什么**，而不是“某个具体类型是什么”。
- 在内核里常见的例子：
  - `trait BlockDevice`：块设备统一接口，磁盘 / RAMDisk / 虚拟设备都可以实现。
  - `trait Console`：输出字符的能力，可以对应串口 / VGA / 日志缓冲等。

```rust
pub trait BlockDevice {
    fn read_block(&self, lba: u64, buf: &mut [u8]);
    fn write_block(&self, lba: u64, buf: &[u8]);
}
```

上层文件系统代码只依赖 `BlockDevice`，至于底层是 virtio-disk 还是 RAM disk，**由启动时的注入/选择决定**。  
这就是“**依赖抽象，不依赖实现**”，也是封装的一种。

### 3.4 newtype：在类型层面“加一层语义”

**newtype 模式**：用一个只有一个字段的 `struct` 包住现有类型，给它新的含义和方法。

```rust
#[repr(transparent)]
struct SV39PhysAddr(usize);

pub type PhysAddr = SV39PhysAddr(usize);


impl PhysAddr {
    pub fn new(addr: usize) -> Self {
        // 保证按页对齐等不变式
        debug_assert!(addr & 0xfff == 0);
        PhysAddr(addr)
    }

    pub fn as_usize(&self) -> usize {
        self.0
    }
}
```

- **调用方不能乱造 `usize` 当成物理地址**，只能通过 `PhysAddr::new`。
- 以后要做检查 / 日志 / 转换，都集中在 `PhysAddr` 的实现里。

在内核里，这个模式随处可见：`Pid`、`PageNumber`、`FrameNumber`、`VirtAddr` ……  
都是“newtype 封装 + 不变式保证”的组合。

### 3.5 错误处理与 `Result<T, E>`：让错误成为 API 的一部分

封装的 API 不只是“做成什么”，也要清楚“可能失败时，怎么表达失败”：

- 返回 `Option<T>`：简单的“有 / 无”（如查找失败）。
- 返回 `Result<T, E>`：明确表达错误原因。

```rust
pub enum AllocError {
    OutOfMemory,
}

pub trait FrameAllocator {
    fn alloc(&mut self) -> Result<PhysFrame, AllocError>;
    fn dealloc(&mut self, frame: PhysFrame);
}
```

比起直接在内部 `panic!` 或 `unwrap()`，把“可能失败”作为类型的一部分，更利于：

- 调用方做降级处理；
- 集成到更高层的错误传播；
- 在文档里清楚告诉别人“这玩意儿会在哪些情况下失败”。

### 3.6 unsafe 封装：对外只暴露 safe API

在内核里不可避免要写 `unsafe`：

- 访问 MMIO 寄存器；
- 修改页表；
- 与汇编 / 裸指针交互。

**好的封装模式**是：

```rust
pub struct PageTable {
    // 内部有裸指针 / 物理地址等
}

impl PageTable {
    pub fn map(
        &mut self,
        va: VirtAddr,
        pa: PhysAddr,
        flags: PageTableFlags,
    ) -> Result<(), MapError> {
        unsafe {
            self.map_inner(va, pa, flags)  // 只在这一处用 unsafe
        }
    }

    // 只在本模块内可见的真正不安全实现
    unsafe fn map_inner(&mut self, va: VirtAddr, pa: PhysAddr, flags: PageTableFlags)
        -> Result<(), MapError>
    {
        // 操作硬件/内存的细节
        Ok(())
    }
}
```

- `unsafe fn map_inner` 的使用范围被限定在当前模块；
- 对外只暴露 safe API `map`，调用方不需要知道细节，也不允许随意用 unsafe。

**阅读别人代码时的一个技巧**：  
找“unsafe 的最外层”，看作者是如何把它包成安全 API 的，这基本就是他想表达的不变式。

### 3.6.1 注释与文档注释：`//`、`///`、`//!` 与跨行

写封装时经常要加“给人看的说明”，Rust 里几种注释含义不同，且会直接影响 `cargo doc` 生成的文档：

| 写法        | 作用                                                        | 是否进入 rustdoc       |
| --------- | --------------------------------------------------------- | ------------------ |
| **`//`**  | 普通行注释：解释本行或后面几行代码，给维护者看。                                  | 否                  |
| **`///`** | 文档注释：修饰**紧接着的**那一项（函数、结构体、枚举、trait 等），说明“这个项是干什么的”。       | 是，出现在该项的文档页        |
| **`//!`** | 文档注释：修饰**当前所在的**项（当前模块或当前 crate），通常放在文件或 `mod { }` 块的最上面。 | 是，出现在模块/crate 的文档页 |

跨行时也有对应三种形式：

| 写法               | 作用                                             | 是否进入 rustdoc |
| ---------------- | ---------------------------------------------- | ------------ |
| **`/* ... */`**  | 普通块注释：一段说明或临时注释掉多行代码。                          | 否            |
| **/\*\* ... */** | 块文档注释：修饰**紧接着的**那一项，和 `///` 等价，只是可多行。          | 是            |
| **`/*! ... */`** | 块文档注释：修饰**当前所在的**项（模块/crate），和 `//!` 等价，只是可多行。 | 是            |

简单示例：

```rust
// 这是普通注释，不会出现在 cargo doc 里

/// 这个函数会出现在文档里，说明“下一项”是干什么的
pub fn public_api() {}

/// 多行时可以用多行 `///`，也可以改用块文档：
/** 这也是对“下一项”的文档，
    适合写很长说明。 */
pub struct MyType;

mod uart {
    //! 这是对“当前模块”的文档，描述整个 mod 的职责。
    //! 多行就多写几行 `//!`。
    //!
    //! 或者用块文档：
    /*!
     * 当前模块的说明可以写很长，
     * 用 /*! ... */ 一块写完。
     */
}
```

**记忆口诀**：`///` 和 `/** */` 是“**下一项**”；`//!` 和 `/*! */` 是“**当前项**”（当前模块/crate）。做封装时，对外暴露的类型和函数用 `///` 写清用途和示例，模块顶部用 `//!` 写清“这一层是干什么的”，文档和示例就能和代码同步。

#### 生成与查看文档：`cargo doc`

写好 `///`、`//!` 之后，用 **`cargo doc`** 可以把这些注释转成 HTML 文档（由 rustdoc 生成），和 docs.rs 上看到的风格一致。

| 命令 | 作用 |
|------|------|
| **`cargo doc`** | 编译当前项目及依赖，生成文档到 `target/doc/`，**不**自动打开浏览器。 |
| **`cargo doc --open`** | 生成文档后**自动在浏览器中打开**当前 crate 的文档首页，最常用。 |
| **`cargo doc --no-deps`** | 只生成**当前 crate** 的文档，不生成依赖的文档；构建更快，适合只关心自己写的 API 时。 |
| **`cargo doc --document-private-items`** | 把**私有项**（未 `pub` 的类型、函数等）也写进文档，方便团队内部查阅内部 API。 |

生成后的文档在 **`target/doc/`** 目录下，首页一般是 `target/doc/你的crate名/index.html`。依赖的 crate 也会各有一个子目录，可以顺着左侧导航在“当前 crate”和“依赖”之间切换。

**典型用法**：在项目根目录执行 `cargo doc --open`，写完一层封装后随时看一遍文档，确认模块说明和公开 API 的注释是否清晰；给别人用之前先让自己能只看文档就上手。

### 3.7 最佳工程实践（整体原则）+ 示例

把上面几小节串起来，可以总结成几条**做封装时的最佳实践**：

- **先画清边界再写代码**：先想清楚“这个模块对外应该暴露哪些类型/函数/trait”，再决定内部怎么实现。
- **对外 API 尽量稳定、简单**：给调用方少量“好记、难用错”的接口；内部细节可以随时重构。
- **unsafe 收口在最小范围**：让 90%+ 的调用代码完全不需要写 `unsafe`；把危险操作集中在少数封装层。
- **错误和不变式体现在类型里**：用 `Result` / `Option` / newtype / enum 让“可能失败”“状态有限”在类型上可见，而不是隐藏在注释里。
- **文档和示例同步维护**：每做一层封装，顺手在模块文档里加“这层的职责是什么、给别人用的最小示例”，方便新人快速上手。

下面用一个**小型“串口输出”封装**示例，把这几条实践体现出来：

```rust
//! 串口输出模块：对外只暴露「写字符串」的 safe API，内部集中处理 MMIO 和 unsafe。
//!
//! # 示例
//!
//! ```ignore
//! use crate::uart::Uart;
//!
//! let uart = Uart::new(0x1000_0000);  // 基地址由板子决定
//! uart.write_str("hello kernel\n").expect("uart init");
//! ```

/// 初始化或写入失败时的错误（类型可见，调用方能分支处理）
#[derive(Debug)]
pub enum UartError {
    NotReady,
}

/// 对外暴露的唯一“句柄”：调用方只拿到 Uart，不能直接碰寄存器
pub struct Uart {
    base_addr: usize,
}

impl Uart {
    /// 构造时只做“记下基地址”，不访问硬件，因此是 safe 的
    pub fn new(base_addr: usize) -> Self {
        Self { base_addr }
    }

    /// 对外唯一写入接口：返回 Result，错误在类型里体现
    pub fn write_str(&self, s: &str) -> Result<(), UartError> {
        for &b in s.as_bytes() {
            self.write_byte(b)?;
        }
        Ok(())
    }

    /// 单字节写入：内部集中一处 unsafe，外部调用方永远不需要写 unsafe
    fn write_byte(&self, byte: u8) -> Result<(), UartError> {
        unsafe {
            let ptr = self.base_addr as *mut u32;
            ptr.write_volatile(byte as u32);
        }
        Ok(())
    }
}
```

对应到几条实践：

| 实践 | 在示例中的体现 |
|------|----------------|
| **先画清边界** | 对外只有 `Uart`、`Uart::new`、`write_str`；寄存器布局、volatile 细节全部在模块内部。 |
| **API 稳定、简单** | 调用方只需 `uart.write_str("...")`，不关心基地址如何换算、哪个偏移是数据寄存器。 |
| **unsafe 收口** | 只有 `write_byte` 里一处 `unsafe`，且为 `fn` 私有，外部无法误用。 |
| **错误在类型里** | `UartError` + `Result<(), UartError>`，调用方可以 `?` 或 `match`，而不是“注释里说可能失败”。 |
| **文档和示例** | 模块顶部的 `//!` 说明了职责，并给出最小使用示例，新人复制即可跑通。 |

这样，**“做好自己的封装”** 既有原则，也有一个可模仿的小例子；组员在做其他硬件抽象（如时钟、GPIO）时，可以按同样套路扩展。

---

## 4. 如何系统地“读懂一个封装”

本小节给一套通用的“读别人代码”的步骤，建议组员照着练几遍。

### 4.1 第一步：从 `Cargo.toml` / `lib.rs` / `main.rs` 入手

拿到一个 crate（包括你们自己的内核仓库）：

1. 先看 `Cargo.toml`：
   - crate 名字是什么？（别人依赖时怎么写）
   - `lib` / `bin` 有哪些？你现在关心的是库还是可执行？
2. 再看 `src/lib.rs` 或 `src/main.rs`：
   - 顶层 `mod` 是怎么划分的（`mod mem; mod trap; mod fs;` 等）。
   - 有哪些 `pub use` 暴露出来，是整个 crate 对外的“门面”。

**经验**：  
**`lib.rs` / `main.rs` 往往就代表作者心目中的“模块分层图”。**  
先搞清楚大致分层，再深入某一块。

### 4.2 第二步：看文档注释和 README / examples

很多封装已经通过文档给你“翻译”过一遍了：

- `///` 风格注释会成为 rustdoc；
- 仓库根目录下的 `README.md`、`examples/` 往往有最小示例。

在 Cursor / IDE 里，移动到某个类型/方法名上，通常可以直接看到它的文档注释。  
**先看注释再看实现，可以减少很多“误解”。**

### 4.3 第三步：从“对外暴露的 API”往里钻

顺序建议：

1. 找这个模块对外 `pub` 的东西：
   - `pub struct Xxx` / `pub trait Xxx` / `pub fn xxx()`。
2. 对每个 `pub` 类型，先只看：
   - 公开字段（如果有的话）；
   - 公开方法的签名（参数 / 返回值 / 是否 `&self` / `&mut self`）。

问自己几个问题：

- **这是在表达什么不变式？**
  - 比如 `&mut self` 明显暗示“独占访问 + 可能修改内部状态”。
- **有哪些错误路径？**
  - 看 `Result` / `Option` / `panic!`，理解它打算在哪一层处理错误。

### 4.4 第四步：顺着 trait / newtype 向下追实现

当你看到：

- 某个 trait：`trait BlockDevice { ... }`；
- 某个 newtype：`pub struct VirtAddr(usize);`

可以这样往下看：

1. 用搜索（如 IDE 的“转到实现”或 `rg "impl\s+BlockDevice" -n`）找**所有实现**。
2. 先挑一个最简单的实现看：
   - 是如何用 unsafe 与底层交互的？
   - 有哪些不变式检查（`assert!` / `debug_assert!` / `if`）？
3. 再回到 trait，看它对上层暴露了怎样的语义。

**小技巧**：  
看到 `unsafe` 块时，不要先盯着细节，而是问：**“作者是想向外界承诺什么？”**  
——这些承诺一般会通过函数签名、注释、命名体现出来。

### 4.5 第五步：结合调用点反推设计意图

光看实现可能会迷糊，可以反过来“顺着调用方看”：

1. 搜索某个方法 / trait 在哪里被调用；
2. 看调用方是如何组合这些 API 的；
3. 反推 API 设计时作者的“假设”和“常用路径”。

例子：

- 看到 `scheduler::spawn(task)` 在多个地方被调用；
- 你可以在所有调用点看它前后有没有统一模式（如必须先初始化页表 / 打开中断）；
- 这往往就是作者心里“正确使用姿势”，可以帮助你理解封装边界。

### 4.6 第六步：如何找到合适的 crate，并只靠 docs 学会用

在工程里，“读懂别人的封装”常常发生在**外部依赖 crate** 上，尤其是我们希望尽量复用成熟的 `no_std` 库。可以给组员一套通用流程：

1. **在 crates.io 上按关键词搜索**：  
   比如想找位标志封装，就搜 `bitflags` / `flags`；想找日志库，就搜 `log`。
2. **在 crates.io 页面上重点看几项信息**：
   - README 里的“Quick start / Usage”示例；
   - `Features` 一栏：是否有 `std` / `no_std` 相关 feature，默认开哪些；
   - `Dependencies`：依赖的库是否也支持 `no_std`（避免一条链拉进 std）。
3. **打开 docs.rs 对应文档**：
   - 先看 crate 级文档（页面最上面的说明和示例）；
   - 再看最顶层模块和最核心的几个类型/宏（一般 README 里会点名）。
4. **本地用 `cargo doc --open` 生成文档**（尤其是自己项目里又包了一层封装时）。

下面用两个常见的内核友好型库来示范：**`bitflags`** 和 **`log`**。

#### 4.6.1 例子：用 `bitflags` 封装寄存器/标志位

`bitflags` 是一个非常轻量的 crate，大部分情况下 **支持 `no_std`**。典型使用流程：

- 在 `Cargo.toml` 里添加依赖时，可以给每个依赖设置多种属性。常见属性及对应写法如下（均写在 `[dependencies]` 下）：

| 属性 | 含义 | 示例 |
|------|------|------|
| **`version`** | 版本要求：精确版本或语义化范围 | `version = "2.3.1"`、`version = "2"`（兼容 2.x）、`version = "^1.0"` |
| **`features`** | 开启该 crate 的**可选特性**（数组） | `features = ["serde"]`、`features = ["std", "derive"]` |
| **`default-features`** | 是否启用该 crate 的**默认特性**（如 `std`） | `default-features = false`（no_std 常用） |
| **`optional`** | 是否作为可选依赖，可由 `[features]` 决定是否启用 | `optional = true`，配合 `[features]` 里 `xxx = ["dep:serde"]` 使用 |
| **`package`** | 依赖的真实包名（与左侧名字不同时用） | 左侧写 `json = { package = "serde_json", version = "1" }`，代码里 `use json::Value` |
| **`path`** | 从**本地路径**引入（不经过 crates.io） | `my_crate = { path = "../my_crate" }` |
| **`git`** | 从 **Git 仓库**引入 | `my_crate = { git = "https://github.com/foo/bar.git" }` |
| **`branch`** / **`tag`** / **`rev`** | 配合 `git`：指定分支、标签或 commit | `branch = "main"`、`tag = "v1.0"`、`rev = "a1b2c3d"` |
| **`registry`** | 使用指定的**索引源**（私有/镜像） | `registry = "company-registry"`（需在 `.cargo/config.toml` 配置） |

组合示例（按需选用）：

```toml
[dependencies]
# 只写版本：从 crates.io 拉取，使用默认特性
bitflags = "2"

# 版本 + 关闭默认特性（no_std 常用）
log = { version = "0.4", default-features = false }

# 版本 + 关闭默认特性 + 开启部分 feature
serde = { version = "1", default-features = false, features = ["derive"] }

# 可选依赖：只有启用 feature "serialize" 时才引入
serde = { version = "1", optional = true }

# 用不同名字依赖同一个包：代码里用 serialization，实际包名是 serde_json
serialization = { package = "serde_json", version = "1" }

# 本地路径（同仓库子目录或兄弟目录）
kernel_hal = { path = "crates/hal" }

# 从 Git 拉取：指定分支
some_lib = { git = "https://github.com/foo/some_lib.git", branch = "no_std" }

# 从 Git 拉取：指定 tag 或 commit
other_lib = { git = "https://github.com/foo/other.git", tag = "v0.1.0" }
# 或：rev = "a1b2c3d4"
```

可选依赖要在 `[features]` 里“挂上”才会被编译进去，例如：

```toml
[features]
default = []
serialize = ["dep:serde"]   # 启用 serialize 时才会拉入 serde
```

上面 `dep:serde` 表示“启用可选依赖 serde”；左侧的 `serialize` 是你们 crate 对外暴露的 feature 名字。

**内核场景下最常见**：`version` + `default-features = false`，必要时加 `features = ["..."]`；调试或适配时可临时用 `path` / `git`。

一个内核场景下的最小例子：

```toml
[dependencies]
bitflags = { version = "2", default-features = false }
```

- 打开 docs.rs 上 `bitflags` 的文档，crate 级文档里会有这样的示例：

```rust
bitflags::bitflags! {
    pub struct Flags: u32 {
        const READ  = 0b00000001;
        const WRITE = 0b00000010;
        const EXEC  = 0b00000100;
    }
}
```

阅读要点：

- `bitflags!` 是一个宏，文档里会告诉你：
  - 支持哪些底层整数类型（`u8`/`u16`/`u32`/...）；
  - 生成的类型有哪些方法（如 `contains`、`insert`、`remove` 等）；
  - 是否 `no_std`，是否自动实现常见 trait（`Copy` / `Clone` / `BitOr` 等）。
- 在内核场景下，你可以**直接用它来描述寄存器字段/页表 flag**：

```rust
bitflags::bitflags! {
    pub struct PageTableFlags: u64 {
        const VALID   = 1 << 0;
        const READ    = 1 << 1;
        const WRITE   = 1 << 2;
        const EXECUTE = 1 << 3;
        const USER    = 1 << 4;
        // ...
    }
}
```

结合 docs.rs：

- 先看 crate 文档里的“Examples”部分，照抄一段本地跑起来；
- 再看 `struct` 文档里列出的所有方法名，挑出和你场景相关的，例如：
  - `flags.contains(PageTableFlags::VALID | PageTableFlags::READ)`；
  - `flags.insert(PageTableFlags::WRITE)`；
  - `flags.bits()` / `from_bits_truncate()`，用来和底层寄存器值互转。

通过这种方式，你几乎可以 **不看源码，只靠 docs 和少量实验代码**，就学会用 `bitflags` 给内核寄存器/页表做类型安全封装。

#### 4.6.2 例子：用 `log` 统一日志接口（即便在 `no_std` 环境）

`log` crate 的定位是**“日志接口标准”**：

- `log` 自己不负责真正的输出，只定义宏和 trait；
- 具体输出行为由“后端”库（logger 实现）决定，比如在内核里你可以实现一个把日志写到串口/缓冲区的 logger。

在 docs.rs 的 crate 文档里，你会看到两个关键部分：

- 给普通调用方的宏：`log!` / `info!` / `warn!` / `error!` / `debug!` / `trace!`；
- 给 logger 实现者的 trait：`Log`，以及 `set_logger` / `set_max_level` 等函数。

一个典型的“只看 docs 也能写出来”的使用方式：

```toml
[dependencies]
log = { version = "0.4", default-features = false }
```

```rust
use log::{info, LevelFilter, Log, Metadata, Record};

struct SimpleLogger;

impl Log for SimpleLogger {
    fn enabled(&self, metadata: &Metadata) -> bool {
        metadata.level() <= LevelFilter::Info
    }

    fn log(&self, record: &Record) {
        if self.enabled(record.metadata()) {
            // 这里可以写到串口、内存缓冲区等
            // 比如 uart_write(format!("{} - {}", record.level(), record.args()));
        }
    }

    fn flush(&self) {}
}

static LOGGER: SimpleLogger = SimpleLogger;

pub fn init_logging() {
    log::set_logger(&LOGGER).unwrap();
    log::set_max_level(LevelFilter::Info);
}
```

使用方只需要在任意地方写：

```rust
info!("hello from kernel, pid = {}", pid);
```

理解 `log` 的方法基本都是从 docs.rs 来的：

- 看 crate 文档顶端的“使用示例”，那里通常就给出了最小实现；
- 打开 `trait Log` 的文档，看每个方法的语义；
- 看宏 `info!` / `error!` 的文档，理解它们会在编译期开关哪些日志等级。

对内核项目来说，这样做的好处是：

- 内核内部所有模块只依赖 `log` 这个统一接口，不直接依赖串口/UART 细节；
- 以后换输出后端（比如从 UART 换成“环形缓冲区 + 用户态读日志”）时，只改 logger 实现，不动调用方。

---

## 5. 内核项目中的典型封装模式示例

本小节列几个在 OS 内核里很常见的封装套路，阅读/写代码时可以刻意感受。

### 5.1 “寄存器访问”封装：从裸指针到安全接口

目标：对外是“读/写某个寄存器字段”的安全函数；对内是 `volatile` 读写和 bit 操作。

典型结构：

```rust
#[repr(C)]
struct UartRegisters {
    dr: u32,   // data register
    // ...
}

pub struct Uart {
    base: *mut UartRegisters,
}

impl Uart {
    /// 对外是 safe API
    pub fn write_byte(&self, byte: u8) {
        unsafe {
            // volatile write，保证不会被优化掉
            core::ptr::write_volatile(&mut (*self.base).dr, byte as u32);
        }
    }
}
```

理解思路：

- 找到 `pub struct Uart` —— 这是对外暴露的设备抽象；
- 看 `impl Uart` 里哪些方法是 `pub` —— 这是“能做的事”；
- 再看 `unsafe` 细节 —— 想想作者如何把它包成安全 API。

### 5.2 “锁”封装：RAII guard + Drop

常见模式：

```rust
pub struct Mutex<T> {
    // 内部是自旋锁/禁中断/原子变量等
    inner: RawSpinLock,
    data: UnsafeCell<T>,
}

pub struct MutexGuard<'a, T> {
    lock: &'a Mutex<T>,
}

impl<T> Mutex<T> {
    pub fn lock(&self) -> MutexGuard<'_, T> {
        self.inner.lock();    // 可能是 unsafe
        MutexGuard { lock: self }
    }
}

impl<'a, T> Deref for MutexGuard<'a, T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        unsafe { &*self.lock.data.get() }
    }
}

impl<'a, T> DerefMut for MutexGuard<'a, T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        unsafe { &mut *self.lock.data.get() }
    }
}

impl<'a, T> Drop for MutexGuard<'a, T> {
    fn drop(&mut self) {
        self.lock.inner.unlock();
    }
}
```

理解思路：

- `Mutex` 自己不直接暴露 `&mut T`，而是通过 `MutexGuard`；
- `MutexGuard` 实现 `Drop`，自动在离开作用域时释放锁；
- 这是典型的 **“用类型系统表达生命周期和所有权”的封装**。

### 5.3 “配置结构体 + builder” 封装

在复杂初始化场景（如启动参数、内存布局配置）中，经常会看到：

```rust
pub struct KernelConfig {
    pub log_level: LogLevel,
    pub heap_size: usize,
    // ...
}

pub struct KernelConfigBuilder {
    cfg: KernelConfig,
}

impl KernelConfigBuilder {
    pub fn new() -> Self {
        Self {
            cfg: KernelConfig {
                log_level: LogLevel::Info,
                heap_size: 4 * 1024 * 1024,
                // 默认值...
            },
        }
    }

    pub fn log_level(mut self, level: LogLevel) -> Self {
        self.cfg.log_level = level;
        self
    }

    pub fn build(self) -> KernelConfig {
        self.cfg
    }
}
```

优点：

- 调用方用链式调用配置选项，可读性强；
- 默认值集中在 builder 里，后续新增字段时不容易漏填。

---

## 7. 本讲小结

| 主题             | 要点                                                                                                                   |
| -------------- | -------------------------------------------------------------------------------------------------------------------- |
| **封装的目标**      | 明确边界、隐藏细节、保证不变式，让大多数人只和“安全 API 层”打交道。                                                                                |
| **Rust 的封装工具** | 模块与可见性（`mod`/`pub`/`pub(crate)`）、私有字段 + `impl`、trait 作为接口、newtype 模式、`Result`/`Option` 作为错误表达、用 safe API 包一层 unsafe。 |
| **阅读封装的套路**    | 从 `Cargo.toml` 和 `lib.rs` 看分层；优先看 `pub` API 和文档；顺着 trait/newtype 找实现；结合调用点理解使用方式；特别留意 `unsafe` 外层的约束。                |
| **内核中的典型模式**   | 寄存器访问封装、锁与 RAII guard、配置结构体与 builder、地址/ID 等用 newtype 表达语义。                                                          |

掌握这些后，在后续几讲（unsafe、宏、配置文件、build.rs 等）里看到大量封装时，会更容易**看懂别人的意图**，也更有能力为内核项目设计出清晰、稳健的模块边界。

