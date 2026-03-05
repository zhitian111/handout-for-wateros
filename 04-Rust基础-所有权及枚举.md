# Rust 基础：所有权及枚举

> 本章深入讲解所有权与借用、嵌套循环标签退出，以及枚举与 Option/Result 的用法。

---

## 上节课补充：切片（Slice）

上节课在讲基本类型和借用时，**切片**没有展开讲，这里补上。切片表示对一段**连续序列**的引用，不拥有数据，属于借用的一种；常见的有**字符串切片 `&str`** 和**数组切片 `&[T]`**。

### 为什么需要切片

- 有时我们只关心数组或字符串的某一段，不想拷贝，也不想拿走所有权。
- 切片提供对“连续一段”的**只读或只写**视图，类型统一（例如函数参数用 `&[i32]` 可以同时接数组和 `Vec` 的切片），且不发生拷贝。

### 字符串切片 `&str`

**字符串切片**的类型是 `&str`，表示对 UTF-8 字符串中某一段的引用。

#### 语法与范围

使用 `[start..end]` 表示从下标 `start` 到 `end`（不包含 `end`）：

```rust
fn main() {
    let s = String::from("hello world");
    let hello = &s[..5];   // "hello"&str
    let world = &s[6..];  // "world"&str
    println!("{} {}", hello, world);
}
```

- **省略开头**：`[..5]` 表示从开头到下标 5（不包含 5）。
- **省略结尾**：`[6..]` 表示从下标 6 到末尾。
- **全部**：`[..]` 表示整段，例如 `&s[..]` 等价于对 `String` 的整段切片。

注意：下标是按**字节**计的，不是字符。一个 UTF-8 字符可能占 1～4 个字节，切在字符中间会 panic：

```rust
fn main() {
    let s = String::from("你好");
    // let bad = &s[0..2];  // 可能 panic：在 UTF-8 字符边界中间
    let ok = &s[0..3];     // "你" 的 UTF-8 是 3 字节
}
```

#### 字符串字面量就是 `&str`

字面量 `"hello"` 的类型就是 `&str`，即指向只读内存的字符串切片：

```rust
fn main() {
    let s: &str = "hello";
    println!("{}", s);
}
```

#### 函数参数常用 `&str`

接受“字符串”时，用 `&str` 可以同时接 `String` 和字面量，无需拷贝：

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &b) in bytes.iter().enumerate() {
        if b == b' ' {
            return &s[0..i];
        }
    }
    &s[..]
}

fn main() {
    let s = String::from("hello world");
    let w = first_word(&s);       // 传 &String，自动转成 &str
    let w2 = first_word("hi");   // 直接传 &str
    println!("{} {}", w, w2);
}
```

### 数组切片 `&[T]`

对数组或 `Vec` 做切片，得到类型 `&[T]`，同样不拥有数据，只是引用一段连续元素。

#### 创建与范围

```rust
fn main() {
    let a = [1, 2, 3, 4, 5];
    let slice = &a[1..4];  // [2, 3, 4]
    println!("{:?}", slice);

    let v = vec![10, 20, 30, 40];
    let vslice = &v[..2];  // [10, 20]
    println!("{:?}", vslice);
}
```

范围规则与字符串一致：`[start..end]`、`[..end]`、`[start..]`、`[..]`。

#### 函数中接受切片

用 `&[T]` 做参数，可以同时接收数组引用和 `Vec` 的切片，调用时写 `&arr` 或 `&vec[..]` 即可：

```rust
fn sum(slice: &[i32]) -> i32 {
    slice.iter().sum()
}

fn main() {
    let a = [1, 2, 3];
    let v = vec![4, 5, 6];
    println!("{}", sum(&a));    // 传 &[i32]
    println!("{}", sum(&v));    // 传 &[i32]
    println!("{}", sum(&v[1..]));  // 只传后半段
}
```

### 切片与所有权、借用

- 切片是**借用**，不拥有数据，不会移动所有权。
- 创建切片后，在切片存活期间，不能通过原变量做会修改或使该段失效的操作（遵守借用规则）。
- 切片本身是**只读视图**（`&[T]`、`&str`）；若需可写，会有对应的“可变切片”形式（如 `&mut [T]`），同样遵守“同一时刻要么多个只读要么一个可写”的规则。

---

## 1. 所有权（Ownership）

所有权是 Rust 最核心的概念，用于在编译期保证内存安全，无需垃圾回收（GC）或手动 free。

### 1.1 栈与堆

- **栈（Stack）**：按值进入、按相反顺序弹出；固定大小、分配快；存放局部变量、函数参数等。
- **堆（Heap）**：动态大小、按需分配；数据通过指针访问；`String`、`Vec` 等的数据在堆上。

Rust 中：

- 基本类型（如 `i32`、`bool`）通常放在栈上。
- `String`、`Vec<T>` 等在堆上分配，变量本身在栈上保存“指针 + 长度 + 容量”等元数据。

### 1.2 所有权三原则

1. **每个值有且仅有一个所有者**。
2. **当所有者离开作用域时，值被丢弃**（相当于自动回收）。
3. **赋值、传参、返回值可能发生“移动”**，原变量不再有效（针对实现 `Move` 语义的类型）。

### 1.3 移动（Move）

对于在堆上分配的类型（如 `String`），赋值是**移动**：所有权交给新变量，原变量不能再被使用。

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;  // s1 的所有权移动给 s2
    // println!("{}", s1);  // 编译错误：s1 已被移动
    println!("{}", s2);     // 正确
}
```

