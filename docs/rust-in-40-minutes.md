---
title: "Rust in 40 minutes"
permalink: /rust-in-40-minutes/
---

# Rust in 40 minutes

会涉及到的内容：

- Rust 的安全性
- Rust 的表达能力

## Preface

- 我们为什么需要 Rust ？

作为一门系统编程语言，Rust 的主要竞争者就是 C/C++。Rust 是如何从 C/C++ 的市场中获得一席之地的呢？

Rust 解决了 C/C++ 中的一些问题：

- 标准库的贫乏
- 缺乏*合理*的模块系统（现在有了一半）和包管理机制
- 安全性（Safety）
- ...

### What is Safety?

安全性相当于编程语言在代码中添加的一些约束，程序员能够依赖这些约束提供的假设来构建能够顺利运行的应用程序

例如静态类型语言的类型系统就是一种约束，它能够避免大多数运行时的类型错误。

而为了保证这些约束的正确性，程序员（或者编译器）需要在编写代码时提供这些约束的 *证明*。

这种证明可以是需要提供的 *具有某种类型的表达式*，也可以是代码中上下文里包含的某种信息

以 Java 中泛型参数的 Type Bounds 为例：

```java
class Foo {
  public static <T extends Comparable<T>> bool greater(T a, T b) {
    return a.compareTo(b) > 0;
  }
}
```

这里我们通过 Type Bounds 规定了类型参数 `T` 的上界，所以我们有理由 *相信*：

- 传入的类型为 `T` 的参数一定也是一个 `Comparable` 的实例

于是我们可以在拥有这个上下文的基础下使用 `a::compareTo` 方法。

而在调用这个函数时，通过显式指定/类型推导出 `T` 的类型满足这个 Type Bounds 的过程，就是关于这个约束的证明。

## C++'s Efforts on Safety issues

C++ 相较于 C 而言已经通过一些机制来提供了一定程度上的安全性，在 *现代C++* 中

- 通过 RAII 实现基于作用域的自动资源回收和权限管理
- 通过禁用 Copy Constructor 表达“独占”概念的 `std::unique_ptr`
- 通过 SFINAE/Concept 对抽象的类型进行检查

但始终仍然缺乏由编程语言本身提供的安全性保证，例如：

- C++ 中的移动语义需要被移动的对象拥有某种 *空* 状态，使得对象能够在被移动之后继续使用；这在很多时候是不符合我们对于移动语义的期待的
- C++ 的 *constness* 无法保证对象的不可变性（常引用仅仅限制的是该引用对对象的修改，而不保证引用的对象不能被修改）
- C++ 多线程数据竞争需要程序员自己来避免，而不是编译器

## Safety in Rust

Rust 的安全性主要通过下面几个方式来实现：

- 通过 Affine/Linear Type System 实现的 Ownership 模型
- 可变引用独占性保证的不可变性
- 通过 Tag Trait 保证的多线程下数据访问的安全性


### Ownership and Borrowing

Rust 语言中最核心的概念之一就是所有权（ownership）:

- Rust 中值都会被一个变量所持有，该变量叫做它的 *Owner*
- 值在同一时刻只能被一个 Owner 所持有
- 当 Owner 离开作用域时，它所拥有的值都会被销毁

```rust
{
    let x = String::from("hello");
    let y = x; // x 的所有权被移交给 y, x 
}
// x 和 y 在这里都不再有效
```

值能够在不同的变量中进行传递，这种传递依赖的就是移动语义。

#### Move Semantics

同样作为提供了移动语义的语言，Rust 的移动相较于 C++ 具有两个特性：

- 普适性：Rust 中所有类型的值都能够被移动（原则上可以视作直接复制源变量的内存数据）
- 破坏性：Rust 中的值在移动之后，将不再可用

也就是说：

- 在 C++ 中，对象默认发生拷贝，需要手动对对象进行移动；
- 在 Rust 中，对象默认发生移动，需要手动对对象进行拷贝（实现了 `Copy` trait 的类型例外）

#### Reference

作为一门系统编程语言，仅仅只有移动语义是不够的，很多时候：

- 我们并不需要一个值的所有权，而只是需要它的一个 *视图*（或者说，只是需要一个借用（Borrowing））
- 我们并不希望将一个值移出来，而只是想修改它的内容

这里就会涉及到变量的引用以及可变性。

和 C++ 中类似，Rust 的引用也类似于一个无法被修改的指针。

```c++
template<typename T>
using Ref<T> = T * const;
```

但不一样的是，作为一门安全的语言，Rust 需要给引用提供约束：

- 引用必须是可用的：在使用引用时，被引用的对象必须处于可用状态
- 不可变引用需要保证在使用引用时，每一次对被引用对象的访问都能得到相同的结果

