---
title: Rust_base.md

date: 2024-1-12 17:00:00 +0800

categories: [Rust]

tags: [Rust]

description: 
---



> 本文用于记录学习Rust重要的语法与机制。

* [Rust圣经](https://course.rs/about-book.html)



# 0 Cargo与Rust模块系统

## 0.1 //TODO: Cargo







## 0.2 模块系统

### Package/Crate

>  `Package` 就是一个项目，因此它包含有独立的 `Cargo.toml` 文件，以及因为功能性被组织在一起的一个或多个包。一个 `Package` 只能包含**一个**库(library)类型的包，但是可以包含**多个**二进制可执行类型的包。
>
> 对于 Rust 而言，包 `crate` 是一个独立的可编译单元，它编译后会生成一个可执行文件或者一个库。一个包会将相关联的功能打包在一起，使得该功能可以很方便的在多个项目中分享。

***典型的 `Package` 结构如下：***

```rust
.
├── Cargo.toml
├── Cargo.lock
├── src
│   ├── main.rs
│   ├── lib.rs
│   └── bin
│       └── main1.rs
│       └── main2.rs
├── tests
│   └── some_integration_tests.rs
├── benches
│   └── simple_bench.rs
└── examples
    └── simple_example.rs
```

* 一个真实项目中典型的 `Package`，会包含多个二进制包，这些包文件被放在 `src/bin` 目录下，每一个文件都是独立的二进制包，同时也会包含一个库包，该包只能存在一个 `src/lib.rs`；
  *  Cargo 有一个惯例：**`src/main.rs` 是二进制包的根文件，该二进制包的包名跟所属 `Package` 相同**；
  * 与 `src/main.rs` 一样，Cargo 知道，如果一个 `Package` 包含有 `src/lib.rs`，意味它包含有一个库类型的Package同名包，该包的根文件是 `src/lib.rs`；
*  `Package` 和包很容易被混淆，因为你用 `cargo new` 创建的 `Package` 和它其中包含的包是同名的。

### Module

如果说包 `crate` 是Rust工程/项目的构成单元，那么模块 `mod` 就是Rust的代码构成单元。

[【翻译】关于Rust模块系统的清晰解释 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/164556350)

[Rust 模块系统 - 掘金 (juejin.cn)](https://juejin.cn/post/7324165965205078066)

[Rust“与众不同”的地方 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/87514788)

[使用 use 关键字将名称引入作用域 - Rust 程序设计语言 简体中文版 (bootcss.com)](https://rust.bootcss.com/ch07-04-bringing-paths-into-scope-with-the-use-keyword.html)

* `pub mod`

* `pub use`

  use可以为当前作用域引入其他模块：类似 `use std::io`。不过 `pub use` 组合起来就可以公开你导入的模块，好处就是对外提供你use的模块的内容。此技术被称为 “重导出”（*re-exporting*），因为这样做将模块引入作用域并同时使其可供其他代码引入自己的作用域。

  ```rust
   mod aaa {
      pub mod bbb {
          pub fn bb() {
              println!("bb");
          }
      }
  }
  
  mod ccc {
      pub use crate::aaa::bbb;
      pub fn cc() {}
  }
  fn main(){
      ccc::cc();
      ccc::bbb::bb();
  }
  ```

# 1  所有权与借用

Rust 之所以能成为万众瞩目的语言，就是因为其内存安全性。在以往，内存安全几乎都是通过 GC 的方式实现，但是 GC 会引来性能、内存占用以及 Stop the world 等问题，在高性能场景和系统编程上是不可接受的，因此 Rust 采用了与众(不同的方式：**所有权系统**。

## 1.1 所有权与堆

与栈不同，对于大小未知或者可能变化的数据，我们需要将它存储在堆上。

当向堆上放入数据时，需要请求一定大小的内存空间。操作系统在堆的某处找到一块足够大的空位，把它标记为已使用，并返回一个表示该位置地址的**指针**, 该过程被称为**在堆上分配内存**，有时简称为 “分配”(allocating)。

接着，该指针会被推入**栈**中，因为指针的大小是已知且固定的，在后续使用过程中，你将通过栈中的**指针**，来获取数据在堆上的实际内存位置，进而访问该数据。

---

当你的代码调用一个函数时，传递给函数的参数（包括可能指向堆上数据的指针和函数的局部变量）依次被压入栈中，当函数调用结束时，这些值将被从栈中按照相反的顺序依次移除。

因为堆上的数据缺乏组织，因此跟踪这些数据何时分配和释放是非常重要的，否则堆上的数据将产生内存泄漏 —— 这些数据将永远无法被回收。这就是 Rust 所有权系统为我们提供的强大保障。

## 1.2 所有权原则

> ***谨记以下规则：***
>
> * Rust 中每一个值都被一个变量所拥有，该变量被称为值的所有者
> * 一个值同时只能被一个变量所拥有，或者说一个值只能拥有一个所有者
> * 当所有者(变量)离开作用域范围时，这个值将被丢弃(drop)
>
> 怎么理解所有者？首先这里我们所说的“值”是**保存该值的内存块**，所谓变量作为该值的所有者指的是**通过一个指针指向这块内存。**那么此时这块内存的维护(释放)工作完全交由这个变量来处理。

### 所有权转移

```rust
let x = 5;
let y = x;

let s1 = String::from("hello");
let s2 = s1;

//error: borrow of moved value: `s1`
println!("{}, rust", s1);
```

* Rust 基本类型都是通过自动拷贝的方式来赋值的；
* 只有涉及堆的相关机制，才存在所有权的说法；
* Rust 这样解决问题：**当 `s1` 赋予 `s2` 后，Rust 认为 `s1` 不再有效，因此也无需在 `s1` 离开作用域后 `drop` 任何东西，这就是把所有权从 `s1` 转移给了 `s2`，`s1` 在被赋予 `s2` 后就马上失效了**。

因此作为维护堆区内存的的动态数据结构，如 `Sring`，类似 `let s2 = s1` 这样的赋值拷贝动作只能是浅拷贝 (即只改变指针的指向)，因为深拷贝会造成不可避免的内存二次释放BUG。

---

> ***关于深拷贝/浅拷贝***

如果我们**确实**需要深度复制 `String` 中堆上的数据，而不仅仅是栈上的数据，可以使用一个叫做 `clone` 的方法。

```rust
let s1 = String::from("hello");
let s2 = s1.clone();

println!("s1 = {}, s2 = {}", s1, s2);
```

---

> ***Copy 特征***

Rust 有一个叫做 `Copy` 的特征，可以用在类似整型这样在栈中存储的类型。如果一个类型拥有 `Copy` 特征，一个旧的变量在被赋值给其他变量后仍然可用。

那么什么类型是可 `Copy` 的呢？有一个通用的规则： **任何基本类型的组合可以 `Copy` ，不需要分配内存或某种形式资源的类型是可以 `Copy` 的**。如下是一些 `Copy` 的类型：

- 所有整数类型，比如 `u32`
- 布尔类型，`bool`，它的值是 `true` 和 `false`
- 所有浮点数类型，比如 `f64`
- 字符类型，`char`
- 元组，当且仅当其包含的类型也都是 `Copy` 的时候。比如，`(i32, i32)` 是 `Copy` 的，但 `(i32, String)` 就不是
- 不可变引用 `&T` 

### 函数传参/返回

将值传递给函数：

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

函数返回值：

```rust
fn main() {
    let s1 = gives_ownership();         // gives_ownership 将返回值
                                        // 移给 s1

    let s2 = String::from("hello");     // s2 进入作用域

    let s3 = takes_and_gives_back(s2);  // s2 被移动到
                                        // takes_and_gives_back 中,
                                        // 它也将返回值移给 s3
} // 这里, s3 移出作用域并被丢弃。s2 也移出作用域，但已被移走，
  // 所以什么也不会发生。s1 移出作用域并被丢弃

fn gives_ownership() -> String {             // gives_ownership 将返回值移动给
                                             // 调用它的函数

    let some_string = String::from("hello"); // some_string 进入作用域.

    some_string                              // 返回 some_string 并移出给调用的函数
}

// takes_and_gives_back 将传入字符串并返回该值
fn takes_and_gives_back(a_string: String) -> String { // a_string 进入作用域

    a_string  // 返回 a_string 并移出给调用的函数
}
```

所有权很强大，避免了内存的不安全性，但是也带来了一个新麻烦： **总是把一个值传来传去来使用它**。Rust的引用/借用机制将解决这个问题。

## 1.3 引用与借用

Rust 通过 `借用(Borrowing)` 这个概念来**获取变量的引用，称之为借用(borrowing)**。

### 不可变引用

```rust
fn main() {
    let s1 = String::from("hello");
    let len = calculate_length(&s1);
    println!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```

`s: &String` 即引用，本质上是一个指针，指向 `main` 函数栈帧的 `s1` 变量，并不直接维护"hello"所在的堆区内存。因此，当引用离开作用域后其指向的内存不会释放 (不会调用 `drop`)：

![&String s pointing at String s1](https://pic1.zhimg.com/80/v2-fc68ea4a1fe2e3fe4c5bb523a0a8247c_1440w.jpg)

### 可变引用

正如变量默认不可变一样，引用指向的值默认也是不可变的，需要 `mut` 声明：

```rust
fn main() {
    let mut s = String::from("hello");
    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

Rust中的借用规则如下：

* 同一作用域，特定数据只能有一个可变引用；

* 可变引用与不可变引用不能同时存在；

  >注意，引用的作用域 `s` 从创建开始，一直持续到它最后一次使用的地方，这个跟变量的作用域有所不同，变量的作用域从创建持续到某一个花括号 `}`；
  >
  >Rust 的编译器一直在优化，早期的时候，引用的作用域跟变量作用域是一致的，这对日常使用带来了很大的困扰，你必须非常小心的去安排可变、不可变变量的借用，免得无法通过编译，例如以下代码:
  >
  >```rust
  >fn main() {
  >   let mut s = String::from("hello");
  >
  >    let r1 = &s;
  >    let r2 = &s;
  >    println!("{} and {}", r1, r2);
  >    // 新编译器中，r1,r2作用域在这里结束
  >
  >    let r3 = &mut s;
  >    println!("{}", r3);
  >} // 老编译器中，r1、r2、r3作用域在这里结束
  >  // 新编译器中，r3作用域在这里结束
  >```
  >
  >在老版本的编译器中（Rust 1.31 前），将会报错，因为 `r1` 和 `r2` 的作用域在花括号 `}` 处结束，那么 `r3` 的借用就会触发 **无法同时借用可变和不可变**的规则。
  >
  >但是在新的编译器中，该代码将顺利通过，因为 **引用作用域的结束位置从花括号变成最后一次使用的位置**，因此 `r1` 借用和 `r2` 借用在 `println!` 后，就结束了，此时 `r3` 可以顺利借用到可变引用。
  >
  >---
  >
  >你的代码或许是违背规则的，这相当于编译器帮你擦了屁股，实际上编译器始终坚持着“借用原则”，即可变引用与不可变引用不能共存，所以在这里编译器强行让 `r1/r2` 在最后一次使用后销毁。
  >
  >对于这种编译器优化行为，Rust 专门起了一个名字 —— **Non-Lexical Lifetimes(NLL)**，专门用于找到某个引用在作用域(`}`)结束前就不再被使用的代码位置。

---

总的来说，借用规则如下：

- 同一时刻，你只能拥有要么一个可变引用, 要么任意多个不可变引用
- 引用必须总是有效的

# 2 一些基础语法

## 2.1 复合类型

### 字符串/切片

顾名思义，字符串是由字符组成的连续集合，但是在上一节中我们提到过，**Rust 中的字符是 Unicode 类型，因此每个字符占据 4 个字节内存空间，但是在字符串中不一样，字符串是 UTF-8 编码，也就是字符串中的字符所占的字节数是变化的(1 - 4)**，这样有助于大幅降低字符串所占用的内存空间。

`str` 类型是硬编码进可执行文件，也无法被修改，但是 `String` 则是一个可增长、可改变且具有所有权的 UTF-8 编码字符串，**当 Rust 用户提到字符串时，往往指的就是 `String` 类型和 `&str` 字符串切片类型，这两个类型都是 UTF-8 编码**。

```rust
fn test_base_use() {
    let my_name = "Pascal"; 				//&str
    let s = String::from("hello world");	//String
    let hello = &s[0..5];					//&str
    let world = &s[6..11];				//&str
    
    let a = [1, 2, 3, 4, 5];
    let slice = &a[1..3];
    assert_eq!(slice, &[2, 3]);
}

fn test_type_switch {
    //&str -> String
    let s1 = String::from("hello_world");
    let s2 = "hello_world".to_string();
    
    //String -> &str
    let s3 = &s1;
    let s4 = &s1[..];
    let s5 = s.as_str();
}

fn test_string_ops {
    let mut s = String::from("hello");
    s.push(' ');
    s.push_str("rust");
    s.insert(5, ','); //hello, rust
    s.insert(6, " i like"); //hello, i like rust
   	//...other methods
}
```

> 注意：在其它语言中，使用索引的方式访问字符串的某个字符或者子串是很正常的行为，但是在 Rust 中就会报错。

---

关于 `String` 和 `&str` 的本质区别？

* 就字符串字面值来说，我们在编译时就知道其内容，最终字面值文本被直接硬编码进可执行文件中；
* 对于 `String` 类型，为了支持一个可变、可增长的文本片段，需要在堆上分配一块在编译时未知大小的内存来存放内容，这些都是在程序运行时完成的；

那么 `String` 对象如何归还堆区内存？

与其它系统编程语言的 `free` 函数相同，Rust 也提供了一个释放内存的函数： `drop`，但是不同的是，其它语言要手动调用 `free` 来释放每一个变量占用的内存，而 Rust 则在变量离开作用域时，自动调用 `drop` 函数。

### 元组

```rust
fn main() {
    let tup = (500, 6.4, 1);
    let (x, y, z) = tup;
    let tup_0 = tup.0;
    let tup_1 = tup.1;
    let tup_2 = tup.2;
}
```

元组在函数返回值场景很常用，可以使用元组返回多个值，但缺陷是含义不清晰。

### 结构体

```rust
struct User {
    active: bool,
    username: String,
    email: String,
    sign_in_count: u64,
}

fn new(email: String, username: String) -> User {
    User {
        email,
        username,
        active: true,
        sign_in_count: 1,
    }
}

fn test_base_use() {
    let email = String::from("...@zz.com");
    let username = String::from("zq");
    let u1 = new(email, username);
    
    let u2 = User {
        email: String::from("...@xx.com");
        ..u1
    };
}

fn test_tuple_struct() {
    struct Color(i3, i32, i32);
    let black = Color(0, 0, 0);
}

fn test_unit_struct() {
    struct UnitStruct;
    let call_obj = UnitStruct;
    impl SomeTrait for UnitStruct {
        //...
    }
}

fn test_debug_struct() {
    #[derive(Debug)]
    struct Range {
        l: u32,
        r: u32.
    }
    
    let x = range {
        l: 1,
        r: 10
    };
    println!("x is {:?}", x);
    println!("x is {:#?}", x);
}
```

* `..` 语法表明凡是我们没有显式声明的字段，全部从 `user1` 中自动获取。需要注意的是 `..user1` 必须在结构体的尾部使用；
* 如果你定义一个类型，但是不关心该类型的内容, 只关心它的行为时，就可以使用 `单元结构体`；
* 如果你想在结构体中使用一个引用，就必须加上生命周期，否则就会报错；

### 枚举

```rust
enum PokerSuit {
    Clubs(u8),
    Spades(u8),
    Diamonds(char),
    Hearts(char),
}

enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

fn test_base_use() {
    let c1 = PokerCard::Spades(5);
    let c2 = PokerCard::Diamonds('A');
    
    let m1 = Message::Quit;
    let m2 = Message::Move{x:1,y:1};
    let m3 = Message::ChangeColor(255,255,0);   
}

/*
	enum Option<T> {
		Some(T),
		None,
	}
*/
fn test_option() {
    fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        None => None,
        Some(i) => Some(i + 1),
    }
    let five = Some(5);
	let six = plus_one(five);
	let none = plus_one(None);
}
```

* `Option<T>` 枚举是如此有用以至于它被包含在了 [`prelude`](https://course.rs/appendix/prelude.html)（prelude 属于 Rust 标准库，Rust 会将最常用的类型、函数等提前引入其中，省得我们再手动引入）之中，你不需要将其显式引入作用域。另外，它的成员 `Some` 和 `None` 也是如此，无需使用 `Option::` 前缀就可直接使用 `Some` 和 `None`；
* 在对 `Option<T>` 进行 `T` 的运算之前必须将其转换为 `T`。通常这能帮助我们捕获到空值最常见的问题之一：期望某值不为空但实际上为空的情况。

### 数组

数组的具体定义很简单：将多个类型相同的元素依次组合在一起，就是一个数组。结合上面的内容，可以得出数组的三要素：

- 长度固定
- 元素必须有相同的类型
- 依次线性排列

```rust
fn test_base_use() {
    //defines
    let a = [1, 2, 3];
    let b: [i32; 3] = [4, 5, 6];
    let c = [1; 3]; //[1, 1, 1]
    
    //access
    let a_0 = a[0];
    let a_1 = a[1];
    
    //error: the trait bound `String: std::marker::Copy` is not satisfie
    /*
    let array = [String::from("rust is good!"); 8];
    println!("{:#?}", array);
    */
    let arr: [String; 8] = std::array::from_fn(|_i| String::from("hello"));
    
    //slice
    let arr_s: [i32; 5] = [1, 2, 3, 4, 5];
    let slice: &[i32] = &a[1..3];
    assert_eq!(slice. &[2, 3]);
}
```

* `let array=[3;5]`底层就是不断的Copy出来的，但很可惜复杂类型都没有深拷贝，只能一个个创建；
* 切片`&[T]` 允许你引用集合中的部分连续片段，而不是整个集合，对于数组也是，数组切片允许我们引用数组的一部分：
* 切片类型[T]拥有不固定的大小，而切片引用类型&[T]则具有固定的大小，因为 Rust 很多时候都需要固定大小数据类型，因此&[T]更有用,`&str`字符串切片也同理；

## 2.2 集合类型

`Vector`、`HashMap` 、`String` 类型，是标准库中最最常用的集合类型。

### Vector

#### 基本操作

包括：创建/更新/读取/迭代/排序等。更多细节，阅读 Vector 的[标准库文档](https://doc.rust-lang.org/std/vec/struct.Vec.html#)。

```rust
fn test_base_use() {
    //create
    let mut v = vec![1, 2, 3, 4, 5];
    let mut v1: Vec<i32> = Vec::new();
    let v2 = vec![1, 2, 3];
    v1.push(1);
    
    //access
    let a: &i32 = &v[0];
    match v.get(0) {
        Some(a) => println!("{}", a),
        None => println!("None"),
    }
    //let does_not_exist = &v[100]; error
    let does_not_exist = v.get(100); 
    
    //iterator
    for i in &mut v {
        *i += 10;
    }
}

fn test_vec_capacity() {
    /*
    	Vector 长度是: 3, 容量是: 10
		Vector（reserve） 长度是: 3, 容量是: 103
		Vector（shrink_to_fit） 长度是: 3, 容量是: 3
    */
    let mut v = Vec::with_capacity(10);
    v.extend([1, 2, 3]);    // 附加数据到 v
    println!("Vector 长度是: {}, 容量是: {}", v.len(), v.capacity());

    v.reserve(100);        // 调整 v 的容量，至少要有 100 的容量
    println!("Vector（reserve） 长度是: {}, 容量是: {}", v.len(), v.capacity());

    v.shrink_to_fit();     // 释放剩余的容量，一般情况下，不会主动去释放容量
    println!("Vector（shrink_to_fit） 长度是: {}, 容量是: {}", v.len(), v.capacity());
}

fn test_vec_methods() {
    let mut v =  vec![1, 2];
    assert!(!v.is_empty());         // 检查 v 是否为空

    v.insert(2, 3);                 // 在指定索引插入数据，索引值不能大于 v 的长度， v: [1, 2, 3] 
    assert_eq!(v.remove(1), 2);     // 移除指定位置的元素并返回, v: [1, 3]
    assert_eq!(v.pop(), Some(3));   // 删除并返回 v 尾部的元素，v: [1]
    assert_eq!(v.pop(), Some(1));   // v: []
    assert_eq!(v.pop(), None);      // 记得 pop 方法返回的是 Option 枚举值
    v.clear();                      // 清空 v, v: []

    let mut v1 = [11, 22].to_vec(); // append 操作会导致 v1 清空数据，增加可变声明
    v.append(&mut v1);              // 将 v1 中的所有元素附加到 v 中, v1: []
    v.truncate(1);                  // 截断到指定长度，多余的元素被删除, v: [11]
    v.retain(|x| *x > 10);          // 保留满足条件的元素，即删除不满足条件的元素

    let mut v = vec![11, 22, 33, 44, 55];
    // 删除指定范围的元素，同时获取被删除元素的迭代器, v: [11, 55], m: [22, 33, 44]
    let mut m: Vec<_> = v.drain(1..=3).collect();    
    let v2 = m.split_off(1);        // 指定索引处切分成两个 vec, m: [22], v2: [33, 44]
}
```

#### 生命周期

`Vector` 类型在超出作用域范围后，会被自动删除。当 `Vector` 被删除后，它内部存储的所有内容也会随之被删除：

```rust
{
    let v = vec![1, 2, 3];
    // ...
} // <- v超出作用域并在此处被删除
```

#### Vector存储不同类型的元素

```rust
trait IpAddr {
    fn display(&self);
}

struct V4(String);
impl IpAddr for V4 {
    fn display(&self) {
        println!("ipv4: {:?}", self.0);
    }
}

struct V6(String);
impl IpAddr for V6 {
    fn display(&self) {
        println!("ipv6: {:?}", self.0);
    }
}

fn main() {
    let v: Vec<Box<dyn IpAddr>> = Vec![
        Box::new(V4("ipv4".to_string())),
        Box::new(V6("ipv6".to_string())),
    ];
    
    for i in v {
        i.display();
    }
}
```

---

### HashMap

`HashMap<K, V>` 中存储的是一一映射的 `KV` 键值对，并提供了平均复杂度为 `O(1)` 的查询方法。

#### 创建/查询/更新

```rust
use std::collections::HashMap;

fn main() {
    /* create */
    let mut hash_1 = HashMap::new();
    hash_1.insert("a", 1);
    hash_2.insert("b", 2);
    hash_3.insert("c", 3);
    
    // from `Vec` -> `HashMap`
    let v =vec![
        ("a".to_string(), 1),
        ("b".to_string(), 2),
        ("c".to_string(), 3),
    ];
    let mut hash_4: HashMap<_, _> = v.into_iter().collect();
    
    /* access */
    let k = String::from("a");
    let val_1: Option<&i32> = hash_4.get(&k);
    let va1_2: i32 = hash_4.get(&k).copied().unwrap_or(0);
    for (k, v) in &hash_4 {
        println!("{}: {}", k, v);
    }
    
    /* update */
    let a_old = hash_4.insert(&k, 100);
    assert_eq!(a_old. Some(1));
    
    let d = String::from("d");
    let d_new = hash_4.entry(&d).or_insert(666);
    assert_eq!(*d_new, 666); //不存在，则插入值
    let d_new = hash_4.entry(&d).or_insert(777);
    assert_eq!(*d_new, 777); //已经存在, 值没有被插入
}

fn test_count_word() {
    let text = "hello world rust hello rust";
    let mut hash = HashMap::new();
    for word in text.split_whitespace() {
        let count = hash.entry(word).or_insert(0);
        *count += 1;
    }
    println!("{:?}}", hash);
}
```

* 以上使用了迭代器和 `clooect` 快速将 `Vec<(String, u32)>` 中的数据快速写入到 `HashMap<String, u32>` 中。

* 对于查询:
  - `get` 方法返回一个 `Option<&i32>` 类型：当查询不到时，会返回一个 `None`，查询到时返回 `Some(&i32)`
  - `&i32` 是对 `HashMap` 中值的借用，如果不使用借用，可能会发生所有权的转移
* `test_count_word` 代码用于统计文本中单词出现的次数。

#### 所有权转移

`HashMap` 的所有权规则与其它 Rust 类型没有区别：

- 若类型实现 `Copy` 特征，该类型会被复制进 `HashMap`，因此无所谓所有权
- 若没实现 `Copy` 特征，所有权将被转移给 `HashMap` 中

**如果你使用引用类型放入 HashMap 中**，请确保该引用的生命周期至少跟 `HashMap` 活得一样久。就像以下代码会报错：

```rust
fn main() {
    use std::collections::HashMap;
    
   	let name = String::from("rust");
    let age = 10;
    let mut hash = HashMap::new();
    hash.insert(&name, age);
    
    std::mem::drop(name);
    println!("{name}");
    println!("{age}");
}
```

## 2.3 流程控制

### if/else if/else

```rust
fn main() {
    let n = 6;
    if n % 4 == 0 {
        println!("number is divisible by 4");
    } else if n % 3 == 0 {
        println!("number is divisible by 3");
    } else if n % 2 == 0 {
        println!("number is divisible by 2");
    } else {
        println!("number is not divisible by 4, 3, or 2");
    }
}
```

程序执行时，会按照自上至下的顺序执行每一个分支判断，一旦成功，则跳出 `if` 语句块。有一点要注意，就算有多个分支能匹配，也只有第一个匹配的分支会被执行。如果代码中有大量的 `else if `会让代码变得极其丑陋，不 `match` 专门用以解决多分支模式匹配的问题。

### for/loop/while/continue/break

```rust
fn main() {
    //for...index
    let arr = [1, 2, 3, 4];
    for (i, v) in arr.iter().enumerate() {
        println!("{}: {}", i, v);
    }
    
    for _ in 0..=10 {
        //...
    }
    
    //continue
    for i in 0..=6 {
        if i == 2 {
            continue;
        }
        if i == 4 {
            break;
        }
        println!("i = {}", i);
    }
    
    //while
    let mut n = 0;
    while n <= 6 {
        println!("{}", n);
        n += 1;
    }
    
    //loop
    let mut cnt = 0;
    let res = loop {
        cnt += 1;
        if cnt == 10 {
            break cnt * 2;
        }
    };
    println!("cnt = {}", cnt);
}
```

* 对于 `for` 一个集合，我们往往使用引用，除非你不想在后面的代码中继续使用该集合：

  | 使用方法                      | 等价使用方式                                      | 所有权     |
  | ----------------------------- | ------------------------------------------------- | ---------- |
  | `for item in collection`      | `for item in IntoIterator::into_iter(collection)` | 转移所有权 |
  | `for item in &collection`     | `for item in collection.iter()`                   | 不可变借用 |
  | `for item in &mut collection` | `for item in collection.iter_mut()`               | 可变借用   |

* 对于loop:

  - **break 可以单独使用，也可以带一个返回值**，有些类似 `return`
  - **loop 是一个表达式**，因此可以返回一个值

## 2.4 模式匹配

### match/if let

在 Rust 中，模式匹配最常用的就是 `match` 和 `if let`：

```rust
fn test_match() {
    enum Action {
        Say(String),
        MoveTo(i32, i32),
        ChangeRGB(u16, u16, u16),
    }
    
    let actions = [
        Action::Say(String::from("rust")),
        Action::MoveTo(1, 2),
        Action::ChangeRGB(6, 6, 6);
    ];
    
    for action in actions {
        match action {
            Action::say(x) => {
                println!("say: {}", x);
            }
            Action::MOveTo(x, y) => {
                println!("moveto: ({}, {})", x, y);
            }
            Action::ChangeRGB(x, y, z) => {
                println!("change_rgb: ({}, {}, {})", x, y, z);
            }
        }
    }
}

fn test_if_let() {
    let v = Some(3u8);
    if let Some(3) = v {
        println!("v = three");
    }
}
```

* 对于 `match`，用于替代 `if/else if/else` 处理多分支情况过于繁琐:
  - `match` 的匹配必须要穷举出所有可能，因此这里用 `_` 来代表未列出的所有可能性
  - `match` 的每一个分支都必须是一个表达式，且所有分支的表达式最终返回值的类型必须相同
* 对于 `if let`，用于只有一个模式的值需要被处理，其它值直接忽略的场景；
* `match` 最好不要使用同名，避免变量遮蔽；

### //TODO: 全模式列表



## 2.5 格式化输出

`println!` 宏接受的是可变参数，第一个参数是一个字符串常量，它表示最终输出字符串的格式，包含其中形如 `{}` 的符号是**占位符**，会被 `println!` 后面的参数依次替换。

```rust
fn main() {
    let s1 = "hello";
    let s2 = format!("{}, rust", s1);
    println!("{}", s2);
}
```

### 关于占位符 {}/{:?}

与其它语言常用的 `%d`，`%s` 不同，Rust 特立独行地选择了 `{}` 作为格式化占位符。它帮助用户减少了很多使用成本，你无需再为特定的类型选择特定的占位符，统一用 `{}` 来替代即可，剩下的类型推导等细节只要交给 Rust 去做。

- `{}` 适用于实现了 `std::fmt::Display` 特征的类型，用来以更优雅、更友好的方式格式化文本，例如展示给用户；
- `{:?}` 适用于实现了 `std::fmt::Debug` 特征的类型，用于调试场景；

```rust
//Debug
#[derive(Debug)]
struct Person {
    name: String,
    age: u8,
}

fn test_debug() {
    let p = Person{name: "rust".to_string(), age: 6};
    println!("{:?}", p);
}

//Display
use std::fmt::{Display, Formatter, Result};
impl Display for Person {
    fn fmt(&self, f: &mut Formatter) -> fmt::Result {
        write!(f, "name:{}, age:{}", self.name, self.age)
    }
}

fn test_display() {
    let p = Person{name: "rust", age: 6};
    println!("{}", p);
}
```

### 输出参数

* 位置参数：还能让指定位置的参数去替换某个占位符
* 具名参数：需要注意的是：**带名称的参数必须放在不带名称参数的后面**
* 格式化参数：进制、指针地址、转义

```rust
fn main() {
    println!("{1}{}{0}{}", 1, 2); //2112
    println!("{name}{}", 1, name = 2) //21
    println!("{:#x}", 27); //0x1b
    println!("{:#X}", 27); //0x1B
    
    let v = vec![1, 2, 4];
    println!("{:p}", v.as_ptr());
    
    println!("hello \"{{rust}}\" ");
}
```

## 2.6 方法Method

### 初识

对比一下Rust和其它语言的结构/类与方法在层次上的区别：

![img](https://pica.zhimg.com/80/v2-0d848e960f3279999eab4b1317f6538e_1440w.png)

### 关联函数(普通方法)

```rust
struct Circle {
    x: f64,
    y: f64,
    radius: f64,
}

impl Circle {
    fn new(x: f64, y: f64, radius: f64) -> Circle {
        Circle {
            x,
            y,
            radius,
        }
    }
    
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.radius * self.radius)
    }
    
    fn add_size(&mut self, x: f64, y: f64, r: f64) {
        self.x += x;
        self.x += y;
        self.radius += r;
    }
}

