# Rust 基础快速入门

> 基于 Rust 语言圣经整理，适合课堂讲述的详细版本

---

## 1. 为什么操作系统内核开发要使用 Rust？

### 1.1 传统系统编程语言的痛点

在操作系统内核开发中，传统上主要使用 C 和 C++ 语言，但这些语言存在以下问题：

1. **内存安全问题**
   - 悬空指针（Dangling Pointer）
   - 缓冲区溢出（Buffer Overflow）
   - 使用已释放的内存（Use After Free）
   - 双重释放（Double Free）
   - 内存泄漏（Memory Leak）

2. **数据竞争问题**
   - 多线程环境下的数据竞争（Data Race）
   - 难以在编译期检测并发问题

3. **缺乏现代语言特性**
   - 类型系统不够强大
   - 缺乏包管理和构建工具
   - 错误处理机制不够优雅

### 1.2 Rust 在操作系统开发中的优势

#### 1.2.1 内存安全保证

Rust 通过**所有权系统**在编译期就保证了内存安全，无需运行时检查：

- **编译期检查**：所有内存安全问题在编译时就被发现
- **零成本抽象**：编译期检查不会带来运行时性能损失
- **无需垃圾回收**：通过所有权系统自动管理内存，无需 GC

```rust
// Rust 编译器会阻止这样的错误代码
fn main() {
    let s = String::from("hello");
    let s2 = s;  // 所有权转移
    println!("{}", s);  // 编译错误！s 已经不再有效
}
```

#### 1.2.2 并发安全

Rust 的所有权系统天然防止数据竞争：

- **编译期检测数据竞争**：借用检查器确保不会出现数据竞争
- **无需锁的并发**：通过类型系统保证线程安全
- **零成本并发抽象**：并发原语在编译期优化

```rust
use std::thread;

fn main() {
    let mut data = vec![1, 2, 3];
    
    // Rust 编译器会阻止这样的数据竞争
    // thread::spawn(|| {
    //     data.push(4);  // 编译错误！无法在多个线程间共享可变数据
    // });
}
```

#### 1.2.3 性能优势

- **零成本抽象**：高级特性不会带来运行时开销
- **无垃圾回收**：没有 GC 暂停，适合实时系统
- **接近 C/C++ 的性能**：编译后的代码性能与 C/C++ 相当
- **栈优先分配**：默认在栈上分配，减少堆分配开销

#### 1.2.4 现代工具链

- **Cargo**：强大的包管理和构建工具
- **rustc**：友好的编译器错误信息
- **rustfmt**：统一的代码格式化工具
- **clippy**：代码检查和建议工具

### 1.3 实际应用案例

- **Redox OS**：完全用 Rust 编写的操作系统
- **Linux 内核模块**：Linux 6.1+ 支持 Rust 编写的内核模块
- **Google Fuchsia**：部分使用 Rust 开发
- **Microsoft**：在 Windows 内核中探索使用 Rust

---

## 2. Rust 的显著优点和特点

### 2.1 核心设计理念

Rust 的设计目标是：**既要安全性，又要灵活性和性能**。

#### 2.1.1 内存安全

- **编译期内存管理**：通过所有权系统在编译期检查内存安全
- **无悬空指针**：编译器保证引用始终有效
- **无数据竞争**：编译期检测并发问题

#### 2.1.2 零成本抽象

- **高级特性，零运行时开销**：泛型、Trait、模式匹配等特性在编译期优化
- **与 C/C++ 相当的性能**：编译后的代码性能接近底层语言

#### 2.1.3 表达力强

- **模式匹配**：强大的模式匹配系统
- **类型推导**：编译器自动推导类型，减少样板代码
- **函数式编程特性**：闭包、迭代器等

#### 2.1.4 友好的开发体验

- **优秀的错误信息**：编译器提供详细的错误提示和建议
- **完善的工具链**：Cargo、rustfmt、clippy 等工具
- **活跃的社区**：丰富的文档和第三方库

### 2.2 Rust 的三大支柱

1. **所有权系统（Ownership）**
   - 每个值都有一个所有者
   - 值在所有者离开作用域时被释放
   - 通过所有权转移和借用管理内存

2. **类型系统（Type System）**
   - 静态类型检查
   - 强大的类型推导
   - 泛型和 Trait 支持

3. **借用检查器（Borrow Checker）**
   - 编译期检查引用有效性
   - 防止数据竞争
   - 确保内存安全

---

## 3. Cargo 项目管理工具

### 3.1 Cargo 简介

Cargo 是 Rust 的官方包管理和构建工具，类似于 Node.js 的 npm、Python 的 pip。

### 3.2 Cargo 的主要操作

