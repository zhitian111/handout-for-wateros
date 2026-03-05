# Rust 基础：泛型与 Trait

> 本章先系统讲解 **Trait**（特征/特质），再讲解 **泛型**。两者结合是 Rust 抽象与复用的核心，务必吃透。

---

## 本章补充：Crate 与模块（为理解孤儿规则所需）

后面讲 **孤儿规则** 时会提到“当前 crate”“外来类型”等概念，这里先补上 **crate** 是什么、如何在 **Cargo.toml** 里配置、以及 **模块** 的最基本概念。模块的完整用法在第 4 章已有，此处只做最小必要回顾。

### 补充 1. Crate 是什么

**Crate** 是 Rust 里**一次编译的最小单元**：编译器按 crate 为单位编译，产生一个库或一个可执行文件。

- **二进制 crate**：有入口函数 `main`，编译成可执行文件。默认的入口文件是 **`src/main.rs`**，该文件及其通过 `mod` 拉进来的代码共同构成一个二进制 crate。
- **库 crate**：没有 `main`，提供函数、类型等给其他 crate 用。默认的入口是 **`src/lib.rs`**，该文件及其通过 `mod` 拉进来的代码共同构成一个库 crate。

一个 **包（Package）** 由 `Cargo.toml` 描述，可以包含：

- 最多一个**库 crate**（对应 `src/lib.rs`）；
- 零个或多个**二进制 crate**（默认有一个对应 `src/main.rs`，也可在 `Cargo.toml` 里配置更多）。

“当前 crate”指的就是你正在写的这个包里的那个库或那个二进制；**其他包** 依赖你时，会把你的**库 crate** 当作一个整体来用，他们自己的代码属于**他们自己的 crate**。

### 补充 2. 在 Cargo.toml 里如何配置 Crate

`Cargo.toml` 里和 crate 相关的常见配置如下。

#### 包名与库/二进制

- **`[package]`**：描述“包”本身，**name** 决定包名（别人依赖时写的名字），**version**、**edition** 等也在这里。

```toml
[package]
name = "my_project"
version = "0.1.0"
edition = "2021"
```

- **库 crate**：若不写 `[lib]`，默认使用 `src/lib.rs` 作为库的根。若要改入口或禁用库，可写：

```toml
[lib]
name = "my_lib"        # 库的名字，默认与 package name 一致
path = "src/lib.rs"    # 默认就是 src/lib.rs
# 若不想生成库，可设置 crate-type = []，一般不需要
```

- **二进制 crate**：默认有一个二进制，名字与包名相同，入口为 `src/main.rs`。要增加或修改二进制，用 **`[[bin]]`**：

```toml
[[bin]]
name = "my_tool"           # 生成的可执行文件名
path = "src/bin/my_tool.rs" # 入口文件
```

这样除了 `main.rs` 对应的默认二进制外，还会多一个以 `my_tool.rs` 为入口的二进制。

#### 小结

- 不写 `[lib]` / `[[bin]]` 时：一个库（`src/lib.rs`）+ 一个二进制（`src/main.rs`）。
- 通过 **`[lib]`** 和 **`[[bin]]`** 可以指定或增加库/二进制及其入口路径；**包名** 在 **`[package] name`** 里配置。

### 补充 3. 模块简要回顾

**模块** 用来在**一个 crate 内部**组织代码，用 **`mod`** 定义，形成一棵以 **crate 根**（`main.rs` 或 `lib.rs`）为根的树。

- **定义模块**：在 crate 根或某模块里写 `mod 名字 { ... }`，或在单独文件中写 `mod 名字;` 并在同名文件中写具体内容。
- **可见性**：默认私有；用 **`pub`** 暴露给父模块或外部。
- **路径**：**绝对路径** 从 crate 根写起（如 `crate::子模块::项`），**相对路径** 从当前模块写起（如 `super::xxx`、`self::xxx`）。

