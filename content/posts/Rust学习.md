---
title: "Rust学习"
date: 2021-11-30T9:38:03+08:00
draft: false
author: "gq"
---

# Rust

### 静态类型语言

Rust是静态类型语言，在编译时就必须知道所有变量的类型。

**动态语言**是指**运行期间**可以改变其结构的语言：例如增加或者删除函数、对象、甚至代码。比如JavaScript、Objective-C、Ruby、Python等，而C、C++等语言则不属于动态语言。**静态语言**与动态语言相反，在运行时不能改变其结构。尽管静态语言可以通过复杂的手段实现动态语言的特性，但是动态语言提供了直接的方法实现语言的动态性。

**动态类型语言**是指在运行期间才去做数据类型检查的语言。在用动态语言编程时，不用给变量指定数据类型，第一次赋值给变量时，在内部将数据类型记录下来。JavaScript、Ruby、Python是典型的动态类型语言。**静态类型语言**与动态类型语言刚好相反，它的数据类型检查发生在在编译阶段，也就是说在写程序时要声明变量的数据类型。C/C++、C#、Java都是静态类型语言的典型代表。

**大部分动态语言是动态类型的，但是不是所有都是。**

## 一、依赖管理--cargo 

### source 

默认路径  ～/.cargo/config

> [source.crates-io]
> registry = "https://github.com/rust-lang/crates.io-index"
> replace-with = 'ustc'
> [source.ustc]
> registry = "git://mirrors.ustc.edu.cn/crates.io-index"

### cargo.lock

> cargo.lock相当于依赖锁，除非在cargo.toml中手动指定了其他依赖版本，否则编译时依赖版本始终保持不变，即项目构建可重现

### cargo update

> cargo  update会忽略cargo.lock，并计算出所有符合 *Cargo.toml* 声明的最新版本。如果成功了，Cargo 会把这些版本写入 *Cargo.lock* 文件。
>
> **注意：（当前版本0.5.5）**
>
> Cargo 默认只会寻找大于 `0.5.5` 而小于 `0.6.0` 的版本。如果 `rand` crate 发布了两个新版本，`0.5.6` 和 `0.6.0`，会更新到0.5.6，如果想要使用 `0.6.0` 版本的 `rand` 或是任何 `0.6.x` 系列的版本，需要修改cargo.toml

### cargo doc --open

> 以文档形式查看本地项目的所有依赖

### Rust 有一个静态强类型系统，同时也有类型推断

> 整数类型的变量默认为i32，不能和string类型进行比较，但是可以和u32类型作比较（rust的类型推断）

## 二、数据类型

### 常量

> 例如：const MAX_POINTS: u32 = 100_000;
>
> 在声明它的作用域之中，常量在整个程序生命周期中都有效，这使得常量可以作为多处代码使用的全局范围的值，例如一个游戏中所有玩家可以获取的最高分或者光速。

### 变量

> ### mut与隐藏
>
> **隐藏**（使用let）：可以改变值的类型并且复用相同的变量名，实际上创建了一个拥有新类型的新变量。隐藏了之前创建的同名变量。
>
> **mut**：如果给可变变量重新赋值，但是前面不加let，只能附相同类型的值

### 标量类型

四种标量类型：整形、浮点型、布尔类型、字符类型

整型：默认类型i32

浮点型：默认类型f64

字符类型：

Rust 的 `char` 类型的大小为四个字节(four bytes)，并代表了一个 Unicode 标量值（Unicode Scalar Value），这意味着它可以比 ASCII 表示更多内容。在 Rust 中，拼音字母（Accented letters），中文、日文、韩文等字符，emoji（绘文字）以及零长度的空白字符都是有效的 `char` 值。Unicode 标量值包含从 `U+0000` 到 `U+D7FF` 和 `U+E000` 到 `U+10FFFF` 在内的值。不过，“字符” 并不是一个 Unicode 中的概念，所以人直觉上的 “字符” 可能与 Rust 中的 `char` 并不符合

### 复合类型

Rust 有两个原生的复合类型：元组、数组

#### 元组

元组是一个将多个其他类型的值组合进一个复合类型的主要方式。元组长度固定：一旦声明，其长度不会增大或缩小。

例如：

>  fn main() {    
>
>    let tup = (500, 6.4, 1);     
>
>    let (x, y, z) = tup;     //解构
>
>    println!("The value of y is: {}", y);   //y=6.4
>
> }