fn main() {
    let mut circle = Circle::new(1.1, 2.2, 3.3);
    circle.add_size(1, 2, 3);
    println!("area = {}", circle.area());
}
```

* 在一个 `impl` 块内，`Self` 指代被实现方法的结构体类型，`self` 指代此类型的实例:
  * `self` 表示 `Rectangle` 的所有权转移到该方法中，这种形式用的较少
  * `&self` 表示该方法对 `Rectangle` 的不可变借用
  * `&mut self` 表示可变借用
* 可以看到 `Circle` 中的 `new` 函数，这种定义在 `impl` 中且没有 `self` 的函数被称之为**关联函数**。因为它没有 `self`，不能用 `circle.area()` 的形式调用，因此它是一个函数而不是方法，它又在 `impl` 中，与结构体紧密关联，因此称为关联函数。
  * Rust 中有一个约定俗成的规则，使用 `new` 来作为构造器的名称，出于设计上的考虑，Rust 特地没有用 `new` 作为关键字；
  * 因为是函数，所以不能用 `.` 的方式来调用，我们需要用 `::` 来调用，比如: `circle.area()`；

### Rust方法调用: 自动引用和解引用

Rust 有一个叫 **自动引用和解引用**的功能。方法调用是 Rust 中少数几个拥有这种行为的地方。

他是这样工作的：当使用 `object.something()` 调用方法时，Rust 会自动为 `object` 添加 `&`、`&mut` 或 `*` 以便使 `object` 与方法签名匹配。也就是说，这些代码是等价的：

```rust
p1.distance(&p2);
(&p1).distance(&p2);
```

第一行看起来简洁的多。这种自动引用的行为之所以有效，是因为方法有一个明确的接收者———— `self` 的类型。在给出接收者和方法名的前提下，Rust 可以明确地计算出方法是仅仅读取（`&self`），做出修改（`&mut self`）或者是获取所有权（`self`）。事实上，Rust 对方法接收者的隐式借用让所有权在实践中更友好。

# 3 泛型和特征

## 3.1 Generics

```rust
//function
fn add<T>(a: T, b: T) -> T 
where
	T: std::ops::Add<Output = T>
{
    a + b
}