“当前 crate” 的边界就是这样：从**当前包的** `main.rs` 或 `lib.rs` 出发，经 `mod` 能到达的代码都属于**同一个 crate**；其他包是**别的 crate**。  
模块的完整用法（多文件拆分、`use`、`pub use` 等）见第 4 章「包和模块」。

---

## 第一部分：Trait

## 1. 什么是 Trait

**Trait** 用来定义类型能做什么——即“能提供哪些方法、满足什么行为”。不同类型可以实现同一个 trait，从而被统一地使用。

- 从使用方看：trait 是**约束**——“只要实现了某 trait 的类型，我都能用”。
- 从实现方看：trait 是**契约**——“实现这个 trait 就必须提供这些方法”。

Rust 没有传统意义上的“继承”，多态和抽象主要通过 **trait** 完成。

---

## 2. 定义 Trait

用 `trait` 关键字定义一组方法签名（也可以带默认实现）。

### 2.1 只有方法签名

```rust
trait Summary {
    fn summarize(&self) -> String;
}
```

实现该 trait 的类型必须提供 `summarize` 方法。

### 2.2 带默认实现

方法可以有默认实现，实现类型可以只实现一部分、或覆盖默认行为：

```rust
trait Summary {
    fn summarize(&self) -> String {
        String::from("(阅读更多...)")
    }
}
```

默认实现里可以调用同 trait 的其他方法（即使它们没有默认实现，实现类型也必须提供）：

```rust
trait Summary {
    fn summarize_author(&self) -> String;

    fn summarize(&self) -> String {
        format!("(来自 {} 的摘要)", self.summarize_author())
    }
}
```

### 2.3 关联函数

Trait 里也可以声明不带 `self` 的关联函数，由实现类型提供：

```rust
trait From<T> {
    fn from(value: T) -> Self;
}
```

---

## 3. 为类型实现 Trait

用 `impl Trait名 for 类型` 为具体类型实现某个 trait：

```rust
trait Summary {
    fn summarize(&self) -> String;
}

struct NewsArticle {
    headline: String,
    location: String,
    author: String,
    content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{} ({})", self.headline, self.location)
    }
}

struct Tweet {
    username: String,
    content: String,
    reply: bool,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}

fn main() {
    let t = Tweet {
        username: String::from("user"),
        content: String::from("hello"),
        reply: false,
    };
    println!("{}", t.summarize());
}
```

- 实现中所有 trait 要求的方法都必须写全（或依赖默认实现）。
- 一个类型可以实现多个 trait；一个 trait 也可以被多个类型实现。

---

## 4. Trait 约束（为泛型铺垫）

函数如果只关心“某种能力”而不关心具体类型，可以要求参数类型**实现了某个 trait**，这就是 **trait 约束**。语法在泛型里会写成 `T: Summary`，这里先建立概念：**“只接受实现了 Summary 的类型”**。

```rust
fn print_summary(item: &impl Summary) {
    println!("{}", item.summarize());
}
```

`&impl Summary` 表示“任意实现了 `Summary` 的类型的引用”。这样 `print_summary` 可以接受 `&NewsArticle`、`&Tweet` 等，只要它们实现了 `Summary`。后面讲泛型时会写成等价形式 `fn print_summary<T: Summary>(item: &T)`。

---

## 5. 标准库中的常用 Trait

标准库有很多基础 trait，建议先记住名字和用途，写代码时再查文档。

### 5.1 格式化与调试

- **Debug**：`{:?}`、`{:#?}` 打印，调试用。可用 `#[derive(Debug)]` 自动实现。
- **Display**：`{}` 打印，面向用户。需手写 `impl Display for MyType`。

```rust
use std::fmt;

struct Point { x: i32, y: i32 }

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}
```

### 5.2 相等与比较

- **PartialEq**、**Eq**：相等（`==`、`!=`）。Eq 表示“等价关系”，浮点数只有 PartialEq 没有 Eq。
- **PartialOrd**、**Ord**：大小比较（`<`、`<=`、`>`、`>=`）。排序、BTreeMap 等要求 Ord。

通常用 `#[derive(PartialEq, Eq, PartialOrd, Ord)]`，必要时手写。

