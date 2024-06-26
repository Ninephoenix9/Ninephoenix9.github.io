+++
title = "Rust 核心语法：所有权与借用"
date = "2024-05-21T18:19:51+08:00"
tags = ["Rust"]
description = "但是，古尔丹，代价是什么呢？"
slug = "rust-ownership"
+++

## 引言

所有编程语言都无法回避的一个问题是内存管理，理想的编程语言应该有以下两个特点：

- 内存对象能在正确的时机及时释放，这使我们能控制程序的内存消耗
- 在内存对象被释放后，不应该继续使用指向它的指针，这会导致崩溃和安全漏洞

由此诞生出两大阵营：1)以 C/C++/Zig 为代表，手动管理内存的申请和释放，但避免内存泄漏和悬空指针是程序员的责任。2)依靠垃圾回收机制(Garbage Collection)自动管理，在所有指向内存对象的指针都消失后，自动释放对应内存。但这会严重影响程序性能，几乎所有现代编程语言，从 Java/Python/Haskell 到 Go/Javascript 都在此列。

为了同时兼顾安全与性能，Rust 选择了第三种方式：由编译器管理内存（编译期 GC），即编译时就决定何时释放内存，并将相关指令在恰当位置写入可执行程序。这种方式不会带来任何运行时开销，也可以保证内存安全和并发安全（虽然 Rust 无法完全避免内存泄漏，但至少大大降低了它发生的概率，这是后话）。

为了满足要求，Rust 语言提出了两个核心概念，即所有权(Ownership)和生命周期(Lifetimes)。这两大概念本质是对语法的一种限制，目的是在混沌中建立足够的秩序，以便让 Rust 在编译期有能力验证程序是否安全。所有权系统解决了内存释放以及二次释放的问题，生命周期系统则解决了悬垂指针问题。

当然，在工程领域，..一切选择皆是权衡..。Rust 不是完美的等边三角，兼顾了安全和性能，则必然要付出一些代价，包括更长的编译时间、更高的心智负担，还有要命的语法限制——你很难用 safe rust 写出一个双向链表或红黑树。但这并非不可接受，不仅因为手写它们的机会越来越少，也因为你可以在 unsafe 中封装这些“不安全”的结构。记住，..unsafe.. 不是 ..nosafe..，只是将安全保证由编译器交到程序员手中，它与 C/C++ 没什么区别，甚至还更安全和现代一点。

## Rust 内存模型

在程序运行时，操作系统会为程序分配内存空间，并将之加载进内存。Rust 尚未严格定义其内存模型，但可以按下图粗略理解，包括了堆区、栈区、静态数据区、只读数据区和只读指令区，如图所示：

![2024-05-23-17-54-54.png](/images/2024-05-23-17-54-54.png)

对于每个部分存储的内容，大致有如下分类：

1. Stack（栈）
   - 栈用于存储函数参数和局部变量，内存对象的数据类型及其大小必须在编译时确定
   - 栈内存分配是连续的，操作系统对栈内存大小有所限制，因此你无法创建过长的数组
   - 栈内存的分配效率要高于堆内存
2. Heap（堆）
   - 程序员主动申请使用，一般做大量数据读写时使用，相比栈，堆分配效率较低
3. Static Data（静态数据区）
   - 存放一般的静态函数、静态局部变量和静态全局变量，在程序启动时初始化
4. Literals（只读数据区）
   - 存放代码的文字常量，比如字符串字面量
5. Instructions（只读代码区）
   - 存放可执行指令

## 所有权规则

Rust 的所有权系统基于以下事实:

1. 编译器能够解析局部变量的生命周期，正确管理栈内存的入栈和出栈
2. 堆内存最终都是通过栈变量（指针和引用）来读取和修改

那么，能否让堆内存管理和栈内存管理一样轻松，成为编译期就生成好的指令呢？根据这个思路，Rust 将堆内存的生命周期和栈变量绑定在一起，当函数栈被回收，局部变量被销毁时，其对应的堆内存（如果有的话）也会被析构函数 `drop()` 回收。

> 在 C++ 中，这种 item 在生命周期结束时释放资源的模式被称作..资源获取即初始化.. Resource Acquisition Is Initialization (RAII)

考虑一个普通的初始化语句：

```Rust
fn main() {
    let Variable: Type = Value;
    // ...
    // Variable 离开作用域
}
```