//struct
struct Point<T> {
    x: T,
    y: T,
}

fn test_struct() {
    let p1 = Point {x: 1, y: 2};
    let p2 = Ponit {x: 1.0, y: 2.0};
}

//methods
impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

impl Point<f32> {
    fn dis_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}

//const
fn display_arr<T: std::fmt::Debug, const N: usize>(arr: [T; N]) {
    println!("{:?}", arr);
}

fn test_const() {
    let arr: [i32; 3] = [1, 2, 3];
    display_arr(arr);
    let arr: [i32; 2] = [1, 2];
    display_arr(arr);
}
```

* 函数：涉及操作符时注意添加特征约束
* 枚举：`Option<T>/Result<T, E>`
* 方法：
  * 需要提前声明：`impl<T>`，只有提前声明了，我们才能在`Point<T>`中使用它，这样 Rust 就知道 `Point` 的尖括号中的类型是泛型而不是具体类型；
  * 我们可以针对特定的泛型类型实现某个特定的方法，对于其它泛型类型则没有定义该方法；
* 针对值的泛型，`const`

---

关于泛型的性能：

在 Rust 中泛型是零成本的抽象，意味着你在使用泛型时，完全不用担心性能上的问题。

但是任何选择都是权衡得失的，既然我们获得了性能上的巨大优势，那么又失去了什么呢？Rust 是在编译期为泛型对应的多个类型，生成各自的代码，因此损失了编译速度和增大了最终生成文件的大小。

## 3.2 Trait_0

### 基本使用

特征定义了**一组可以被共享的行为，只要实现了特征，你就能使用这组行为**。

```rust
pub trait Say {
    fn saying(&self) -> String;
}