### 5.3 拷贝与克隆

- **Copy**：按位复制，赋值不移动。只能给“全栈、无资源”的类型实现，且不能和 Drop 同时存在。
- **Clone**：显式克隆，`clone()`。有堆数据时实现 Clone，需要时再考虑 Copy。

### 5.4 默认值

- **Default**：默认值，`T::default()`。可 `#[derive(Default)]` 或手写。

### 5.5 线程安全（标记 trait）

- **Send**：类型可以安全地跨线程转移所有权。
- **Sync**：类型的引用可以安全地在线程间共享（`&T` 是 Send）。

多数类型自动实现 Send + Sync；若包含裸指针或非线程安全类型，可能只实现其一或都不实现。后续写并发、内核时会经常遇到。

---

## 6. 派生（derive）与自动实现

很多 trait 可以**自动推导实现**，用 `#[derive(...)]` 即可：

```rust
#[derive(Debug, Clone, PartialEq)]
struct User {
    id: u64,
    name: String,
}
```

常见可 derive：`Debug`, `Clone`, `Copy`, `Default`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`, `Serialize`, `Deserialize` 等。  
编译器会按字段递归实现；若某字段不支持（例如没实现 Eq），该 derive 会报错。

---

## 7. 父 Trait（Supertrait）

一个 trait 可以**依赖**另一个 trait：要求“实现当前 trait 的类型必须先实现某个 trait”，语法是 `trait Child: Parent`。

```rust
trait OutlinePrint: fmt::Display {
    fn outline_print(&self) {
        let s = self.to_string();  // 来自 Display
        let len = s.len();
        println!("{}", "*".repeat(len + 4));
        println!("*{}*", " ".repeat(len + 2));
        println!("* {} *", s);
        println!("*{}*", " ".repeat(len + 2));
        println!("{}", "*".repeat(len + 4));
    }
}

impl OutlinePrint for Point {}  // 只有 Point 实现了 Display 才能写这句
```

这样在 `OutlinePrint` 的方法里可以直接用 `Display` 的方法（如 `to_string()`）。

---

## 8. 多个 Trait 约束与 `+` 语法

类型可以同时要求实现多个 trait，用 `+` 连接：

```rust
fn notify(item: &(impl Summary + Display)) { ... }
```

泛型形式（后面会写）是 `T: Summary + Display`。  
多个约束较多时，用 `where` 子句更清晰（见下）。

---

## 9. `where` 子句

当泛型参数或 trait 约束较多时，把约束写在函数签名末尾的 `where` 里，可读性更好：

```rust
fn some_function<T, U>(t: &T, u: &U) -> i32
where
    T: Summary + Clone,
    U: Display + Debug,
{
    // ...
}
```

Trait 定义里也可以为关联类型或方法加 `where` 约束。

---

## 10. 关联类型（Associated Type）

Trait 不仅可以声明方法，还可以声明**关联类型**：由实现者指定具体类型，在 trait 内部用这个名字即可，调用方无需再写类型参数。

### 10.1 定义与实现

```rust
trait Container {
    type Item;  // 关联类型：由实现者指定

    fn get(&self, index: usize) -> Option<&Self::Item>;
    fn len(&self) -> usize;
}

impl Container for Vec<i32> {
    type Item = i32;

    fn get(&self, index: usize) -> Option<&Self::Item> {
        self.as_slice().get(index)
    }
    fn len(&self) -> usize {
        self.len()
    }
}
```

使用 trait 时写 `Container`，内部用 `Self::Item`；实现时用 `type Item = i32` 等具体类型。

### 10.2 与泛型参数的区别

- **泛型 trait**：`trait Foo<T>` —— 一个类型可以实现多次，如 `impl Foo<i32> for Bar` 和 `impl Foo<u32> for Bar`。
- **关联类型**：`trait Foo { type T; }` —— 一个类型只实现一次，`T` 在实现时固定。  
标准库中 `Iterator` 用 `type Item`，`Add` 用 `type Output`，都是关联类型，表示“和具体类型强绑定”的产出类型。

---

## 11. Trait 对象（`dyn Trait`）与动态分发

有时我们希望在运行时才决定具体类型（例如多种实现选一种），而不是在编译期用泛型写死。这时可以用 **trait 对象**：`&dyn Trait`、`Box<dyn Trait>` 等。

### 11.1 基本用法

```rust
trait Draw {
    fn draw(&self);
}