`Variable` 被称为变量，`Type` 是其类型，而 `Value` 被称为..内存对象..，也叫做值。每一个赋值操作称为值..绑定..，因为此时不仅仅对变量进行了赋值，我们还把..内存对象的所有权..一并给予了变量。此处的内存对象 `Value` 可以是栈内存，也可以是堆内存（但它一定有一个栈指针）。

> 重点辨析：既然堆内存对象都是由栈上指针进行管理的，那么当 `Value` 包含 `String::from("xxx")` 或 `Box::new(xxx)` 这样的堆内存时，严格来说，`Variable` 拥有的是栈上指针的所有权，而非堆内存字符串(`Value`)。但因为 `Variable` 实现了指向内存的释放逻辑，`Variable` 实质上拥有指向内存的所有权。Rust 所有权的本质，就是..明确谁负责释放资源的责任..。

Rust 所有权的核心规则很简单：

1. 每一个内存对象，在任意时刻，都有且只有一个称作所有者(owner)的变量
2. 当所有者（变量）离开作用域时，这个内存对象将被释放

编译器知道本地变量 `Variable` 何时离开作用域，自然也就知道何时执行对内存对象 `Value` 的回收。而所有者唯一，保证了不会出现二次释放同一内存的错误。

切记，所有权是一个编译器抽象的概念，它不存在于实际的代码中，仅仅是一种思想和规则。

## 所有权转移

其他语言往往有深拷贝和浅拷贝两个概念，浅拷贝是只拷贝数据对象的引用，深拷贝是根据引用递归到最终的数据并拷贝数据。

Rust 为了适应所有权系统，没有采用深浅拷贝，而是提出了移动(Move)、拷贝(Copy)和克隆(Clone)的概念。

### Move 语义

考虑以下代码：

```Rust
let mut s1 = String::from("big str");
let s2 = s1;

// 下面将报错 error: borrow of moved value: `s1`
println!("{},{}", s1, s2); 

// 重新赋值
// "big str" 被自动释放，为 "new str" 分配新的堆内存
s1 = String::from("new str");
```

将 `s1` 赋值给 `s2` 会发生什么？如果将 `s1` 宽指针复制一份给 `s2`，这违反了所有者唯一的规则，会导致内存的二次释放；如果拷贝一份 `s1` 指向的堆内存交给 `s2`，这又违背了 Rust 性能优先的原则。

实际上，这时候 Rust 会进行所有权转移(Move)：直接让 `s1` 无效（`s1` 仍然存在，只是失去所有权，..变成未初始化的变量..，只要 `s1` 是可变的，你还可以重新为其初始化），同时将 `s1` 栈内存对象复制一份，再将内存对象的所有权交给 `s2`，这样 `s2` 就指向了同一个堆内存对象，如图所示：

![2024-05-23-20-48-24.png](/images/2024-05-23-20-48-24.png)

编译期的 `mut` 标识是作用在变量名上，而不是那个内存对象。因此下面的例子中 `s1` 不可变，并不妨碍我们定义另外一个可变的变量名 `s2` 来写这块内存：

```Rust
// s1 不可变
let s1 = String::from("big str");

let mut s2 = s1;
s2.push('C');  // 正确
```

#### 所有权检查

所有权检查是编译期的静态检查，编译器通常不会考虑你的程序将怎样运行，而是基于代码结构做出判断，这使得它看上去不那么聪明。比如依次写两个条件互斥的 `if`，编译器不会考虑那么多，直接告诉你不能移动 `x` 两次：

```Rust
fn foobar(n: isize, x: Box<i32>) {
    if n > 1 {
        let y = x;
    }
    if n < 1 {
        let z = x; // error[E0382]: use of moved value: `x`
    }
}
```

甚至把 Move 操作放在循环次数固定为 1 的 `for` 循环内，编译器也傻傻看不出来：

```Rust
fn foobar(x: Box<i32>) {
    for _ in 0..1 {
        let y = x; // error[E0382]: use of moved value: `x`
    }
}
```

#### 编译器优化

Move 语义不仅出现在变量赋值过程中，在函数传参、函数返回数据时也会发生，因此，如果将一个大对象（例如过长的数组，包含很多字段的 `struct`）作为参数传递给函数，可能会影响程序性能。