~~这两点在 C++ 中都能轻易地被打破。~~

为了保证前一点，Rust 为引用引入了生命周期类型参数。

```rust
type Ref<T, 'a> = * const T; 
// 示意代码：'a 代表被引用的 T 类型变量的生命周期
// 代码中写作：&'a T
```

#### Lifecycle

什么是生命周期？

变量的生命周期对应于它在代码中保持 *存活* 的一段区间

- 开始于它被声明或作为参数传入
- 结束于它被移动或是离开作用域

```rust
{
  let r;                // ---------+-- 'a
  {                     //          |
    let x = 5;          // -+-- 'b  |
    r = &x;             //  |       |
  }                     // -+       |
  println!("r: {}", r); //          |
}                       // ---------+
```

引用的可用意味着我们要提供这样一个证明：变量引用的生命周期需要短于（或者说被包含于）被引用变量的生命周期

当引用仅仅是在一个函数内部使用时，编译器会自动计算出引用的对象的生命周期参数，并且视图构造下面的命题的证明：

- 生成的引用作为变量的生命周期短于该引用类型中标注的生命周期

那么当引用需要作为参数传递呢？我们则需要把函数定义为一个泛型函数，而它的泛型参数为生命周期：

```rust
fn get<'a>(a: &'a i32) -> i32 {
  *a
}

{
  let x: i32 = 1; // --+-- 'b
  get(&x);        //   |
}                 // --+
```

这样在这个函数调用时，我们实际上会得到一个 `'a` 的实际类型为 `'b` 的 `get` 实例，而且函数参数 `a` 也会拥有实际的类型： `&'b i32` 

那么我们现在就可以放心地在 `get` 里使用这个引用了。

根据生命周期定义中的 *包含* 语义，我们可以发现生命周期天然具有 subtyping 特性，即：

生命周期 `'a` 包含另一个生命周期 `'b`（`'a` 比 `'b` 存活的时间更长），那么我们可以认为 `'a` 可以当做 `'b` 来使用。

而生命周期又同时是引用类型的参数，所以我们可以发现引用类型的生命周期参数实际上是协变的。下面一个例子：

```rust
fn longest<'a>(x: &'a String, y: &'a String) -> &'a String { 
  if x.len() > y.len() { x } else { y }
}
{
  let x = "x".to_string();        // --------+ 'd
  let result;                     // --------+ 'e
  {                               //         |
    let y = "y".to_string();      // --+ 'c  |
    result = longest(&x, &y);     //   |     |
  }                               // --+     |
  println!("{}", result);         //         |
}                                 // --------+
```

我们可以发现，`x` 和 `y` 的生命周期其实并不一样，即 `&x` 和 `&y` 分别拥有类型 `&'d String` 和 `&'c String`。

不过根据生命周期参数的协变特性，我们可以认为 `&'d String` 是 `&'c String` 的子类型，就可以把 `&x` 和 `&y` 同时作为 `&'c String` 传入 `longest` 函数中。

但这时我们再检查一下变量 `result` 的生命周期，我们会发现拥有生命周期 `'e` 的 `result` 却持有了一个生命周期为 `'c` 的对象的引用，这打破了我们之前对于引用安全的假设：

> 生成的引用作为变量 `result` 的生命周期理应短于该引用类型中标注的生命周期 `'c`，然而 `'e` 却要长于 `'c`

编译器于是会生成一个编译错误，阻止这样不安全的程序进入后面的编译流程。

当然，并不是所有的生命周期参数都需要以参数的形式显式地标记在函数上。

Rust 编译器提供了很多情况下的省略生命周期参数功能，用来减少程序员在 *提供证明* 时需要完成的工作。

#### Mutability

Rust 中默认声明的变量或函数参数都是不可变的，除非添加上 `mut` 修饰符。

变量的不可变性是一个非常好的特性：虽然我们无法修改它的值，但是可以被保证：该变量在其生命周期里的每一次访问，都能得到相同的结果。

虽然在 Rust 里我们也可以只使用不可变数据结构，但作为一门系统编程语言（而不是能够轻易容忍函数调用和递归的函数式语言），很难不和这种副作用产生联系。

那么 Rust 该如何保证 *安全的* 变量可变性呢？

如果没有引用的话，那么这个这个问题会相当简单：除了这个变量本身，没有其它方式来对它所持有的值进行修改。我们的所有操作只能通过移动来实现。

而一旦引入了引用，问题就会变得相当复杂。

对于不可变引用来说，我们希望它能够 *安全地代表* 一个不可变的值的视图，也就是说