pub struct Speaker {
    name: String,
    age: u32,
    word: String,
}

impl Say for Speaker {
    fn saying(&self) -> String {
        format!("name:{}, saying:{}", self.name, self.word)
    }
}

pub struct Video {
    id: u32,
    content: String,
}

impl Say for Video {
    fn saying(&self) -> String {
        format!("video({self.id}), saying:{self.content}")
    }
}

fn test_trait_base_use() {
    let s1 = Speaker {
        name: "tom".to_string(),
        age: 16,
        word: "rust".to_string(),
    }
    
    let s2 = Video {
        content: "jack".to_string(),
        id: 666,
    }
    
    println!("{}", s1.saying());
    println!("{}", s2.saying());
}
```

* 你可以在特征中定义具有**默认实现**的方法，这样其它类型无需再实现该方法，或者也可以选择重载该方法；

### 特征约束

```rust
pub fn notify<T: Summary>(item1: &T, item2: &T) {}
pub fn notify<T: Summary + Display>(item: &T) {}

//where
fn test_where<T: Display + Clone, U: Copy + Debug>(t: &T, u: &U) -> i32 {}
fn test_where<T, U>(t: &T, u: &U) -> i32
where T: Display + Clone
	  U: Copy + Debug
{}

//example1: 为自定义类型实现加法(+)操作
use std::ops::Add;
#[derive(Debug)]
struct Point<T: Add<T, Output=T>> {
    x: T,
    y: T,
}