因此 Rust 编译器会对 Move 语义的行为做出一些优化，简单来说，当数据量较大且不会引起程序正确性问题时，它会优化为传递大对象的指针而非内存拷贝。[^1]

总之，Move 语义虽然发生了栈内存拷贝，但性能并不会受太大影响。

### Copy 语义

你可能会想，如果每次赋值都要令原变量失效，是否太麻烦了？为此，Rust 提出了 Copy 语义，和 Move 语义的唯一区别是，Copy 后..原变量仍然可用..。换言之，Copy 等同于“浅拷贝”，会对栈内存做按位复制，而不对任何堆内存负责，原变量和新变量各自绑定独立的栈内存，..并拥有其所有权..。显然，如果变量负责管理堆内存对象，Copy 语义会导致二次释放的错误，因而 Rust 默认使用 Move 语义，只有实现了 `Copy` Trait 的类型赋值时 Copy。

例如，标准库的 `i32` 类型已经实现了 `Copy` Trait，因此它在进行所有权转移的时候，会自动使用 Copy 而非 Move 语义，即赋值后原变量仍可用。

Rust 默认实现 `Copy` Trait 的类型，包括但不限于：

- 所有整数类型
- 所有浮点数类型
- 布尔类型
- 字符类型
- 元组，当且仅当其包含的类型也都实现 `Copy` 的时候。比如 `(i32, i32)` 是 `Copy` 的，但 `(i32, String)` 不是
- 共享指针类型 `*const T` 或共享引用类型 `&T`（无论 T 是否实现 `Copy`）

对于那些没有实现 `Copy` Trait 的自定义类型，可以通过 `#[derive]` 手动派生实现(必须同时实现 `Clone` Trait)，方式很简单：

```Rust
#[derive(Copy, Clone)]
struct MyStruct(i32, i32);
```

你也可以手动实现：

```Rust
struct MyStruct;

impl Copy for MyStruct { }

impl Clone for MyStruct {
    fn clone(&self) -> MyStruct {
        *self
    }
}
```

注意，只有当某类型的所有成员都实现了 `Copy`，该类型才能实现 `Copy`。“成员(Member)”的含义取决于类型，例如：结构体的字段、枚举的变量、数组的元素、元组的项，等等。

Rust 标准库文档提到[^2]，一般来说，如果你的类型可以实现 `Copy`，它就应该实现。但实现 `Copy` 是你的类型公共 API 的一部分。如果该类型可能在未来变成非 `Copy`，那么现在省略 `Copy` 的实现可能会是明智的选择，以避免 API 的破坏性改变。

那么哪些类型不能实现 `Copy` 呢？`Copy` 与 `Drop` 是两个互斥的 Trait，任何自身或部分实现了 `Drop` 的类型都不可实现 `Copy`，如 `String` 和 `Vec<T>` 类型。

`Drop` Trait 的定义如下：

```Rust
pub trait Drop {
    // Required method
    fn drop(&mut self);
}
```

当一个值不再被需要（比如离开作用域时），Rust 会运行 "destructor" 将其释放。如果该类型实现了 `Drop` Trait，destructor 会调用 `Drop::drop` 析构函数，但即使没有实现 `Drop`，destructor 也会自动生成 "drop glue"，递归地为这个值的所有成员调用析构函数。因此多数情况下，你不必为你的类型实现 `Drop`。

但是在某些情况下，手动实现是有用的，例如对于直接管理资源的类型。这个资源可能是内存，也可能是文件描述符，也可能是网络套接字。一旦不再使用该类型的值，它应该通过释放内存或关闭文件或套接字来“清理”其资源。这就是 destructor 的工作，因此也就是 `Drop::drop` 的职责。

除此之外，Copy 一个可变引用 `&mut T` 也不被允许，这违背了引用的借用规则。

### Clone 语义

看见上面的 `Clone` Trait 了吗？它也是一个常见的 Trait，实现了 `Clone` 的类型变量可以调用 `clone()` 方法手动拷贝内存对象，它对复本的整体有效性负责，所以【栈】与【堆】都是 `clone()` 的复制目标，这相当于“深拷贝”。`Clone` 还是 `Copy` 的 Supertrait，看看 `Copy` Trait 的定义：

```Rust
pub trait Copy: Clone { }
```

这表明，实现 `Copy` 的类型必须先实现 `Clone`。这是因为实现了 `Copy` 的类型在赋值时，会自动调用其 `clone()` 方法。如果一个类型是 `Copy` 的，那么它的 `clone()` 实现只需要返回 `*self`（参见上例）。