> fn main() {    
>
> let x: (i32, f64, u8) = (500, 6.4, 1);     
>
> let five_hundred = x.0;         //元组的第一个索引值是 0
>
> let six_point_four = x.1;    
>
>  let one = x.2; 
>
> }        

#### 数组

与元组不同，数组中的每个元素的类型必须相同；

Rust 中的数组是固定长度的：一旦声明，它们的长度不能增长或缩小。

数组是一整块分配在栈上的内存，可以使用索引来访问数组的元素。

例如：

> let a = [1, 2, 3, 4, 5];
>
> let a: [i32; 5] = [1, 2, 3, 4, 5]; //类型+长度
>
> let a = [3; 5];

vector 类型是标准库提供的一个 **允许** 增长和缩小长度的类似数组的集合类型。当不确定是应该使用数组还是 vector 的时候，你可能应该使用 vector

## 三、函数与控制流

#### 命名规则：

Rust 代码中的函数和变量名使用 *snake case* 规范风格。在 snake case 中，所有字母都是小写并使用下划线分隔单词

> println!("The value of x is: {}", x);
>
> `println!` 为宏

因为 Rust 是一门基于表达式（expression-based）的语言，这是一个需要理解的（不同于其他语言）重要区别

`let y = 6` 语句并不返回值，所以没有可以绑定到 `x` 上的值。这与其他语言不同，例如 C 和 Ruby，它们的赋值语句会返回所赋的值。在这些语言中，可以这么写 `x = y = 6`，这样 `x` 和 `y` 的值都是 `6`；Rust 中不能这样写

```rust
fn main() {
    let y = {
        let x = 3;
        x + 1
    };

    println!("The value of y is: {}", y);
}
其中：
{
    let x = 3;
    x + 1
}   
为代码块
x+1为表达式，表达式的结尾没有分号
如果在表达式的结尾加上分号，它就变成了语句，而语句不会返回值
```

#### 具有返回值的函数

```rust
fn five() -> i32 {
    5
}

fn main() {
    let x = five();

    println!("The value of x is: {}", x);
}
//在 Rust 中，函数的返回值等同于函数体最后一个表达式的值。使用 return 关键字和指定值，可从函数中提前返回；但大部分函数隐式的返回最后的表达式
```

#### 控制流

```rust
fn main() {
    let condition = true;
    let number = if condition {
        5
    } else {
        6
    };

    println!("The value of number is: {}", number);
}
//if的每个分支的可能的返回值都必须是相同类型
//Rust需要在编译时就确切的知道 number 变量的类型，这样它就可以在编译时验证在每处使用的 number 变量的类型是有效的。Rust 并不能够在 number 的类型只能在运行时确定的情况下工作；这样会使编译器变得更复杂而且只能为代码提供更少的保障，因为它不得不记录所有变量的多种可能的类型。
--->rust为静态类型语言
```

```rust
fn main() {
    let a = [10, 20, 30, 40, 50];

    for element in a.iter() {
        println!("the value is: {}", element);
    }
}
//遍历数组
fn main() {
    for number in (1..4).rev() {
        println!("{}!", number);
    }
    println!("LIFTOFF!!!");
}
//输出3 2 1
```

## 四、所有权

所有权（系统）是 Rust 最为与众不同的特性，它让 Rust 无需垃圾回收（garbage collector）即可保障内存安全

#### 所有权规则

> 1. Rust 中的每一个值都有一个被称为其 **所有者**（*owner*）的变量。
> 2. 值在任一时刻有且只有一个所有者。
> 3. 当所有者（变量）离开作用域，这个值将被丢弃。

前面介绍的类型都是存储在栈上的并且当离开作用域时被移出栈

String类型被分配到堆上

```rust
let s = String::from("hello");
```

在有 **垃圾回收**（*garbage collector*，*GC*）的语言中， GC 记录并清除不再使用的内存，而我们并不需要关心它；Rust 采取了一个不同的策略：内存在拥有它的变量离开作用域后就被自动释放。

rsut在作用域结尾，自动调用drop，释放内存

#### 变量与数据交互的方式（一）：移动

```rust
let s1 = String::from("hello");
let s2 = s1;
//只是在栈上复制
println!("{}, world!", s1);
//为防止二次释放，导致内存污染，rust认为s1已经无效，s1已经被移动到了s2，作用域结束时，rust只需要释放s2即可
```