#### 3.2.1 创建新项目

```bash
# 创建二进制项目
cargo new my_project

# 创建库项目
cargo new --lib my_lib

# 在现有目录初始化项目
cargo init
```

**创建的项目结构：**
```
my_project/
├── Cargo.toml      # 项目配置文件
├── src/
│   └── main.rs     # 主程序入口
└── .gitignore      # Git 忽略文件
```

#### 3.2.2 编译项目

```bash
# 编译项目（调试模式）
cargo build

# 编译项目（发布模式，优化）
cargo build --release

# 快速检查代码是否能编译（不生成二进制文件）
cargo check
```

**编译输出：**
- 调试模式：`target/debug/my_project`
- 发布模式：`target/release/my_project`

#### 3.2.3 运行项目

```bash
# 编译并运行（调试模式）
cargo run

# 编译并运行（发布模式）
cargo run --release

# 传递命令行参数
cargo run -- arg1 arg2
```

#### 3.2.4 测试

```bash
# 运行所有测试
cargo test

# 运行特定测试
cargo test test_name

# 运行测试并显示输出
cargo test -- --nocapture
```

#### 3.2.5 依赖管理

```bash
# 添加依赖（需要 cargo-edit 插件）
cargo add serde

# 添加开发依赖
cargo add --dev tokio-test

# 更新依赖
cargo update

# 查看依赖树
cargo tree
```

#### 3.2.6 代码质量工具

```bash
# 代码格式化
cargo fmt

# 代码检查和建议
cargo clippy

# 生成文档
cargo doc

# 在浏览器中打开文档
cargo doc --open
```

#### 3.2.7 其他常用命令

```bash
# 清理编译产物
cargo clean

# 查看项目信息
cargo metadata

# 安装二进制工具
cargo install tool_name

# 查看 Cargo 版本
cargo --version
```

### 3.3 Cargo.toml 配置文件

`Cargo.toml` 是项目的配置文件，定义了项目的基本信息和依赖：

```toml
[package]
name = "my_project"
version = "0.1.0"
edition = "2021"  # Rust 版本：2015, 2018, 2021

[dependencies]
serde = "1.0"
tokio = { version = "1.0", features = ["full"] }

[dev-dependencies]
tokio-test = "0.4"

[profile.release]
opt-level = 3      # 优化级别
lto = true         # 链接时优化
```

### 3.4 Cargo 对项目的作用

1. **统一的项目结构**
   - 标准化的目录布局
   - 便于团队协作

2. **依赖管理**
   - 自动下载和编译依赖
   - 版本锁定（Cargo.lock）
   - 依赖冲突检测

3. **构建系统**
   - 增量编译
   - 并行编译
   - 条件编译（features）

4. **开发工具集成**
   - 测试框架
   - 文档生成
   - 代码格式化

5. **跨平台支持**
   - 自动处理平台差异
   - 交叉编译支持

---

## 4. 变量绑定与解构

### 4.1 变量绑定

在 Rust 中，我们使用 `let` 关键字进行**变量绑定**，而不是"赋值"。

```rust
fn main() {
    let x = 5;  // 将值 5 绑定给变量 x
    println!("x = {}", x);
}
```

#### 4.1.1 变量默认不可变

Rust 中的变量默认是**不可变的**（immutable），这是 Rust 安全性的重要保证：

```rust
fn main() {
    let x = 5;
    x = 6;  // 编译错误！不能修改不可变变量
}
```

**为什么默认不可变？**
- **安全性**：避免意外的变量修改
- **性能**：编译器可以进行更多优化
- **并发安全**：不可变数据天然线程安全

#### 4.1.2 可变变量

如果需要修改变量，使用 `mut` 关键字：

```rust
fn main() {
    let mut x = 5;
    x = 6;  // 可以修改
    println!("x = {}", x);
}
```

**使用 `mut` 的原则：**
- 只在需要修改时才使用 `mut`
- 显式声明可变性，提高代码可读性

#### 4.1.3 变量遮蔽（Shadowing）

Rust 允许使用相同的名字重新声明变量，这叫做"遮蔽"：

```rust
fn main() {
    let x = 5;
    let x = x + 1;  // 遮蔽，创建新的变量
    let x = x * 2;  // 再次遮蔽
    println!("x = {}", x);  // 输出：12
    
    // 遮蔽可以改变类型
    let spaces = "   ";
    let spaces = spaces.len();  // 从字符串变为数字
}
```

**遮蔽 vs 可变变量：**
- 遮蔽：创建新变量，可以改变类型
- `mut`：修改现有变量，不能改变类型

### 4.2 变量解构

Rust 支持从复杂数据结构中解构出值：