struct Button;
struct Select;

impl Draw for Button {
    fn draw(&self) { println!("绘制按钮"); }
}
impl Draw for Select {
    fn draw(&self) { println!("绘制选择框"); }
}

fn main() {
    let components: Vec<Box<dyn Draw>> = vec![
        Box::new(Button),
        Box::new(Select),
    ];
    for c in components {
        c.draw();
    }
}
```

- `dyn Draw` 表示“任意实现了 Draw 的类型”，在运行时通过虚表查找具体方法，称为**动态分发**。
- Trait 对象有**对象安全**限制：trait 不能返回 `Self`、不能带泛型方法等，这样编译器才能生成统一的虚表。

### 11.2 与泛型的对比

- **泛型**：编译期单态化，每种类型一份代码，无额外间接调用，适合“类型在编译期已知”。
- **Trait 对象**：运行时多态，一份代码处理多种类型，有虚表开销，适合“类型集合在运行时才确定”或需要减小代码体积时。

先把两者概念分清；实际项目里会大量用泛型 + trait 约束，trait 对象在需要“类型擦除”时再用。

---

## 12. 一致性（孤儿规则）

Rust 规定：**要么 trait 在当前 crate 定义，要么类型在当前 crate 定义**，才能写 `impl Trait for Type`。不能为外来类型实现外来 trait（避免依赖冲突和任意覆盖）。这叫 **孤儿规则**。

因此无法在标准库类型上随意加自己的 trait 实现，但可以：

- 用 newtype 包一层：`struct Wrapper(T);`，然后 `impl MyTrait for Wrapper`。

---

## 第二部分：泛型

## 13. 为什么需要泛型

同一逻辑希望复用于多种类型（例如“比较大小”“打印”“容器元素”），而不想为每种类型写一遍函数或结构体。**泛型**就是在类型或函数上引入**类型参数**，由编译器在编译期具化为具体类型，实现零成本抽象。

---

## 14. 泛型函数

### 14.1 类型参数

在函数名后加 `<T>` 表示类型参数，参数或返回值类型可以用 `T`：

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list.iter() {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn main() {
    let nums = [3, 1, 4, 1, 5];
    let chars = ['a', 'z', 'm'];
    println!("{}", largest(&nums));
    println!("{}", largest(&chars));
}
```

这里 `T: PartialOrd` 是 **trait 约束**：只有实现了 `PartialOrd` 的类型才能比较大小，所以 `T` 才能用 `>`。

### 14.2 多个类型参数

可以有多个类型参数，并分别加约束：

```rust
fn mix<T: Clone, U: Clone>(a: T, b: U) -> (T, U) {
    (a.clone(), b.clone())
}
```

---

## 15. 泛型结构体与枚举

