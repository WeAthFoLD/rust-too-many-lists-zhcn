# 测试

我们现在已经实现了`push`和`pop`，就可以测试我们的栈了！Rust和cargo把测试作为一个一级特性来实现，所以写起测试来会很轻松。我们需要做的只是写一个函数，然后用`#[test]`标记它。

通常在Rust社区中，我们会把测试代码放在它所测试的部分的附近。不过我们通常会为测试创建单独的命名空间，来让它不与“真正的”代码产生冲突。就像我们用`mod`来表明`first.rs`应该被包含在`lib.rs`中，可以使用`mod`来*内联的*创建一个新文件：


```rust
// in first.rs

mod test {
    #[test]
    fn basics() {
        // TODO
    }
}
```

之后，我们通过`cargo test`调用它。

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

好的，我们的什么都不做的测试通过了！我们来让它实际做些事吧。我们会使用`assert_eq!`宏来进行测试。这不是什么特殊的测试魔法。它所做的仅仅是比较你给它的两个值，并且让在它们不相等的情况下让程序panic。没错，你通过崩溃来指出测试中的失败！

```rust
mod test {
    #[test]
    fn basics() {
        let mut list = List::new();

        // 检查空列表行为正确
        assert_eq!(list.pop(), None);

        // 填充列表
        list.push(1);
        list.push(2);
        list.push(3);

        // 检查通常的移除
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(2));

        // 推入更多元素来确认没有问题
        list.push(4);
        list.push(5);

        // 检查通常的移除
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), Some(4));

        // 检查完全移除
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), None);
    }
}
```

```text
> cargo test
   Compiling lists v0.1.0 (file:///Users/ABeingessner/dev/too-many-lists/lists)
src/first.rs:47:24: 47:33 error: failed to resolve. Use of undeclared type or module `List` [E0433]
src/first.rs:47         let mut list = List::new();
                                       ^~~~~~~~~
src/first.rs:47:24: 47:33 error: unresolved name `List::new` [E0425]
src/first.rs:47         let mut list = List::new();
                                       ^~~~~~~~~
error: aborting due to 2 previous errors
```

噢！因为我们做了一个新的模块，所以需要把List显式的导入进来才能使用它。

```rust
mod test {
    use super::List;
    // 其他不变
}
```

```text
> cargo test
   Compiling lists v0.1.0 (file:///Users/ABeingessner/dev/too-many-lists/lists)
src/first.rs:45:9: 45:20 warning: unused import, #[warn(unused_imports)] on by default
src/first.rs:45     use super::List;
                        ^~~~~~~~~~~
     Running target/debug/lists-5c71138492ad4b4a

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests lists

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

好极了！

那个警告又是怎么回事呢……？我们清楚的在测试里用了List！

……但仅仅在测试的过程中！为了平息编译器（以及对库的使用者友好），我们应该指明test模块只会在运行测试的过程中编译。

```
#[cfg(test)]
mod test {
    use super::List;
    // everything else the same
}
```

这就是关于测试的所有要点了！