- 拥有不可变引用这个类型的变量不能够对它所引用的值进行修改
- 拥有不可变引用这个类型的变量所引用的值也不能被其它任何东西所修改

而对于可变引用而言，我们希望它能 *安全地代表* 一个可（以被这个引用所改）变的值的视图，即：

- 拥有不可变引用这个类型的变量可以对它所引用的值进行修改
- 拥有不可变引用这个类型的变量所引用的值不能被其它任何东西所修改

很容易发现，这几条规则和读写锁非常类似：

- 不可变引用类似于读锁，可以与其它的不可变引用兼容
- 可变引用类似于写锁，与其它任何引用都不兼容

只不过这些 '锁' 和约束仅仅适用于编译期，并不会对运行时产生影响。

下面是一个示例：

```rust
fn foo(&)
{
  let mut x = "x".to_string();
  {
    let y = &x;
    // 我们在 y 的生命周期内无法再对 x 进行修改/移动
    // 但我们仍然能创建 x 的不可变引用
    let z = &x;
    println!("{}", z);
    println!("{}", y);
  }
  {
    let y = &mut x;
    // 我们在 y 的生命周期内不能再对 x 进行修改/移动，也不能再对 x 创建任何引用
    y.push_str("y");
    println!("{}", y);
  }
}
```

在满足了这些约束的情况下，我们才能说：我对这个变量、或者这个引用所引用的变量的修改，是不会破坏我们之前提供的这些假设的。

#### Heap Allocation And Interior Mutability

在 Rust 中，对象的移动虽然是 trivial 的，但并不一定是廉价的。无论是从性能，还是从实践上的问题来看，一个过于庞大的对象都不适合直接在栈中进行分配和移动。

所以我们会需要在堆上进行对象的分配。而为了保证对于这些操作的安全性，Rust 使用了和 C++ 类似的智能指针进行管理。

最简单的智能指针就是 `Box`，它对应于 C++ 中的 `std::unique_ptr`，表示对于存储在堆区域中的一个对象的 *独占性引用*。

不过和 Rust 原生引用不同在于，`Box` 所引用的对象和 `Box` 本身拥有 **相同** 的生命周期

- `Box` 创建时会在堆中分配一块和它绑定的内存
- `Box` 在销毁时会同时销毁掉它在堆中引用的对象
- `Box` 在移动时会将堆中对象的所有权一并 *移动* 到目标变量中

而且 Rust 中还为 `Box` 实现了 `Deref` trait，使得在使用 `Box` 对象时可以直接把它当做它所引用的对象来使用（并拥有更小的移动开销）

拥有 `Box` 之后我们就能实现一些递归的数据结构了。

例如最简单的 Lisp 风格链表，在没有 `Box` 时我们只能这样写：

```rust
enum List<T> {
  Nil, Cons(T, List<T>) // recursive without indirection
}
```

看上去非常正确，但是这份代码是无法通过编译的：

> error[E0072]: recursive type `List` has infinite size

`List` 的对象在创建时我们需要为它分配一块内存，而枚举类型的实例大小则至少等于它最大的一种构造器的大小，这就会产生递归的大小计算并最终得到发散的结果。

而如果把 `Cons` 里的 `List` 包装在 `Box` 中，由于 `Box` 具有固定的大小（只是一个指针罢了），最终得到的 `List` 对象大小也是合理的。

到这里可能有小朋友就要说了：`Box` 不是仅仅是个指针吗，我对 `Box` 所指向的内容进行修改并没有对指针的值进行修改，为什么也需要 `Box` 本身的可变性呢？

答案很简单：这是由 `Box` 的 **语义** 决定的。Rust 希望 `Box` *直接代表* 它所指向的数据，引用堆内存带来的间接性是对于 `Box` 的使用者透明的（实现了 `Deref` 也是这个目的。

那么有没有一种方式能够实现这种 *间接* 的引用呢？这就需要引入一种设计模式：Interior Mutability（内部可变性）。

Rust 中唯一可以打破 “同时被多个变量引用的值无法被改变” 这个约束的类型就是 `std::cell::UnsafeCell`，而所有的内部可变性数据结构都必须使用 `UnsafeCell` 实现。

`UnsafeCell` 仅仅是在它的类型参数对应的类型上添加的一层没有任何额外开销的包装，但它提供了一个方法能够强行从不可变引用中获取可变引用：

```rust
pub struct UnsafeCell<T: ?Sized> {
    value: T,
}

impl<T: ?Sized> UnsafeCell<T> {
  pub const fn get(&self) -> *mut T {
    self as *const UnsafeCell<T> as *const T as *mut T
  }
}
```

也就是说，`UnsafeCell` 是 Rust 官方提供的编译期可变性检查上的洞，而在使用这个洞的同时如何保证之前的安全性约束，则需要通过 `UnsafeCell` 的使用者来提供。

