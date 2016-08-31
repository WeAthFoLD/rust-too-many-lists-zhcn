# 基本布局

好吧，所以链表是什么东西呢？大致上说，它是一大片相互指向的数据，以顺序连接起来（嘘，Linux内核！）。链表是过程式程序员应该用一切代价避免的东西，而函数程序员则在所有情况下使用它。那么，看起来向函数式程序员询问链表的定义是很公平的。他们可能会给你类似这样的定义：

```haskell
List a = Empty | Elem a (List a)
```

它可以大致读成“一个列表要么是空的，要么是一个元素接着一个列表”。这是用一个复合类型（sum type）表达的递归定义，而复合类型只是“一个可以拥有多种类型的值的类型”的酷炫叫法。Rust把复合类型称作`enum`！如果你是从一个C系语言过来的，那么这正是你所熟知并热爱的枚举类型，但是强大了许多。让我们把这个函数式的链表定义转录到Rust吧！

现在为了保持说明简单，我们会避开泛型。我们暂时只支持有符号的32位整数：

```rust,ignore
// 在 first.rs 中

// pub 表明我们想让这个模块外之外的人使用这个 List
pub enum List {
    Empty,
    Elem(i32, List),
}
```

呼，这一点也不麻烦嘛。我们继续下去编译它吧：

```text
> cargo build
   Compiling lists v0.1.0 (file:///Users/ABeingessner/dev/lists)
src/first.rs:1:1: 4:2 error: illegal recursive enum type; wrap the inner value in a box to make it representable [E0072]
src/first.rs:1 pub enum List {
src/first.rs:2    Empty,
src/first.rs:3    Elem(T, List),
src/first.rs:4 }
error: aborting due to previous error
Could not compile `lists`.
```

不！！！！函数式程序员欺骗了我们！它让我们做了*不合法*的东西！这是圈套！

……

我冷静下来了。你冷静了么？如果我们真正去检查错误消息（而不是像我们中的某些人一样，准备逃出这个国家），我们就会发现rustc实际上在告诉我们如何解决这个问题：

> illegal recursive enum type; wrap the inner value in a box to make it representable
> 不合法的递归枚举类型；把值包装在一个box里让它变得可表示
好吧，`box`。那是什么东西？让我们 google `rust box`……