```Rust
let s1 = String::from("big str");
// 克隆 s1 之后，变量 s1 仍然绑定原始数据
let s2 = s1.clone();
println!("{},{}", s1, s2);
```

`Copy` 的细节被封装在编译器内，无法自行定制和实现，不可重载；而 `Clone` 可由开发者自行实现。所以调用 `Clone` 的默认实现时，操作的性能是较低的。但你可以实现自己的克隆逻辑，也不一定总是会效率低。比如 `Rc`，它的 `clone()` 用于增加引用计数，同时只拷贝少量数据，效率并不低。

### 小结

总结一下，Move 语义等于“浅拷贝” + 原变量失效，复制栈内存并移动所有权；Copy 语义只进行“浅拷贝”，复制栈内存和所有权；Clone 语义必须显式调用，进行“深拷贝”，复制栈内存、堆内存和所有权。

另外，在 1)赋值 2)参数传入 3)返回值传出时 Move 和 Copy 行为被隐式触发，而 Clone 行为必须显示调用 `Clone::clone(&self)` 成员方法。

## 引用

和 C/C++ 一样，Rust 有裸指针(Pointer)类型和引用(Reference)类型，分别是共享引用（不可变引用） `&T` 和可变引用 `&mut T`，常量裸指针（不可变指针） `*const T` 和可变裸指针 `*T`，他们的值都是 `T` 类型对象的地址，都可以通过解引用操作指向内存对象。

区别在于，Rust 认为裸指针是不安全的操作，所以它只能在 `unsafe` 块中使用，引用则是被编译器加了限制的裸指针，遵循借用规则(Borrowing Rules)并由编译器检查，以保证安全。

为了使用方便，Rust 引用的作用域比普通变量更短：普通变量的作用域从初始化持续到最近的花括号 `}`；引用的作用域从借用开始，一直持续到它最后一次使用的地方。这种优化行为被称为非词法作用域生命周期(Non-Lexical Lifetimes, NLL)。

创建一个引用的行为称为借用(Borrowing)，代表着引用会借用（而非获得）原变量对内存对象的所有权。当你只想使用变量，而不想转移所有权时，可以通过借用访问内存对象，例如：

```Rust
fn main() {
    let s1 = String::from("hello"); // s1 本质是一个指向堆内存的指针

    let len = calculate_length(&s1); // 发生了不可变借用

    println!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize { // s 是指向 s1 的引用
    s.len()
}
```

传入 `calculate_length()` 的是 `s1` 的共享引用，参数 `s` 会借用 `s1` 的所有权，如图所示：

![2024-05-24-15-42-23.png](/images/2024-05-24-15-42-23.png)

Rust 的借用规则很有趣：

1. 在同一时刻，要么只有一个可变引用(`&mut T`)，要么有任意数量的共享引用(`&T`)。可变引用与共享引用不能同时存在，也不能同时存在多个可变引用。正因如此，可变引用 `&mut T` 不能实现 `Copy`，这会违反借用规则。当然，共享引用是可 Copy 的，即把共享引用赋值给另一个共享引用后，可以继续使用
2. 引用必须总是有效的，即引用的生命周期不能超过原变量的生命周期。所以当存在借用时，原变量不能 Move，但可以 Copy 或 Clone

为什么会有这样的规定呢？因为 Rust 希望在同一时刻，..一份资源只能被至多一个变量名读写，或者被多个变量名读取..。由此：

- 对于不可变借用(`let y = &x`)：1)引用 `y` 是只读的；2)在 `y` 作用域结束之前，`x` 可读不可写（因为存在 `y` 可读），因此 `x` 能被不可变借用但不能被可变借用
- 对于可变借用(`let y = &mut x`)：1)引用 `y` 可读写；2)在 `y` 作用域结束之前，`x` 不可读不可写（因为存在 `y` 可写），因此 `x` 不能再次被借用

下面这段代码会报错：

```Rust
fn main() {
    let mut x = 0;
    let y = &mut x; // y 可变借用于 x
    if x == 0 { // 报错：存在 x 的可变引用 y，此时不能通过原变量 x 读取值（也不可写入值）
        *y += 1; // y 的作用域到此结束
        println!("{}", x); // x 可以正常读取
    }
}
```

## 解引用

