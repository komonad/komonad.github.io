---
title: "IoC 与 DI —— 背景、Rust 实现与使用方式"
permalink: /ioc-di/
---

# IoC 与 DI —— 背景、Rust 实现与使用方式

## 什么是 IoC？

IoC（Inversion of Control）是一种用来解耦的*设计模式*。

顾名思义， IoC 对 “Control” 进行了 “Inverse”，那么什么是 “Control”，为什么需要对它进行反转？

这里的 Control 指的是控制流（即在命令式程序中按照顺序调用的语句、指令或函数调用）；而反转，则指的是将控制流中这种顺序关系进行调整，使其从某些角度上呈现出和原本不同的控制流关系。

一个更加形象的描述是：在传统的命令式程序中如果我们希望使用某个库实现一个功能时，我们往往会：

```rust
use ext_lib::{a, b, c};
a(1); b(2); c(3);
```

在这种场景下，我们拥有对 ext_lib 所提供的这些函数调用的控制流，我们可以决定何时调用这些方法。

而控制反转则将这种能力移交到其它地方：

```rust
use ext_lib::{when_a, when_b, when_c};
when_a(|a| a(1)); when_b(|b| b(2)); when_c(|c| c(3));
```

我们在库中注册的回调*移交*了对于 `a/b/c` 的调用时机的控制权，现在是由这个库来调用我们的方法、而不是我们去调用它的方法。