impl<T: Add<T, Output=T>> Add for Point<T> {
    type Output = Point<T>;
    fn add(self, p: Point<T>) -> Point<T> {
        x: self.x + p.x,
        y: self.y + p.y,
    }
}

fn add<T: Add<T, Output=T>>(a: T, b: T) -> T {
    a + b
}

fn test_add_point() {
    let p1 = Point {x: 1.1f32, y: 2.2f32};
    let p2 = Point {x: 3.3f32, y: 4.4f32};
    println!("{:?}", add(p1, p2));
}

//example2
use std::fmt::{self, Display, Formatter, Result}

#[derive(Debug)]
enum FileState {
    Open,
    Closed,
}

impl Display for FileState {
    fn fmt(&self, f: &mut Formatter) -> Result {
        write!(f, 
            match *self {
                FileState::Open => "Open",
                FileState::Closed => "Closed",
            }
        )
    }
}

#[derive(Debug)]
struct File {
    name: String,
    data: Vec<u8>,
    state: FileState,
}

impl Display for File {
    fn fmt(&self, f: &mut Formatter) -> Result {
        write!(f, "<{}, ({})>", self.name, self.state)
    }
}

impl File {
    fn new(name: &str) -> Self {
        Self {
            name: name.to_string(),
            data: Vec::new(),
            state: FileState::Closed,
        }
    }
}

fn main() {
    let f =  File::new("hello.txt");
    println!("{:?}", f);
    println!("{}", f);
}
```

## 3.3 特征对象(动态分发)

### 基本使用

将多个对象渲染在屏幕上，这些对象属于不同的类型，存储在列表中，渲染的时候，需要循环该列表并顺序渲染每个对象，在 Rust 中的实现：

```rust
trait Draw {
    fn draw(&self);
}

struct Screen {
    components: Vec<Box<dyn Draw>>,
}

impl Screen {
    fn run(&self) {
        for x in self.components.iter() {
            x.draw();
        }
    }
}

struct Button {
    width: u32,
    height: u32,
    label: String,
}

impl Draw for Button {
    fn draw(&self) {
        println!("lablel: {}", self.label);
    }
}

struct SelectBox {
    width: u32,
    height: u32,
    options: Vec<String>,
}

impl Draw for SelectBox {
    fn draw(&self) {
        for (i, v) in self.options.iter().enumerate() {
            println!("option_{}: {}", i, v);
        }
    }
}

fn main() {
    let screen = Screen {
        components: vec![
            Box::new(Button {
                width: 10,
                height: 20,
                label: String::from("start"),
            }),
            Box::new(SelectBox {
                width: 30,
                height: 40,
                options: vec![
                    "OFF".to_string(),
                    "INIT".to_string(),
                    "DIRTY".to_string()
                ],
            }),
        ],
    };
    
    screen.run();
}
```

### 动态分发

Rust中的两种多态：泛型和特征对象。

* 泛型是在编译期完成处理的：编译器会为每一个泛型参数对应的具体类型生成一份代码，这种方式是**静态分发(static dispatch)**，因为是在编译期完成的，对于运行期性能完全没有任何影响。
* 与静态分发相对应的是**动态分发(dynamic dispatch)**，在这种情况下，直到运行时，才能确定需要调用什么方法。关键字 `dyn` 正是在强调这一“动态”的特点。

![img](https://pic1.zhimg.com/80/v2-b771fe4cfc6ebd63d9aff42840eb8e67_1440w.jpg)

>- **特征对象大小不固定**：这是因为，对于特征 `Draw`，类型 `Button` 可以实现特征 `Draw`，类型 `SelectBox` 也可以实现特征 `Draw`，因此特征没有固定大小；
>- **几乎总是使用特征对象的引用方式**，如 `&dyn Draw`、`Box<dyn Draw>`；

---

注意，不是所有特征都能拥有特征对象，只有对象安全的特征才行。当一个特征的所有方法都有如下属性时，它的对象才是安全的：

- 方法的返回类型不能是 `Self`
- 方法没有任何泛型参数



## //TODO: 3.4 Trait_1



# 4 生命周期

## 4.1 初识

## 4.2 深入

## 4.3 关于static



# 5 闭包/迭代器

## 5.1 Closure

### 基本使用

闭包是**一种匿名函数，它可以赋值给变量也可以作为参数传递给其它函数，不同于函数的是，它允许捕获调用者作用域中的值**。

```rust
/*
	|param1, param2,...| {
    	语句1;
    	语句2;
    	返回表达式
	}
*/

fn main() {
    let sum = |x, y| x + y;
    let v = sum(1, 2);
}
```

* 编译器在闭包的使用过程中会进行自动类型推导，但注意闭包不是泛型，**当编译器推导出一种类型后，它就会一直使用该类型**：

  ```rust
  // error 
  let example_closure = |x| x;
  
  let s = example_closure(String::from("hello"));
  let n = example_closure(5);
  ```

### 结构体的闭包使用

假设我们要实现一个简易缓存，功能是获取一个值，然后将其缓存起来，那么可以这样设计：

- 一个闭包用于获取值
- 一个变量，用于存储该值

调用 `value` 获取缓存中的值，如果有值则直接返回，没有值则将传入的参数 `arg` 存入：

```rust
struct Cacher<T, E> 
where:
	T: Fn(E) -> E,
	E: Copy
{
    query: T,
    value: Option<E>,
}

impl<T, E> Cacher<T, E>
where
	T: Fn(E) -> E,
	E: Copy
{
    fn new(query: T) -> Cacher<T, E> {
        Cacher {
            query,
            value: None,
        }
    }
    
    fn value(&mut self, arg: E) -> E {
        match self.value {
            Some(v) => v,
            None => {
                let v = (self.query)(arg);
                self.value = Some(v);
            }
        }
    }
}

fn main() {}

#[test]
fn test_closure() {
    let mut cacher = Cacher::new(|a| a);
    let v1 = cacher.value(1);
    let v2 = cacher.value(2);
    assert_eq!(v2, 1);
}
```

### 捕获作用域的值

闭包捕获变量有三种途径，恰好对应函数参数的三种传入方式：转移所有权、可变借用、不可变借用，因此相应的 `Fn` 特征也有三种：

* `FnOnce`: 该类型的闭包会拿走被捕获变量的所有权
* `FnMut`: 以可变借用的方式捕获环境中的值，因此可以修改该值
* `Fn`: 以不可变借用的方式捕获环境中的值

```rust
//FnOnce
fn fn_once<F>(func: F)
where
	F: FnOnce(usize) -> bool + Copy,
{
    println!("{}", func(3));
    println!("{}", func(4));
}

//FnOnce/FnMut/Fn
/*
	实际上，一个闭包并不仅仅实现某一种 Fn 特征，规则如下：
        所有的闭包都自动实现了 FnOnce 特征，因此任何一个闭包都至少可以被调用一次
        没有移出所捕获变量的所有权的闭包自动实现了 FnMut 特征
        不需要对捕获变量进行改变的闭包自动实现了 Fn 特征
*/
fn test_fn_fnmut_fnonce() {
    let s = String::new();
    let update_string = || println!("{}", s);
    exec(update_string);
    exec1(update_string);
    exec2(update_string);
}

fn exec<F: FnOnce()>(f: F) {
    f()
}

fn exec1<F: FnMut()>(f: F) {
    f()
}