**为什么是移动而不是浅拷贝？**  
若只是浅拷贝，`s1` 和 `s2` 会指向同一块堆内存，离开作用域时会被 drop 两次，导致双重释放。Rust 通过“移动”保证同一块堆数据只有一个所有者。

### 1.4 克隆（Clone）

若需要独立的一份数据，应显式克隆：

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();  // 堆上数据被复制一份
    println!("{} {}", s1, s2);  // 都有效
}
```

`clone()` 可能较贵，只在需要时使用。

### 1.5 复制（Copy）

对于纯栈上、拷贝成本低的类型，Rust 会实现 `Copy` trait，赋值时是**复制**而不是移动，原变量仍可用。

```rust
fn main() {
    let x = 5;
    let y = x;  // 复制，x 和 y 都有效
    println!("{} {}", x, y);
}
```

常见 `Copy` 类型：整数、浮点、`bool`、`char`、只包含 `Copy` 类型的元组等。  
一旦类型或它的字段里有 `Drop`（如 `String`），就不能再实现 `Copy`。

### 1.6 所有权与函数

- **传入函数**：传值会发生移动或复制；移动后，调用处的原变量失效。
- **从函数返回**：返回值会移动给调用者。

```rust
fn take_ownership(s: String) {
    println!("{}", s);
}  // s 离开作用域，对应堆内存被释放

fn make_string() -> String {
    let s = String::from("hello");
    s
}

fn main() {
    let s1 = String::from("hello");
    take_ownership(s1);  // s1 被移动进函数
    // println!("{}", s1);  // 错误：s1 已移动

    let s2 = make_string();  // 返回值移动给 s2
    println!("{}", s2);
}
```

---

## 2. 借用与引用（Borrowing）

不获取所有权而访问值，通过**引用**完成：**借用**不会移动数据，只是临时使用。

### 2.1 不可变引用（&T）

```rust
fn main() {
    let s = String::from("hello");
    let len = calculate_length(&s);  // 借用 s，不移动
    println!("{} 的长度是 {}", s, len);  // s 仍可用
}

fn calculate_length(s: &String) -> usize {
    s.len()
}  // s 是引用，不拥有数据，不会 drop 堆内存
```

- `&s` 创建不可变引用；函数签名 `s: &String` 表示“借用一个 String”。
- 借用期间，所有者仍保留所有权，且不能通过其他引用修改该值。

### 2.2 可变引用（&mut T）

需要修改时使用可变引用：

```rust
fn main() {
    let mut s = String::from("hello");
    append_world(&mut s);
    println!("{}", s);  // hello world
}