> [std::boxed::Box - Rust](https://doc.rust-lang.org/std/boxed/struct.Box.html)

看看接下来是啥……

> `pub struct Box<T>(_);`
>
> A pointer type for heap allocation.
> See the [module-level documentation](https://doc.rust-lang.org/std/boxed/) for more.

*点击链接*

> `Box<T>`，或被随意的称为`box`，提供了Rust中最简单的堆内存分配的形式。Box提供了当次内存分配的所有权，并在退出作用域时销毁存放的内容。
>
> 示例
>
> 创建一个box
>
> `let x = Box::new(5);`
>
> 创建一个递归数据结构：
>
```
#[derive(Debug)]
enum List<T> {
    Cons(T, Box<List<T>>),
    Nil,
}
```
>
```
fn main() {
    let list: List<i32> = List::Cons(1, Box::new(List::Cons(2, Box::new(List::Nil))));
    println!("{:?}", list);
}
```
>
> 这会打印 `Cons(1, Box(Cons(2, Box(Nil))))`.
>
> 递归的结构必须使用box包装，因为如果Cons的定义如下这样：
>
> `Cons(T, List<T>),`
>
> 是不会工作的。这是因为List的大小由其中的元素数量所决定，所以我们无法决定为一个Cons分配多少内存。通过引入一个具有固定大小的Box，我们才知道Cons需要占用多少内存。

哇哦。这或许是我见过的最相关最有帮助的文档了。在里面的第一个东西就是*我们正在尝试写的东西，为什么它不能工作，以及如何修复它*。好耶，文档。

OK，我们来完成它：

```rust
pub enum List {
    Empty,
    Elem(i32, Box<List>),
}
```

```text
> cargo build
   Compiling lists v0.1.0 (file:///Users/ABeingessner/dev/lists)
```

嘿，它成功编译了！

……但这实际上是一个非常蠢的List的定义，出于以下的一些原因。

考虑一个拥有两个元素的列表：

```text
[] = 栈
() = 堆

[Elem A, ptr] -> (Elem B, ptr) -> (Empty *垃圾*)
```

这里有两个关键问题：

* 我们创建了一个“实际上不是个节点”的节点
* 其中的一个节点根本没分配在堆里

在表面上，这两个问题似乎相互抵消。我们分配了一个多余的节点，但有一个节点完全无需在堆里分配。然而，考虑我们链表的一个潜在的内存布局：

```text
[ptr] -> (元素A, ptr) -> (元素B, *null*)
```

在这个布局里，我们在堆里分配所有的元素。和第一个布局相比，核心的区别是多余的*垃圾*的消失。这个垃圾到底是什么？为了理解它，我们需要看一看enum是如何在内存中布局的。

通常的，如果我们有像这样的一个enum：

```rust
enum Foo {
    D1(T1),
    D2(T2),
    ...
    Dn(Tn),
}
```

一个Foo需要保存一个整数，来指出它实际表示的是哪一个*变体*（`D1`, `D2`, .. `Dn`）。这是enum的*标签*（tag）。它也需要足够大的空间，来存储`T1`, `T2`, .. `Tn`中的最大元素（以及用来满足内存对齐要求的附加空间）。

一个很大的缺陷是，尽管`Empty`只存储了一位的信息，它却消耗了一个指针和一个元素的内存空间，因为它要随时准备成为一个`Elem`。因此第一种布局在堆里分配了一个充满垃圾的多余元素，比第二种布局消耗更多的空间。

让我们的一个元素不在堆中分配，或许也比所有元素都在堆中分配更糟。这是因为它给了我们一个*不一致的*节点内存布局。在推入和弹出节点时这并无问题，但在分割和合并列表时确实会有影响。

考虑在两种布局里分别分割一个列表：

```text
布局 1：

[Elem A, ptr] -> (Elem B, ptr) -> (Elem C, ptr) -> (Empty *junk*)

在C点分割：

[Elem A, ptr] -> (Elem B, ptr) -> (Empty *junk*)
[Elem C, ptr] -> (Empty *junk*)
```

```text
布局 2：

[ptr] -> (Elem A, ptr) -> (Elem B, ptr) -> (Elem C, *null*)

在C点分割：

[ptr] -> (Elem A, ptr) -> (Elem B, *null*)
[ptr] -> (Elem C, *null*)
```

布局2的分割仅仅涉及将B的指针拷贝到栈上，并把原值设置为null。布局1最终还是做了同一件事，但是还得把C从堆中拷贝到栈中。反过来执行上述操作，就是合并列表。

链表的优点之一就是可以在节点中构建元素，然后在列表中随意调换它的位置而不需移动它的内存。你只需要调整指针，元素就被“移动了”。第一个布局毁掉了这个特点。

好吧，我现在很确信布局1是糟糕的。我们要怎么重写List呢？可以这么做：

```rust
pub enum List {
    Empty,
    ElemThenEmpty(i32),
    ElemThenNotEmpty(i32, Box<List>),
}
```

或许你觉得这看起来更糟了。一个问题是，这让逻辑变得更复杂了。具体地说，现在出现了一个完全无效的状态：`ElemThenNotEmpty(0, Box(Empty))`。它也*仍*被内存分配模式不一致的问题所困扰。

不过它确实有*一个*有趣的特性：它完全避免了在堆里分配Empty，让堆内存分配的数量减少了1。不幸的是，这么做反而浪费了*更多空间*！因为之前的布局利用了*空指针优化*。

我们之前了解到每个enum需要存储一个*标签*，来指明它代表哪一个enum的*变体*。然而，如果我们有如下特殊类型的enum：

```rust
enum Foo {
    A,
    B(ContainsANonNullPtr),
}
```

空指针优化就会发挥作用，*消除标签所占用的内存空间*。如果变体是A，那整个enum就被设置为0。否则，变体是B。这可以工作，因为B存放了一个非空指针，永远不可能为0。真聪明！

 你还能想到能进行这种优化的enum和类型么？实际上有很多！这就是为什么Rust没有详细描述enum的内存布局。悲伤的是，现在实现的优化只有空指针优化——尽管它很重要！这意味着`&`, `&mut`, `Box`, `Rc`, `Arc`, `Vec`，以及其他一些Rust中的重要类型，在放到一个 `Option` 中时没有多余开销！（上面这些概念的大部分，我们在适当的时候都会接触到）


所以我们要如何避免多余垃圾，统一的分配内存，*并且*从空指针优化中获益呢？我们需要更好的将存储元素和分配列表这两个想法分开。要做到它，我们该像C语言看齐：struct！

enum让我们定义了一种可以存放多个变体中的一个的类型，而struct则让我们定义可以同时存放多种元素的类型。让我们把List分成两个类型吧：一个List，和一个Node。

和之前一样，一个List要么是Empty，要么是一个元素跟着一个List。不过，要通过另外一种类型来表示“一个元素跟着一个List”，我们可以将Box提升到一个更理想的位置：

```rust
struct Node {
    elem: i32,
    next: List,
}

pub enum List {
    Empty,
    More(Box<Node>),
}
```

让我们检查各个条目：

* 列表末尾不分配多余垃圾：通过！
* `enum` 享受美妙的空指针优化：通过！
* 所有元素的内存分配一致：通过！

好的！我们创建的正是用来指明第一种内存布局（官方Rust文档中所建议的那种）有问题的第二种内存布局。

```text
> cargo build
   Compiling lists v0.1.0 (file:///Users/ABeingessner/dev/lists)
src/first.rs:8:11: 8:18 error: private type in exported type signature
src/first.rs:8    More(Box<Node>),
                           ^~~~~~~
error: aborting due to previous error
Could not compile `lists`.
```

:(
    
Rust又对我们发飙了。我们将`List`标记为public（因为我们想让其他人使用它），却没有公开`Node`。问题在于，`enum`的内部是完全公开的，所以在其中包含内部类型是不允许的。我们可以让整个`Node`都成为公开的，但是通常在Rust中，我们倾向于让实现细节私有化。让我们把`List`改造成一个struct，这样我们就可以隐藏实现细节：


```rust
pub struct List {
    head: Link,
}

enum Link {
    Empty,
    More(Box<Node>),
}

struct Node {
    elem: i32,
    next: Link,
}
```

因为`List`是一个单值的struct，它的大小和该值完全相同。零代价抽象超赞！

```text
> cargo build
   Compiling lists v0.1.0 (file:///Users/ABeingessner/dev/lists)
src/first.rs:2:2: 2:15 warning: struct field is never used: `head`, #[warn(dead_code)] on by default
src/first.rs:2    head: Link,
                  ^~~~~~~~~~~~~
src/first.rs:6:2: 6:7 warning: variant is never used: `Empty`, #[warn(dead_code)] on by default
src/first.rs:6    Empty,
                  ^~~~~
src/first.rs:7:2: 7:20 warning: variant is never used: `More`, #[warn(dead_code)] on by default
src/first.rs:7    More(Box<Node>),
                  ^~~~~~~~~~~~~~~~~~
src/first.rs:11:2: 11:9 warning: struct field is never used: `elem`, #[warn(dead_code)] on by default
src/first.rs:11   elem: i32,
                  ^~~~~~~
src/first.rs:12:2: 12:15 warning: struct field is never used: `next`, #[warn(dead_code)] on by default
src/first.rs:12   next: Link,
                  ^~~~~~~~~~~~~
```

好吧，终于编译了！Rust非常生气，因为我们现在写的东西完全无用：我们从不使用`head`，并且因为它是私有的；使用我们库的人也无法使用它。进而Link和Node也毫无用处。让我们来解决它吧！为我们的List实现一些代码！