### 15.1 泛型结构体

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let p_i = Point { x: 1, y: 2 };
    let p_f = Point { x: 1.0, y: 2.0 };
    // let p_mix = Point { x: 1, y: 2.0 };  // 错误：x 和 y 必须是同一 T
}
```

若希望 x、y 类型不同，可再引入一个类型参数：

```rust
struct Point<T, U> {
    x: T,
    y: U,
}
```

### 15.2 泛型枚举

标准库里的 `Option<T>`、`Result<T, E>` 就是泛型枚举：

```rust
enum Option<T> {
    Some(T),
    None,
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

---

## 16. 泛型 impl 块

可以为泛型类型写 `impl`，在 `impl` 上声明类型参数，并在方法里使用；也可以为**具体类型**实现**泛型 trait**（如 `impl From<f64> for MyType`）。

### 16.1 为泛型结构体实现方法

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}
```

所有 `Point<T>` 都有 `x()` 方法。

### 16.2 带约束的 impl

只为满足某约束的 `T` 实现方法：

```rust
impl<T: Display + PartialOrd> Point<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("较大的是 x: {}", self.x);
        } else {
            println!("较大的是 y: {}", self.y);
        }
    }
}
```

只有“实现了 Display 和 PartialOrd”的 `Point<T>` 才有 `cmp_display`。

### 16.3 为具体类型实现泛型 trait

例如为标准库类型实现标准库 trait（在孤儿规则允许的前提下）：

```rust
impl From<i32> for MyType {
    fn from(n: i32) -> Self {
        MyType { value: n as f64 }
    }
}
```

---

## 17. 单态化（Monomorphization）

泛型在 Rust 里是**编译期**展开的：编译器为每个用到的具体类型生成一份代码（例如 `largest::<i32>`、`largest::<char>` 各一份），这叫**单态化**。  
结果是：没有运行时类型擦除、没有虚表，和手写多份函数在性能上等价，即**零成本抽象**。

---

## 18. Trait 约束的完整写法

### 18.1 约束形式小结

- `T: Trait`：T 必须实现 Trait。
- `T: A + B`：T 必须同时实现 A 和 B。
- `where T: Trait`：把约束写在 where 子句里。
- `impl Trait` 语法：`fn f(x: impl Trait)` 等价于泛型 + 约束，但无法在签名里重复使用同一类型（此时必须用泛型 `fn f<T: Trait>(x: T, y: T)`）。

### 18.2 约束与关联类型

泛型函数或 impl 里若用到了某 trait 的关联类型，需要写清约束，并在需要时用 `T::Item` 等形式引用：

```rust
trait Container {
    type Item;
    fn get(&self, i: usize) -> Option<&Self::Item>;
}

fn first<C: Container>(c: &C) -> Option<&C::Item> {
    c.get(0)
}
```

---

## 19. 泛型与 Trait 结合：综合示例

```rust
use std::fmt::Display;

trait Summary {
    fn summarize(&self) -> String;
}

fn print_many<T: Summary + Display>(items: &[T]) {
    for item in items {
        println!("{} -> {}", item, item.summarize());
    }
}

struct Wrapper<T>(T);

impl<T: Display> Summary for Wrapper<T> {
    fn summarize(&self) -> String {
        format!("Wrapper({})", self.0)
    }
}

impl<T> Display for Wrapper<T>
where
    T: Display,
{
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "{}", self.0)
    }
}
```

这里：泛型结构体 `Wrapper<T>`、为带约束的 `T` 实现多个 trait、函数参数使用 `T: Summary + Display`，是常见模式。

---

## 20. Const 泛型（简介）

Rust 还支持**常量泛型**：用常量作为参数，常用于数组长度等：

```rust
fn foo<const N: usize>(arr: [i32; N]) {
    println!("长度: {}", N);
}

struct Buffer<const N: usize> {
    data: [u8; N],
}
```

`N` 在编译期确定，不同 `N` 会单态化成不同类型（如 `[i32; 3]` 和 `[i32; 4]` 是不同类型）。

---

## 21. 小结

| 主题 | 要点 |
| --- | --- |
| **Trait** | 定义行为契约；默认实现；为类型实现 trait；trait 约束（`impl Trait`、`T: Trait`）。 |
| **常用 trait** | Debug、Display、Clone、Copy、PartialEq、Eq、Ord、Default、Send、Sync 等；derive 自动实现。 |
| **高级 trait** | 父 trait、关联类型、trait 对象（`dyn Trait`）、孤儿规则。 |
| **泛型** | 类型参数 `<T>`；泛型函数、结构体、枚举、impl；trait 约束与 where。 |
| **单态化** | 编译期展开，零成本；const 泛型用于编译期常量。 |

先掌握 trait 与泛型的基础和常见写法，后续在复杂 trait 约束、组件化设计中会反复用到这些概念。