Rust 永远也不会自动创建数据的 “深拷贝”。因此，任何 **自动** 的复制可以被认为对运行时性能影响较小。

#### 变量与数据交互的方式（二）：克隆

```rust
let s1 = String::from("hello");
let s2 = s1.clone();
//在堆上复制
println!("s1 = {}, s2 = {}", s1, s2);
```

#### 只在栈上的数据：拷贝

```rust
let x = 5;
let y = x;

println!("x = {}, y = {}", x, y);
//像整型这样的在编译时已知大小的类型被整个存储在栈上，所以拷贝其实际的值是快速的。这意味着没有理由在创建变量 y 后使 x 无效。换句话说，这里没有深浅拷贝的区别.
```

那么什么类型是 `Copy` 的呢？可以查看给定类型的文档来确认，不过作为一个通用的规则，任何简单标量值的组合可以是 `Copy` 的，不需要分配内存或某种形式资源的类型是 `Copy` 的

> - 所有整数类型，比如 `u32`。
> - 布尔类型，`bool`，它的值是 `true` 和 `false`。
> - 所有浮点数类型，比如 `f64`。
> - 字符类型，`char`。
> - 元组，当且仅当其包含的类型也都是 `Copy` 的时候。比如，`(i32, i32)` 是 `Copy` 的，但 `(i32, String)` 就不是。

#### 所有权与函数

```rust
fn main() {
    let s = String::from("hello");  // s 进入作用域

    takes_ownership(s);             // s 的值移动到函数里 ...
                                    // ... 所以到这里不再有效

    let x = 5;                      // x 进入作用域

    makes_copy(x);                  // x 应该移动函数里，
                                    // 但 i32 是 Copy 的，所以在后面可继续使用 x

} // 这里, x 先移出了作用域，然后是 s。但因为 s 的值已被移走，
  // 所以不会有特殊操作

fn takes_ownership(some_string: String) { // some_string 进入作用域
    println!("{}", some_string);
} // 这里，some_string 移出作用域并调用 `drop` 方法。占用的内存被释放

fn makes_copy(some_integer: i32) { // some_integer 进入作用域
    println!("{}", some_integer);
} // 这里，some_integer 移出作用域。不会有特殊操作
```

#### 引用与借用

 & 符号就是 **引用**，它们允许你使用值但不获取其所有权

当函数使用引用而不是实际值作为参数，无需返回值来交还所有权，因为就不曾拥有所有权。

我们将获取引用作为函数参数称为 **借用**（*borrowing*）。

正如变量默认是不可变的，引用也一样。（默认）不允许修改引用的值。

##### 可变引用：

```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

不过可变引用有一个很大的限制：在特定作用域中的特定数据只能有一个可变引用

```rust
-----------------------------------------
let mut s = String::from("hello");

let r1 = &mut s;
let r2 = &mut s;
//编译不通过，存在数据竞争
println!("{}, {}", r1, r2);
let mut s = String::from("hello");

-----------------------------------------
{
    let r1 = &mut s;

} // r1 在这里离开了作用域，所以我们完全可以创建一个新的引用

let r2 = &mut s;
//可以使用大括号来创建一个新的作用域，以允许拥有多个可变引用，只是不能 同时 拥有
-----------------------------------------
let mut s = String::from("hello");

let r1 = &s; // 没问题
let r2 = &s; // 没问题
let r3 = &mut s; // 大问题

println!("{}, {}, and {}", r1, r2, r3);
//不能在拥有不可变引用的同时拥有可变引用;多个不可变引用是可以的，因为没有哪个只能读取数据的人有能力影响其他人读取到的数据。
-----------------------------------------
let mut s = String::from("hello");

let r1 = &s; // 没问题
let r2 = &s; // 没问题
println!("{} and {}", r1, r2);
// 此位置之后 r1 和 r2 不再使用

