# Rust 进阶：生命周期与智能指针

> 本章涵盖：**闭包**与 **Fn 系列 Trait**（含关联类型与标准写法）、**迭代器**及其 trait 与方法、**生命周期**，以及 Rust 中的**全部智能指针**。

---

## 1. 闭包（Closure）

闭包是可以**捕获其所在作用域中变量**的匿名函数。与普通函数不同，闭包能够“记住”定义时的环境，可作为参数传递、存入结构体、跨线程使用，是函数式风格和回调的基础。

### 1.1 语法形式

闭包由 **`|参数列表|`** 和 **函数体** 组成，参数与返回值类型多数情况下可省略，由编译器根据调用处推导。

**单行表达式**（可省略 `{}`）：

```rust
let add_one = |x| x + 1;
let add = |a, b| a + b;
```

**多行块**（用 `{}`，最后为表达式时作为返回值）：

```rust
let complex = |x: i32| {
    let y = x * 2;
    if y > 10 { y - 10 } else { y }
};
```

**显式标注类型**（在编译器无法推导或需要明确契约时）：

```rust
let add_one = |x: i32| -> i32 { x + 1 };
let none: Option<i32> = None;
let wrap = |x: i32| -> Option<i32> { Some(x) };
```

**无参闭包**（常用于延迟求值或捕获环境）：

```rust
let x = 42;
let get_x = || x;
println!("{}", get_x());
```

### 1.2 捕获方式（环境如何进入闭包）

闭包会**自动**从环境中捕获用到的变量，捕获方式分为三种；编译器根据闭包**体内如何使用**这些变量来决定用哪种方式，必要时可显式加 **`move`** 强制按值捕获。

| 捕获方式    | 闭包内用法    | 实现的 Trait | 说明                  |
| ------- | -------- | --------- | ------------------- |
| 不可变借用   | 只读环境变量   | `Fn`      | 可多次调用，不修改环境         |
| 可变借用    | 修改环境变量   | `FnMut`   | 可多次调用，需 `&mut self` |
| 按值（所有权） | 移动进闭包或消费 | `FnOnce`  | 至少实现；可能只能调一次        |

示例：不可变借用

```rust
let x = 4;
let equal_to_x = |z| z == x;   // 只读 x，捕获 &x
assert!(equal_to_x(4));
// x 仍可用
```

示例：可变借用

```rust
let mut list = vec![1, 2, 3];
let mut push_four = || list.push(4);  // 修改 list，捕获 &mut list
push_four();
push_four();
// list 仍可用，且已被修改
```

示例：按值捕获（move）

在闭包前加 **`move`**，会强制**按值**捕获：相关变量的所有权移入闭包。适合把闭包传到线程、或需要闭包比当前作用域活得更久时。

```rust
let list = vec![1, 2, 3];
let take = move || {
    println!("{:?}", list);  // list 被移动进闭包
};
take();
// take();  // 若闭包消费了 list，第二次调用可能报错（取决于是否实现 FnOnce）
// list 在这里已不可用
```

move 与线程

`thread::spawn` 要求闭包为 `'static`，且其捕获不能依赖当前栈；因此跨线程传闭包时几乎总要写 **`move`**，把需要的数据“搬进”闭包。

```rust
use std::thread;
let v = vec![1, 2, 3];
let handle = thread::spawn(move || {
    println!("{:?}", v);  // v 被移进闭包，在子线程中使用
});
handle.join().unwrap();
```

### 1.3 闭包作为函数参数

函数可以接受闭包，通过泛型 + **Fn / FnMut / FnOnce** 约束（见下一节），或 **`impl Fn(...)`** 语法：

```rust
fn apply_twice<F>(f: F, x: i32) -> i32
where
    F: Fn(i32) -> i32,
    // Fn<(i32,), Output=i32>
{
    f(f(x))
}

fn main() {
    let double = |x: i32| -> i32 {x * 2};
    println!("{}", apply_twice(double, 5));  // 20
}
```

若希望**多次调用**且闭包内部**会修改**自己的状态，参数类型应为 **`F: FnMut(...)`**，且调用时传 **`&mut closure`**：

```rust
fn call_n_times<F>(f: &mut F, n: u32)
where
    F: FnMut() -> (),
{
    for _ in 0..n {
        f();
    }
}
```

### 1.4 闭包作为返回值与存储

闭包类型是**匿名**的，大小不固定，因此：

- **返回闭包**：通常返回 **`impl Fn(...)`** 或 **`Box<dyn Fn(...)>`**（当需要类型擦除、跨函数边界时）。
- **在结构体中存储闭包**：用泛型参数（如 `F: Fn(...)`）或 **`Box<dyn Fn(...)>`**。

**返回 impl Trait：**

```rust
fn make_adder<f>(n: i32) -> f
where f : Fn(i32) -> i32
{
    move |x| x + n  // move 使 n 被移进闭包，闭包可安全返回
}
let add_10 = make_adder(10);
println!("{}", add_10(2));  // 12
```

**返回 Box\<dyn Fn\>（类型擦除）：**

```rust
fn make_multiplier(factor: i32) -> Box<dyn Fn(i32) -> i32> {
    Box::new(move |x| x * factor)
}
```

**结构体中存闭包：**

```rust
struct Cacher<F>
where
    F: Fn(u32) -> u32,
{
    calculation: F,
    value: Option<u32>,
}
```

### 1.5 小结：何时用哪种捕获与 Fn