`UnsafeCell` 上最常见的封装就是 `RefCell` 和 `Cell` 了。

`RefCell` 提供了针对引用的内部可变性，而 `Cell` 提供了针对值和移动语义的内部可变性。

```rust
let value = RefCell::new(1);;
let x = &value;
let y = &value;
*x.borrow_mut() += 2;
assert_eq!(*y.borrow(), 3);
*y.borrow_mut() += 2;
assert_eq!(*x.borrow(), 5);
```

这里我们对 `value` 创建了两个不可变引用，但通过 `RefCell` 实现的内部可变性，我们能够通过 `borrow_mut` 来获得它间接引用的值的可变引用，并进行修改和访问。

`Cell` 的话则稍微有一点区别：

```rust
let value = Cell::new(1);
let x = &value;
let y = &value;
x.set(2);
assert_eq!(2, y.get());
y.set(3);
assert_eq!(3, x.get());
value.take();
assert_eq!(0, x.get());
```

比起 `RefCell` 通过 `borrow_mut` 拿到的可变引用，`Cell` 能够直接将其包装的值移动出去，或是替换为新的值并将原本的值销毁。

但内部可变性并不意味着放弃了对于引用的安全性，即对 `RefCell` 的引用并不等于对它所代表的对象的引用。

通过 `RefCell::borrow` 和 `RefCell::borrow_mut` 创建出的 *引用* 依然需要遵守前面的类似于读写锁的约束。

对于同一个 `RefCell` 创建的：

- 不同 `std::cell::Ref` 之间能够同时存在
- `std::cell::RefMut` 与其它 `Ref` 和 `RefMut` 都不能同时存在

编译器是无法在编译时保证程序满足这个约束，于是退而求其次，`RefCell` 在运行时会对其进行检查：如果在已经被借用的情况下需要获得可变引用，则会使得代码 panic。

#### Shared Ownership

通过 `Box` 和 `RefCell` 能够解决我们在堆上分配内存并进行修改的一部分问题，但故事并没有这么结束。

既然 `Box` 只是一个指针，那么我们对指针的复制应该也是平凡的。为什么不能直接这么做呢？

答案就是 `Box` 需要保证它拥有的对象的所有权。指针虽然能够复制，但是所有权却不能复制。

复制出的指针并不拥有它所指向的对象的所有权，所以它注定无法成为一个 `Box` 对象。

那是不是就没有办法解决这种需求呢？

类似于 C++ 的 `shared_ptr`，Rust 使用了基于引用计数的 `std::rc::Rc` 来解决这种 *被分享的所有权* 问题。

简而言之，使用引用计数后的，对象的生命周期：

- 开始于创建该对象，此时该对象的引用计数为 1
- 此后 `Rc` 的每次复制都会导致引用计数增加 1、每次销毁都会导致引用计数减 1
- 结束于该对象的引用计数变为 0

Rust 的安全性禁止一个被分享的引用具有可变性，`Rc` 同理。所以 `Rc` 只能能够分享一个堆中的对象的 *不可变引用*。 

但结合我们上面提到过的内部可变性模式，我们就能使用一个能够在不同地方被读取和修改的对象了。

一个例子：

```rust
fn mutate(x: Rc<RefCell<i32>>) {
  *x.borrow_mut() += 2;
}
{
  let x = Rc::new(RefCell::new(1));
  mutate(Rc::clone(&x));
  assert_eq!(3, *x.borrow());
}
```

一个常见的练习就是使用这套 API 来实现一个双向链表，各位有兴趣可以写一下。