fn append_world(s: &mut String) {
    s.push_str(" world");
}
```

### 2.3 借用规则（重要）

1. **同一时刻**，要么只能有**多个不可变引用**，要么只能有**一个可变引用**，不能混用。
2. 引用必须**始终有效**（不能悬垂）。

```rust
fn main() {
    let mut s = String::from("hello");
    let r1 = &s;
    let r2 = &s;
    // let r3 = &mut s;  // 编译错误：已有不可变引用时不能再借可变引用
    println!("{} {}", r1, r2);
    // r1、r2 使用结束后，下面才可以有 &mut
    let r3 = &mut s;
    //let r4 = &mut s;
    //let r5 = &s;
    
    r3.push_str("!");
}
```

这些规则在编译期消除数据竞争和悬垂引用。

### 2.4 悬垂引用

Rust 不允许返回指向已失效数据的引用：

```rust
// fn dangle() -> &String {
//     let s = String::from("hello");
//     &s  // 错误：s 在函数结束时被 drop，返回 &s 会悬垂
// }
```

正确做法是返回所有权（例如直接返回 `String`），或让调用者持有数据并传入引用。

### 2.5 切片与借用的关系

切片（`&str`、`&[T]`）本质上是对一段连续数据的**借用**，不拥有数据，也不发生拷贝。具体语法、范围写法和在函数中的用法已在本章开头的**「上节课补充：切片」**中说明。

---

## 3. 嵌套循环与标签退出

在多重循环中，若想从内层直接退出外层，可以使用**循环标签**。

### 3.1 语法

在循环前写 `'标签名:`，在需要退出的地方写 `break '标签名;`。`loop`、`while`、`for` 都支持标签。

### 3.2 在 loop 中使用标签

```rust
fn main() {
    let mut count = 0;
    'outer: loop {
        let mut inner = 0;
        loop {
            inner += 1;
            if inner > 3 {
                break;  // 只退出内层 loop
            }
            if count == 2 {
                break 'outer;  // 直接退出外层 loop
            }
        }
        count += 1;
    }
    println!("count = {}", count);  // count = 2
}
```

### 3.3 在 for 循环中使用标签

```rust
fn main() {
    'outer: for i in 1..=3 {
        for j in 1..=3 {
            if i == 2 && j == 2 {
                break 'outer;  // 跳出外层 for
            }
            println!("i={}, j={}", i, j);
        }
    }
}
```

### 3.4 在 while 循环中使用标签

```rust
fn main() {
    let mut a = 0;
    'outer: while a < 3 {
        let mut b = 0;
        while b < 3 {
            if a == 1 && b == 1 {
                break 'outer;
            }
            println!("a={}, b={}", a, b);
            b += 1;
        }
        a += 1;
    }
}
```

### 3.5 典型用途

- 二维或多维遍历中，找到目标后一次性退出所有层。
- 多重循环中的“失败即整体退出”逻辑，避免用很多 `bool` 或深层 `if`。

---

## 4. 枚举（Enum）的定义与使用

枚举用于表示**一组可选值中的一个**，每个可选值称为**变体（variant）**。

### 4.1 基本定义

```rust
enum Direction {
    Up,
    Down,
    Left,
    Right,
}

fn main() {
    let d = Direction::Up;
}
```

### 4.2 带数据的枚举

变体可以携带不同类型、不同数量的数据：

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}
```

- `Quit`：无数据。
- `Move { x, y }`：具名字段。
- `Write(String)`：单个类型。
- `ChangeColor(i32, i32, i32)`：元组形式。

### 4.3 枚举与 match

用 `match` 对所有变体做穷尽匹配：

```rust
fn handle_message(msg: Message) {
    match msg {
        Message::Quit => println!("退出"),
        Message::Move { x, y } => println!("移动到 ({}, {})", x, y),
        Message::Write(s) => println!("写入: {}", s),
        Message::ChangeColor(r, g, b) => println!("颜色 ({},{},{})", r, g, b),
    }
}
```

枚举 + `match` 是 Rust 里表达“多种情况”的标准方式。下面两个枚举是标准库中最常用的。

---

## 5. 包和模块

包（Package）和模块（Module）是 Rust 组织代码的方式，用于控制代码的作用域和私有性。

### 5.1 包（Package）

**包**是一个或多个 crate 的集合，提供一组功能。一个包包含：

- 一个 `Cargo.toml` 文件，描述如何构建这些 crate
- 最多一个库 crate（library crate）
- 可以有多个二进制 crate（binary crate）

#### 5.1.1 包的结构

使用 `cargo new` 创建项目时，会创建一个包：

```bash
cargo new my_project
```

创建的项目结构：

```text
my_project/
├── Cargo.toml
└── src/
    └── main.rs
```

#### 5.1.2 Crate

**Crate** 是 Rust 编译器一次编译的最小代码单元，可以是：

- **二进制 crate**：可执行程序，必须有 `main` 函数
- **库 crate**：提供功能供其他项目使用，没有 `main` 函数

**创建库 crate：**

```bash
cargo new --lib my_lib
```

库 crate 的入口是 `src/lib.rs`。

### 5.2 模块（Module）

**模块**让我们可以将 crate 中的代码组织成逻辑单元，并控制项的可见性。

#### 5.2.1 定义模块

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

#### 5.2.2 模块树

模块形成一棵树，以 crate 为根：

```text
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

#### 5.2.3 在模块树中引用项

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

### 5.3 可见性

默认情况下，模块中的项是**私有的**，只能在同一模块及其子模块中访问。

#### 5.3.1 pub 关键字

使用 `pub` 关键字使项变为**公开**：

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}
```