- 只读环境、可多次调用 → 自动不可变借用，实现 **Fn**。
- 修改环境、可多次调用 → 自动可变借用，实现 **FnMut**；调用方需 `&mut closure`。
- 环境被移动或消费、或只打算调一次 → **FnOnce**；加 **move** 可强制按值捕获，便于传线程或返回闭包。

---

## 2. Fn 系列 Trait（Fn / FnMut / FnOnce）

所有可调用的闭包都至少实现 **`FnOnce`**；根据捕获方式，还会实现 **`FnMut`** 或 **`Fn`**。这三个 trait 的关系是：`Fn` 是 `FnMut` 的子约束，`FnMut` 是 `FnOnce` 的子约束。下面先说明 **关联类型** 和 **Fn 的标准写法**，再分别看三个 trait 的用法，最后给出**为 Struct 用泛型存闭包并实现 Fn** 的完整例子。

### 2.1 关联类型与 Fn 的标准定义

**关联类型**是 trait 里由**实现者**指定的类型（如 **`Self::Item`**），调用方不写类型参数。Fn 系列里，**返回值类型** 就是 trait 的**关联类型 `Output`**，不是方法上的泛型。

**FnOnce**（可调用一次，消费 self）：

```rust
// 标准库中的概念形式（略去 extern "rust-call" 等细节）
trait FnOnce<Args> {
    type Output;
    fn call_once(self, args: Args) -> Self::Output;
}
```

**FnMut**（可多次调用，`&mut self`）：

```rust
trait FnMut<Args>: FnOnce<Args> {
    fn call_mut(&mut self, args: Args) -> Self::Output;  // Output 继承自 FnOnce
}
```

**Fn**（可多次调用，`&self`）：

```rust
trait Fn<Args>: FnMut<Args> {
    fn call(&self, args: Args) -> Self::Output;
}
```

你写的 **`closure(arg1, arg2)`** 是语法糖，编译器会翻译成对 **`call` / `call_mut` / `call_once`** 之一的调用，参数打包成元组，返回值类型就是 **`Self::Output`**。

### 2.2 FnOnce

- **调用方式**：`self` 按值（即闭包只能被调用一次）。
- **含义**：闭包可能消耗掉捕获的变量（移动进闭包或只调用一次），所以调用时“用掉”自己。
- **典型**：用了 `move` 且捕获了需要消费的值，或只打算调用一次时。

```rust
fn call_once<F>(f: F)
where
    F: FnOnce(),
{
    f();
}
```

### 2.3 FnMut

- **调用方式**：`&mut self`，可多次调用，每次可修改捕获。
- **含义**：闭包可能修改捕获的变量，但不消费自己。
- **典型**：捕获了 `&mut` 或 `mut` 变量，且没有 move 走需要消费的东西。

```rust
fn call_mut<F>(f: &mut F)
where
    F: FnMut(i32) -> i32,
{
    println!("{}", f(1));
    println!("{}", f(2));
}
```

### 2.4 Fn

- **调用方式**：`&self`，可多次调用，且不修改捕获。
- **含义**：闭包只对环境做不可变借用，最“温和”，可无限次调用。
- **典型**：只读捕获，或无捕获的纯函数式闭包。

```rust
fn call_many<F>(f: F, x: i32)
where
    F: Fn(i32) -> i32,
{
    println!("{}", f(x));
    println!("{}", f(x));
}
```

### 2.5 如何选择

- 只调用一次、或闭包内部会消费捕获 → 用 **`FnOnce`**。
- 需要多次调用且可能改捕获 → 用 **`FnMut`**。
- 需要多次调用且从不改捕获 → 用 **`Fn`**。

在泛型里写约束时，通常写**能满足需求的最宽松的一个**：例如能接受 `Fn` 就写 `F: Fn(...)`，这样 `FnMut`/`FnOnce` 的实现也能用；若必须可修改捕获，再写 `F: FnMut(...)`。

### 2.6 带泛型的函数：自己定义“接受闭包”的函数

“接受闭包”的本质是：参数类型是**实现了 Fn（或 FnMut / FnOnce）的某个类型**。用**泛型 + trait 约束**即可写出这样的函数，无需写死具体闭包类型。

例如：写一个 **“对 x 连续应用两次 f”** 的函数，要求 **f** 是“接受一个 i32、返回 i32”的可调用对象（闭包或实现了 Fn 的类型）：

```rust
/// 对 x 应用两次 f：先算 f(x)，再算 f(f(x))。
/// 泛型 F 约束为 Fn(i32) -> i32，表示“任何实现了 Fn(i32) -> i32 的类型”都可以传进来。
fn apply_twice<F>(f: F, x: i32) -> i32
where
    F: Fn(i32) -> i32,
{
    f(f(x))
}

fn main() {
    let double = |x| x * 2;
    let another = |x| x * 4;
    let result = apply_twice(double, 5);
    println!("{}", result);  // 20：先 5*2=10，再 10*2=20

    // 也可以传不同的闭包
    let add_ten = |x| x + 10;
    println!("{}", apply_twice(add_ten, 0));  // 20：0+10=10，10+10=20
}
```

**要点**：

- **`F: Fn(i32) -> i32`** 表示：`F` 是“可以像函数一样被调用、接受一个 i32、返回 i32”的类型，闭包自动满足。
- 函数体内直接 **`f(x)`** 调用即可，和调用普通函数写法一致。
- 这样我们就**自己定义了一个接受闭包的函数**；需要“只调一次”或“可修改状态”时，把约束改成 **`FnOnce`** 或 **`FnMut`** 即可。

### 2.7 为 Struct 实现 Fn：用泛型指明“接受闭包”

