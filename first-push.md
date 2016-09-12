# Push

我们来实现将一个值推入列表的功能吧。`push`*改变了*列表，因此我们需要`&mut self`。还需要提供一个推入的i32参数：

```rust
impl List {
    pub fn push(&mut self, elem: i32) {
        // TODO
    }
}
```

首先，我们需要创建一个节点来存储我们的元素：

```rust
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: ?????
        };
    }
```

`next`那里存储的是什么呢？呃，是整个原先的列表！我们...可以这么做么？

```rust
impl List {
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: self.head,
        };
    }
}
```

```text
> cargo build
   Compiling lists v0.1.0 (file:///Users/ABeingessner/dev/lists)
src/first.rs:19:10: 19:14 error: cannot move out of borrowed content
src/first.rs:19           next: self.head,
                                ^~~~
error: aborting due to previous error
Could not compile `lists`.
```

不不不。Rust在告诉我们正确的事，但是它的实际意义和修复它的方法并不明了：

> cannot move out of borrowed content （无法移动出租借的内容）

我们尝试把`self.head`字段移动到`next`中，但是Rust不想让我们做这件事。这会导致在我们的租借结束，值返回到其所有者的时候，`self`只是部分初始化的。正如我们之前所说，这是你不能通过`&mut`做的*唯一一件事*：这样做是非常粗野的，而Rust则打算尽量保持礼貌（同时这也是极其危险的，但是并不是Rust关心它的原因）。

如果我们把东西放回去呢？具体的说，就是我们创建的节点：


```rust
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: self.head,
    });

    self.head = Link::More(new_node);
}
```

```text
> cargo build
   Compiling lists v0.1.0 (file:///Users/ABeingessner/dev/lists)
src/first.rs:19:10: 19:14 error: cannot move out of borrowed content
src/first.rs:19           next: self.head,
                                ^~~~
error: aborting due to previous error
Could not compile `lists`.
```

不行。在原则上，Rust是可以接受这样的行为的，但是它不会（出于数个原因——最重要的是安全性）我们需要某种方法得到head，而不让Rust发现它已经消失了。我们转而向声名狼藉的Rust黑客Indiana Jones寻求建议：

![Indy准备进行mem::replace](indy.gif)

噢没错，Indy建议采用`mem::replace`这一招。这个极其有用的函数让我们通过*替换*将一个值从租借中偷出来。让我们先在文件顶部把`std::mem`拉进来，这样`mem`就在局部作用域了：

```rust
use std::mem;
```

然后恰当的使用它：

```rust
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: mem::replace(&mut self.head, Link::Empty),
    });

    self.head = Link::More(new_node);
}
```

在将self.head替换为列表的新头部之前，我们临时的将它`replace`为Link::Empty。我不会撒谎：非要这么去做真是很糟糕。悲伤的是，我们必须这样（至少现在）。

不过，这样子`push`就完成了！或许吧。说真的，我们应该测试一下它。现在来说，最简单的测试方法是实现`pop`，然后确认它给出了正确的结果。