#### 4.2.1 元组解构

```rust
fn main() {
    let tup = (500, 6.4, 1);
    
    // 解构元组
    let (x, y, z) = tup;
    println!("x = {}, y = {}, z = {}", x, y, z);
}
```

#### 4.2.2 结构体解构

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 0, y: 7 };
    
    // 解构结构体
    let Point { x, y } = p;
    println!("x = {}, y = {}", x, y);
    
    // 部分解构
    let Point { x, .. } = p;
    println!("x = {}", x);
}
```

#### 4.2.3 枚举解构

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
}

fn main() {
    let msg = Message::Move { x: 3, y: 4 };
    
    match msg {
        Message::Move { x, y } => {
            println!("移动到 ({}, {})", x, y);
        }
        _ => {}
    }
}
```

### 4.3 未使用变量

Rust 编译器会警告未使用的变量，可以使用下划线前缀忽略：

```rust
fn main() {
    let _x = 5;  // 下划线前缀，忽略未使用警告
    let y = 10;  // 会有警告
}
```

---

## 5. 基本类型

Rust 是**静态类型语言**，这意味着编译器必须在编译时知道所有变量的类型。

### 5.1 数值类型

#### 5.1.1 整数类型

Rust 有多种整数类型，根据大小和是否有符号分类：

| 长度 | 有符号 | 无符号 |
|------|--------|--------|
| 8位  | i8     | u8     |
| 16位 | i16    | u16    |
| 32位 | i32    | u32    |
| 64位 | i64    | u64    |
| 128位| i128   | u128   |
| 架构 | isize  | usize  |

**示例：**
```rust
fn main() {
    let x: i32 = 42;        // 32位有符号整数
    let y: u64 = 100;       // 64位无符号整数
    let z: isize = -10;     // 架构相关大小（32位或64位）
}
```

**整数字面量：**
```rust
fn main() {
    let decimal = 98_222;        // 十进制：98222
    let hex = 0xff;              // 十六进制：255
    let octal = 0o77;            // 八进制：63
    let binary = 0b1111_0000;    // 二进制：240
    let byte = b'A';             // 字节（仅限 u8）：65
}
```

#### 5.1.2 浮点数类型

Rust 有两种浮点数类型：

```rust
fn main() {
    let x: f32 = 3.14;  // 32位浮点数（单精度）
    let y: f64 = 2.718; // 64位浮点数（双精度，默认）
    
    let z = 3.0;  // 默认是 f64
}
```

**浮点数运算：**
```rust
fn main() {
    let sum = 5 + 10;           // 整数加法
    let difference = 95.5 - 4.3; // 浮点数减法
    let product = 4 * 30;       // 整数乘法
    let quotient = 56.7 / 32.2; // 浮点数除法
    let remainder = 43 % 5;     // 整数取余
}
```

#### 5.1.3 数值运算

```rust
fn main() {
    // 加法
    let sum = 5 + 10;
    
    // 减法
    let difference = 95.5 - 4.3;
    
    // 乘法
    let product = 4 * 30;
    
    // 除法
    let quotient = 56.7 / 32.2;
    let floored = 2 / 3;  // 整数除法，结果为 0
    
    // 取余
    let remainder = 43 % 5;
}
```

### 5.2 布尔类型

布尔类型只有两个值：`true` 和 `false`：

```rust
fn main() {
    let t = true;
    let f: bool = false;  // 显式类型标注
}
```

### 5.3 字符类型

Rust 的 `char` 类型表示一个 Unicode 标量值，占 4 个字节：

```rust
fn main() {
    let c = 'z';
    let z = 'ℤ';
    let heart_eyed_cat = '😻';
    
    println!("字符: {}, {}, {}", c, z, heart_eyed_cat);
}
```

**注意：** `char` 使用单引号，字符串使用双引号。

### 5.4 字符串类型

Rust 有两种主要的字符串类型：

#### 5.4.1 字符串字面量（&str）

```rust
fn main() {
    let s = "hello";  // 类型是 &str
    println!("{}", s);
}
```

`&str` 是不可变的字符串切片，通常指向字符串字面量或 String 的切片。

#### 5.4.2 String 类型

```rust
fn main() {
    let mut s = String::from("hello");
    s.push_str(", world!");
    println!("{}", s);
}
```

`String` 是可变的、拥有所有权的字符串类型。

### 5.5 单元类型

单元类型 `()` 表示"无值"：

```rust
fn main() {
    let unit = ();  // 单元类型
}
```

没有返回值的函数实际上返回 `()`：

```rust
fn greet() {
    println!("Hello!");
    // 隐式返回 ()
}
```

### 5.6 类型推导与标注

#### 5.6.1 类型推导