有时需要**在结构体里存一个闭包**，并让该结构体**像函数一样被调用**（例如缓存器、策略对象）。做法是：用**泛型参数**约束“这是一个闭包类型”，在结构体里保存该闭包，再为结构体 **impl FnOnce / FnMut / Fn**，在方法里**委托**给内部闭包的 `call_once` / `call_mut` / `call`。

下面例子：**`Caller<F>`** 存一个闭包 `F`，用 **`F: FnOnce(i32) -> i32`** 指明“接受一个 i32、返回 i32 的闭包”；为 **`Caller<F>`** 实现 **`FnOnce(i32) -> i32`**，在 **`call_once`** 里把参数转给内部的 **`self.f.call_once(args)`**。这样创建 **`Caller { f: |x| x + 1 }`** 后，既可以 **`caller(2)`** 调用，也可以显式写 **`call_once(caller, (2,))`**。

```rust
use std::ops::FnOnce;

/// 包装一个闭包，使结构体本身也可被“调用”。
/// 泛型 F 约束为 FnOnce(i32) -> i32，表示“接受 i32、返回 i32 的闭包”。
struct Caller<F>
where
    F: FnOnce(i32) -> i32,
{
    f: F,
}

impl<F> Caller<F>
where
    F: FnOnce(i32) -> i32,
{
    fn new(f: F) -> Self {
        Caller { f }
    }
}

// 为 Caller<F> 实现 FnOnce，委托给内部闭包；这样可以用 caller(arg) 语法调用
impl<F> FnOnce<(i32,)> for Caller<F>
where
    F: FnOnce(i32) -> i32,
{
    type Output = i32;
    fn call_once(self, args: (i32,)) -> Self::Output {
        (self.f)(args.0)
    }
}

fn main() {
    let c = Caller::new(|x| x + 10);
    println!("{}", c(2));  // 12，语法糖
    // 标准写法等价于：Caller::call_once(c, (2,))；注意 call_once 消费 self，所以 c 只能调一次
}
```

若希望结构体**可被多次调用**，应存 **`F: Fn(i32) -> i32`**（或 FnMut），并为 **`Caller<F>`** 实现 **`Fn`**（或 **`FnMut`**），在 **`call`** 里委托 **`self.f.call(args)`**。这样 **`caller(2)`** 就会多次合法。

---

## 3. 迭代器（Iterator）

迭代器负责按顺序产生一系列值，在 Rust 里通过 **`Iterator`** trait 和 **`IntoIterator`** 统一抽象；大量集合和流式数据都通过迭代器处理，且是**惰性**的，只有消费时才真正计算。

### 3.1 Iterator trait 与 next