> Aka [Hollywood principle](http://en.wikipedia.org/wiki/Hollywood_principle), 'don't call us, we'll call you'.

常见的 IoC 实现包含：

- 回调函数
- 事件循环、事件处理
- 依赖注入
- 模板方法
- ……

## 为什么需要 IoC？代价和收益是什么？

IoC 能够将 **做什么** 和 **什么时候做** 进行分离，使得两者之间失去*直接*的依赖关系；按照这种原则，我们就可以将程序模块化，每个模块：

- 仅知道自己需要提供的功能中每一步“什么时候做”
- 不知道自己需要提供的功能中每一步“具体做什么”

那么对于每一个模块，我们都可以单独实现、单独测试、切换某些功能或算法的实现，只要每一个模块遵循它们提供的接口中所包含的“契约”。

而 IoC 的代价则往往体现在某些对于控制流中的顺序关系有需求的场景，IoC 会将整个可能相对完整的运行流程切碎并分派到不同的模块中，对于调试或者代码浏览可能会不太方便；同时过于频繁的方法调用*可能*会带来一些性能上的损失（但一般的性能瓶颈都不会发生在这些地方，还是要以 profiling 结果为准）。

---

依赖注入是一种非常典型的贯彻了 IoC 思想的设计模式，我们接下来关于 IoC 的讨论也会建立在依赖注入上。

## 依赖注入是什么？

顾名思义、“依赖注入”指的是将某个依赖的功能或实体，注入到需要此依赖的地方，从而对“构造”和“使用”的行为进行解耦。

>  通过依赖注入编写的程序，在方法和对象中不需要了解如何构造希望使用的*服务*；这些工作被移交到了一些外部的代码（即注入器）进行完成

换一种说法：将一个对象可能需要被抽象的行为从外部传入而不是由该对象自行决定；这个传入的方式也可以有很多思路：在对象构造时直接传入、构造后通过 setter 传入等等。

一般而言，依赖注入中有几个概念：

- Service 与 Client

  任何提供了功能的对象都*可以*作为 Service（但这并不代表它们一定是 Service）、任何需要使用到 Service 的对象都*可以*作为 Client；一个对象可以同时为 Service 与 Client

  我们需要将 Service 注入到依赖于该 Service 的 Client 中

- Interface

  Service 的抽象能力可以通过 Interface 进行抽象，使得 Client 可以在不知道 Service 的具体类型的情况下获取实现

- Container/Injector/Factory

  Container 需要负责构造 Service 和 Client 并将依赖的 Service 注入到对应的 Client 中；或者说：Service 把自己的构造*托管*给了 Container。在这个前提下，用户一般需要自行构造的只有一些数据对象（因为离开了 Container 你无法知道如何给 Client 注入 Service）

根据上面提到的 Client 和 Service 的性质，我们可以把所有托管到 Container 的对象均称为 Service

## Rust 中的依赖注入实现

观前提醒：

- 以下内容与实际实现并不完全一致，仅作为思路展示
- API 的设计对 [InversifyJS](https://github.com/inversify/InversifyJS) 有*非常多*的参考。

### 初步尝试

在实现之前，我们必须了解我们的需求是什么。

- 我们需要在 Container 中维护 Service 的构造器
- 一个 Service 可能有多种构造方法
- Service 的类型可能是基于 Interface 的抽象类型
- Service 的构造可能会依赖于其它的 Service

那么接下来就开始尝试实现一个依赖注入框架吧：

---

先从最简单的 Container 开始：

```rust
pub struct Container { /*...*/ }
impl Container {
    pub fn new() -> Self { ... }
}
```

我们希望维护 Service 与 Client 之间的关系，那么首先需要一个 Service Id 来对 Service 进行标注。

正好 Rust 提供了 `std::any::TypeId` 来在编译期获取任何类型的独一无二的 ID，我们可以直接将 `TypeId` 作为我们的 `ServiceId`：

```rust
pub type ServiceId = std::any::TypeId;
```

而由于 Container 需要负责 Service 的构造，所以 Container 中也需要维护 Service 的构造器。

Service 的构造器类型*应该*长什么样？有几点我们能够肯定：

- 构造器类型可以等效为一个函数
- 所有 Service 的构造器类型是相同的：需要对构造器做类型擦除
- 构造出来的 Service 可能是 Unsized：构造出来的对象需要装箱

虽然 Rust 中并没有可以抽象成 `[Object] -> Object` 的构造器，但我们可以把这些逻辑直接放在构造器里面，而构造器本身可能只需要 Container 用来构造其它 Service，这样我们就解决了构造器的参数的类型问题。

但另一个问题并没有解决：构造器返回的到底是个啥？

虽然 Rust 提供了 `std::any::Any` 这个 trait 来提供*最基本*的类型擦除能力，但我们并不能直接返回一个 `Box<dyn Any>`：`Box<dyn Any>` 无法变成 `Box<T>`。而我们也不能直接返回 `Box<T>` 对应的裸指针，因为 `T` 如果是 Unsized 的话，`Box<T>` 也会是一个胖指针。

一个非常朴素的解决方案是：在 `Box<T>` 上再包一层 `Box`：

```rust
pub struct UniqueAny(* const ());

pub fn to_unique_any<T: ?Sized>(any: Box<T>) -> UniqueAny {
    UniqueAny(Box::into_raw(Box::new(any)) as * const ())
}
pub fn from_unique_any<T: ?Sized>(any: UniqueAny) -> Box<T> {
    unsafe { *Box::from_raw(any.0 as *mut Box<T>) }
}
```

构造器大概就可以长这个样子：

```rust
pub type ServiceConstructor = dyn Fn(&mut Container) -> UniqueAny;
```

通过 Container 注册 Service 构造器和获取 Service 的 API 就可以长这个样子：

```rust
impl Container {
    pub fn register(&mut self, id: ServiceId, ctor: Box<ServiceConstructor>) {}
    pub fn get<T: ?Sized>(&mut self, id: &ServiceId) -> Box<T> { todo!() }
}
```

至于实现当然也可以很简单，只需要维护一个 `HashMap<ServiceId, BoxCtor>`，然后 `register` 时往里面添加，`get` 时获取到构造器后把 `self` 传进去——等等，拿到的 `Constructor` 似乎包含了 `self` 的引用，我们不能对已经借用过的 `self` 再次进行可变借用。

还是把构造器放在 `Shared` 里吧，这样就可以先复制一个构造器再把 `self` 传进去了：

```rust
impl Container {
    pub fn get<T: ?Sized>(&mut self, id: ServiceId) -> Box<T> {
        let ctor = self.ctors.get().unwrap().clone();
        from_unique_any::<T>(ctor(self))
    }
}
```

简单的说，Container 可以理解为一个 Service Id 到 Service Constructor 的映射，它通过自己维护的 Constructor 来处理“Service 实例化”的请求。

### 不同的 Service 实现

在上一节里面，我们很轻松地就实现了一个依赖注入 Container，但有几个问题：

- 每次 `get` 时都需要调用注册的构造器并返回一个新的实例，而在很多（甚至是绝大多数）场景下，我们希望拿到的 Service 始终是同一个（即所谓的单例）
- 对于 unsized 类型我们需要直接提供该类型的构造器，而不能通过 unsize coerce 来复用已经存在的构造器

如果把最开始我们提供的 Service 实现叫做 Transient Binding 的话，那么我们还需要提供 Singleton Binding 和 Service Binding：前者保证 Container 中始终最多只会进行一次该 Service 实现的构造，而且每次获取均能够拿到同一个实例；后者能够将对一个 Service 的获取转发到另一个 Service 上。

#### Singleton

先来看看 Singleton Binding。

既然是单例，那通过 Container 拿到的是什么？

引用无法比较方便地注入到其它的 Service 中；裸指针又不安全，可能会导致内存安全问题；那么还是用 ~~`std::shared_ptr`~~ `std::rc::Rc` 来保存吧；不过在后面的讨论中，我们会用 `Shared` 来指代具体的引用计数指针实现。

但共享生命周期的对象擦除类型可能没那么方便：在擦除后我们还需要保证它能够正常地复制、析构，转换回正常的 `Shared` 对象。

所以除了和 `UniqueAny` 相似的保留一个 `* const Shared<T>` 以外，还需要维护一个“虚表”，提供对 `* const Shared<T>` 复制和析构的实现，这里为了方便直接使用了 Closure 类型进行处理。

```rust
pub struct SharedAny {
    pointer: * const (), // released from Box<Shared<T>>
    ty: TypeId, // TypeId::of::<T>()，used for type checking
    copier: Shared<dyn Fn(* const ()) -> * const ()>, 
    deleter: Shared<dyn Fn(* const ()) -> ()>,
}
```

单例只会构造一次，那么 Singleton 构造器的类型就可以设为：

```rust
pub type SingletonConstructor = dyn FnOnce(&mut Container) -> SharedAny;
```

与 Transient 不同的是，我们需要额外添加字段来保存通过构造器生成的单例，并在后续的调用时直接复制该实例。

由于到单例无法生成 `Box`，`Container` 也需要添加一个 API 用于获取 `Shared<T>` 的服务：

```rust
impl Container {
    pub fn get_shared<T: ?Sized>(&mut self, id: ServiceId) -> Shared<T>;
}
```

#### Service Forwarding

Rust 允许一个 struct 实现多个 trait，那么一个很自然的需求就是：我希望将一个 struct 单例实现的 Service 绑到其它的 trait 对应的 Service 上，这样在提供不同 Service 功能的同时，还能够保证它们最终会落到同一个对象上。

在 nightly Rust 中提供了 `CoerceUnsized` 这个 auto-trait，我们能够通过它把下面这段代码的 `i32` 和 `dyn ToString` 作为泛型类型参数提取到外面：

```rust
let x: &i32 = &1;
let y: &dyn ToString = x;
/// into

```

但由于一些原因这个 RFC 目前还没能 stable，所以这里只提一嘴。

对于 Service Forwarding，在注册时除了 `ServiceId` 还需要提供一个 Forwarding Constructor，用于通过另一个 Service 的实例来创建自己所需的 Service 实例；对输入和输出进行类型擦除后，它的类型也可以很简单：

```rust
pub type ForwardingConstructor = dyn Fn(SharedAny) -> SharedAny;
```

#### Constant 

有的时候我们甚至不希望 Container 来帮我们完成 Service 构造过程，或者构造过程非常平凡不需要一个可调用对象来维护；我们为这些 Service 提供了 Constant 绑定：

```rust
impl Container {
    pub fn register_constant::<T>(&mut self, id: ServiceId, value: Shared<T>);
}
```

为了支持 Unsized 类型，我们还是不得不使用了 `Shared` 对 Service 进行包装。



### 循环依赖

目前由于所有的构造器都是由外界传入的，如果两个 Service 互相依赖，则会在构造时直接发生死循环；我们希望能够在构造时处理这种异常情况；这个问题的模型也很简单，判断 Service 与 Service 之间依赖关系形成的图是否存在环。

不过 Container 也没有啥性能上的要求（大概），循环依赖也不是一个应该在运行时被解决的异常，我们完全可以把判环的时机推迟到 Service 实例化时；对于已经检查过的服务也可以在下次检查时直接跳过。

那么有没有方法可以让我们支持服务之间的循环依赖呢，当然有，不过我们放在后面再说；

---

### API 改良与自动注入

为了让添加 Service 实现的代码与英语语法更加贴合，我们添加了类似于下面这种 API：

```rust
container.bind::<i32>().to_transient(vec![], Shared::new(|_| Box::new(10)));
```

但即便如此，用户在为一个 Service 注入其它 Service 时仍然需要编写很多重复代码：

```rust
container.bind::<X>()
	.to_singleton(vec![], Box::new(|c| Shared::new(
        X { a: c.get::<A>(), b: c.get::<B>(), d: c.get::<D>(), }
)))
```

用户自定义的构造器提供了非常灵活的 Service 构造行为，不过我们还是需要提供某种“默认”的构造方法、作为不需要这种灵活性时的选择。这时最简单的选择就是使用宏来生成绑定时需要的*数据*与*行为*。

我们提供了类似于下面的自动注入 API：

```rust
#[injectable]
struct SomeService {
    #[inject(ThisService)]
    thisService: Shared<ThisService>,
    #[inject(ThatService)]
    thatService: Shared<ThatService>,
    someField: Field,
}
```

`#[injectable]` 需要生成一个签名为 `(&mut Container) -> SomeService` 的构造器，和一个用户获取该 Service 所依赖的其它 Service 的函数。

由于 Rust 的过程宏发生在语义分析之前，在 `#[injectable]` 的实现中我们无法**直接获取**每个字段的类型；所以这里我们只能让用户自己提供它们希望注入的服务类型

对于所有通过 `#[inject(X)]` 标注的字段，我们可以假设该字段的类型为 `Shared<X>`，然后在生成的 `SomeService` struct expression 中通过传入的 `Container` 获取 `X` 的实例：

```rust
SomeService {
    thisService: container.get::<ThisService>(),
    ...
}
```

对于没有通过 `#[inject]` 标注的字段，直接使用 `Default::default()`（如果需要更复杂的构造过程，还是直接自己实现构造器吧）

在上文中我们为 `Container` 支持了获取不同类型的服务，考虑到 Service 注入的字段可能也有不同种类的需求，这里也需要考虑直接注入 Box 或者其它类型的服务

```rust
#[injectable]
struct A {
    #[inject_instance(B)] b: B,
    #[inject_box(C)] c: Box<C>,
    #[inject_custom(expr)] e: E,
}
```

通过 `#[injectable]` 生成的代码，在 `Container` 中注册 Service 的代码被简化为：

```rust
container.bind::<SomeService>()
	.to_singleton(
        SomeService::dependencies(),
        Box::new(|c| Shared::new(SomeSerivce::constructor(c))),
);
```

不过还是要写很多内容，但现在的结构已经可以让我们通过过程宏来表达了：

```rust
macro_rules! bind_singleton {
    ($container: expr, $service_ty: ty) => {
        $container.bind::<$service_ty>().to_singleton(
            <$service_ty>::__dependencies__(),
            Box::new(|__container__| {
                Shared::new(<$service_ty>::__constructor__(__container__))
            })),
        )
    };
}
```

提供绑定时也只需要：

```rust
bind_singleton!(container, SomeService);
```

### 模块化

我们可以把一系列的注册行为包装在一个对象里，称之为模块。一个最简单的模块签名如下：

```rust
pub trait ServiceModule {
    fn register(&self, _container: &mut Container) {}
}
```

那么在 `Container` 中*装载*一个模块的行为也很平凡：

```rust
impl Container {
    pub fn load_module<T: ServiceModule>(&mut self, module: T) {
        module.register(self);
    }
}
```

很明显，一个签名为 `(&mut Container) -> ()` 的函数理应是一个 `ServiceModule`：

```rust
impl ServiceModule for for<'r> fn(c: &'r mut Container) -> () {
    fn register(&self, container: &mut Container) {
        (self)(container);
    }
}
```

 但可惜 rustc 暂时还推不出来，我们不得不使用 workaround：

```rust
impl Container {
    pub fn load_fn_module(&mut self, module: for<'r> fn(&'r mut Container)) {
        module(self);
    }
}
```

### 杂项

#### Named Service

当然你可能说：Service Id 使用类型来区分的话会导致一些常用的类型很容易发生抢占，从而不得不使用一些 new type idiom；而另一种选择就是为类型添加额外的标注；为了保证 Service Id 的可读性、我们添加了一种新的 Service Id：

```rust
enum ServiceId {
    Type(TypeId),
    NamedType(TypeId, String),
}
```

通过在绑定/获取时添加名字的形式，我们可以在 Container 中提供类似于配置之类的功能：

```rust
container.bind_named::<i32>("x".to_string()).to_constant(Arc::new(1));
```

相应的、在进行注入和绑定时都可以使用 Named Service Id：

```rust
#[injectable]
pub struct SomeService {
    #[inject(OtherService["for-some-service"])]
    other_service: Shared<OtherService>,
}
pub fn some_module(container: &Container) {
    bind_singleton!(container, OtherService["for-some-service"]);
}
```

#### Multiple Binding

一种常见的需求是：我希望有很多针对于某个 Service 的实现并拿到它们；目前我们在获取 Service 时如果出现了对于同一个 Service Id 的多个实现，则会认为是一个错误并返回相应的异常；但这种需求是合理的并且应该有相应的支持（具体的应用在下面也会提到），所以在 Container 中我们也添加了相应的 API：

```rust
impl Container {
    pub fn get_all<T>(&mut self, id: ServiceId) -> Vec<Shared<T>>;
}
```

但这里并没有对所有类型的实现都提供支持，而是只允许获取每个实现的 Shared Instance（需要的话当然也可以提供，不过考虑到 API 的复杂性以及使用范围这里还是选择了只提供一种）

#### Deferred Injection

在 [上面](#循环依赖) 我们提到了循环依赖会导致 Service 实例化过程中出现死循环，但这种情况实际上是可以避免的，正如引用计数指针都提供了对应的弱引用版本来避免直接的循环依赖，服务之间的依赖关系当然也可以通过类似的手段解决。

假设我们有两个服务 A 和 B，它们原本的依赖形式如下：

```rust
struct A { b: Shared<B> }
struct B { a: Shared<A> }
```

不依赖于一些 unsafe 黑魔法我们是无法同时构建出它们中的任意一个的

但我们可以引入下面这种机制：

```rust
struct B { a: IntMut<Option<Shared<A>>> } // IntMut stands for 'Interior Mutability', maybe an alias for std::rc::RefCell
```

这样 `B` 就能够在 `A` 还不存在时构造出一个处于*不合法*状态的实例，然后在 `A` 构造完成后将其通过内部可变性传入 `B` 的字段中。在此之后，双方都会处于“各自拥有对方实例”的*合法*状态中。

那么我们可以定义这么一个 struct 来表示这种“延迟注入”的 Service

```rust
pub struct Deferred<T: ?Sized> {
    inner: Mutex<Option<Shared<T>>>,
}
impl<T: ?Sized> Deferred<T> {
    pub fn setup(this: &Self, value: Shared<T>) {
        *this.inner.lock() = Some(value);
    }
}
impl<T: ?Sized> Deref for Deferred<T> {
    type Target = Shared<T>;
    fn deref(&self) -> &Self::Target { ... }
}
```

为了让 Container 支持这种行为，我们为 Singleton 绑定的构造器提供了一个额外的可选参数（仅支持 Singleton 是因为 Transient 绑定的语义会导致注入 Service 的身份无法保证和自己一致）：

```rust
type SingletonBinding = (SharedSingletonConstructor, Option<DeferredInitializer>);
```

`DeferredInitializer` 用于表示“对 Service 中的 Deferred 依赖进行初始化”的动作：

```rust
type DeferredInitializer = Box<dyn FnOnce(&mut Container, SharedAny)>;
```

Container 在对 Singleton Service 进行实例化时，我们会将所有的“对 Deferred Service 进行填充”的逻辑放在“最外层”的构造请求中，这样能够保证在尝试获取 Deferred 依赖时，该 Service 已经被实例化一个（可能不合法）的对象。

实现逻辑也很简单，维护一个计数器来表示嵌套的构造请求层数，一个栈表示待初始化的服务：

- 在进行构造前增加计数器、构造完后减少计数器
- 将 `DeferredInitializer` 和构造出的 `SharedAny` 填入栈中
- 如果当前构造是顶层的服务实例化请求，那么清空栈、并将所有栈中的服务进行初始化

当然在 `#[injectable]` 中我们也需要提供支持：

```rust
#[injectable]
struct SomeService {
    #[inject_deferred(OtherService)]
    other_service: Deferred<OtherService>,
}
```

## 如何在 Rust 中使用依赖注入 ？

依赖注入是一个对架构影响非常大的设计模式，会对代码的维护和组织形式产生非常大的影响（这也意味着你原本写代码的方式在使用了依赖注入后不一定合理，需要进行一些重构）。

依赖注入中*规定*了“所有提供了功能的实体都可以被认为是 Service”，但提供的功能性质不同也导致了我们在设计时对不同的 Service 并不能一视同仁；所以我们还需要一些设计模式中的设计模式来指导我们对依赖注入的使用。

~~*opinion-based 警告*~~

一般而言，程序的结构可以表示为：

```rust
fn main() {
	something_else();
    let mut container = Container::new();
    container.load_module(service_modules);
    let app = container.get_typed::<Application>();
    app.start();
}
```

我们创建了一个 `Container`，通过装载 `service_modules` 获取了所有需要的 Service，并通过一个预先写好的 `Application` 作为整个程序的入口。

通过自定义 `service_modules` 中的内容，我们可以很轻松地对程序的行为进行**模块化地**修改。

### 该如何选择 Service Binding 的种类？

对于 Application 而言，我们使用什么都没有本质区别；但对于 Application 中的其它 Service 又该如何呢？

先说结论：

- 在默认情况下可以直接使用 Singleton 绑定
- 如果 Service 包含可变的内部状态、且需要具有相同实现的不同的 Service 实例来处理不同的数据，则使用 Transient 绑定
- 对于需要抽象的 Service ，通过 Service 实现的 Trait 使用 Service 绑定
- 对于简单的配置数据或者不希望通过 Container 进行构造的 Service ，使用 Constant 绑定

一个例子：

```rust
bind_singleton!(container, FileServiceImpl);
bind_transient!(container, FileReader);
bind_service!(container, dyn FileService => FileServiceImpl);
bind_constant!(container, usize["file_service.MAX_FILE_SIZE"], Shared::new(1024));
```

我们需要一个 `FileService` 来提供一些和文件系统相关的能力，但并不希望它的实现和某个非常复杂的实体产生直接的依赖关系，于是将 `FileService` 声明为一个 object-safe 的 trait 并通过 Container 来获取它；

`FileServiceImpl` 是我们的默认的 `FileService` 实现，它可能包含一个**内置的**日志系统，用于统计文件操作的性能数据和异常情况，并在进行处理之后上报给某个服务器。我们希望所有的 `FileService` 行为都通过一个 `FileServiceImpl` 的实例进行获取，那么这里可以对 `FileServiceImpl` 使用 Singleton 绑定。

假设出于性能因素，`FileServiceImpl` 在每次执行大文件读写时都会需要一个 `FileReader` 来完成相应的操作，但 `FileReader` 在处理文件读入行为时会需要对自身的内部状态进行修改；考虑到 `FileReader` 可能会依赖于其它的 Service，我们可以通过 Transient 绑定来获取一个新的 reader 实例。

而为了判断什么是一个大文件，`FileServiceImpl` 需要一个文件大小阈值；`usize` 本身是一个非常 trivial 的对象，这里我们可以直接使用 Constant 绑定。

### Contribution Point

依赖注入一个非常常见的设计模式就是 Contribution Point 了。一个“综合性”的 Service（例如示例代码中的 `Application`） 往往需要足够多的灵活性，只基于 Service Forwarding 虽然能够抽象**某个**行为，但无法进行更多的抽象；

不过我们在 [这里](#multiple-binding) 提供了对于同一个 Service Id 多个实现的支持，那么我们就可以借助这个机制来提供更复杂的自定义行为支持，例如：

1. 提供可扩展的 Service 行为：`FileService` 中通过维护不同 `Scheme` 的 `FilesystemProvider` 来支持各种文件 URI

   ```rust
   #[injectable]
   struct FileService {
       #[inject_all]
       contributions: Vec<Shared<dyn FileServiceContribution>>,
   }
   impl FileService {
       pub fn init(&self) {
           self.contributions.iter().map(|x| x.contribute(self));
       }
   }
   ```

2. `Application` 提供 `ApplicationContribution` 作为部分行为的生命周期钩子

   ```rust
   impl Application {
       pub fn start(&self) {
           self.contributions.iter().map(|x| x.before_start());
           ...
       }
   }
   ```

和 Service/DI 类似、Contribution 本身提供的功能并不是不可替代的，例如你完全可以自己维护一个 `Vec` 在运行时一个个添加；但 Contribution 的意义在于它提供了一种和 Service Module 绑定的“前运行时”的行为自定义能力，我们可以通过它来实现一些对于灵活性不太依赖的 Service 个性化需求。


## Disucssion & Reference

https://stackoverflow.com/questions/6550700/inversion-of-control-vs-dependency-injection

https://stackoverflow.com/questions/871405/why-do-i-need-an-IoC-container-as-opposed-to-straightforward-di-code

https://en.wikipedia.org/wiki/Dependency_injection

https://en.wikipedia.org/wiki/Inversion_of_control

https://www.martinfowler.com/articles/injection.html

http://blog.gtiwari333.com/2011/05/understanding-dependency-injection-and.html

https://github.com/inversify/InversifyJS