# Drop

我们现在可以创建一个栈，推入元素，弹出元素，甚至确认了一切都可以正常的工作！

我们需要担心列表元素的清理么？严格的说，根本不用！就像C++，Rust使用析构器来自动的处理使用完毕的资源。如果一个类型实现了叫做 Drop 的*特性（Trait）*，它就拥有一个析构器。特性是Rust对接口的特别术语。Drop特性有如下的接口：

```
pub trait Drop {
    fn drop(&mut self);
}
```

基本上是这个意思：“当对象退出作用域的时候，我会给你清理事务的第二次机会”。

如果你的类型里存放有实现了Drop的其他类型，而你想要调用它们的析构器，是不需要实际实现Drop的。对于List来说，我们想做的不过是把列表头丢弃，之后或许会接着丢弃一个`Box<Node>`。所有这些都会自动在一瞬间处理完成。

自动处理会很糟糕。

让我们考虑这个简单的列表。

```text
list -> A -> B -> C
```

当列表被丢弃时，它会先丢弃A，然后尝试丢弃B，然后会尝试丢弃C。现在你可能已经紧张起来了。这是递归代码，而递归代码会把栈爆掉！

```rust
impl Drop for List {
    fn drop(&mut self) {
        // 注意：在实际Rust代码中你不能显式调用`drop`，
        // 我们假装自己是编译器！
        list.head.drop(); // 尾递归——好！
    }
}

impl Drop for Link {
    fn drop(&mut self) {
        match list.head {
            Link::Empty => {} // 完成！
            Link::More(ref mut boxed_node) => {
                boxed_node.drop(); // 尾递归——好！
            }
        }
    }
}

impl Drop for Box<Node> {
    fn drop(&mut self) {
        self.ptr.drop(); // 糟糕，不是尾递归！
        deallocate(self.ptr);
    }
}

impl Drop for Node {
    fn drop(&mut self) {
        self.next.drop();
    }
}
```

我们不能在释放内存之后再丢弃Box的内容，所以没有办法以尾递归的形式进行drop！作为替代，我们必须为`List`手动编写一个迭代drop，来把节点从box中拿出来。

```rust
impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, Link::Empty);
        // `while let` == “在这个模式不匹配之前持续循环”
        while let Link::More(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, Link::Empty);
            // boxed_node在这里退出作用域然后被丢弃；
            // 但是其节点的`next`字段被设置为 Link::Empty
            // 所以没有多层递归产生。
        }
    }
}
```

```text
> cargo test
   Compiling lists v0.1.0 (file:///Users/ABeingessner/dev/too-many-lists/lists)
     Running target/debug/lists-5c71138492ad4b4a

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests lists

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

棒极了！