fn exec2<F: Fn()>(f: F) {
    f()
}

///////////////////
fn main() {
    let x = vec![1, 2, 3];
    fn_once(|z| {z == x.len()} )
}
```

//闭包的所有权、闭包捕获变量的所有权转移

**一个闭包实现了哪种 Fn 特征取决于该闭包如何使用被捕获的变量，而不是取决于闭包如何捕获它们**。

#### 疑问

> 这章感觉 [闭包 Closure - Rust语言圣经(Rust Course)](https://course.rs/advance/functional-programing/closure.html) 讲得并不清晰，尤其是后面将 `move` 和 `FnOnce/FnMut/Fn` 混在了一起，但后面的评论有一些高质量回答，这里总结一下。

闭包相对于普通函数的一个优势在于：可以捕获同一作用域下的变量，关键在于捕获方式，当前环境中的变量的所有权是否转移到了闭包内部，或者只是以可变/不可变引用的方式进行捕获。Rust为闭包提供了三种trait: `FnOnce/FnMut/Fn`，三者是如何被闭包实现的文中提到过，`FnOnce` 被所有闭包默认实现，但并非所有闭包都以 "转移所有权“ 的方式捕获上下文环境中的变量，Rust有一些隐式实现完全取决于闭包内部的实际逻辑，以下闭包：

* `|| return s;` 闭包内部将s所有权转移，意味着该闭包必须从上下文环境中获取到s的所有权，所以只实现了 `FnOnce`；
* `|| s.push_str("xx");` 使用可变引用的方式即可实现闭包内部逻辑，那Rust默认以可变引用进行捕获；
* `|| println!("{}", s);` 使用不可变引用即可实现闭包内逻辑，那Rust默认以不可变引用进行捕获；
* 除了第一种只能转移所有权的闭包实现外，另外两种都存在以可变/不可变引用捕获变量的默认行为，那么可不可以改变这种行为呢？可以，这时候 `move` 关键字可以强制获取环境变量的所有权，改变了默认行为。 

### 闭包作为函数返回值

返回不同的闭包类型：

```rust
fn factory(x:i32) -> Box<dyn Fn(i32) -> i32> {
    let num = 5;

    if x > 1{
        Box::new(move |x| x + num)
    } else {
        Box::new(move |x| x - num)
    }
}
```

## 5.2 Iterator

[迭代器 Iterator - Rust语言圣经(Rust Course)](https://course.rs/advance/functional-programing/iterator.html)

### 基本使用

* for循环与迭代器

* next方法

* Iterator trait

  * into_iter/iter/iter_mut

* 消费者与适配器

  * 消费者适配器

    只要迭代器上的某个方法 `A` 在其内部调用了 `next` 方法，那么 `A` 就被称为**消费性适配器**。消费者适配器是消费掉迭代器，然后返回一个值；

  * 迭代器适配器

    那么迭代器适配器，顾名思义，会返回一个新的迭代器，这是实现链式方法调用的关键：`v.iter().map().filter()...`；

    与消费者适配器不同，迭代器适配器是惰性的，意味着你**需要一个消费者适配器来收尾，最终将迭代器转换成一个具体的值**。

    ```rust
    let v1: Vec<i32> = vec![1, 2, 3];
    let v2: Vec<_> = v1.iter().map(|x| x + 1).collect();
    assert_eq!(v2, vec![2, 3, 4]);
    ```

  * collect

    使用它可以将一个迭代器中的元素收集到指定类型中，比如说 `Vec<_>/HashMap<_, _>`；

  * 闭包作为迭代器适配器参数

    使用闭包来作为迭代器适配器的参数，它最大的好处不仅在于可以就地实现迭代器中元素的处理，还在于可以捕获环境值：

    ```rust
    struct Shoe {
        size: u32,
        style: String,
    }
    
    fn select_shoes_by_size(shoes: Vec<shoe>, shoe_size: usize) -> Vec<Shoe> {
        shoes.into_iter().filter(|s| s.size == shoe_size).collect()
    }
    ```

  * 迭代器获取索引 `enumerate`

    首先 `v.iter()` 创建迭代器，其次 调用 `Iterator` 特征上的方法 `enumerate`，该方法产生一个新的迭代器，其中每个元素均是元组 `(索引，值)`：

    ```rust
    let v = vec![1, 2, 3, 4, 5, 6];
    let val = v.iter()
    	.enumerate()
    	.filter(|&(idx,_)| idx % 2 == 0)
    	.map(|(_, val)| val)
    	.fold(0u64, |acc, x| acc + x);
    ```

### 为自定义类型实现Iterator

```rust
struct Counter {
    count: i32,
}

impl Counter {
    fn new() -> Self {
        Counter { count: 0 }
    }
}

impl Iterator for Counter {
    type Item = u32;
    