> 当然也可以看一看 [这本书](https://rust-unofficial.github.io/too-many-lists/)

### Multi-Threading

上面我们关于安全性的讨论一直都没有讨论一个话题，那就是 多线程下的安全性。

但实在是过于复杂，我们改日再谈。

## Rust is Expressive

约束虽然能够给代码编写者带来可靠的依仗，但或多或少都会对表达能力产生限制。

Rust 就拥有着非常强力的约束，自然也会有各种各样的限制。

不过在拥有如此多的限制的情况下，Rust 仍然具有非常强大的表达能力。

下面我 *稍微* 介绍一下 Rust 的语言特性给我们带来的可能性。

### Algebraic Data Types

代数数据类型作为早在 1970 年就被引入的语言特性，理应是 *现代语言* 的标配（在很大程度上也事实如此），不过我们这里还是稍微讲一讲。

为什么叫代数数据类型？这里面的 "代数性/Algebraic" 是怎么来的？

简单的来说，代数数据类型就是由其它类型 *组合* 起来的类型。而这种代数性指的是这种组合的方式是类似于某种 代数操作。

我们可以认为，每种类型对应于这种代数中的一个数，例如  `Unit` 对应于 1，而它们支持通过一些操作进行运算：

- 求和：两个类型加在一起代表代表这个值可以是两个类型中的任意一种，例如 `Either<A, B>` 相当于 `A` 与 `B` 的和
- 求积：类似于集合的笛卡尔积，这个类型的值需要同时包含两个类型的值，例如 `(A, B)` 相当于 `A` 和 `B` 的积
- 求幂：类似于集合间的映射，例如 `fn(A) -> B` 相当于 `B` 的 `A` 次幂

> 在 Rust 中用来定义代数数据类型的就是 `enum` 了，但由于 Rust 需要保证 `enum` 的大小是有限的，我们并不能在定义的类型中 **直接** 使用自己或者其它没有固定大小的类型。
> 
> 同时，Rust 的安全性使得它对函数有着诸多限制，所以这里我们仅以前两种组合形式进行讲解

通过简单的这几种操作我们可以组合出相当多的可能性：

```rust
enum Void {} // 0
struct Unit; // 1

enum Option<T> { None, Some(T) } // X + 1
enum Either<L, R> { Left(L), Right(R) } // X + Y

type Product2<L, R> = (L, R); // X * Y

enum List<T> { Nil, Cons(T, Box<List<T>>) } // f(X) = 1 + X * f(X)
enum BinaryTree<T> { Leaf, Node(Box<BinaryTree<T>>, T, Box<BinaryTree<T>>) } // f(X) = 1 + f(X) * X * f(X)
enum BinaryTree2<T> { Leaf(T), Node(Box<BinaryTree2<T>>, Box<BinaryTree2<T>>) } // f(X) = X + f(X) * f(X) 
```

和类型一般是通过 Tagged Union 实现的：union 占据各种分支中最大的内存空间，以保存不同类型的值，并通过标记来判断 union 中内容的实际类型。

Rust 也为和类型提供了模式匹配语法用于获取其不同分支的值，可以看做 if-else 和解构赋值的一种语法糖

```rust
fn unwrapString(value: Option<String>) -> String {
  match value {
    Some(s) => s,
    None => "".to_string(),
  }
}
```

Nothing new, right? 

### Parametric Polymorphism

编程中一个非常重要的理念就是避免重复，重复意味着对人力产生了浪费、同时增加了维护成本。

而避免重复的最重要的手段就是 *抽象*：将会被重复的东西从逻辑中剥离出来，将剩下的部分通过抽象进行组织，从而避免重复。

而在这种抽象中，最为直观的就是 *参数化多态（Parametric Polymorphism）*，即将需要抽象的东西通过额外的函数参数提取出来，只保留需要的核心逻辑。

这在 Rust 中对应的就是 *泛型*。

一个简单的案例：

```rust
fn create_box_i32(x: i32) -> Box<i32> { Box::new(i32) }
fn create_box_i64(x: i64) -> Box<i64> { Box::new(i64) }
fn create_box_f32(x: f32) -> Box<f32> { Box::new(f32) }
fn create_box_f64(x: f64) -> Box<f64> { Box::new(f64) }

fn create_box<T>(x: T) -> Box<T> { Box::new(x) }

{
  let x = create_box(1);
  let y = create_box("123".to_string());
}
```

我们把函数参数类型和返回值类型通过类型参数添加到 `create_box` 上，就能获得一个简单的泛型函数。

而我们刚才讲生命周期时也提到了，直接或间接使用到了引用类型作为参数的函数都会实现为泛型函数，对于它们而言的泛型参数就是生命周期参数了。

由于 Rust 中的泛型是静态多态，所以 Rust 对于泛型参数的一个要求是：

它是一个编译期能够确认的类型/常量/生命周期，因为除了能够被擦除掉的泛型参数，大多数都还是需要针对实际的泛型参数来进行代码生成。

和函数类似，类型当然也拥有参数化多态：

```rust
type Id1<T> = T; // type alias

enum Id2<T> { Identity(T) } // enumeration declaration

struct Id3<T> { identity: T } // structure declaration

type Pair<L, R> = (L, R); // tuple-construction
```

在给 `struct`/`enum`/`union` 实现方法时，我们也可以使用参数化多态：

```rust
struct A<T> { _marker: PhantomData<T> }
impl<T> A<T> {
  fn foo(&self) {}
}
```

那我们有没有办法对参数化多态中的泛型参数进行限制、从而能够得到对泛型内容的更强的约束呢？

第一种方法就是依赖于生命周期参数的子类型特性，保证一部分生命周期参数比另一些生命周期参数更长：

```rust
fn choose_first<'a: 'b, 'b>(first: &'a i32, _: &'b i32) -> &'b i32 {
  first
}

fn main() {
  let first = 2; // Longer lifetime
  {
    let second = 3; // Shorter lifetime
    println!("{} is the first", choose_first(&first, &second));
  };
}
```

而另一种则是通过接下来需要讨论的 Trait 系统实现的。

### Trait System

Rust 的 Trait System 作为类型类（[Type Class](https://en.wikipedia.org/wiki/Type_class)）的一种实现，为 Rust 语言提供了极其强大的表达能力。

首先需要明确的一点：Trait **并不是类型**，而是一种对于类型的 *约束*，仅仅在语法层面我们就能区分出类型和 Trait。

和上面提到的参数化多态不同的是，Trait 是一种 *特设多态（Ad-hoc Polymorphism）*。（常见的特设多态还有函数重载）

如果你曾经用过 Rust，你肯定知道在实现某个 Trait 时，语法形如 `impl SomeTrait for SomeType`。

注意到其中的 `for`，它代表 “**为**某个类型实现某个 Trait”，也就是说这种实现是仅提供给这个类型的。

参数化多态需要针对所有（满足约束）的类型提供相同的行为，而特设多态 *可以* 只给特定的类型实现不同的逻辑。

```rust
trait Foo { fn foo(&self) {} }

impl Foo for i32 {}

impl Foo for i64 {
  fn foo(&self) {
    println!("i64 here");
  }
}
```

Trait 作为一种约束，使得我们可以为泛型中抽象的类型参数提供额外的信息，以实现 *有意义的* 的泛型代码：

> 对泛型参数没有约束的泛型代码几乎无法解决任何实际问题，因为它们只能够使用到语言中对于所有类型都通用的规则，而这部分规则往往非常少

```rust
fn foo<T>(x: T, y: T) -> T {
  x + y // 编译错误：我们不能把抽象类型 T 的实例相加
}

fn foo2<T: Add<Output = T>>(x: T, y: T) -> T {
  x + y // OK.
}
```

但 Trait（或者说类型类）也有一个问题：如果对一个类型和 Trait 同时出现了多个实现（某些实现还可能在别的编译单元、也就是 crate 里），该如何处理？

```rust
trait Foo { fn foo(&self); }

impl<T> Foo for T {
  fn foo(&self) {}
}

impl Foo for i32 {
  fn foo(&self) { println!("i32 here"); } 
}

{
  let x: i32 = 1;
  x.foo(); // ???
}
```

这个问题一般而言有几种解决方案：

1. 禁止出现重叠（Rust 和 Haskell 采用的就是这种规则），如果出现多个重叠的 Trait 实现则会编译报错
2. 将 Trait 实现时的方法表以对象的形式进行管理，在使用 Trait 实现时则需要指定使用的对象（例如 Scala 的 implicit object）
3. 按照某种顺序使得 Trait 的实现能够互相覆盖

Rust 采取的策略并不能完全解决问题：

- 不同的 crate 可能对于某个类型和某个 Trait 提供了不同的实现；为了保证不出现重叠，那么我们就不能同时引用这两个 crate
- 在对发布的 crate 进行更新时将无法获得这样的保证：我为本 crate 里定义的 Trait 实现一定不会与在其它 crate 中的实现相重叠

为此 Rust 通过 Orphan Rules 来防止这些情况的发生。

Orphan Rules 规定了，实现一个 Trait 必须满足下面两个条件中的至少一个

- 实现的 Trait 定义在当前编译单元
- 实现的类型定义在当前编译单元

但这些限制并不是无法绕过的，一种常见的对于 Orphan Rules 的 workround 就是 newtype 模式：

> 对现有的类型进行一层包装，使其成为一个新类型，从而避免出现重叠

那么同样是特设多态，函数重载和 Trait 有什么区别呢？这里就需要先讲函数重载是怎么一回事了

函数重载同样是一个 **非常** 麻烦的功能（尤其是需要支持类型推导和各种隐式转换规则时），它的逻辑一般可以描述为：

1. 获得需要被调用的（且没有显式地指定函数签名的）函数
2. 找到上下文中所有同名的重载函数，这些函数被称为候选者
3. 筛选掉参数个数不匹配的候选者
4. 对函数类型和参数类型进行 unify，剔除掉不匹配的候选者
5. 确认最终只剩下一个合理的候选者

简单的来说，就是根据参数的类型和个数进行匹配，获得实际的函数类型。

而 Trait 中提供的方法则不同，它能够根据方法的返回值进行匹配：

```rust
struct A;

impl Into<i32> for A {
  fn into(self) -> i32 {
    1
  }
}

fn foo(x: i32) { 
  println!("{}", x);
}

{
  foo(A.into()); // 这是无法通过函数重载实现的功能
}
```

除此以外，Trait 还提供了关联类型来增加表达能力。什么是关联类型？

简单的来说就是 Trait 中可以包含一个和 Trait 的**具体实现**相关联的 *抽象类型*。

和基于泛型的 Trait 类似，它们都能够表达 “方法类型是抽象的” 这种概念，但是关联类型能够只暴露出抽象的内容中一部分作为代表：

```rust
trait Convert<B> {
  fn convert(self) -> B;
}

trait Convert2 {
  type B;
  fn convert2(self) -> Self::B;
}
```

可以发现，虽然两者都能够提供对于转换过程中 Target Type 的抽象，但前者将类型编码在了 Trait 的类型中，后者则编码在了 Trait 的实现里。

常见的关联类型使用案例有：

- `std::ops::Add` 中的 `Output`，用来表示 `add` 作为操作符的返回类型
- 在 `std::ops::Deref` 中的 `Target`，用来表示解引用的结果类型

### Dynamic Dispatch

上面讲到的表达能力基本都是静态的多态性。静态的多态性能够解决一部分问题，但并不是万能的。

那么 Rust 又是如何支持动态的多态性的呢？

这部分需要分两个部分说起：

1. 函数指针与闭包
1. trait object

#### Function Pointer and Closure

（字面意义上的）函数指针是实现动态派遣的基础，它使得我们能够调用一个由 **运行时决定** 的函数。

这对于语言的表达能力具有极大的提升，我们能够将路径选择与实际的业务逻辑进行分离，提供更强的扩展性。

Rust 中提供了对于函数指针的支持：

```rust
fn id_i32(x: i32) -> i32 { x }
fn add_1_i32(x: i32) -> i32 { x }
fn generic_id<T>(x: T) -> T { x }

let mut transform_i32: fn(i32) -> i32 = id_i32; 
transform_i32 = add_1_i32;
transform_i32 = generic_id::<i32>;
```

形如 `fn(...) -> ...` 的类型即为函数指针，我们可以把现成的函数绑定到这些类型的变量上，从而达到动态派遣的效果。

我们知道，Rust 是支持 lambda 表达式的：

```rust
let id = |x: i32| x;
assert_eq!(id(1), 1);
```

那么这些 lambda 能绑定到函数指针上吗？

```rust
let mut pointer: fn(i32) -> i32;
pointer = |x: i32| x; // ok

let a = 123;
pointer = |x: i32| x + a; // 编译错误
```

第一个赋值操作顺利地通过了编译，而第二个则会出现编译错误：

```
expected fn pointer `fn(i32) -> i32`, found closure `[closure@src\main.rs:67:9: 67:23]`
note: closures can only be coerced to `fn` types if they do not capture any variables
```

Rust 中的 lambda 表达式均会拥有一个匿名的 closure 类型。

- 当 lambda 没有 *捕获* 任何局部变量时，它能够自动 *转换（Coerce）* 为一个函数指针

这很容易理解，毕竟匿名函数/字面量函数最终还是被编译为顶层函数和某个包含捕获变量的结构体，这个结构体会以函数额外的参数的形式传入。

而在没有捕获的场合，这个参数可以省略掉，也就是可以直接使用相同签名的函数指针来引用这个 lambda。

那么对于产生了捕获的 lambda，我们该如何把它作为可以描述的类型变量进行保存和传递呢？

之前我们以函数为主的形式描述了 lambda 的编译方式，但我们也可以把 lambda 当成一个 *可调用对象* 来考虑：每个 lambda 对应于一个 closure 对象，而在被调用时则是对这个对象上的一个方法进行调用。

Rust 根据 closure 的方法在调用时需要传入的 closure 种类为这些 lambda 自动实现了一些 trait：

- `Fn`：传入 closure 对象的不可变引用，对应于没有捕获以及捕获变量引用的 lambda
- `FnMut`：传入 closure 对象的可变引用，对应于捕获变量可变引用的 lambda
- `FnOnce`：传入 closure 对象本身，对应于移动了捕获变量的 lambda

这些 Trait 从上到下依次具有 sub-trait 关系（也很容易理解），于是我们便可以通过抽象类型的方式来传递这些具有匿名 closure 类型的对象了：

```rust
fn apply<T, F, R>(x: T, f: F) -> R
    where F: FnOnce(T) -> R {
    f(x)
}

let v = vec![1, 2, 3];
let mut v2 = vec![4, 5, 6];

let x = move |v: Vec<i32>| { // Anonymous Type: FnOnce(Vec<i32>) -> Vec<i32>
  for i in 0 .. v.len().min(v2.len()) {
    v2[i] += v[i];
  }
  v2
};

let res = apply(v, x);

println!("{:?}", res)
```

但通过泛型函数我们仍然只能通过静态的方式对 closure 进行调用，那么我们该如何把这些 lambda 保存在具名的类型中呢？

这就必须要使用到 trait object 了

#### Trait Object

之前我们提到，Trait 实际上并不是类型，而只是对类型的约束。上面关于静态多态的部分里，我们能够通过 Trait 描述这种行为：

- 这是一个编译期能够确定的类型，虽然在当前上下文里我并不知道它具体是什么，我希望它能够满足某个约束以使用一些方法

我们能不能拿掉 “编译期能够确定” 的这个要求呢？Rust 用 trait object 给出了答案：（在一部分场景下）可以

```rust
trait Foo {
  fn foo(&self);
}

impl Foo for i32 { fn foo(&self) { println!("i32") } }

impl Foo for i64 { fn foo(&self) { println!("i64") } }

type DynamicFoo = dyn Foo;

{
  let x = 1i32;
  let y = 1i64;
  let mut foo: &DynamicFoo;
  foo = &x; foo.foo(); // print i32
  foo = &y; foo.foo(); // print i64
}
```

那么这个 `DynamicFoo` 或者说 `dyn Foo` 究竟是个什么类型？它的引用又代表着什么？

简单的来说，`DynamicFoo` 这种 trait object 代表着所有可能实现了某些 trait 的类型，是一个在运行时才能知道大小的类型（Dynamically Sized Type），我们不能直接直接在变量、函数参数等地方使用，且不能 *直接创建* 这种类型的实例，因为它们的实例实际上是其它类型的实例。

但我们可以创建对 trait object 的指针/引用！

动态大小类型的引用和具有固定大小的类型的引用在大小和实现上一般都会有不同，trait object 也是。

一般而言，指向 trait object 的 *指针* 由两个字段组成：

- 指向实际对象的指针，该对象的类型需要满足 trait object 的要求
- 虚表指针，虚表包含了实际类型中对于 trait object 对应的若干 trait 中的方法的函数指针

那么在上面的例子中，把 `i32` 作为 `dyn Foo` 进行取引用时，除了 `x` 的地址以外，还拿到了为 `i32` 实现的 `foo` 方法所在的虚表指针，一并作为 trait object 的引用保存在局部变量 `foo` 里。

在 trait object 引用执行的方法调用也会变成对于虚表中保存的具体的函数指针的调用。

那么是不是所有的 trait 都能够作为 trait object 里的 trait 呢？

trait object 对于在类型约束的要求为：

- 有且只有一个的 object-safe 的 trait 作为 base trait
- 一系列可能的 auto traits
- 最多一个的生命周期参数

所谓 object-safe 是对于一个 trait 的限制，其内容包含：

- 该 trait 所有的 supertrait 必须是 object-safe 的
- `Sized` 不应该是其 supertrat
- 不能包含任何 *关联常量*
- 所有的方法必须满足下面两种条件之一
  - 持有 `where Self: Sized` 约束的方法（以 `self` 为方法接受者的方法会自动持有此约束）会在 trait object 被禁用
  - 同时满足下面这些条件：
    - 不包含任何除了生命周期以外的泛型参数
    - 方法签名中除了接受者位置以外不使用 `Self` 
    - 接受者必须为以下几种：
      - `&self` 或 `&mut self`
      - `Box<Self>`
      - `Rc<Self>` 和 `Arc<Self>`
      - 上述类型与 `Pin` 的嵌套

在满足这些限制之后，我们才能保证：这些方法的签名是一个能够在编译期表达的类型，并可以通过 trait object 的引用、在不破坏类型系统的限制下调用他们。

## Summary

本文主要从安全性和表达能力这两个角度稍微介绍了一下 Rust 语言的设计思路以及一些语言特性的使用方式。

不过限于篇幅，还有很多和 Rust 相关的话题没能在这里讲到：

- Rust 对于线程安全的约束方式
- 通过 Unsafe Rust 绕过的限制，以及 Safe Rust 是如何控制它们的
- Rust 里的 GAT/TAIT 等
- ……

## Reference

https://doc.rust-lang.org/book/title-page.html

https://doc.rust-lang.org/rust-by-example/index.html

https://doc.rust-lang.org/stable/reference/

https://terbium.io/2021/02/traits-typeclasses/

https://github.com/Ixrec/rust-orphan-rules

https://www.fpcomplete.com/blog/monads-gats-nightly-rust/

https://bartoszmilewski.com/2014/09/22/parametricity-money-for-nothing-and-theorems-for-free/
