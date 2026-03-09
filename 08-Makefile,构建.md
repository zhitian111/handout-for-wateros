# Makefile 与构建：依赖图、脚本集与一键构建

> 本章目标：让你**真正理解 Make/Makefile 的本质**——它是一个维护「构建产物依赖图」的工具，也是一个**打包好的脚本集**；它自己**不懂如何编译内核**，只负责在「哪些文件变了」的前提下，**决定该执行哪几条你写好的命令**。学完后，你应该能在现有构建流程上，自信地**加自己的构建目标和小工具**。

---

## 1. 为什么内核项目需要 Makefile？

在写普通 Rust 程序时，`cargo build` 往往已经够用。但在做内核开发时，构建流程一般是这样的：

1. 用 `rustc` / `cargo` **编译**出目标架构的 ELF 可执行文件或静态库；
2. 用链接器、`objcopy` 等工具 **重链接 / 转换格式**，得到裸机可引导镜像；
3. 把镜像交给 **QEMU / 真机 / 仿真板** 去启动；
4. 附加步骤：生成调试符号文件、打包测试程序、生成文档、清理产物……  

如果把这些步骤都记在脑子里、每次手敲命令：

- 容易打错、顺序弄反；
- 每次全量重来，浪费时间；
- 队友之间流程不统一，很难协作。

我们需要一个东西来做两件事：

- **记住「要生成哪些构建产物」以及它们之间的依赖关系**；
- **在需要重建时，帮我们自动跑好那几条命令**。

这就是 **Make + Makefile** 要干的事。

---

## 2. Make 的本质：依赖图 + shell 命令 + 打包好的脚本集

从“原理”的角度看，Make/Makefile 可以抽象成三部分：

- **依赖图（DAG）**：  
  - 每个「目标文件」（target）是图中的一个节点；
  - 每个目标列出它的「前置条件/依赖」（prerequisites）；
  - 这构成了一张「谁依赖谁」的有向无环图。
- **规则（recipe）**：  
  - 对于每个目标，写上一组命令：**「当依赖更新了，需要重建这个目标时，要怎么做」**；
  - 这些命令本质上就是你平时在 shell 里敲的那些命令。
- **调度器（Make 程序本身）**：  
  - 它比较目标和依赖的**时间戳**，判断哪些目标「过期」了；
  - 然后按依赖关系排序，决定命令的执行顺序。

注意几个关键点：

- **Make 完全不理解你的编译“语义”**  
  它不知道什么叫“编译 Rust 程序是对还是错”，也不知道“链接参数该怎么写才合理”。  
  它只知道：  
  > 「当我需要重建 `kernel.bin` 时，我应该执行你写在 Makefile 里的这几行命令。」

- **Make 本质就是一个「打包好的脚本集」**  
  你可以理解为：  
  - 把一堆平时要手敲的 shell 命令，写在一个文件里，给它取名字（如 `all` / `run` / `debug`）；  
  - 将这些「脚本」之间的依赖关系描述清楚；  
  - 以后只要打一行 `make run`，Make 就会**按依赖顺序帮你自动执行这一整套脚本**。

换句话说：

> **Make 不关心“构建过程”，只关心“构建产物”和“它们之间的依赖关系”。  
> 构建过程本身，就是你写在规则里的那堆 shell 命令。**

理解了这一点之后，你就能在现有流程上自信地扩展：  
**只要你能在命令行里敲对一串命令，就一定能把它“打包”成一个新的 Make 目标。**

---

## 3. 最小可用示例：从零写一个小 Makefile

先看一个极简的例子（为了专注于原理，这里用 C 演示，换成 Rust 只是命令不同而已）：

```make
app: app.c app.h
	@gcc -o app app.c
app.c: app.md
	
```
说明：

- `app`：**目标文件**（target），我们想要生成的可执行文件；
- `app.c`：**依赖**（prerequisite），也就是生成 `app` 所需要的源文件；
- 下一行开始，以 Tab 开头的是 **规则中的命令（recipe）**，表示：
  > 如果 `app` 需要重建，就执行 `gcc -o app app.c`。

使用方式：

- 第一次运行：`make app`  
  - 若当前目录下没有 `app`，Make 认为目标「不存在」，需要重建；
  - 它会执行那条 `gcc` 命令，生成 `app`。