#### 5.3.2 私有性规则

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

### 5.4 使用 use 关键字引入路径

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

#### 5.4.1 引入函数

通常引入函数的父模块，而不是直接引入函数：

```rust
// 好的做法
use std::collections::HashMap;

// 不太好的做法
use std::collections::HashMap::new;
```

#### 5.4.2 引入其他项

对于结构体、枚举等，通常直接引入：

```rust
use std::collections::HashMap;

fn main() {
    let mut map = HashMap::new();
    map.insert(1, 2);
}
```

#### 5.4.3 使用 as 关键字重命名

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

#### 5.4.4 使用 pub use 重导出

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

### 5.5 将模块拆分为多个文件

当模块变大时，可以将模块拆分为多个文件。

#### 5.5.1 模块文件分离

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

#### 5.5.2 模块目录结构

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

### 5.6 标准库的常用模块

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

### 5.7 使用外部包

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

### 5.8 模块最佳实践

1. **组织相关功能**：将相关的函数、结构体等组织到同一个模块中
2. **控制可见性**：使用 `pub` 只暴露必要的接口
3. **使用 `use` 简化路径**：减少重复的长路径
4. **模块文件分离**：当模块变大时，拆分为多个文件

### 5.9 完整示例

**项目结构：**

```text
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

## 6. Option\<T\> 详解

`Option<T>` 表示“可能有值，可能没有”，用于避免空指针问题。

### 6.1 定义与变体

```rust
enum Option<T> {
    Some(T),  // 有值，值为 T
    None,     // 无值
}
```

标准库已预定义，无需自己写，直接使用 `Option::Some`、`Option::None`，或简写 `Some`、`None`。

### 6.2 为什么需要 Option

很多语言用 `null` 表示“无值”，容易忘记检查导致运行时错误。Rust 没有 `null`，用 `Option<T>` 强制在类型层面处理“可能没有”的情况。

### 6.3 基本使用

```rust
fn main() {
    let some_number = Some(5);
    let some_string = Some("hello");
    let absent: Option<i32> = None;  // 必须注明类型，因为 None 无法推断 T
}
```

### 6.4 用 match 处理 Option

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

### 6.5 unwrap 与 expect（慎用）

- **unwrap()**：若是 `Some(v)` 返回 `v`，若是 `None` 则 **panic**。
- **expect("msg")**：同上，但 panic 时带上自定义消息。

适合原型或确定一定有值的场景，生产代码更推荐用 `match` 或 `?` 显式处理。

```rust
fn main() {
    let some = Some(42);
    let v = some.unwrap();  // 42
    // let n: Option<i32> = None;
    // n.unwrap();  // panic
    let v2 = some.expect("应该有值");  // 42
}
```

### 6.6 if let 简化单分支

当只关心 `Some` 一种情况时，可用 `if let`：

```rust
fn main() {
    let opt = Some(7);
    if let Some(x) = opt {
        println!("值是 {}", x);
    }
    // 若需要 else，可写：
    // if let Some(x) = opt { ... } else { ... }
}
```

### 6.7 常用方法

- **unwrap_or(default)**：`None` 时返回 `default`。
- **unwrap_or_else(f)**：`None` 时用闭包 `f()` 得到默认值。
- **map(f)**：`Some(x)` → `Some(f(x))`，`None` → `None`。
- **and_then(f)**：链式处理，`f` 接收值并返回 `Option<U>`。

```rust
fn main() {
    let some: Option<i32> = Some(2);
    let none: Option<i32> = None;

    assert_eq!(some.unwrap_or(0), 2);
    assert_eq!(none.unwrap_or(0), 0);

    assert_eq!(some.map(|x| x * 2), Some(4));
    assert_eq!(none.map(|x| x * 2), None);

    let double_then_sqrt = |x: i32| -> Option<f64> {
        if x >= 0 { Some((x as f64).sqrt()) } else { None }
    };
    assert_eq!(Some(4).and_then(double_then_sqrt), Some(2.0));
}
```

---

## 7. Result\<T, E\> 详解

`Result<T, E>` 表示可能成功得到 `T`，也可能失败得到错误 `E`，是 Rust 中错误处理的基础。

### 7.1 定义与变体

```rust
enum Result<T, E> {
    Ok(T),   // 成功，结果为 T
    Err(E),  // 失败，错误为 E
}
```

标准库已定义，直接使用 `Ok`、`Err` 即可。

### 7.2 基本使用

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt");
    match f {
        Ok(file) => { /* 使用 file */ }
        Err(e) => println!("打开文件失败: {:?}", e),
    }
}
```