**`Iterator`** 的核心是 **`next(&mut self) -> Option<Self::Item>`**：有下一个元素返回 `Some(item)`，没有则返回 `None`。实现类型需声明 **关联类型 `Item`**。

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    // 下面大量方法有默认实现：size_hint, map, filter, fold, collect, ...
}
```

只要实现 **`next`**，就会自动获得 `map`、`filter`、`fold`、`collect`、`sum`、`enumerate`、`zip` 等几十个方法。

### 3.2 三种迭代方式：iter / into_iter / iter_mut

对集合类型，通常有三种获取迭代器的方式：

| 方法                | 元素类型     | 是否消费集合   | 典型用途       |
| ----------------- | -------- | -------- | ---------- |
| **`iter()`**      | `&T`     | 否        | 只读遍历       |
| **`into_iter()`** | `T`      | 是（所有权移出） | 消费集合、取得所有权 |
| **`iter_mut()`**  | `&mut T` | 否        | 就地修改元素     |

```rust
let v = vec![1, 2, 3];
for x in v.iter() { println!("{}", x); }       // &i32，v 仍可用
for x in v.iter_mut() { *x += 1; }            // 修改 v
for x in v.into_iter() { println!("{}", x); } // 消费 v，之后 v 不可用
```

**`for x in collection`** 会调用 **`collection.into_iter()`**，因此 `for` 会消费集合（除非集合实现的是借用的 `IntoIterator`，如某些 slice 的 `into_iter()` 等价于 `iter()`）。

### 3.3 IntoIterator 与 for 循环

**`IntoIterator`** 表示“可被转换为迭代器”：

```rust
pub trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Self::Item>;
    fn into_iter(self) -> Self::IntoIter;
}
```

为类型实现 **`IntoIterator`** 后，就可以写 **`for x in value`**；`Vec`、数组、`Option`、`Result`、范围等都已实现。若希望 **`for x in &collection`** 不消费集合，需要为 **`&T`** 或 **`&mut T`** 实现 `IntoIterator`（如 `for x in &v` 使用 `(&v).into_iter()`）。

### 3.4 常见适配器（返回新迭代器）：用法与示例

适配器在原有迭代器上再包一层，返回**新的迭代器**；本身是**惰性**的，只有在你**消费**新迭代器（如 `collect`、`for`、`next`）时才会真正执行。

下面说明常用适配器的**用法**，并给出**可运行的例子和对应结果**。

| 方法            | 用法                         | 含义                          |
| ------------- | -------------------------- | --------------------------- |
| **map**       | `iter.map(\|x\| 表达式)`      | 对每个元素做变换，得到新迭代器，元素类型可变。     |
| **filter**    | `iter.filter(\|x\| 布尔表达式)` | 只保留谓词为 `true` 的元素。          |
| **take(n)**   | `iter.take(n)`             | 最多取前 n 个元素。                 |
| **skip(n)**   | `iter.skip(n)`             | 跳过前 n 个元素。                  |
| **enumerate** | `iter.enumerate()`         | 产生 `(索引, 元素)`，索引从 0 开始。     |
| **zip**       | `iter1.zip(iter2)`         | 两个迭代器逐对组合成 `(a, b)`，以较短者为准。 |
| **chain**     | `iter1.chain(iter2)`       | 先遍历 iter1，再遍历 iter2。        |

**示例（适配器 + collect，并写出结果）**：

```rust
fn main() {
    // map：每个数乘 2；filter：只保留大于 4 的；collect 收集成 Vec
    let v: Vec<i32> = (1..=5).map(|x| x * 2).filter(|x| *x > 4).collect();
    println!("{:?}", v);
    // 运行结果：[6, 8, 10]

    // enumerate：得到 (下标, 值)
    let with_idx: Vec<_> = (10..13).enumerate().collect();
    println!("{:?}", with_idx);
    // 运行结果：[(0, 10), (1, 11), (2, 12)]

    // zip：两个迭代器逐对配对
    let zipped: Vec<_> = (1..4).zip(vec!['a', 'b', 'c']).collect();
    println!("{:?}", zipped);
    // 运行结果：[(1, 'a'), (2, 'b'), (3, 'c')]

    // chain：前后两段范围接在一起
    let chained: Vec<_> = (1..3).chain(10..12).collect();
    println!("{:?}", chained);
    // 运行结果：[1, 2, 10, 11]

    // take：只取前 3 个；skip：跳过前 2 个
    let taken: Vec<_> = (1..=10).take(3).collect();
    let skipped: Vec<_> = (1..=10).skip(2).take(3).collect();
    println!("{:?}", taken);   // [1, 2, 3]
    println!("{:?}", skipped); // [3, 4, 5]
}
```

### 3.5 常见消费型方法（消耗迭代器得到值）：用法与示例

消费型方法会**推进**迭代器并**消耗**它，得到最终的一个值（或 `Option`/`Result`）。调用后迭代器通常已被部分或全部消费，不能指望再从头用。

| 方法            | 用法                                            | 返回类型           | 含义                                   |
| ------------- | --------------------------------------------- | -------------- | ------------------------------------ |
| **next**      | `iter.next()`                                 | `Option<Item>` | 取下一个元素，没有则 `None`。                   |
| **collect**   | `iter.collect()` 或 `iter.collect::<Vec<_>>()` | 由目标类型决定        | 把迭代器收集成集合（Vec、String、HashMap 等）。     |
| **sum**       | `iter.sum()`                                  | 数值类型           | 对所有元素求和。                             |
| **product**   | `iter.product()`                              | 数值类型           | 对所有元素求积。                             |
| **count**     | `iter.count()`                                | `usize`        | 元素个数（会消费完迭代器）。                       |
| **fold**      | `iter.fold(初始值, \|acc, x\| 新acc)`             | 与初始值同类型        | 从左到右折叠：第一次用初始值和第一个元素，之后用上一次结果和下一个元素。 |
| **find**      | `iter.find(\|x\| 布尔)`                         | `Option<Item>` | 第一个满足谓词的元素。                          |
| **any**       | `iter.any(\|x\| 布尔)`                          | `bool`         | 是否存在一个元素满足谓词。                        |
| **all**       | `iter.all(\|x\| 布尔)`                          | `bool`         | 是否全部元素都满足谓词。                         |
| **nth(n)**    | `iter.nth(n)`                                 | `Option<Item>` | 跳过前 n 个，返回第 n 个（从 0 计）。              |
| **last**      | `iter.last()`                                 | `Option<Item>` | 最后一个元素。                              |
| **partition** | `iter.partition(\|x\| 布尔)`                    | `(Vec, Vec)`   | 按谓词拆成两个 Vec。                         |

**示例（消费型方法，并写出结果）**：

```rust
fn main() {
    let sum: i32 = (1..=10).sum();
    println!("{}", sum);
    // 运行结果：55

    let product: i32 = (1..=5).product();
    println!("{}", product);
    // 运行结果：120

    let count = (0..10).count();
    println!("{}", count);
    // 运行结果：10

    // fold(初始值, 闭包)：acc 是累积值，x 是当前元素
    let folded: i32 = (1..=5).fold(0, |acc, x| acc + x);
    println!("{}", folded);
    // 运行结果：15  （0+1+2+3+4+5）

    let first_even = (1..).find(|x| *x % 2 == 0);
    println!("{:?}", first_even);
    // 运行结果：Some(2)

    let has_gt_5 = (1..=5).any(|x| x > 5);
    let all_positive = (1..=5).all(|x| x > 0);
    println!("{} {}", has_gt_5, all_positive);
    // 运行结果：false true

    let third: Option<i32> = (10..20).nth(2);
    println!("{:?}", third);
    // 运行结果：Some(12)  （跳过 10,11，取第 2 个即 12）

    let (even, odd): (Vec<_>, Vec<_>) = (1..=5).partition(|x| x % 2 == 0);
    println!("{:?} {:?}", even, odd);
    // 运行结果：[2, 4] [1, 3, 5]
}
```

### 3.6 创建迭代器的常见方式

- **范围**：`1..5`（不含 5）、`1..=5`（含 5）、`0..`（无穷，需配合 take 等）。
- **集合**：`vec.iter()`、`vec.into_iter()`、`vec.iter_mut()`；数组同理。
- **空 / 单元素**：`std::iter::empty()`、`std::iter::once(x)`。
- **重复**：`std::iter::repeat(x)`（无穷）、`iter::repeat(x).take(n)`。
- **FromIterator**：`FromIterator` trait 的 `from_iter` 常通过 **`.collect()`** 调用，把迭代器收集成 `Vec`、`String`、`HashMap` 等。

### 3.7 其他相关 trait

- **DoubleEndedIterator**：除 `next` 外还有 **`next_back`**，可从尾部取元素；`rev()` 依赖它。
- **ExactSizeIterator**：`len()` 精确，表示剩余元素个数已知。
- **FromIterator\<A\>**：类型可从 `Iterator<Item = A>` 构建，如 `Vec::from_iter(iter)` 或 `iter.collect::<Vec<_>>()`。

### 3.8 为自定义类型实现 Iterator 与 IntoIterator

实现 **`Iterator`** 只需提供 **`Item`** 和 **`next`**；若希望支持 **`for x in my_type`**，再为类型实现 **`IntoIterator`**（通常 `into_iter(self)` 返回 `Self` 的迭代器）。

```rust
struct Counter { count: u32, max: u32 }