    fn next(&mut self) -> Option<Self::Item> {
        if self.count < 5 {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}

fn main() {
    let mut counter1 = Counter::new();
    
    assert_eq!(counter.next(), Some(1));
    assert_eq!(counter.next(), Some(2));
    assert_eq!(counter.next(), Some(3));
    assert_eq!(counter.next(), Some(4));
    assert_eq!(counter.next(), Some(5));
    assert_eq!(counter.next(), None);
    
    let sum: i32 = Counter::new()	 //[1, 2, 3, 4, 5]/[2, 3, 4, 5]
    	.zip(Counter::new().skip(1)) //[(1,2),(2,3),(3,4),(4,5)]
    	.map(|a, b| a * b)			 //[2,6,12,20]
    	.filter(|x| x % 3 == 0)		 //[6,12]
    	.sum();
    assert_eq!(18, sum);
}
```

# 6 智能指针

Rust中的指针类型有三种：

* **裸指针**：使用受限，需要使用unsafe封装

* **引用：**仅借用数据，不拥有数据

* **智能指针：**本质上是一个封装了裸指针的结构体，拥有数据，如 `String/Vec`都是容器类型的智能指针。

  智能指针实现了 `Deref` 和 `Drop` 特征：

  - `Deref` 可以让智能指针像引用那样工作，这样你就可以写出同时支持智能指针和引用的代码，例如 `*T`
  - `Drop` 允许你指定智能指针超出作用域后自动执行的代码，例如做一些数据清除等收尾工作

---

Rust中常用的智能指针类型：

- `Box<T>`，可以将值分配到堆上
- `Rc<T>`，引用计数类型，允许多所有权存在
- `Ref<T>/RefMut<T>`，允许将借用规则检查从编译期移动到运行期进行

## 6.1 Box堆内存分配

### 基本使用

```rust
fn main() {
    let a = Box::new(3);
    println!("a = {}", a);
    //let b = a + 1;
}
```

从以上代码可获知：

- `println!` 可以正常打印出 `a` 的值，是因为它隐式地调用了 `Deref` 对智能指针 `a` 进行了解引用
- 最后一行代码 `let b = a + 1` 报错，是因为在表达式中，我们无法自动隐式地执行 `Deref` 解引用操作，你需要使用 `*` 操作符 `let b = *a + 1`，来显式的进行解引用
- `a` 持有的智能指针将在作用域结束（`main` 函数结束）时，被释放掉，这是因为 `Box<T>` 实现了 `Drop` 特征

### 使用场景

* **避免栈上数据拷贝**

  栈上数据在拷贝的时候会逐字节的迁移过去，而堆对象的栈上持有者在拷贝时仅转移所有权 (只做了一次地址赋值操作)，因此大块数据尽量放在堆上，不只是因为堆空间足够大，也出于拷贝性能的考量。

* **将动态类型变为大小确定的类型 (数组存储不同类型的元素)**

  利用特征对象，实际上就是Box封装的智能指针指向不同的堆对象 `Box<dyn Draw>`，智能指针是一个结构体，因此大小可在编译期确定。

* `Vec<Box<i32>>` 内存布局

  智能指针 `vec2` 依然是存储在栈上，然后指针指向一个堆上的数组，该数组中每个元素都是一个 `Box` 智能指针，最终 `Box` 智能指针又指向了存储在堆上的实际值。

* `Box::leak`

  一个简单的场景，**你需要一个在运行期初始化的值，但是可以全局有效，也就是和整个程序活得一样久**，那么就可以使用 `Box::leak`，例如有一个存储配置的结构体实例，它是在运行期动态插入内容，那么就可以将其转为全局有效。

  ```rust
  fn main() {
      let s = gen_static_str();
      println!("static_str = {}", s);
  }
  
  fn gen_static_str() -> &'static str {
      let mut s = "hello, ".to_string();
      s.push_str("rust");
      
      Box::leak(s.into_boxed_str())
  }
  ```

## 6.2 Deref解引用

> 这部分需要理解的是Rust的自动 (隐式) 解引用是怎么做的，这决定了你能否看懂或写出更简洁的Rust代码。

自定义一个 “智能指针 `MyBox`”，并实现 `Deref` 特征：

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}

impl<T> Deref for MyBox<T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        &self.0s
    }
}

fn main() {
    let x = MyBox::new(666);
    assert_eq!(*x, 666);
}
```

* 关于Rust的自动解引用，[Deref 解引用 - Rust语言圣经(Rust Course)](https://course.rs/advance/smart-pointer/deref.html#函数和方法中的隐式-deref-转换) 讲的有点乱，首先自动解引用和 `Deref` 特征两者之间并没有直接的映射关系，Rust在使用方法 `methods` 的时候会对 `&mut T/&T` 进行自动解引用操作，这和 `T` 是否实现了 `Deref` 特征没关系，如：

  ```rust
  fn main() {
      let x = 1;
      let y = &x;
      println!("{}", y); //*(&x)
  }
  ```

  而你如果实现了 `Deref` 特征，那么就会出现 `*(y.deref())` 这样的隐式自动调用，如：

  ```rust
  fn main() {
      let s = MyBox::new(String::from("hello world"));
      display(&s)
  }
  
  fn display(s: &str) {
      println!("{}",s);
  }
  ```

## 6.3 Drop资源释放

### 资源回收顺序

```rust
struct HasDrop1;
struct HasDrop2;
impl Drop for HasDrop1 {
    fn drop(&mut self) {
        println!("Dropping HasDrop1!");
    }
}
impl Drop for HasDrop2 {
    fn drop(&mut self) {
        println!("Dropping HasDrop2!");
    }
}
struct HasTwoDrops {
    one: HasDrop1,
    two: HasDrop2,
}
impl Drop for HasTwoDrops {
    fn drop(&mut self) {
        println!("Dropping HasTwoDrops!");
    }
}

struct Foo;

impl Drop for Foo {
    fn drop(&mut self) {
        println!("Dropping Foo!")
    }
}

fn main() {
    let _x = HasTwoDrops {
        two: HasDrop2,
        one: HasDrop1,
    };
    let _foo = Foo;
    println!("Running!");
}

/*
    Running!
    Dropping Foo!
    Dropping HasTwoDrops!
    Dropping HasDrop1!
    Dropping HasDrop2!
*/
```

可以得出以下关于 `Drop` 顺序的结论

- **变量级别，按照逆序的方式**，`_x` 在 `_foo` 之前创建，因此 `_x` 在 `_foo` 之后被 `drop`
- **结构体内部，按照顺序的方式**，结构体 `_x` 中的字段按照定义中的顺序依次 `drop`

### 手动回收

* 对于 Rust 而言，不允许显式调用析构函数，因为Drop特征中的 `drop` 函数是以可变借用的方式处理回收逻辑，这有可能让后续代码访问到并不存在的数据。你可以尝试使用 `std::mem::drop` 以转移目标值所有权的方式进行回收，这种实现保证了后续的使用必定会导致编译错误，因此非常安全。
* Rust的动态内存管理使用起来更加简洁，但相比于C隐藏起来更多细节，你甚至在定义堆对象的时候连分配堆内存的接口都是透明的，因为Rust将堆内存分配的操作都封装在各种动态类型 `Box/String/Vec::new()` 的内部；
* 自定义 `Drop` 特征的情况较少，因为你无需担心堆内存回收工作，Rust默认的 `drop` 帮你完成了。除非存在一些特定的系统共享资源，例如文件描述符、网络 socket 等，当这些值超出作用域不再使用时，就需要进行关闭以释放相关的资源。
* `Copy/Drop` 特征是互斥的：我们无法为一个类型同时实现 `Copy` 和 `Drop` 特征。因为实现了 `Copy` 的特征会被编译器隐式的复制，因此非常难以预测析构函数执行的时间和频率。因此这些实现了 `Copy` 的类型无法拥有析构函数。

## 6.4 *Rc/Arc多所有权机制

### Rc<>

> 如何理解所有权，资源所有权支持者和借用者在资源使用管理上有什么区别？
>
> * 对资源类型T，存在三种使用方式：`Box<T>/Rc<T>/&T`
>   * `&T`
>     * 不涉及资源清理工作，引用也是有生命周期的，但引用销毁了仅仅是该引用结束了，资源依旧存在
>   * `Box<T>`
>     * 发生赋值拷贝动作时，资源所有权转移，从始至终资源持有者 `Box<T>` 只有一个，即Rust不允许第二个资源持有者 `Box<T>` 存在
>     * 销毁时回收指向的内存资源
>   * `Rc<T>`
>     * 发生赋值拷贝动作时，资源的引用计数器+1，内部封装的指针也会指向资源（作了浅拷贝）。这种情况下，Rust允许多个 `Rc<T>` 真正的指向同一块内存。
>     * 销毁时，资源的引用计数器+1，引用计数减至0时回收内存资源
>
> 以上就是Rust维护资源的三种常用方式，分析一下它们各自的使用场景：
>
> * `&T`：仅仅是访问需求，关于资源何时被回收、能否及时被回收，`&T` 并不关心；
>
> * `Box<T>`：可以确保唯一的资源持有者。
>
>   > ***加深对所有权的理解：***
>   >
>   > 之前有过一个疑问 `let x = Box::new(3); let mut y = &x;` 这样的代码中，`x/y` 都可以访问到资源，在使用的时候看起来一片和谐。实际上，`y` 的可访问时间段极其不自由，因为它完全却决于资源何时被释放掉，那么谁来管理资源释放呢？没错，正是这个资源持有者 (拥有资源所有权) 的 `x`，因此 `y` 是完全听命于 `x` 的。Rust系统规定引用生命周期总是短于资源生命周期，这的确保证了安全性(避免悬空引用)但也牺牲了灵活性，这意味着引用不能随处使用了，始终活在 `y` 资源持有者所圈定的区间中。
>
> * `Rc<T>`：允许存在多个资源持有者owner，owner之间互不干扰，他们可以随处访问资源，因为他们掌管资源。

---

当我们**希望在堆上分配一个对象供程序的多个部分使用且无法确定哪个部分最后一个结束时，就可以使用 `Rc` 成为数据值的所有者**。

> 考虑一个场景，有很多小工具，每个工具都有自己的主人，但是存在多个工具属于同一个主人的情况，此时使用 `Rc<T>` 就非常适合：

```rust
use std::rc::Rc;

struct Owner {
    name: string,
}

struct Gadget {
    id: i32,
    owner: Rc<Owner>,
}

fn main() {
    let owner: Rc<Owner> = Rc::new(Owner {
        name: "rust".to_string(),
    });
    
    let g1 = Gadget {
        id: 0,
        owner: Rc::clone(&owner),
    };
    
    let g1 = Gadget {
        id: 1,
        owner: Rc::clone(&owner),
    };
    
    drop(owner);
    println!("{}, {}", g1.id, g1.owner.name);
    println!("{}, {}", g2.id, g2.owner.name);
}
```

* 事实上，`Rc<T>` 是指向底层数据的不可变的引用，因此你无法通过它来修改数据，这也符合 Rust 的借用规则：要么存在多个不可变借用，要么只能存在一个可变借用；
* `Rc<T>` 是一个智能指针，实现了 `Deref` 特征，因此你无需先解开 `Rc` 指针，再使用里面的 `T`，而是可以直接使用 `T`，例如上例中的 `gadget1.owner.name` ;
* `Rc` 只能用于同一线程内部，想要用于线程之间的对象共享，你需要使用 `Arc`；
  * 由于 `Rc<T>` 需要管理引用计数，但是该计数器并没有使用任何并发原语，因此无法实现原子化的计数操作，最终会导致计数错误；
  * `Arc` 和 `Rc` 并没有定义在同一个模块，前者通过 `use std::sync::Arc` 来引入，后者通过 `use std::rc::Rc`；

---

### Arc<>

`Arc` 和 `Rc` 拥有完全一样的API：

```rust
use std::sync:Arc;
use std::thread;

fn main() {
    let s_arc = Arc::new(String::from("rust"));
    for _ in 0..10 {
        let s = Arc::clone(&s_arc);
        let handle = thread::spawn(move || {
            println!("{}". s)
        });
    }
}
```

## 6.5 *Cell/RefCell内部可变性

> `Cell/RefCell` 在功能上没有区别，区别在于 `Cell<T>` 只适用于 `T` 实现 `Copy` 的情况，因此不常用。

Rust的借用规则是极其严格的，它考虑到了几乎所有场景，但在用户定制化的场景下有些Rust认为的“unsafe”操作却是合理的，智能指针相当于Rust开辟的一块灵活性更强的“异世界”，在这里有很多规则得到了扩展，和Rust原本的世界很不一样：

| Rust 规则                            | 智能指针带来的额外规则                  |
| ------------------------------------ | --------------------------------------- |
| 一个数据只有一个所有者               | `Rc/Arc`让一个数据可以拥有多个所有者    |
| 要么多个不可变借用，要么一个可变借用 | `RefCell`实现编译期可变、不可变引用共存 |
| 违背规则导致**编译错误**             | 违背规则导致**运行时`panic`**           |

- 与 `Cell` 用于可 `Copy` 的值不同，`RefCell` 用于引用
- `RefCell` 只是将借用规则从编译期推迟到程序运行期，并不能帮你绕过这个规则
- `RefCell` 适用于编译期误报或者一个引用被在多处代码使用、修改以至于难于管理借用关系时
- 使用 `RefCell` 时，违背借用规则会导致运行期的 `panic`

### 内部可变性

虽然基本借用规则是 Rust 的基石，然而在某些场景中，**一个值可以在其方法内部被修改，同时对于其它代码不可变，**是很有用的。

```rust
use std::cell::RefCell;

// 定义在外部库中的特征
pub trait Messenger {
    fn send(&self, msg: String);
}

// --------------------------
// 我们的代码中的数据结构和实现
struct MsgQueue {
    msg_cache: RefCell<Vec<String>>,
}

impl Messenger for MsgQueue {
    fn send(&self, msg: String) {
        self.msg_cache.borrow_mut().push(msg);
    }
}

fn main() {
    let mq = MsgQueue {
        msg_cahce: Refcell::new(Vec::new());
    };
    mq.send("rust".to_string());
}
```

这个 MQ 功能很弱，但是并不妨碍我们演示内部可变性的核心用法：通过包裹一层 `RefCell`，成功的让 `&self` 中的 `msg_cache` 成为一个可变值，然后实现对其的修改。

### Rc+RefCell

在 Rust 中，一个常见的组合就是 `Rc` 和 `RefCell` 在一起使用，前者可以实现一个数据拥有多个所有者，后者可以实现数据的可变性：

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let s = Rc::new(RefCell::new("rust".to_string()));
    let s1 = s.clone(); //s1 = Rc::clone(&s);
    let s2 = s.clone();
    s2.borrow_mut().push_str("is best!");
}
```

---

* `RefCell` 适用于编译器误报或者一个引用被在多个代码中使用、修改以至于难于管理借用关系时，还有就是需要内部可变性时；
* 从性能上看，`RefCell` 由于是非线程安全的，因此无需保证原子性，性能虽然有一点损耗，但是依然非常好；
* `Rc` 跟 `RefCell` 结合使用可以实现多个所有者共享同一份数据，非常好用，但是潜在的性能损耗也要考虑；

## 6.6 *Weak与循环引用

> 注意: Weak能优雅地解决循环引用问题，但其存在的意义不止于此。

总的来说，Weak应用于以下两种场景：

* 持有一个 `Rc` 对象的临时引用，并且不在乎引用的值是否依然存在**（更加常见）**
* 阻止 `Rc` 导致的循环引用，因为 `Rc` 的所有权机制，会导致多个 `Rc` 都无法计数归零

使用方式简单总结下：**对于父子引用关系，可以让父节点通过 `Rc` 来引用子节点，然后让子节点通过 `Weak` 来引用父节点**。

---

```rust
use std::rc::Rc;

fn main() {
    // 创建Rc，持有一个值5
    let five = Rc::new(5);

    // 通过Rc，创建一个Weak指针
    let weak_five = Rc::downgrade(&five);

    // Weak引用的资源依然存在，取到值5
    let strong_five: Option<Rc<_>> = weak_five.upgrade();
    assert_eq!(*strong_five.unwrap(), 5);

    // 手动释放资源`five`
    drop(five);

    // Weak引用的资源已不存在，因此返回None
    let strong_five: Option<Rc<_>> = weak_five.upgrade();
    assert_eq!(strong_five, None);
}
```

* 通过 `use std::rc::Weak` 引入
* 可访问，但没有所有权，不增加引用计数，因此不会影响被引用值的释放回收
* 可由 `Rc<T>` 调用 `downgrade` 方法转换成 `Weak<T>`
* `Weak<T>` 可使用 `upgrade` 方法转换成 `Option<Rc<T>>`，如果资源已经被释放，则 `Option` 的值是 `None`

# 7 全局变量

## 7.1 编译期初始化

* 无论是常量还是静态变量，两者定义的时候必须赋值为**在编译期间就可以计算出来的值 (常量表达式/数学表达式)，而不能是运行时才能计算出的值 (如函数)。**

  * 常量、字符串字面值无需 `static` 声明，它们已经被打包到二进制可执行文件中；

    ```rust
    const MAX_VALUE: usize = usize::MAX / 2;
    fn main() {
        println!("{}", MAX_VALUE);
    }
    ```

  * 静态变量允许声明一个全局的变量，常用于全局数据统计：

    ```rust
    static mut GLOBAL_MUT: usize = 0;
    fn main() {
        unsafe {
            GLOBAL_MUT += 1;
        }
    }
    ```

    为什么加 `unsafe`，因为Rust怀疑你可能在多线程场景下访问这个全局变量，这是不安全行为。如果你一定要这么做，且不像使用 `unsafe` 封装，可以使用原子类型：

    ```rust
    use std::sync::atomic::{AtomicUsize, Ordering};
    static REQUEST_RECV: AtomicUsize  = AtomicUsize::new(0);
    fn main() {
        for _ in 0..100 {
            REQUEST_RECV.fetch_add(1, Ordering::Relaxed);
        }
    }
    ```

    如果面对不能使用 `std` 的场景，比如写一个裸机程序，可以选择用 `RefCell<T>` 来实现内部可变性。

  * 静态变量和常量的区别？

    * 静态变量不会被内联，在整个程序中，静态变量只有一个实例，所有的引用都会指向同一个地址
    * 存储在静态变量中的值必须要实现 Sync trait


## 7.2 运行期初始化

静态变量在声明时就必须进行初始化，且需要给出一个编译期就能确定的值，像通过函数初始化这种方式就不行。为什么不行？因为Rust 的借用和生命周期规则限制了我们做到这一点，试图将一个局部生命周期的变量赋值给全局生命周期的`CONFIG`，这明显是不安全的。

> 全局变量在程序运行过程中存在随时被修改的需求可以理解，但初始化的时间节点为什么要在运行时，有这种场景吗？当然，**一个全局的动态配置，它在程序开始后，才加载数据进行初始化，最终可以让各个线程直接访问使用。**

### lazy_static!

静态变量的运行时初始化可以用 `lazy_static!` 实现：

```rust
use lazy_static::lazy_static;
use std::collections::HashMap;

lazy_static! {
    static ref HASH_MAP: HashMap<u32, &str> = {
        let mut hash = HashMap::new();
        hash.insert(0, "a");
        hash.insert(1, "b");
        hash.insert(2, "c");
        hash
    }
}

fn main() {
    println!("{}", HASH_MAP.get(&0).unwrap());
}
```

### Box::leak

`Rust` 还提供了 `Box::leak` 方法，但并不常用。它可以将一个变量从内存中泄漏，然后将其变为 `'static` 生命周期，最终该变量将和程序活得一样久，因此可以赋值给全局静态变量 `CONFIG`。

```rust
#[derive(Debug)]
struct Config {
    a: String,
    b: String,
}
static mut CONFIG: Option<&mut Config> = None;

fn init() -> Option<&'static mut Config> {
    let c = Box::new(Config {
        a: "hello".to_string(),
        b: "rust".to_string(),
    });
    Some(Box::leak(c))
}

fn main() {
    unsafe {
        CONFIG = init();
        println!("{:?}", CONFIG)
    }
}
```

### OnceCell