- 修改 `app.c` 之后，再次运行 `make app`：  
  - Make 会比较 `app.c` 和 `app` 的时间戳：  
    - 如果 `app.c` 比 `app` 新，表示源文件改过，目标过期了，需要重建；  
    - 它会再执行一次 `gcc` 命令。
- 如果你没有修改 `app.c`，又敲了一次 `make app`：  
  - Make 会发现 `app` 比 `app.c` 新（已经是最新版本），  
  - 什么也不做，直接输出 `make: 'app' is up to date.`。

这个极简案例已经展示了 Make 的核心思想：

1. 你**用 Makefile 描述依赖关系和构建命令**；
2. Make **自动判断哪些目标需要重建**，再去执行对应命令；
3. 它不会“聪明”到替你分析代码，只看**文件变化**。

---

## 4. Makefile 语法速览：变量、规则和常用技巧

上面的例子只是展示了 Make 的“工作模式”。这一节我们把 **常用语法** 过一遍，后面你看到项目里的 Makefile 就不会懵。

### 4.1 规则的基本结构

一条最常见的规则长这样：

```make
target: prerequisites
<Tab>command1
<Tab>command2
```

- `target`：目标，可以是文件名，也可以是逻辑目标（比如 `run`、`clean`）；  
- `prerequisites`：依赖文件，多个时用空格分隔；  
- 每一行命令都必须以 **Tab** 开头；  
- 多行命令会按顺序执行。

注意一个容易踩坑的点：**每一行命令默认在“单独的 shell”里执行**。比如：

```make
wrong:
	cd build
	cargo build
```

`cd build` 生效的只是在那一行的 shell，下一行又回到了原来的目录。正确写法通常是：

```make
ok:
	cd build && cargo build
```

或者直接在命令里写路径：

```make
ok:
	cargo build --target-dir build
```

很多诡异的“make 里 cd 不生效”的 bug，都是因为忘了这一点。

### 4.2 变量：抽象出“可配置的部分”

Makefile 里的变量类似于“宏替换”：

```make
CC      := riscv64-unknown-elf-gcc
CFLAGS  := -O2 -Wall

app: app.c
	$(CC) $(CFLAGS) -o app app.c
```

- 定义变量：`NAME := value` 或 `NAME = value`（见下文区别）；  
- 使用变量：`$(NAME)`。

**变量的几种赋值方式：**

| 写法           | 含义说明                                        |
| ------------ | ------------------------------------------- |
| `VAR := val` | **立即展开**：右边在定义时就算出结果，推荐日常使用。                |
| `VAR = val`  | **延迟展开**：右边在每次被引用时才求值，可能用到后面才定义的变量。         |
| `VAR ?= val` | **条件赋值**：仅当 `VAR` 尚未定义时才赋值为 `val`（适合提供默认值）。 |
| `VAR += val` | **追加**：在原有值后面追加 `val`，常用于收集多个来源的选项。         |

示例：

```make
CC      := gcc
CFLAGS  := -Wall -O2
CFLAGS  += -DDEBUG          # 在已有 CFLAGS 上追加 -DDEBUG
CROSS   ?= riscv64-unknown-elf-   # 若未在命令行或环境中设置，才用这个默认值
```

- **立即 vs 延迟**：初学用 `:=` 即可；`=` 在复杂 Makefile 里可能产生“变量展开时机”的坑，需要时再细究。
- 可通过命令行覆盖变量，例如：`make CC=riscv64-linux-gnu-gcc`，在内核/交叉编译里很常见。
- 若希望**不被命令行覆盖**，可写 `override VAR := value`。

### 4.3 常用自动变量：像“参数”一样复用规则

为了避免在命令里不停重复文件名，Make 提供了几个 **自动变量**，最常用的是：

- `$@`：当前规则的 **目标名**（target）；  
- `$<`：当前规则的 **第一个依赖**；  
- `$^`：当前规则的 **所有依赖（去重后）**。

例子：

```make
app: app.c
	$(CC) $(CFLAGS) -o $@ $<
```

等价于前面的 `-o app app.c`，但当你把 `app` 改成别的名字时，就不需要改命令里的字符串。

