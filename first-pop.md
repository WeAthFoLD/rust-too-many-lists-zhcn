# Pop

和`push`一样，`pop`想要改变列表；除此之外，我们还想返回结果。然而`pop`还得处理一个特殊的边界情况：如果列表是空的呢？为了表示这个情况，我们使用可靠的`Option`类型：

```rust
pub fn pop(&mut self) -> Option<i32> {
    //TODO
}
```

`Option<T>`是一个表示一个值可能存在也可能不存在的enum。它要么是`Some(T)`，要么是`None`。我们也可以像Link一样创建一个自己的enum，但是我们想让用户了解我们的返回类型到底是什么，而Option是如此的无处不在，每个人都知道它。实际上，因为它是如此的基本，它被隐式的导入到了每一个源文件的作用域中，也包括它的两个变体：`Some`和`None`（这样我们就不用写`Option::None`）。

在`Option<T>`尖括号里的部分指出Option实际上是一个泛型，它的泛型参数是T。这意味着你可以创建一个任何类型的Option！

所以，我们有这个`Link`了，我们怎么检查它是Empty还是More呢？使用`match`进行模式匹配：

```rust
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
}
```

```text
> cargo build
   Compiling lists v0.1.0 (file:///Users/ABeingessner/dev/lists)
src/first.rs:27:2: 36:3 error: not all control paths return a value [E0269]
src/first.rs:27   pub fn pop(&mut self) -> Option<i32> {
src/first.rs:28       match self.head {
src/first.rs:29           Link::Empty => {
src/first.rs:30               // TODO
src/first.rs:31           }
src/first.rs:32           Link::More(node) => {
              ...
error: aborting due to previous error
Could not compile `lists`.
```

啊，`pop`必须返回一个值，我们还没做这件事。我们*可以*直接返回`None`，但是在这情况下，返回`unimplemented!`来指出我们没有完成该函数的实现会更好。`unimplemented!`是一个宏（`!代表一个宏`），它会在被调用的时候让整个程序panic（基本上也就是以可控的方式崩溃）。

```rust
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
    unimplemented!()
}
```

无条件panic是一个[发散函数(diverging function)][diverging]的例子。发散函数永远不会返回到调用者，所以无论一个地方期待何种类型的返回值，它的返回值都能拿来用。在这里，`unimplemented!`被使用在期待一个`Option<T>`的地方。

注意到我们不需要在程序里写`return`。函数中的最后一个表达式也就隐式的成为它的返回值。这让我们可以更精炼的表达简单的逻辑。你也可以像C系语言一样，显式的`return`返回。

```text
> cargo build
   Compiling lists v0.1.0 (file:///Users/ABeingessner/dev/lists)
src/first.rs:28:9: 28:13 error: cannot move out of borrowed content
src/first.rs:28       match self.head {
                            ^~~~
src/first.rs:32:15: 32:19 note: attempting to move value to here
src/first.rs:32           Link::More(node) => {
                                     ^~~~
src/first.rs:32:15: 32:19 help: to prevent the move, use `ref node` or `ref mut node` to capture value by reference
error: aborting due to previous error
Could not compile `lists`.
```

噢，Rust，别纠缠不休了！和往常一样，Rust对我们非常生气。值得感谢的是，这次它还给出了深入的信息！

```text
src/first.rs:17:9: 17:13 error: cannot move out of borrowed content
src/first.rs:17       match self.head {
                            ^~~~
src/first.rs:21:15: 21:19 note: attempting to move value to here
src/first.rs:21           Link::More(node) => {
                                     ^~~~
```

模式匹配默认会移动匹配的值，所以这就是它如此生气的原因。

```text
help: to prevent the move, use `ref node` or `ref mut node` to capture value by reference
帮助：要避免移动，使用`ref node`或`ref mut node`来引用捕获该值
```

为了避免移动，我们使用`ref`关键词来指明我们想要把`node`进行引用绑定。来试试吧：

```rust
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(ref node) => {
            // TODO
        }
    };
    unimplemented!()
}
```

```text
> cargo build
   Compiling lists v0.1.0 (file:///Users/ABeingessner/dev/lists)
src/first.rs:13:2: 13:9 warning: struct field is never used: `elem`, #[warn(dead_code)] on by default
src/first.rs:13   elem: i32,
                  ^~~~~~~
src/first.rs:14:2: 14:15 warning: struct field is never used: `next`, #[warn(dead_code)] on by default
src/first.rs:14   next: Link<T>,
                  ^~~~~~~~~~~~~
src/first.rs:32:15: 32:23 warning: unused variable: `node`, #[warn(unused_variables)] on by default
src/first.rs:32           Link::More(ref node) => {
                                     ^~~~~~~~
```

好耶，又编译了！现在让我们搞清楚实现逻辑。我们要创建一个Option，所以要为这个预留一个变量。在Empty情况下要返回None，在More情况下需要返回`Some(i32)`，并且改变列表的head。来吧：

```rust
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match self.head {
        Link::Empty => {
            result = None;
        }
        Link::More(ref node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
> cargo build
   Compiling lists v0.1.0 (file:///Users/ABeingessner/dev/too-many-lists/lists)
src/first.rs:39:29: 39:33 error: cannot move out of borrowed content
src/first.rs:39                 self.head = node.next;
                                            ^~~~
src/first.rs:39:17: 39:38 error: cannot assign to `self.head` because it is borrowed
src/first.rs:39                 self.head = node.next;
                                ^~~~~~~~~~~~~~~~~~~~~
src/first.rs:37:24: 37:32 note: borrow of `self.head` occurs here
src/first.rs:37             Link::More(ref node) => {
                                       ^~~~~~~~
error: aborting due to 2 previous errors
Could not compile `lists`.
```