Rust 编译器可以根据上下文自动推导类型：

```rust
fn main() {
    let x = 5;        // 推导为 i32
    let y = 3.14;     // 推导为 f64
    let z = true;     // 推导为 bool
}
```

#### 5.6.2 类型标注

当编译器无法推导类型时，需要显式标注：

```rust
fn main() {
    // 无法推导，需要标注
    let guess: u32 = "42".parse().expect("不是数字");
    
    // 可以推导，但也可以标注
    let x: i32 = 5;
}
```

#### 5.6.3 类型后缀

也可以使用类型后缀：

```rust
fn main() {
    let x = 42i32;    // i32 类型
    let y = 3.14f64;  // f64 类型
    let z = 100u8;    // u8 类型
}
```

---

## 6. 流程控制

### 6.1 if 表达式

`if` 表达式根据条件执行不同的代码：

```rust
fn main() {
    let number = 3;
    
    if number < 5 {
        println!("条件为真");
    } else {
        println!("条件为假");
    }
}
```

#### 6.1.1 多个条件

```rust
fn main() {
    let number = 6;
    
    if number % 4 == 0 {
        println!("能被 4 整除");
    } else if number % 3 == 0 {
        println!("能被 3 整除");
    } else if number % 2 == 0 {
        println!("能被 2 整除");
    } else {
        println!("不能被 4、3 或 2 整除");
    }
}
```

#### 6.1.2 在 let 语句中使用 if

因为 `if` 是表达式，所以可以在 `let` 语句中使用：

```rust
fn main() {
    let condition = true;
    let number = if condition {
        5
    } else {
        6
    };
    
    println!("number 的值是: {}", number);
}
```

**注意：** `if` 的每个分支必须返回相同类型的值。

### 6.2 循环

Rust 提供了三种循环：`loop`、`while` 和 `for`。

#### 6.2.1 loop 循环

`loop` 创建一个无限循环，直到遇到 `break`：

```rust
fn main() {
    loop {
        println!("再次执行！");
        break;  // 退出循环
    }
}
```

**从循环返回值：**

```rust
fn main() {
    let mut counter = 0;
    
    let result = loop {
        counter += 1;
        
        if counter == 10 {
            break counter * 2;  // 返回值
        }
    };
    
    println!("结果是: {}", result);  // 输出：20
}
```

**循环标签：**

```rust
fn main() {
    let mut count = 0;
    'counting_up: loop {
        let mut remaining = 10;
        
        loop {
            if remaining == 9 {
                break;
            }
            if count == 2 {
                break 'counting_up;  // 退出外层循环
            }
            remaining -= 1;
        }
        
        count += 1;
    }
    println!("最终计数: {}", count);
}
```

#### 6.2.2 while 循环

`while` 循环在条件为真时执行：

```rust
fn main() {
    let mut number = 3;
    
    while number != 0 {
        println!("{}!", number);
        number -= 1;
    }
    
    println!("LIFTOFF!!!");
}
```

#### 6.2.3 for 循环

`for` 循环用于遍历集合：

```rust
fn main() {
    let a = [10, 20, 30, 40, 50];
    
    for element in a.iter() {
        println!("值是: {}", element);
    }
}
```

**使用范围：**

```rust
fn main() {
    // 1..4 表示 1, 2, 3（不包含 4）
    for number in 1..4 {
        println!("{}!", number);
    }
    
    // 1..=4 表示 1, 2, 3, 4（包含 4）
    for number in 1..=4 {
        println!("{}!", number);
    }
}
```

**倒计时示例：**

```rust
fn main() {
    for number in (1..4).rev() {
        println!("{}!", number);
    }
    println!("LIFTOFF!!!");
}
```

### 6.3 表达式 vs 语句

Rust 区分**表达式**和**语句**：

- **表达式**：有返回值，末尾不加分号
- **语句**：执行操作，无返回值，末尾加分号

```rust
fn main() {
    let x = 5;  // 语句
    
    let y = {
        let x = 3;
        x + 1  // 表达式，没有分号
    };  // y 的值是 4
    
    println!("y = {}", y);
}
```

---

## 7. 模式匹配

模式匹配是 Rust 最强大的特性之一，用于解构数据并处理不同的情况。

### 7.1 match 表达式

`match` 表达式允许将一个值与一系列模式进行比较，并根据匹配的模式执行相应的代码：

```rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}
```

#### 7.1.1 绑定值的模式

```rust
#[derive(Debug)]
enum UsState {
    Alabama,
    Alaska,
    // ... 等等
}

enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter(UsState),  // 绑定值
}

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter(state) => {
            println!("州是: {:?}", state);
            25
        }
    }
}
```