在更复杂的内核 Makefile 里，这些自动变量会大量出现，一定要认识。

其他常用自动变量（供查阅）：`$*`（匹配 stem）、`$?`（比目标新的依赖）、`$|`（order-only 依赖）。

**多行变量与 `define`**：需要把多行内容赋给一个变量时，可用 `define`：

```make
define RUN_QEMU
	$(QEMU) -kernel $(KERNEL_BIN) -nographic
	@echo "QEMU exited."
endef

run:
	$(RUN_QEMU)
```

用 `define` 定义的是“命令列表”，引用时用 `$(RUN_QEMU)`，会原样展开为多行命令。若希望当作**单行**使用，可写 `define VAR ... endef` 后配合 `$(call ...)` 等。

### 4.4 条件判断：根据情况选择不同定义或规则

Makefile 支持简单的条件判断，在**解析阶段**生效（不是运行时），用来在不同平台、不同配置下选不同的变量或规则。

常用形式：

```make
# 相等 / 不相等（注意括号与空格）
ifeq ($(ARCH),riscv64)
    CC := riscv64-unknown-elf-gcc
else
    CC := gcc
endif

# 是否定义（常用于可选参数）
ifdef DEBUG
    CFLAGS += -O0 -g
else
    CFLAGS += -O2
endif

# 未定义时
ifndef INSTALL_PREFIX
    INSTALL_PREFIX := /usr/local
endif
```

- **ifeq / ifneq**：比较两个值是否相等（字符串），注意 `ifeq (a,b)` 里逗号后的空格也算进右边。
- **ifdef / ifndef**：判断变量是否“已定义”（非空即视为已定义）。
- 条件块必须写在同一 Makefile 里，且 **else / endif 不能丢**。

典型用法：根据 `TARGET` 或环境选择工具链、根据 `V`（verbose）决定是否回显命令等。

### 4.5 循环与批量处理：`foreach` 与 `shell` 的配合

Makefile 里没有“循环规则”的语法，但可以用**函数**做批量处理：

**1. `$(foreach var, list, text)`**  
对列表中的每个元素，展开一次 `text`（其中用 `$(var)` 表示当前元素），结果再拼成一段文本。常用于生成一列文件名或参数：

```make
SRCS := main.c util.c foo.c
# 把 .c 换成 .o：可用替换语法 $(VAR:.c=.o)，或 foreach
OBJS := $(SRCS:.c=.o)              # 得到 main.o util.o foo.o
OBJS := $(foreach s,$(SRCS),$(s:.c=.o))   # 等价写法

# 更典型的 foreach：给每个文件加路径前缀
BUILD_OBJS := $(foreach f,main util,$(BUILD_DIR)/$f.o)
```

常用文本函数：`$(VAR:a=b)` 表示把 `VAR` 中所有以 `a` 结尾的词替换成 `b`（如 `$(SRCS:.c=.o)`）；`$(addprefix pre,$(LIST))` 给列表中每项加前缀；`$(wildcard pattern)` 做文件名通配。

**2. `$(call func, arg1, arg2, ...)`：调用“用户自定义函数”**  
用 `define` 可以定义带参数的“模板”，再用 `call` 传入实参并展开。在模板里用 `$(1)`、`$(2)`、`$(3)` … 表示第 1、2、3 … 个参数：

```make
# 定义“函数”：根据名字生成对应的 .o 路径
define obj_from
$(BUILD_DIR)/$(1).o
endef

# 调用：$(call obj_from,main) 展开为 $(BUILD_DIR)/main.o
OBJS := $(call obj_from,main) $(call obj_from,util)

# 更实用：和 foreach 结合，批量生成
NAMES := main util foo
OBJS  := $(foreach n,$(NAMES),$(call obj_from,$n))
# 得到 $(BUILD_DIR)/main.o $(BUILD_DIR)/util.o $(BUILD_DIR)/foo.o
```

再例如，封装“生成一条编译命令”的模板（多行时用 `define`）：

```make
define compile_c
	$(CC) $(CFLAGS) -c $(1) -o $(2)
endef

%.o: %.c
	$(call compile_c,$<,$@)
```