*头*

*桌*

我们现在有了两个不同的的错误。。首先，我们在仅仅拥有一个共享引用的情况下就尝试把值移动出`node`。其次，在我们已经租借了`node`的引用的时候，还在尝试改变`self.head`的值！

真是一堆纠缠不清的乱东西。

我们应该后退一步，思考我们要做什么。我们想要：

* 检查列表是否为空。
* 如果是空的，返回None
* 如果是非空
    * 移除list头部
    * 移除该头部的`elem`
    * 将列表的head替换为`next`
    * 返回`Some(elem)`

重要的一点事我们想要*删除*东西，这意味着我们需要*按值*获取list的head。我们肯定不能通过由`ref node`获取的共享引用来做这件事。我们也“只”拥有一个可变引用，所以能移动东西的唯一方法就是*替换它*。看来我们又在做Empty替换那一套了！来试试吧：

```rust
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => {
            result = None;
        }
        Link::More(node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
cargo build
   Compiling lists v0.1.0 (file:///Users/ABeingessner/dev/too-many-lists/lists)
```

我 的 天 哪

它编译了，一个警告都没有！！！！！

这里我要给出我的优化提示了：我们现在返回的是result变量的值，但实际上根本不用这么做！就像一个函数的结果是它的最后一个表达式，每个代码块的结果也是它的最后一个表达式。通常我们使用分号来阻止这一行为，这会让代码块的值变成空元组（tuple）`()`。这实际上也是不声明返回值的函数——例如`push`——返回的。

因此，我们可以将`pop`修改为：

```rust
pub fn pop(&mut self) -> Option<i32> {
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => None,
        Link::More(node) => {
            self.head = node.next;
            Some(node.elem)
        }
    }
}
```

这更简洁，也更符合语言惯例。注意到Link::Empty分支只需要求值一个表达式，所以我们把大括号也去掉了。这是对于简单情况的简便处理。

```text
> cargo build
   Compiling lists v0.1.0 (file:///Users/ABeingessner/dev/too-many-lists/lists)
src/first.rs:36:22: 36:31 error: use of moved value: `node` [E0382]
src/first.rs:36                 Some(node.elem)
                                     ^~~~~~~~~
src/first.rs:35:29: 35:38 note: `node` moved here (through moving `node.next`) because it has type `first::Link`, which is non-copyable
src/first.rs:35                 self.head = node.next;
                                            ^~~~~~~~~
error: aborting due to previous error
```

啥？别这样啊。

为什么我们的代码不工作了？！

实际上，我们之前的代码只是侥幸通过了编译，借助Copy的魔法。当我们介绍[所有权][ownership]的时候说过，当你移动值的时候，就无法再使用它。对于某些类型，这是完全合理的。我们的好朋友Box为我们管理堆中的内存分配，而我们显然不想让两段代码认为它们应该释放相同的一块内存。

但是对某些类型这简直*糟透了*。整数可没有所有权语义：它们只是毫无意义的数字！这也正是为什么整数被标记为Copy。Copy类型可以通过按位复制进行完整的拷贝。因此，它们拥有一个超能力：当被移动的时候，老的值仍然是可用的。作为结果，你可以将一个Copy类型从引用移出而不需替换！

所有rust中的基本数字类型（i32, u64, bool, f32, char, etc...)都是Copy。同时，共享引用也是Copy，这很有用！只要一个自定类型的所有字段都是Copy，你也可以将该类型声明为Copy。

总之，回到代码：出了什么错？在第一次迭代过程中，我们对result进行赋值时在进行*拷贝*，因此node没被改变，可以被继续用于下一个操作。现在我们在移动`next`（它不是Copy），而这在我们能碰到elem之前消耗掉了整个Box里的值。

现在，我们可以重新调整代码来只拿到`elem`，但我们只是使用i32作为某种数据的占位符。晚些时候我们会处理非Copy类型的数据，所以最好现在就研究研究怎么做。

正确的答案是将*整个*节点从Box中取出来，这样就可以安全的将它拆开了。我们通过显式解引用操作来做这件事：

```rust
pub fn pop(&mut self) -> Option<i32> {
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => None,
        Link::More(boxed_node) => {
            let node = *boxed_node;
            self.head = node.next;
            Some(node.elem)
        }
    }
}
```

在这之后，Rust就可以足够好的理解一个栈上的值，来让你一步步分解它了。

```text
> cargo build
   Compiling lists v0.1.0 (file:///Users/ABeingessner/dev/too-many-lists/lists)
```

不错。

Box在Rust里真的很特殊，因为它是固定于语言里的一部分，编译器可以让你对它做一些其它任何类型都不能做的事。我们实际上一直在做这件事：`DerefMove`。当你拥有一个指针类型时，可以通过`*`或`.`来获得它的内容。通常你可以获得一个`Deref`或者一个`DerefMut`，分别对应共享和可变引用。

但是因为Box完全拥有它的内容，你可以通过解引用*将内容移出*。这是完完全全的魔法，因为其他任何类型都无法实现这个操作。编译器还知道如何在Box上实现很多很多的酷炫技巧，只因为它*就是*Box，但是它们都被阻挡在了1.0版本的实现目标之外，等待进一步的设计。理想的，Box会在未来完全可自定义化。



[ownership]: first-ownership.html
[diverging]: http://doc.rust-lang.org/nightly/book/functions.html#diverging-functions