#### 7.1.2 匹配 Option\<T\>

```rust
fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        None => None,
        Some(i) => Some(i + 1),
    }
}

fn main() {
    let five = Some(5);
    let six = plus_one(five);
    let none = plus_one(None);
}
```

#### 7.1.3 通配模式和 _ 占位符

```rust
fn main() {
    let dice_roll = 9;
    match dice_roll {
        3 => add_fancy_hat(),
        7 => remove_fancy_hat(),
        other => move_player(other),  // 捕获其他值
    }
}

fn add_fancy_hat() {}
fn remove_fancy_hat() {}
fn move_player(num_spaces: u8) {}
```

如果不想使用捕获的值，可以使用 `_`：

```rust
fn main() {
    let dice_roll = 9;
    match dice_roll {
        3 => add_fancy_hat(),
        7 => remove_fancy_hat(),
        _ => reroll(),  // 忽略其他值
    }
}

fn add_fancy_hat() {}
fn remove_fancy_hat() {}
fn reroll() {}
```

或者使用 `()` 表示什么都不做：

```rust
fn main() {
    let dice_roll = 9;
    match dice_roll {
        3 => add_fancy_hat(),
        7 => remove_fancy_hat(),
        _ => (),  // 什么都不做
    }
}
```

### 7.2 if let 简洁控制流

`if let` 是 `match` 的语法糖，用于只关心一种模式的情况：

```rust
fn main() {
    let some_u8_value = Some(3u8);
    
    // 使用 match
    match some_u8_value {
        Some(3) => println!("三"),
        _ => (),
    }
    
    // 使用 if let（更简洁）
    if let Some(3) = some_u8_value {
        println!("三");
    }
}
```

**if let else：**

```rust
fn main() {
    let mut count = 0;
    if let Coin::Quarter(state) = coin {
        println!("州是: {:?}", state);
    } else {
        count += 1;
    }
}
```

### 7.3 while let 条件循环

`while let` 在模式匹配成功时继续循环：

```rust
fn main() {
    let mut stack = Vec::new();
    
    stack.push(1);
    stack.push(2);
    stack.push(3);
    
    while let Some(top) = stack.pop() {
        println!("{}", top);
    }
}
```

---

## 8. 方法 Method

方法是与特定类型关联的函数。在 Rust 中，使用 `impl` 块来定义方法。

### 8.1 定义方法

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    // 方法：第一个参数是 &self
    fn area(&self) -> u32 {
        self.width * self.height
    }
    
    // 方法：可变引用
    fn set_width(&mut self, width: u32) {
        self.width = width;
    }
    
    // 方法：获取所有权
    fn take_ownership(self) {
        println!("获取了 Rectangle 的所有权");
    }
}

fn main(){
    let rect = Rectangle {
        width: 30,
        height: 50,
    };
    
    println!("面积是: {}", rect.area());
}
```

### 8.2 关联函数

关联函数（Associated Functions）是定义在 `impl` 块中但不以 `self` 作为参数的函数：

```rust
impl Rectangle {
    // 关联函数（类似静态方法）
    fn new(width: u32, height: u32) -> Rectangle {
        Rectangle {
            width,
            height,
        }
    }
    
    // 另一个关联函数
    fn square(size: u32) -> Rectangle {
        Rectangle {
            width: size,
            height: size,
        }
    }
}

fn main() {
    let rect = Rectangle::new(30, 50);
    let sq = Rectangle::square(10);
}
```

### 8.3 多个 impl 块

可以为同一个类型定义多个 `impl` 块：

```rust
impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }
}

impl Rectangle {
    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }
}
```

### 8.4 方法调用语法

```rust
fn main() {
    let rect = Rectangle {
        width: 30,
        height: 50,
    };
    
    // 方法调用
    println!("面积: {}", rect.area());
    
    // 关联函数调用
    let rect2 = Rectangle::new(20, 40);
}
```

---

## 9. 集合类型

Rust 标准库提供了多种集合类型，用于存储多个值。

### 9.1 Vector（Vec\<T\>）

`Vec<T>` 是一个可增长的数组类型。

#### 9.1.1 创建 Vector

```rust
fn main() {
    // 使用 Vec::new 创建
    let v: Vec<i32> = Vec::new();
    
    // 使用 vec! 宏创建（更常用）
    let v = vec![1, 2, 3];
    
    // 使用 Vec::with_capacity 创建（预分配容量）
    let mut v = Vec::with_capacity(10);
}
```

#### 9.1.2 更新 Vector

```rust
fn main() {
    let mut v = Vec::new();
    
    v.push(1);
    v.push(2);
    v.push(3);
}
```

#### 9.1.3 读取元素

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];
    
    // 使用索引（可能 panic）
    let third: &i32 = &v[2];
    println!("第三个元素是: {}", third);
    
    // 使用 get 方法（返回 Option）
    match v.get(2) {
        Some(third) => println!("第三个元素是: {}", third),
        None => println!("没有第三个元素"),
    }
}
```