解引用操作 `*r` 会得到一个被称为影子变量的东西，可以理解为没有所有权的变量别名。它可以用来对内存对象进行读写，但不能通过它转移所有权（但可以 Copy 或 Clone），这会影响本体对于内存对象的掌控：

```Rust
struct MyType<T> {
    val: T
}

fn main() {
    let num1 = 1;
    let num1_ref = &num1;
    // i32 类型实现了 Copy，因此 i32 类型的影子变量会进行 Copy 操作，这不会影响本体的所有权
    let num2 = *num1_ref;
    let x = MyType{val: 1};
    let y = &x;
    // 这里报错，因为无法通过解引用得到的影子变量移动所有权
    let z = *y;
}
```

## 引用类型的所有权

Rust 中所有的值都有所有权，引用类型的值也不例外。引用不拥有指向对象的所有权，但引用变量拥有自身地址值的所有权。参考下面这段代码：

```Rust
fn main() {
    let mut s = String::from("value");
    let r = &mut s; // r 是 s 的可变引用
    let r1 = r; // move 而非 copy
    println!("{}", r); // 报错 borrow of moved value: `r`
}
```

上文提到，共享引用实现了 `Copy`，自然也实现了 `Clone`，而下面的结构体 `Person` 没有实现 `Clone`，因此 `b.clone()` 只能复制引用 `b`，不能复制引用指向的内存对象。虽然这能通过编译，但 clippy 不建议我们这样做，因为它的行为相当于 `Copy` 操作，很可能不是我们希望的克隆效果。

```Rust
struct Person;

let a = Person;
let b = &a;
let c = b.clone();  // c 的类型是 &Person
```

但如果为结构体 `Person` 实现 `Clone`，再去 `clone()` 引用类型，将没有错误提示：

```Rust
#[derive(Clone)]
struct Person;

let a = Person;
let b = &a;
let c = b.clone();  // 此时 c 的类型是 Person，而不是 &Person
```

前后两个示例的区别，仅在于引用所指向的类型 `Person` 有没有实现 `Clone`。所以得出结论：

- 没有实现 `Clone` 时，引用类型的 `clone()` 将等价于 Copy
- 实现了 `Clone` 时，引用类型的 `clone()` 将克隆并得到引用所指向的类型

这是因为，方法调用时会先查找与调用者类型匹配的方法，查找过程具有优先级，找到即停。由于 `.` 操作可以自动引用/解引用，如果引用/解引用前后的两种类型都实现了同一方法(如 `clone()`)，Rust 编译器将按照查找顺序来决定调用哪个类型上的方法。[^3]

如果 `b` 是没有实现 `Copy` 和 `Clone` 的可变引用，`b.clone()` 只能得到 `Person` 类型（前提是 `Person` 实现了 `Clone`）。

## 所有权树

Rust 每个拥有所有权的容器类型(`tuple`/`array`/`vec`/`struct`/`enum`等)变量和它的成员（以及成员的成员），会形成一棵所有权树。树中任何一个成员（假设叫 `A`）离开作用域或转移所有权，其全部子成员将与其保持行为一致——销毁内存对象或 Move 给新变量。

当成员 `A` 所有权转移后，除非你为 `A` 重新初始化，否则 `A` 的所有父成员（包括树的根成员）将失去所有权（但不属于 `A` 子成员的仍然可用）。

```Rust
fn main() {
    let mut tup = (5, String::from("hello"));

    let (x, y) = tup; // tup.1 转移所有权给 y，tup.0 copy 给 x
    println!("{},{}", x, y); // 正确
    println!("{}", tup.0); // 正确
    // tup.1 = String::from("world"); // 重新初始化 tup 可以修正错误
    println!("{}", tup.1); // 错误
    println!("{:?}", tup); // 错误
}
```

## 重借用

上文提到，借用检查不允许对一个实例的多个可变引用，也不能同时存在共享和可变引用。但对解引用得到的影子变量进行借用（重借用）却是可行的：

```Rust
let mut s = String::from("ABC");
let r1 = &mut s;
{
    let r2 = &mut *r1; // 重借用
    r2.push('2');
    println!("{}", r2); // r2 的作用域到此结束
}
println!("{}", r1); // r1 的作用域到此结束
```