### 7.3 用 match 处理 Result

```rust
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err(String::from("除数不能为 0"))
    } else {
        Ok(a / b)
    }
}

fn main() {
    match divide(10.0, 2.0) {
        Ok(v) => println!("结果是 {}", v),
        Err(e) => println!("错误: {}", e),
    }
}
```

### 7.4 unwrap 与 expect

与 `Option` 类似：

- **unwrap()**：`Ok(v)` 返回 `v`，`Err` 时 panic。
- **expect("msg")**：同上，panic 时带自定义消息。

```rust
fn main() {
    let res: Result<i32, &str> = Ok(42);
    let v = res.unwrap();  // 42
    // Err("错了").unwrap();  // panic
}
```

### 7.5 错误传播与 ? 运算符

在返回 `Result` 的函数中，用 `?` 可以把“错误”直接向上返回，成功则得到内部值：

```rust
use std::fs::File;
use std::io;
use std::io::Read;

fn read_username_from_file() -> Result<String, io::Error> {
    let mut f = File::open("hello.txt")?;  // 若 Err，则整个函数返回该 Err
    let mut s = String::new();
    // f.read_to_string(&mut s)?;
    match f.read_to_string(&mut s){
	    Ok(_) => ();
	    Err(e) => return Err(e);
    }
    Ok(s)
}
```

- `?` 只能用在返回类型为 `Result`（或 `Option`）的函数里。
- 等价于：遇到 `Err(e)` 就 `return Err(e)`，遇到 `Ok(v)` 就得到 `v`。

### 7.6 Option 与 Result 的 ? 用法

在返回 `Option` 的函数里，也可以对 `Option` 使用 `?`：

```rust
fn first_element(v: &[i32]) -> Option<i32> {
    let first = v.get(0)?;  // 若为 None，则函数返回 None
    Some(*first)
}
```

### 7.7 Result 常用方法

- **unwrap_or(default)**：`Err` 时返回 `default`。
- **unwrap_or_else(f)**：`Err(e)` 时用 `f(e)` 得到默认值。
- **map(f)**：`Ok(x)` → `Ok(f(x))`，`Err` 原样保留。
- **map_err(f)**：`Err(e)` → `Err(f(e))`，`Ok` 原样保留。
- **and_then(f)**：链式调用，`f` 接收值并返回 `Result<U, E>`。

```rust
fn main() {
    let ok: Result<i32, &str> = Ok(2);
    let err: Result<i32, &str> = Err("失败");

    assert_eq!(ok.unwrap_or(0), 2);
    assert_eq!(err.unwrap_or(0), 0);

    assert_eq!(ok.map(|x| x * 2), Ok(4));
    assert_eq!(err.map(|x| x * 2), Err("失败"));
}
```

### 7.8 组合 Option 与 Result

实际代码中经常混用：例如先得到 `Option`，再转成 `Result`，或用 `ok_or` / `ok_or_else`：

```rust
fn main() {
    let opt: Option<i32> = Some(42);
    let res: Result<i32, i32> = opt.ok_or(456);
    assert_eq!(res, Ok(42));

    let none: Option<i32> = None;
    let res2 = none.ok_or("没有值");
    assert_eq!(res2, Err("没有值"));
}
```

---

## 8. 小结

| 主题 | 要点 |
| --- | --- |
| **切片（补充）** | `&str` 字符串切片、`&[T]` 数组切片；范围 `[start..end]`；不拥有数据，属于借用。 |
| **所有权** | 每个值一个所有者；移动/复制/克隆；函数传参与返回值也遵循所有权规则。 |
| **借用** | `&T` 不可变引用，`&mut T` 可变引用；遵守借用规则，避免悬垂引用。 |
| **嵌套循环** | 使用 `'label:` 和 `break 'label` 从内层直接退出外层。 |
| **枚举** | 定义变体、带数据、用 `match` 穷尽匹配。 |
| **包和模块** | 包与 crate、模块定义与模块树、可见性、`use` 与 `pub use`、模块拆分为多文件。 |
| **Option\<T\>** | 表示“有/无值”；用 `match`、`if let`、`map`、`and_then`、`unwrap_or` 等安全处理。 |
| **Result\<T, E\>** | 表示“成功/失败”；用 `match`、`?` 传播错误，配合 `map`、`map_err`、`and_then` 等。 |

建议多写小例子体会所有权、借用与 Option/Result 的配合，这是写出正确、健壮 Rust 代码的基础。