let r3 = &mut s; // 没问题
println!("{}", r3);
//注意一个引用的作用域从声明的地方开始一直持续到最后一次使用为止
//不可变引用 r1 和 r2 的作用域在 println! 最后一次使用之后结束，这也是创建可变引用 r3 的地方。它们的作用域没有重叠，所以代码是可以编译的。
-----------------------------------------
```

##### 悬垂引用：

在具有指针的语言中，很容易通过释放内存时保留指向它的指针而错误地生成一个 **悬垂指针**（*dangling pointer*），所谓悬垂指针是其指向的内存可能已经被分配给其它持有者。相比之下，在 Rust 中编译器确保引用永远也不会变成悬垂状态：当你拥有一些数据的引用，编译器确保数据不会在其引用之前离开作用域。

```rust
fn main() {
    let reference_to_nothing = dangle();
}

fn dangle() -> &String {
    let s = String::from("hello");

    &s
}
//因为 s 是在 dangle 函数内创建的，当 dangle 的代码执行完毕后，s 将被释放。不过我们尝试返回它的引用。这意味着这个引用会指向一个无效的 String，这可不对！Rust 不会允许我们这么做。
//解决
fn no_dangle() -> String {
    let s = String::from("hello");

    s
}
```

##### 引用规则

> - 在任意给定时间，**要么** 只能有一个可变引用，**要么** 只能有多个不可变引用。
> - 引用必须总是有效的。

#### Slice类型

另一个没有所有权的数据类型是 *slice*。slice 允许你引用集合中一段连续的元素序列，而不用引用整个集合。

```rust
//“字符串 slice” 的类型声明写作 &str
//Slice存储第一个集合元素的引用和一个集合总长度
fn first_word(s: &String) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

字符串字面值就是slice

```rust
let s = "Hello, world!";
//字符串字面值被储存在二进制文件
//这里 s 的类型是 &str：它是一个指向二进制程序特定位置的 slice。这也就是为什么字符串字面值是不可变的；&str 是一个不可变引用
```

## 五、结构体

### 定义并实例化结构体

###### 变量与字段同名时的字段初始化简写语法

```rust
fn build_user(email: String, username: String) -> User {
    User {
        email,
        username,
        active: true,
        sign_in_count: 1,
    }
}
```

###### 使用结构体更新语法从其他实例创建实例

```rust
let user2 = User {
    email: String::from("another@example.com"),
    username: String::from("anotherusername567"),
    active: user1.active,
    sign_in_count: user1.sign_in_count,
};

let user2 = User {
    email: String::from("another@example.com"),
    username: String::from("anotherusername567"),
    ..user1
};
```

###### 使用没有命名字段的元组结构体来创建不同的类型

```rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);

let black = Color(0, 0, 0);
let origin = Point(0, 0, 0);
```

###### 没有任何字段的类单元结构体

我们也可以定义一个没有任何字段的结构体！它们被称为 **类单元结构体**（*unit-like structs*）因为它们类似于 `()`，即 unit 类型。类单元结构体常常在你想要在某个类型上实现 trait 但不需要在类型中存储数据的时候发挥作用

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let rect1 = Rectangle { width: 30, height: 50 };

    println!("rect1 is {:#?}", rect1);
}
//输出
rect1 is Rectangle {
    width: 30,
    height: 50,
}

```

### 方法语法

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }
}

fn main() {
    let rect1 = Rectangle { width: 30, height: 50 };

    println!(
        "The area of the rectangle is {} square pixels.",
        rect1.area()
    );
}
//Rust 有一个叫 自动引用和解引用（automatic referencing and dereferencing）的功能,当使用 object.something() 调用方法时，Rust 会自动为 object 添加 &、&mut 或 * 以便使 object 与方法签名匹配。也就是说，这些代码是等价的：
p1.distance(&p2);
(&p1).distance(&p2);
```

带有更多参数的方法

```rust
impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }

    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }
}
fn main() {
    let rect1 = Rectangle { width: 30, height: 50 };
    let rect2 = Rectangle { width: 10, height: 40 };
    let rect3 = Rectangle { width: 60, height: 45 };

    println!("Can rect1 hold rect2? {}", rect1.can_hold(&rect2));
    println!("Can rect1 hold rect3? {}", rect1.can_hold(&rect3));
}
```

###### 关联函数

```rust
impl Rectangle {
    fn square(size: u32) -> Rectangle {
        Rectangle { width: size, height: size }
    }
}
let sq = Rectangle::square(3);
//这个方法位于结构体的命名空间中：:: 语法用于关联函数和模块创建的命名空间
```

每个结构体都允许拥有多个 `impl` 块

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