这段代码的大括号内，同时存在 `r1` `r2` 两个指向同一变量 `s` 的可变引用，但编译器不会报错。这是因为编译器看到 `*r1` 的时候，通常很难确定解引用得到的实际对象是什么，所以借用检查不会把 `*r1` 跟 `s` 当成同一个对象，自然不会报错。

关于重借用，唯一的限制是不能在 `r2` 的生命周期中使用 `r1`，这样就不会违反借用规则：

```Rust
let mut s = String::from("ABC");
let r1 = &mut s;
{
    let r2 = &mut *r1;

    let l = r1.len(); // 错误 Cannot borrow `*r1` as immutable because it is also borrowed as mutable
    println!("{}", l);
    r2.push('2');
    r1.push('3'); // 错误 Cannot borrow `*r1` as mutable more than once at a time
    println!("{:?}", r1); // 错误 Cannot borrow `r1` as immutable because it is also borrowed as mutable
    println!("{}", r2);
}
println!("{}", r1);
```

你可能会好奇，明明传入方法的是引用类型，为什么前两条报错信息中会显示 `*r1`？这是因为发生了隐式重借用，`r1.len()` 实际上是 `String::len(&*r1)`，同理 `r1.push('3')` 实际上是 `String::push(&*r1, '3')`。

隐式重借用并非多此一举，`len()` 和 `push()` 的方法签名分别是 `pub fn len(&self) -> usize` `pub fn push(&mut self, ch: char)`。没有隐式重借用，可变引用 `r1` 将无法调用 `len()`，而 `r1.push('3')` 会转移可变引用 `r1` 的所有权，导致 `r1` 之后无法使用。

事实上，隐式重借用几乎无处不在：

```Rust
let mut s = String::from("ABC");
let r1 = &mut s;
// 不标注 r2 的类型，会 Move 而非隐式重借用，之后 r1 失效
let r2 = r1;
// 手动标注 r2 的类型，会进行非隐式重借用，比如调用函数时传参
let r3: &mut String = r2; // 相当于 let r3: &mut String = &mut *r2;
println!("{:?}", r3);
println!("{:?}", r2); // 打印 r3 r2 的顺序不能颠倒
```

对共享引用 `&T`，可以认为发生了重借用, 也可以认为直接发生 `Copy`，因为效果完全一样。

### 手写重借用

下面这两种情况[^4]，`from()` 函数不会自动重借用：

```Rust
struct X;

impl From<&mut i32> for X {
    fn from(i: &mut i32) -> Self {
        X
    }
}

let mut i = 4;
let r = &mut i;

fn from_auto_reborrow<'a, F, T: From<&'a mut F>>(f: &'a mut F) -> T {
    T::from(f)
}
let x: X = from_auto_reborrow(r); // 可以自动重借用
let x: X = from_auto_reborrow(r); // 可以自动重借用

fn from<F, T: From<F>>(f: F) -> T {
    T::from(f)
}
let x: X = from(&mut *r); // 必须显式重借用, 创建 x 的 reborrow 不会 move x
let x: X = from(r); // 此处不会自动重借用, 导致 Move x
let x: X = from(r); // 编译失败
```

```Rust
struct I(i32);

struct X1;
impl From<&mut I> for X1 {
    fn from(p: &mut I) -> X1 {
        p.0 = 1;
        X1
    }
}
// 必须引入这个中间函数
fn x1(p: &mut I) -> X1 {
    X1::from(p)
}

// value used here after move
fn from_twice_fail(p: &mut I) {
    let x11 = X1::from(p); // p Move
    let x12 = X1::from(p); // 编译失败
}

fn from_twice(p: &mut I) {
    let x11 = x1(p); // 此处不会自动重借用, 导致 Move p
    let x12 = x1(p); // 编译通过
}
```

## 总结

这张图展示了变量、类型、内存对象、值，引用、解引用和裸指针的概念：

![2024-05-24-17-17-25.png](/images/2024-05-24-17-17-25.png)

---

[^1]: 参考：<https://stackoverflow.com/questions/30288782/what-are-move-semantics-in-rust>
[^2]: 参考：<https://doc.rust-lang.org/std/marker/trait.Copy.html#when-should-my-type-be-copy>
[^3]: 参考：<https://rustc-dev-guide.rust-lang.org/method-lookup.html?highlight=lookup#method-lookup>
[^4]: 参考：<https://rustcc.cn/article?id=28fedcbc-d0c9-41e1-8d95-de73a578a078>