impl Counter {
    fn new(max: u32) -> Self {
        Counter { count: 0, max }
    }
}

impl Iterator for Counter {
    type Item = u32;
    fn next(&mut self) -> Option<Self::Item> {
        if self.count < self.max {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}

impl IntoIterator for Counter {
    type Item = u32;
    type IntoIter = Counter;
    fn into_iter(self) -> Self::IntoIter {
        self
    }
}

fn main() {
    for n in Counter::new(5) {
        println!("{}", n);
    }
}
```

---

## 4. 生命周期（Lifetime）

生命周期标注解决的是**引用在多大范围内有效**的问题：编译器需要知道每个引用**不会比它指向的数据先失效**，否则就会报错或要求你显式写出 **生命周期参数**（如 **`'a`**）。

### 4.1 为什么需要生命周期

函数若**返回引用**，该引用要么来自某个参数，要么来自全局/静态数据。若来自参数，返回的引用**不能比该参数所引用的数据活得更久**，否则就会悬垂。编译器无法自动推断“返回的引用和哪个参数同寿”，因此要求你用 **`'a`** 等名字把这种关系标出来。

```rust
// 编译错误示例：返回的引用可能悬垂
// fn longest(x: &str, y: &str) -> &str {
//     if x.len() > y.len() { x } else { y }
// }
// 正确：用 'a 标明“返回的引用与 x、y 中较短命的那一个同寿”
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

### 4.2 生命周期参数 `'a` 的声明与使用

**声明**：在函数名或 `impl` 块后、泛型参数中用 **`'a`**、**`'b`** 等名字声明（习惯用短名字，且以 `'` 开头）。

**使用**：在引用类型上写成 **`&'a T`**、**`&'a mut T`**，表示“该引用的存活时间由 `'a` 描述”。

- **`'a`** 本身不是“某一段具体时间”，而是**一种约束关系**：所有带 **`'a`** 的引用“活一样长”（或更准确地说，编译器会取它们生命周期的交集，保证返回的引用不会比输入引用活得更久）。

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let s1 = String::from("short");
    let s2 = String::from("longer");
    let r = longest(s1.as_str(), s2.as_str());
    println!("{}", r);
}
```

### 4.3 多个生命周期参数：`'a` 与 `'b`

当有多个引用且它们的“寿命”可能不同时，可以声明多个生命周期参数，并在返回类型或结构体字段上标明“返回的引用与哪一个同寿”。

```rust
fn choose_first<'a, 'b>(x: &'a str, _y: &'b str) -> &'a str {
    x
}
```

这里返回的引用只与 `x` 同寿，与 `'b` 无关；若写成 `-> &'b str` 就错了，因为返回的是 `x`，不能比 `x` 活得更久。

### 4.4 结构体中的引用与生命周期

若结构体**持有引用**，必须在结构体名字后声明生命周期（如 **`struct Foo<'a>`**），并在每个引用字段类型上使用（如 **`&'a str`**）。含义：**该结构体实例不能比其引用字段指向的数据活得更久**。

```rust
struct ImportantExcerpt<'a> {
    part1: &'a str,
    part2: &'a str,
    part3: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().expect("找不到句号");
    let i = ImportantExcerpt { part: first_sentence };
    // i 不能比 first_sentence 引用的数据活得更久
}
```

**多个引用字段**可以共用同一个 `'a`，也可以分别用 `'a`、`'b`：

```rust
struct TwoRefs<'a, 'b> {
    first: &'a str,
    second: &'b str,
}
```

### 4.5 impl 块与方法上的生命周期

在 **`impl<'a>`** 和 **方法** 上也要声明并使用与结构体一致的生命周期参数。

```rust
impl<'a> ImportantExcerpt<'a> {
    fn level(&self) -> i32 {
        3
    }
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("{}", announcement);
        self.part
    }
}
```

若方法返回引用且来自 `self`，通常返回类型写 **`&'a ...`** 或依赖省略规则（见下）；若返回的引用来自某个参数，则需在方法上声明生命周期并标明关系，例如 **`fn foo<'b>(&'a self, s: &'b str) -> &'b str`**。

### 4.6 生命周期省略规则（elision）

在部分**固定模式**下，编译器允许不写生命周期，按以下规则**自动推断**（仅用于函数与方法的签名）：

1. **每个引用参数**各自得到一个**独立的**生命周期参数。
2. 若**只有一个**输入生命周期参数，则该生命周期被赋给**所有**输出生命周期。
3. 若有**多个**输入生命周期参数，但其中一个是 **`&self`** 或 **`&mut self`**，则 **`self`** 的生命周期被赋给所有输出生命周期。

