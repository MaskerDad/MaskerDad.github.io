---
title: Rust_base.md

date: 2024-1-12 17:00:00 +0800

categories: [Rust]

tags: [Rust]

description: 
---

# 0 参考

> 本文用于记录学习Rust重要的语法与机制。

* [Rust圣经](https://course.rs/about-book.html)



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







## 3.4 Trait_1











# 4 生命周期

## 4.1 初识

## 4.2 深入

## 4.3 关于static



# 5 闭包/迭代器



# 6 智能指针



# 7 全局变量



# 8 模块系统