#### 9.1.4 遍历 Vector

```rust
fn main() {
    let v = vec![100, 32, 57];
    
    // 不可变引用遍历
    for i in &v {
        println!("{}", i);
    }
    
    // 可变引用遍历
    let mut v = vec![100, 32, 57];
    for i in &mut v {
        *i += 50;  // 解引用并修改
    }
}
```

#### 9.1.5 使用枚举存储多种类型

```rust
enum SpreadsheetCell {
    Int(i32),
    Float(f64),
    Text(String),
}

fn main() {
    let row = vec![
        SpreadsheetCell::Int(3),
        SpreadsheetCell::Text(String::from("blue")),
        SpreadsheetCell::Float(10.12),
    ];
}
```

### 9.2 字符串（String）

Rust 的字符串比较复杂，因为字符串是 UTF-8 编码的。

#### 9.2.1 创建字符串

```rust
fn main() {
    // 创建空字符串
    let mut s = String::new();
    
    // 从字面量创建
    let s = "initial contents".to_string();
    
    // 使用 String::from
    let s = String::from("initial contents");
}
```

#### 9.2.2 更新字符串

```rust
fn main() {
    let mut s = String::from("foo");
    
    // push_str 追加字符串切片
    s.push_str("bar");
    
    // push 追加单个字符
    s.push('l');
    
    // 使用 + 运算符
    let s1 = String::from("Hello, ");
    let s2 = String::from("world!");
    let s3 = s1 + &s2;  // 注意 s1 被移动了
    
    // 使用 format! 宏
    let s1 = String::from("tic");
    let s2 = String::from("tac");
    let s3 = String::from("toe");
    let s = format!("{}-{}-{}", s1, s2, s3);
}
```

#### 9.2.3 字符串索引

Rust 的字符串不支持直接索引，因为字符串是 UTF-8 编码的：

```rust
fn main() {
    let s = String::from("hello");
    // let h = s[0];  // 编译错误！
    
    // 使用切片
    let hello = "Здравствуйте";
    let s = &hello[0..4];  // 前 4 个字节
}
```

#### 9.2.4 遍历字符串

```rust
fn main() {
    let s = "नमस्ते";
    
    // 遍历字符
    for c in s.chars() {
        println!("{}", c);
    }
    
    // 遍历字节
    for b in s.bytes() {
        println!("{}", b);
    }
}
```

### 9.3 HashMap

`HashMap<K, V>` 存储键值对。

#### 9.3.1 创建 HashMap

```rust
use std::collections::HashMap;

fn main() {
    // 创建空 HashMap
    let mut scores = HashMap::new();
    
    // 插入键值对
    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Yellow"), 50);
}
```

#### 9.3.2 访问值

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();
    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Yellow"), 50);
    
    // 使用 get 方法（返回 Option）
    let team_name = String::from("Blue");
    let score = scores.get(&team_name);
    
    // 遍历
    for (key, value) in &scores {
        println!("{}: {}", key, value);
    }
}
```

#### 9.3.3 更新 HashMap

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();
    
    // 覆盖值
    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Blue"), 25);  // 覆盖
    
    // 只在键不存在时插入
    scores.entry(String::from("Yellow")).or_insert(50);
    scores.entry(String::from("Blue")).or_insert(50);  // 不会覆盖
    
    // 基于旧值更新
    let text = "hello world wonderful world";
    let mut map = HashMap::new();
    
    for word in text.split_whitespace() {
        let count = map.entry(word).or_insert(0);
        *count += 1;
    }
    
    println!("{:?}", map);
}
```

---

## 10. 格式化输出

### 10.1 println! 和 print! 宏

`println!` 和 `print!` 用于输出到标准输出：

```rust
fn main() {
    // println! 输出并换行
    println!("Hello, world!");
    
    // print! 输出但不换行
    print!("Hello, ");
    print!("world!");
}
```

### 10.2 格式化占位符

使用 `{}` 作为占位符：

```rust
fn main() {
    let x = 5;
    let y = 10;
    
    println!("x = {} 且 y = {}", x, y);
}
```

### 10.3 格式化选项

#### 10.3.1 Debug 格式化