- **注意**：`call` 的实参之间用逗号分隔；若实参本身含逗号，需先赋给变量，再 `$(call func,$(VAR))` 传入。
- 和 `define` 搭配后，可以实现“一段可复用的带参模板”，避免在规则里重复写相同命令。

**3. 用 `$(shell ...)` 调用 shell 做循环**  
若需要“跑一段 shell 循环”，可以写在 `$(shell ...)` 里，结果会作为变量内容：

```make
# 列出某目录下所有 .c 文件（示例）
C_SRCS := $(shell find src -name '*.c')
```

复杂循环逻辑（如按行处理）一般放在 shell 脚本里，在 Makefile 里用 `$(shell ./script.sh)` 或单独一条规则调用脚本。

**4. “循环”效果通过规则依赖实现**  
最常见的“批量”是：多个目标用同一套模式规则（如 `%.o: %.c`），Make 会根据依赖关系自动对每个 `.c` 应用规则，无需手写 foreach 遍历——见下一节模式规则。

### 4.6 模式规则：批量描述“一类文件怎么生成”

当有很多类似的文件需要用同一种方式生成时，可以用 **模式规则（pattern rule）**：

```make
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@
```

含义是：

- 对于任何一个 `xxx.o`，如果存在同名的 `xxx.c`，  
- 那么要生成 `xxx.o`，就执行下面那条命令。

在 Rust/内核场景下，也可以用类似思路，比如：

```make
build/%.bin: src/%.rs
	cargo build --bin $* ...
```

（示意用，具体写法要结合你项目的结构来定。）  
模式规则的核心就是：**用一次规则，描述一大类文件的生成方式**。

### 4.7 伪目标 .PHONY：把“纯脚本”目标标记出来

像 `run`、`clean` 这类目标其实并不对应真实文件，我们一般会把它们声明为 **伪目标**：

```make
.PHONY: all run clean
```

这样做的好处是：

- 即使当前目录下刚好有一个叫 `run` 的文件，`make run` 也会执行规则里的命令；  
- 不会受时间戳干扰，始终会按你写的方式执行。

### 4.8 小技巧：`@`、`-`、续行和注释

- 命令前加 `@`：不回显命令本身，只显示命令输出（适合防止日志太吵）：

  ```make
  run:
  	@echo "Running kernel..."
  	@$(QEMU) ...
  ```

- 命令前加 `-`：即使命令出错也继续执行后续命令（谨慎使用，一般用于清理）：

  ```make
  clean:
  	-rm -f $(KERNEL_BIN)
  ```

- 行尾 `\`：在逻辑上把一行命令写成多行，便于阅读：

  ```make
  run:
  	@$(QEMU) \
  		-kernel $(KERNEL_BIN) \
  		-nographic
  ```

- `#`：注释，从 `#` 开始到行尾都是注释（但注意不要出现在命令需要的参数里）。

掌握了这些语法，你已经具备了“把日常脚本封装成 Makefile 语言”的基本能力。

---

## 5. 内核项目中的典型 Makefile 结构

在我们的内核项目里，Makefile 通常会包含这些元素：

- **工具相关的变量**：  
  - `CARGO`：用哪个 Cargo（有时需要指定路径）；  
  - `RUSTC`：用哪个 Rust 编译器（有需要时）；  
  - `OBJCOPY` / `OBJDUMP` / `NM`：来自交叉工具链的工具；  
  - `QEMU`：使用的 QEMU 命令，如 `qemu-system-riscv64`。

- **路径与产物名字**：  
  - `TARGET`：目标架构三元组，如 `riscv64imac-unknown-none-elf`；  
  - `BUILD_DIR`：编译产物目录；  
  - `KERNEL_ELF` / `KERNEL_BIN`：中间 ELF 与最终内核镜像的路径。

- **常见伪目标（phony targets）**：  
  - `all`：构建所有主要目标，一般会依赖 `kernel` 或 `run`；  
  - `kernel`：只构建内核镜像，不跑；  
  - `run`：在 `kernel` 的基础上调用 QEMU，跑起来；  
  - `debug`：以便于 GDB 连接的方式启动 QEMU；  
  - `clean`：删除构建产物。

示意（伪代码，非完整 Makefile）：