**不满足**上述规则时，必须手写生命周期（例如多个输入且无 `self`、或输出引用与不同参数有关）。

```rust
fn first_word(s: &str) -> &str {
    // 规则 1：s 得到 'a；规则 2：唯一输入，输出也是 'a
    s.split_whitespace().next().unwrap_or("")
}
```

### 4.7 静态生命周期 `'static`

**`'static`** 表示引用在**整个程序运行期间**都有效。例如：

- 字符串字面量的类型是 **`&'static str`**。
- 若某数据真正是“程序全程存在”（如静态变量、字面量），其引用才应标为 **`'static`**。

不要为了消除编译错误就随意把引用标成 **`'static`**；只有确定生命周期确实是全局的才用。

```rust
let s: &'static str = "hello";
fn get_static() -> &'static str {
    "world"
}
```

### 4.8 小结：生命周期 `'a` 要点

- **`'a`** 是“名字”，在 **`<>`** 里声明，在 **`&'a T`** 等处使用，表示“该引用的有效期”。
- 函数/方法返回引用时，返回类型上的 **`'a`** 必须与某个**输入引用**的生命周期一致（或更短），不能无中生有。
- 结构体持引用时，必须在结构体上写 **`struct Foo<'a>`**，并在字段类型上用 **`'a`**。
- **impl** 和**方法**上也要写 **`impl<'a> Foo<'a>`** 并在方法签名里按需使用 **`'a`**。
- 能由编译器按**省略规则**推断的可以不写；推断不出时再显式标注 **`'a`**、**`'b`** 等。

---

## 5. `dyn` 与 `ref` 关键字

在接触智能指针和 trait 对象之前，先明确两个关键字：**`dyn`**（用于类型）和 **`ref`**（用于模式）。它们与“引用”“所有权”紧密相关，后面讲 **Box\<dyn Trait\>** 时会反复用到 **dyn**。

### 5.1 `dyn`：trait 对象与动态分发

**`dyn`** 是 **dynamic** 的缩写，用在类型里表示“**在运行时才确定具体类型**”的 trait 实现。写成 **`dyn Trait`**，读作“实现了 Trait 的某种类型”，但**不写出具体类型名**，只约定“能调用该 Trait 的方法”。这种类型叫做 **trait 对象**。

- **动态分发**：具体调用哪个类型的实现，在运行时通过 **虚表（vtable）** 查找，而不是编译期单态化。
- **类型擦除**：编译期只知道“它是某个实现了 Trait 的类型”，无法知道大小，因此 **`dyn Trait` 不能单独作为变量类型**，必须出现在**引用**或**智能指针**后面，例如：
  - **`&dyn Trait`**：借用一个 trait 对象；
  - **`Box<dyn Trait>`**：堆上分配 trait 对象，拥有所有权；
  - **`Rc<dyn Trait>`** / **`Arc<dyn Trait>`**：共享同一个 trait 对象。

这样可以在**同一集合里放多种具体类型**（如 `Vec<Box<dyn Draw>>`），或函数返回“任意实现某 Trait 的类型”而不写死具体类型。下一节智能指针会具体讲 **Box\<dyn Trait\>** 等用法。

### 5.2 `ref`：在模式中按引用绑定

在 **`let`**、**`match`**、**`if let`**、**`while let`** 等模式中，默认会**移动**被匹配的值。若只想**得到引用、不移动**，在模式里加 **`ref`**（不可变引用）或 **`ref mut`**（可变引用）。

**`let` 绑定：**

```rust
let x = 42;
let ref r = x;   // r 的类型是 &i32，x 未被移动
assert_eq!(*r, 42);

let mut y = 10;
let ref mut rm = y;  // rm 是 &mut i32
*rm += 1;
assert_eq!(y, 11);
```

**`match` 中避免移动：**

```rust
let s = String::from("hello");
match s.chars().next() {
    Some(ref c) => println!("首字符: {}", c),  // c 是 &char，不消费迭代器项
    None => {}
}
// 若写成 Some(c)，则 c 会取得所有权（若类型非 Copy）
```

**在结构体/元组模式中：**

```rust
let point = (1, 2);
let (ref a, ref b) = point;  // a、b 为 &i32，point 仍可用
```

**`ref` 与 `&` 的异同**

- **相同点**：二者都能得到“引用”类型。在简单 `let` 下，**`let ref r = x`** 与 **`let r = &x`** 效果等价：`r` 的类型都是 `&i32`，`x` 未被移动。
- **不同点**：
  - **`&`** 是**表达式**里的“取引用”运算符，用在等号**右边**或函数实参等：`let r = &x`、`foo(&v)`。在**模式**里出现时，**`&`** 表示“匹配一个引用”或“解构引用”，例如 **`let &y = &42`** 里 `y` 绑定的是**引用指向的值**（即 `i32`），而不是引用本身。
  - **`ref`** 是**仅用于模式**的绑定方式，用在等号**左边**或 match 的臂里：表示“**不要移动**被匹配的值，用引用绑定到变量”。例如 **`match opt { Some(ref v) => ... }`** 里 `v` 是 `&T`，原 `opt` 里的值未被移出；若写 **`Some(&v)`** 则要求 `Option` 里装的是引用，且 `v` 绑定的是引用指向的内容。

因此：在**表达式**里用 **`&`** 取引用；在**模式**里若只想“拿引用、不移动”，用 **`ref`** / **`ref mut`**；若模式要“匹配引用并解出内部值”，用 **`&`**。