## 六、枚举与模式匹配

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}
//这个枚举有四个含有不同类型的成员：
Quit 没有关联任何数据。
Move 包含一个匿名结构体。
Write 包含单独一个 String。
ChangeColor 包含三个 i32。
```

### Option枚举和其相对于空值的优势

Rust 并没有空值，不过它确实拥有一个可以编码存在或不存在概念的枚举。这个枚举是 `Option<T>`，而且它[定义于标准库中](https://doc.rust-lang.org/std/option/enum.Option.html)，如下:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

为了拥有一个可能为空的值，你必须要显式的将其放入对应类型的 `Option<T>` 中。接着，当使用这个值时，必须明确的处理值为空的情况。只要一个值不是 `Option<T>` 类型，你就 **可以** 安全的认定它的值不为空

## 七、使用包、Crate和模块管理不断增长的项目

### 包中所包含的内容由几条规则来确立

> 一个包中至多 **只能** 包含一个库 crate(library crate)；
>
> 包中可以包含任意多个二进制 crate(binary crate)；
>
> 包中至少包含一个 crate，无论是库的还是二进制的。

`src/main.rs` 和 `src/lib.rs` 叫做 crate 根。之所以这样叫它们是因为这两个文件的内容都分别在 crate 模块结构的根组成了一个名为 `crate` 的模块，该结构被称为 *模块树*（*module tree*）。

模块不仅对于你组织代码很有用。他们还定义了 Rust 的 *私有性边界*（*privacy boundary*）：这条界线不允许外部代码了解、调用和依赖被封装的实现细节。所以，如果你希望创建一个私有函数或结构体，你可以将其放入模块。

Rust 中默认所有项（函数、方法、结构体、枚举、模块和常量）都是私有的。

如果我们在一个结构体定义的前面使用了 `pub` ，这个结构体会变成公有的，但是这个结构体的字段仍然是私有的.

枚举成员默认就是公有的。结构体通常使用时，不必将它们的字段公有化，因此结构体遵循常规，内容全部是私有的，除非使用 `pub` 关键字。

### 使用use关键字将名称引入作用域

要想使用 `use` 将函数的父模块引入作用域，我们必须在调用函数时指定父模块，这样可以清晰地表明函数不是在本地定义的，同时使完整路径的重复度最小化

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```

另一方面，使用 `use` 引入结构体、枚举和其他项时，习惯是指定它们的完整路径。

```rust
use std::collections::HashMap;

fn main() {
    let mut map = HashMap::new();
    map.insert(1, 2);
}
```

将两个具有相同名称但不同父模块的 `Result` 类型引入作用域，以及如何引用它们。

```rust
use std::fmt;
use std::io;

fn function1() -> fmt::Result {
    // --snip--
}

fn function2() -> io::Result<()> {
    // --snip--
}
或者使用as 
use std::fmt::Result;
use std::io::Result as IoResult;

fn function1() -> Result {
    // --snip--
}

fn function2() -> IoResult<()> {
    // --snip--
}
```

### 指定嵌套的路径在一行中将多个带有相同前缀的项引入作用域

```rust
use std::cmp::Ordering;
use std::io;
写做：
use std::{cmp::Ordering, io};
--------------------------------
use std::io;
use std::io::Write;
写做
use std::io::{self, Write};
--------------------------------
use std::collections::*;
//将一个路径下所有公有项引入作用域，可以指定路径后跟 *
```

## 八、常见集合

不同于内建的数组和元组类型，这些集合指向的数据是储存在堆上的，这意味着数据的数量不必在编译时就已知，并且还可以随着程序的运行增长或缩小

### vector

```rust
let v: Vec<i32> = Vec::new();
```

### 字符串

`String` 是一个 `Vec<u8>` 的封装

## 九、错误处理

阅读 backtrace 的关键是从头开始读直到发现你编写的文件。这就是问题的发源地。这一行往上是你的代码所调用的代码；往下则是调用你的代码的代码。

为了获取带有这些信息的 backtrace，必须启用 debug 标识。当不使用 `--release` 参数运行 cargo build 或 cargo run 时 debug 标识会默认启用

### ？运算符可被用于返回Result的函数

```rust
use std::error::Error;
use std::fs::File;

fn main() -> Result<(), Box<dyn Error>> {
    let f = File::open("hello.txt")?;

    Ok(())
}
```

## 十、泛型、trait与生命周期