```make
TARGET      := riscv64imac-unknown-none-elf
KERNEL_ELF  := target/$(TARGET)/debug/kernel
KERNEL_BIN  := target/$(TARGET)/debug/kernel.bin
QEMU        := qemu-system-riscv64

.PHONY: all kernel run clean

all: run

kernel: $(KERNEL_BIN)

$(KERNEL_BIN): $(KERNEL_ELF)
	@rust-objcopy --strip-all -O binary $(KERNEL_ELF) $(KERNEL_BIN)

$(KERNEL_ELF):
	@cargo build --target $(TARGET)

run: $(KERNEL_BIN)
	@$(QEMU) -kernel $(KERNEL_BIN) -nographic ...

clean:
	@cargo clean
	@rm -f $(KERNEL_BIN)
```

你可以把上面的每个目标都看作是一个「打包好的脚本」：

- `make kernel`：= 先 `cargo build`，再 `objcopy`；  
- `make run`：= 如果需要就先 `make kernel`，然后再跑 QEMU；  
- `make clean`：= 执行一组清理命令。

---

## 6. 与 Cargo、工具链的协作：Make 只是“外层总控”

在 Rust 内核项目中，**Cargo 仍然是负责编译 Rust 代码的主角**：

- 依赖管理：`Cargo.toml` 和 crates.io；
- 构建配置：profile、features、目标架构；
- 编译步骤：`cargo build` / `cargo test` 等。

而 Make 的角色，是把这些**本来就可以在命令行直接敲的 Cargo 命令**，再加上一些额外工具（`objcopy`、`qemu-system-...`、打包脚本），统筹成一套：

- 一键构建：`make` / `make all`；
- 一键运行：`make run`；
- 一键调试：`make debug`；
- 一键清理：`make clean`。

你可以这样理解两者关系：

- **Cargo 管理的是“Rust 代码 → 某个 ELF/库”的构建细节**；  
- **Make 管理的是“多个工具之间的调用顺序 + 产物之间的依赖关系”**。

所以，只要你先学会在命令行里手动敲：

- `cargo build --target ...`  
- `rust-objcopy ...`  
- `qemu-system-... -kernel ...`  

那就可以很自然地把它们“打包”进 Makefile 里，让 Make 来帮你记住和调度这些命令。

---

## 7. 可扩展思路：在现有流程上加自己的规则

理解了前面的原理，你就可以开始做一件对团队非常有价值的事：  
**在现有 Makefile 基础上，扩展出适合自己和团队的新构建目标。**

典型的扩展方向包括：

- **加一个只做自己开发用的运行模式**  
  - 例如：`make run-log`，在原有 `run` 命令基础上，多加日志输出到文件的参数；  
  - 或者 `make run-fast`，关闭某些调试选项、用更快的 QEMU 参数。

- **加一个调试辅助目标**  
  - 例如：`make debug`  
    - 依赖 `kernel`；  
    - 用 `-S -s` 方式启动 QEMU；  
    - 在终端输出“现在可以用 GDB 连接到 :1234”之类的提示。

- **加一个检查/格式化目标**  
  - 例如：`make fmt` 调用 `cargo fmt`；  
  - `make clippy` 调用 `cargo clippy`，统一团队代码风格和静态检查。

在设计这些目标时，可以遵循一个简单的思路：

1. **先在命令行里把这组命令跑通**；  
2. 把这组命令复制进 Makefile，给它取一个有意义的名字；  
3. 想一想它依赖哪些已有目标（比如先要 `kernel`），把依赖关系也写进去；
4. 尝试通过 `make xxx` 来触发这组命令，看效果是否一致。

最后再次强调本章想传达的核心观念：

- **Makefile 是「依赖图 + 一组打包好的脚本集」**；  
- Make 不懂编译语言的语义，只看“哪些文件变了”；  
- 你日常在终端里敲的任何一串构建/运行/调试命令，**都可以变成一个新的 Make 目标**；
- 只要按照「目标 → 依赖 → 命令」的形式写清楚，Make 就能帮你自动调度这些脚本。

带着这个视角，你在阅读项目中的 Makefile 时，就不再害怕——  
看到的每一块内容，都可以直接翻译成一句话：  
> 「当我需要得到某个产物时，Make 会帮我按顺序跑哪几条命令？」