小结：**`ref`** / **`ref mut`** 在模式里表示“按引用绑定”，不夺走所有权，适合在 match/if let 里只读或只改一部分、且希望保留原值不被移动的场景。

---

## 6. 智能指针概览

Rust 中“智能指针”通常指实现了 **`Deref`**（有时还有 **`Drop`**）的类型，行为像指针但带有额外语义。下面先说明**智能指针与 `dyn Trait` 的搭配**，再按单线程 → 多线程 → 特殊用途把常见智能指针过一遍。

### 6.1 `dyn` 与 trait 对象（在智能指针中的用法）

**`dyn Trait`** 的含义与用法已在前节（5.1）说明。这里强调：trait 对象**必须**放在引用或智能指针后面，因为实现 Trait 的类型大小不固定，栈上无法为“dyn Trait”分配固定空间。

- **`&dyn Trait`**：借用一个 trait 对象，不拥有。
- **`Box<dyn Trait>`**：在堆上放一个 trait 对象，拥有所有权，大小固定（指针 + 虚表指针）。
- **`Rc<dyn Trait>`** / **`Arc<dyn Trait>`**：多份共享同一个 trait 对象。

这样就能在**同一集合里存放多种具体类型**（只要它们都实现同一个 Trait），例如 `Vec<Box<dyn Draw>>` 里可以同时放不同的图形类型。

### 6.2 智能指针与 `Box<dyn Trait>` 等

智能指针常用来**持有 trait 对象**，因为 trait 对象本身是“不定大小”的，必须放在指针后面：

- **`Box<dyn Trait>`**：最常用。在堆上分配“实现了 Trait 的某个具体值”，栈上只保留固定大小的 `Box`（指针 + 虚表），适合作为字段、返回值或集合元素，例如 **`Box<dyn Fn(i32) -> i32>`**、**`Box<dyn Error>`**。
- **`Rc<dyn Trait>`** / **`Arc<dyn Trait>`**：需要多份共享同一 trait 对象时用（单线程用 Rc，多线程用 Arc）。
- **`&dyn Trait`**：不拥有，只是借用，不涉及智能指针，但同样是“持有”trait 对象的一种方式。

示例：用 **`Box<dyn Trait>`** 存多种实现同一 trait 的类型：

```rust
trait Shape {
    fn area(&self) -> f64;
}
struct Circle { r: f64 }
struct Rect { w: f64, h: f64 }
impl Shape for Circle {
    fn area(&self) -> f64 { std::f64::consts::PI * self.r * self.r }
}
impl Shape for Rect {
    fn area(&self) -> f64 { self.w * self.h }
}

fn main() {
    let shapes: Vec<Box<dyn Shape>> = vec![
        Box::new(Circle { r: 1.0 }),
        Box::new(Rect { w: 2.0, h: 3.0 }),
    ];
    for s in &shapes {
        println!("{}", s.area());
    }
    // 输出：3.141592653589793  和  6
}
```

后面讲 **Box**、**Rc** 等时，**`Box<dyn T>`**、**`Rc<dyn T>`** 指的就是“用该智能指针装一个 trait 对象”的用法。

---

## 7. Box\<T\>

**作用**：在堆上分配一个 `T`，拥有其所有权；用于递归类型、大对象、或**需要固定大小的 trait 对象**（即 **`Box<dyn Trait>`**，见 6.1 节）。

```rust
let b = Box::new(42);
// 存 trait 对象：多种类型统一成 Box<dyn Trait>
let f: Box<dyn Fn(i32) -> i32> = Box::new(|x| x + 1);
println!("{}", f(2));  // 3
```

- **Deref**：`*b` 得到 `T`。
- **Drop**：离开作用域时释放堆内存。

---

## 8. Rc\<T\>（Reference Counting）

**作用**：**共享所有权**，通过引用计数在多个地方持有同一份数据；**仅用于单线程**。

```rust
use std::rc::Rc;
let a = Rc::new(String::from("hello"));
let b = Rc::clone(&a);
let c = Rc::clone(&a);
// 三个 Rc 指向同一块数据，计数为 3
```

- **Rc::clone(&rc)** 只增加计数，不深拷贝数据。
- **Drop**：计数减一，减到 0 时释放内部数据。
- 不可变共享；若需要内部可变，配合 **RefCell**（见下）。

---

## 9. RefCell\<T\> 与内部可变性

**作用**：在**不可变引用**的前提下，在运行时做**可变借用**；借用规则在运行时检查，违反则 **panic**。**仅用于单线程**。

```rust
use std::cell::RefCell;
let x = RefCell::new(42);
*x.borrow_mut() += 1;
println!("{}", *x.borrow());
```

- **borrow()**：不可变借用，返回 `Ref<T>`。
- **borrow_mut()**：可变借用，返回 `RefMut<T>`。
- 同一时刻要么多个 `borrow`，要么一个 `borrow_mut`，否则运行时 panic。

常见组合：**`Rc<RefCell<T>>`** —— 多份 Rc 共享同一数据，且每份都能在需要时通过 RefCell 做可变访问。

### Cell\<T\>

**作用**：单线程下的**内部可变性**，不提供引用，只通过 **`get`**（要求 `T: Copy`）或 **`set`** 做整体读/写；无运行时借用检查，适合简单类型（如 `Cell<i32>`）。

```rust
use std::cell::Cell;
let c = Cell::new(42);
c.set(43);
println!("{}", c.get());
```

---

## 10. Weak\<T\>（弱引用）