 `cell::OnceCell` 和 `sync::OnceLock`，前者用于单线程，后者用于多线程，它们用来存储堆上的信息，并且具有最多只能赋值一次的特性。如实现一个多线程的日志组件 `Logger`：

```rust
use std::{syc::OnceLock, thread};

#[derive(Debug)]
struct Logger;

static LOGGER: OnceLock<Logger> = OnceLock::new();

impl Logger {
    fn global() -> &'static Logger {
        Logger.get_or_init(|| {
            println!("Logger init");
            Logger
        })
    }
    
    fn log(&self, info: String) {
        println!("{}", info);
    }
}

fn main() {
    let logger = Logger::global();
    logger.log("hello".to_string());
    logger.log("rust".to_string());
    
    let handle = thread::spawn(|| {
        let logger = Logger::global();
        logger.log("thread info".to_string());
    });
    
    handle.join().unwrap();
}
```

---

总结一下，全局变量可以分为两种：

- 编译期初始化的全局变量
  * `const`创建常量
  * `static`创建静态变量
  * `Atomic`创建原子类型
- 运行期初始化的全局变量
  * `lazy_static`用于懒初始化
  * `Box::leak`利用内存泄漏将一个变量的生命周期变为`'static`

# 8 Rust难点

## 8.1 切片/切片引用



## 8.2 Eq/PartialEq