使用 `{:?}` 或 `{:#?}` 进行 Debug 格式化：

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 3, y: 4 };
    println!("{:?}", p);      // Point { x: 3, y: 4 }
    println!("{:#?}", p);     // 美化输出
}
```

#### 10.3.2 数字格式化

```rust
fn main() {
    let x = 255;
    
    println!("十进制: {}", x);        // 255
    println!("十六进制: {:x}", x);    // ff
    println!("十六进制: {:X}", x);    // FF
    println!("八进制: {:o}", x);      // 377
    println!("二进制: {:b}", x);      // 11111111
}
```

#### 10.3.3 宽度和对齐

```rust
fn main() {
    println!("{:<10}", "left");      // 左对齐
    println!("{:>10}", "right");     // 右对齐
    println!("{:^10}", "center");    // 居中
    println!("{:*^10}", "center");   // 居中并用 * 填充
}
```

#### 10.3.4 精度

```rust
fn main() {
    let pi = 3.14159265359;
    
    println!("{:.2}", pi);      // 3.14（保留 2 位小数）
    println!("{:.5}", pi);      // 3.14159（保留 5 位小数）
}
```

### 10.4 format! 宏

`format!` 宏返回格式化的字符串，而不是打印：

```rust
fn main() {
    let s = format!("Hello, {}!", "world");
    println!("{}", s);
}
```

### 10.5 eprintln! 和 eprint!

用于输出到标准错误：

```rust
fn main() {
    eprintln!("这是一个错误消息");
    eprint!("错误: ");
}
```

---

## 11. 注释和文档

### 11.1 普通注释

```rust
fn main() {
    // 这是单行注释
    
    /* 这是
       多行注释 */
}
```

### 11.2 文档注释

文档注释使用 `///`（为接下来的项生成文档）或 `//!`（为包含该注释的项生成文档）：

```rust
/// 计算两个数的和
/// 
/// # 示例
/// 
/// ```
/// let result = add(2, 3);
/// assert_eq!(result, 5);
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

//! # 我的 Crate
//! 
//! 这是一个库的文档
```

### 11.3 文档测试

文档注释中的代码块可以作为测试运行：

```rust
/// ```
/// let x = 5;
/// assert_eq!(x, 5);
/// ```
pub fn example() {}
```

运行 `cargo test` 时会运行文档测试。

---

## 12. 包和模块

包（Package）和模块（Module）是 Rust 组织代码的方式，用于控制代码的作用域和私有性。

### 12.1 包（Package）

**包**是一个或多个 crate 的集合，提供一组功能。一个包包含：

- 一个 `Cargo.toml` 文件，描述如何构建这些 crate
- 最多一个库 crate（library crate）
- 可以有多个二进制 crate（binary crate）

#### 12.1.1 包的结构

使用 `cargo new` 创建项目时，会创建一个包：

```bash
cargo new my_project
```

创建的项目结构：

```
my_project/
├── Cargo.toml
└── src/
    └── main.rs
```

#### 12.1.2 Crate

**Crate** 是 Rust 编译器一次编译的最小代码单元，可以是：

- **二进制 crate**：可执行程序，必须有 `main` 函数
- **库 crate**：提供功能供其他项目使用，没有 `main` 函数

**创建库 crate：**

```bash
cargo new --lib my_lib
```

库 crate 的入口是 `src/lib.rs`。

### 12.2 模块（Module）

**模块**让我们可以将 crate 中的代码组织成逻辑单元，并控制项的可见性。

#### 12.2.1 定义模块

使用 `mod` 关键字定义模块：

```rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}
        fn seat_at_table() {}
    }

    mod serving {
        fn take_order() {}
        fn serve_order() {}
        fn take_payment() {}
    }
}
```

#### 12.2.2 模块树

模块形成一棵树，以 crate 为根：

```
crate
 └── front_of_house
     ├── hosting
     │   ├── add_to_waitlist
     │   └── seat_at_table
     └── serving
         ├── take_order
         ├── serve_order
         └── take_payment
```

#### 12.2.3 在模块树中引用项

使用路径（path）来引用模块树中的项：

- **绝对路径**：从 crate 根开始，以 crate 名或字面量 `crate` 开头
- **相对路径**：从当前模块开始，使用 `self`、`super` 或当前模块的标识符

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // 绝对路径
    crate::front_of_house::hosting::add_to_waitlist();
    
    // 相对路径
    front_of_house::hosting::add_to_waitlist();
}
```

### 12.3 可见性

默认情况下，模块中的项是**私有的**，只能在同一模块及其子模块中访问。

#### 12.3.1 pub 关键字

使用 `pub` 关键字使项变为**公开**：

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}
```

#### 12.3.2 私有性规则

- 如果项是私有的，只有同一模块内的代码可以访问
- 如果项是公开的，任何可以访问其父模块的代码都可以访问它

```rust
mod outer_module {
    pub mod inner_module {
        pub fn inner_public() {}
        fn inner_private() {}
    }
    