**作用**：与 **Rc** 配合，形成**弱引用**，不增加引用计数，用于打破循环引用，避免内存泄漏。

```rust
use std::rc::{Rc, Weak};
let strong = Rc::new(42);
let weak = Rc::downgrade(&strong);
drop(strong);
assert!(weak.upgrade().is_none());  // 强引用已无，upgrade 得 None
```

- **Rc::downgrade(&rc)** 得到 `Weak<T>`。
- **weak.upgrade()** 得到 `Option<Rc<T>>`，若强计数已为 0 则为 `None`。

在树形或图结构里，子节点用 `Rc` 指父、父节点用 `Weak` 指子，可避免循环引用。

---

## 11. Arc\<T\>（Atomic Reference Counting）

**作用**：**线程安全**的引用计数，多线程下共享同一份数据时使用。

```rust
use std::sync::Arc;
use std::thread;
let a = Arc::new(0);
let a2 = Arc::clone(&a);
thread::spawn(move || {
    // 在子线程中使用 a2
});
```

- API 与 `Rc` 类似（`Arc::clone`、`Arc::new`），但内部用原子操作维护计数。
- 若需在多线程下“共享且可改”，通常用 **Arc\<Mutex\<T\>\>** 或 **Arc\<RwLock\<T\>\>**。

---

## 12. Mutex\<T\>（互斥锁）

**作用**：**互斥访问**，同一时刻只允许一个线程持有锁，用于保护共享可变状态。

```rust
use std::sync::Mutex;
let m = Mutex::new(0);
{
    let mut guard = m.lock().unwrap();
    *guard += 1;
}
```

- **lock()** 返回 **Result\<MutexGuard\<T\>, PoisonError\<...\>\>**；拿到 guard 后通过解引用修改数据，离开作用域时自动释放锁。
- 若持有锁的线程 panic，锁会“中毒”，其他线程再 `lock` 会得到 `Err`（可选择 `into_inner` 恢复）。

---

## 13. RwLock\<T\>（读写锁）

**作用**：**多读单写**——多个读者可同时持有，或一个写者独占；适合读多写少的共享数据。

```rust
use std::sync::RwLock;
let rw = RwLock::new(0);
{
    let r1 = rw.read().unwrap();
    let r2 = rw.read().unwrap();
}
{
    let mut w = rw.write().unwrap();
    *w += 1;
}
```

- **read()**：读锁，可多个并存。
- **write()**：写锁，独占；与读锁、写锁互斥。

---

## 14. Cow\<B\>（Clone on Write）

**作用**：写时复制——可能持有借用或拥有数据；仅在**需要修改**时才做克隆，避免不必要的拷贝。

```rust
use std::borrow::Cow;
fn process(s: &str) -> Cow<str> {
    if s.contains("bad") {
        Cow::Owned(s.replace("bad", "good"))
    } else {
        Cow::Borrowed(s)
    }
}
```

- **Cow::Borrowed**：包装引用。
- **Cow::Owned**：包装拥有所有权的数据。
- 常见于字符串/切片处理 API（如 `std::path::Path` 的某些方法返回 `Cow<Path>`）。

---

## 15. Pin\<P\>

**作用**：**固定**指针指向的值，禁止其被移动；用于自引用结构、**async/await** 中的 Future 等，保证安全。

```rust
use std::pin::Pin;
let mut x = 5;
let p = Pin::new(&mut x);
// 被 Pin 住后，不能通过 p 把 x move 走
```

- 通常不会直接手写很多 `Pin` 代码，但在理解 async、Future、自引用时会出现。
- 规则：若类型实现了 **`Unpin`**（大多数类型默认实现），则即使被 `Pin` 住也可以安全移动；若未实现 `Unpin`，则不能移出 `Pin`。

---

## 16. 智能指针的共性：Deref 与 Drop

- **Deref**：智能指针可被**解引用**为内部类型（`*ptr`、方法调用时的自动解引用）。
- **Drop**：离开作用域时**自动清理**（释放堆内存、释放锁、减少引用计数等）。

自定义智能指针时，实现 **`Deref`** 和 **`Drop`** 即可融入 Rust 的指针与资源管理习惯。

---

## 17. 小结表

| 主题 | 要点 |
| --- | --- |
| **闭包** | 捕获环境、`move`、作为参数/返回值；与 Fn 系列配合。 |
| **Fn / FnMut / FnOnce** | 按调用次数与是否修改捕获选择；约束写能满足需求的最宽松即可。 |
| **迭代器** | `Iterator`（`next`）、`IntoIterator`；`iter`/`into_iter`/`iter_mut`；消费型与适配器方法。 |
| **生命周期** | 标注引用关系、结构体持引用、省略规则、`'static`。 |
| **dyn** | trait 对象类型，动态分发；必须放在引用或智能指针后，如 `&dyn T`、`Box<dyn T>`。 |
| **ref / ref mut** | 在模式中按引用绑定，不移动值；用于 let、match、if let 等。 |
| **Box** | 堆分配、单一所有权；**Box<dyn Trait>** 常用来存 trait 对象。 |
| **Rc / Weak** | 单线程共享所有权、引用计数；Weak 破环、防泄漏。 |
| **RefCell / Cell** | 单线程内部可变性；RefCell 借出引用，Cell 仅 get/set。 |
| **Arc** | 多线程引用计数；常与 Mutex/RwLock 组合。 |
| **Mutex / RwLock** | 多线程互斥/读写锁。 |
| **Cow** | 写时复制，减少不必要的克隆。 |
| **Pin** | 禁止移动，用于自引用与 async。 |