    fn outer_private() {
        inner_module::inner_public();  // 可以访问
        inner_module::inner_private();  // 可以访问（同一模块）
    }
}

fn main() {
    outer_module::inner_module::inner_public();  // 可以访问
    // outer_module::inner_module::inner_private();  // 编译错误！
}
```

### 12.4 使用 use 关键字引入路径

`use` 关键字创建一个短路径，用于引入项到作用域：

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

// 使用 use 引入
use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();  // 可以直接使用
}
```

#### 12.4.1 引入函数

通常引入函数的父模块，而不是直接引入函数：

```rust
// 好的做法
use std::collections::HashMap;

// 不太好的做法
use std::collections::HashMap::new;
```

#### 12.4.2 引入其他项

对于结构体、枚举等，通常直接引入：

```rust
use std::collections::HashMap;

fn main() {
    let mut map = HashMap::new();
    map.insert(1, 2);
}
```

#### 12.4.3 使用 as 关键字重命名

使用 `as` 关键字为引入的项提供别名：

```rust
use std::fmt::Result;
use std::io::Result as IoResult;

fn function1() -> Result {
    // ...
}

fn function2() -> IoResult<()> {
    // ...
}
```

#### 12.4.4 使用 pub use 重导出

使用 `pub use` 可以将项引入作用域并重新导出，使其他代码也可以使用：

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
}
```

### 12.5 将模块拆分为多个文件

当模块变大时，可以将模块拆分为多个文件。

#### 12.5.1 模块文件分离

假设有以下模块结构：

```rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}
    }
}
```

可以拆分为：

**src/lib.rs（或 main.rs）：**
```rust
mod front_of_house;

pub use crate::front_of_house::hosting;
```

**src/front_of_house.rs：**
```rust
pub mod hosting;
```

**src/front_of_house/hosting.rs：**
```rust
pub fn add_to_waitlist() {}
```

#### 12.5.2 模块目录结构

也可以使用目录结构：

**src/lib.rs：**
```rust
mod front_of_house;
```

**src/front_of_house/mod.rs：**
```rust
pub mod hosting;
```

**src/front_of_house/hosting.rs：**
```rust
pub fn add_to_waitlist() {}
```

### 12.6 标准库的常用模块

Rust 标准库提供了许多有用的模块：

```rust
// 集合类型
use std::collections::HashMap;
use std::collections::BTreeMap;

// 文件 I/O
use std::fs;
use std::io;

// 字符串
use std::string::String;

// 线程
use std::thread;
use std::sync::mpsc;
```

### 12.7 使用外部包

在 `Cargo.toml` 中添加依赖：

```toml
[dependencies]
serde = "1.0"
tokio = { version = "1.0", features = ["full"] }
```

然后在代码中使用：

```rust
use serde::{Serialize, Deserialize};
use tokio::runtime::Runtime;
```

### 12.8 模块最佳实践

1. **组织相关功能**：将相关的函数、结构体等组织到同一个模块中
2. **控制可见性**：使用 `pub` 只暴露必要的接口
3. **使用 `use` 简化路径**：减少重复的长路径
4. **模块文件分离**：当模块变大时，拆分为多个文件

### 12.9 完整示例

**项目结构：**
```
restaurant/
├── Cargo.toml
└── src/
    ├── main.rs
    ├── front_of_house.rs
    └── front_of_house/
        └── hosting.rs
```

**src/main.rs：**
```rust
mod front_of_house;

use crate::front_of_house::hosting;

fn main() {
    hosting::add_to_waitlist();
}
```

**src/front_of_house.rs：**
```rust
pub mod hosting;
```

**src/front_of_house/hosting.rs：**
```rust
pub fn add_to_waitlist() {
    println!("添加到等待列表");
}
```

---

## 总结

本章介绍了 Rust 的基础语法和核心概念：

1. **为什么选择 Rust**：内存安全、并发安全、高性能
2. **Rust 的特点**：所有权系统、零成本抽象、强大的类型系统
3. **Cargo 工具**：项目管理和构建工具
4. **基础语法**：变量、类型、流程控制、模式匹配
5. **方法**：为类型定义方法
6. **集合类型**：Vec、String、HashMap
7. **格式化输出**：println!、format! 等宏
8. **注释和文档**：代码文档化
9. **包和模块**：代码组织和可见性控制

在后续章节中，我们将深入学习：
- **所有权和借用**（第 4 章）
- **泛型和 Trait**（第 5 章）
- **生命周期**（第 7 章）
- **包和模块的高级用法**（第 17 章）
