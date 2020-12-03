[原文](https://seed-rs.org/0.8.0/rust/)
1. Rust的编译器是你的朋友。

2. 给你的代码编写[Document ](https://doc.rust-lang.org/book/ch14-02-publishing-to-crates-io.html#making-useful-documentation-comments)，写出富有表现力的名字、可读性高的文档；最好可以在注释里测试你的代码片段。

3. 了解在哪里使用 [Result](https://doc.rust-lang.org/std/result/)，在哪里使用[panic](https://doc.rust-lang.org/book/ch09-03-to-panic-or-not-to-panic.html#to-panic-or-not-to-panic)。

4. 尊重[命名约定(naming conventions)](https://rust-lang.github.io/api-guidelines/naming.html)。

5. 了解这些著名的关系对：

   - [From ](https://doc.rust-lang.org/std/convert/trait.From.html)和 [Into ](https://doc.rust-lang.org/std/convert/trait.Into.html)+ [TryFrom ](https://doc.rust-lang.org/std/convert/trait.TryFrom.html)和 [TryInto](https://doc.rust-lang.org/std/convert/trait.TryInto.html)
   - [String](https://doc.rust-lang.org/std/string/struct.String.html) 和 [str](https://doc.rust-lang.org/std/primitive.str.html)
   - [Display](https://doc.rust-lang.org/std/fmt/trait.Display.html) 和 [ToString](https://doc.rust-lang.org/std/string/trait.ToString.html)
   - [fn](https://doc.rust-lang.org/std/primitive.fn.html) 和 [Fn](https://doc.rust-lang.org/std/ops/trait.Fn.html)
   - [FromStr](https://doc.rust-lang.org/std/str/trait.FromStr.html) 和 [parse](https://doc.rust-lang.org/std/primitive.str.html#method.parse)
   - [Rc](https://doc.rust-lang.org/std/rc/struct.Rc.html) 和 [RefCell](https://doc.rust-lang.org/stable/std/cell/struct.RefCell.html)
   - [Vec](https://doc.rust-lang.org/std/vec/struct.Vec.html) 和 [vec!](https://doc.rust-lang.org/std/macro.vec.html)
   - [Deref](https://doc.rust-lang.org/std/ops/trait.Deref.html) 和 [DerefMut](https://doc.rust-lang.org/std/ops/trait.DerefMut.html)

6. 如果可以的话使用[early returns](https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html#recoverable-errors-with-result)。

7. 不要每件事情都用一个编程范型，学习在哪里使用[循环](https://doc.rust-lang.org/book/ch03-05-control-flow.html#repetition-with-loops)，在哪里使用[迭代器](https://doc.rust-lang.org/std/iter/index.html)。

8. 尽可能的使用[引用](https://doc.rust-lang.org/std/primitive.reference.html)，比如在使用[String](https://doc.rust-lang.org/std/string/struct.String.html)的地方可以用[&str](https://doc.rust-lang.org/std/primitive.str.html)代替；在使用[Vec](https://doc.rust-lang.org/std/vec/struct.Vec.html)的地方可以用[&[T]](https://doc.rust-lang.org/std/primitive.slice.html)来代替。

9. 尽可能少的去调用`clone`，当你真的需要使用`clone`时，可以试试`Rc::clone(&value)` [[rc docs\]](https://doc.rust-lang.org/std/rc/index.html)

10. Rust的一个特点是安全可变性，但是只有当你写出的[不可变变量](https://doc.rust-lang.org/book/ch03-01-variables-and-mutability.html#variables-and-mutability)和[纯函数](https://en.wikipedia.org/wiki/Pure_function)使你的代码不可读，太慢或者容易出错的时候，才使用它。

11. “提前优化是一切灾祸的根源。”——特别是在Rust这种运行速度很快的语言当中。不要做额外的事除非你的基准测试已经准备好要去测试你的代码。个人的一些案列：

    - 当我在用Rust写代理服务器的时候，有两个因素使得代理服务器的速度减慢了好几倍，一个是忘记删除存在于热路径（hot path）中的`println`调用，而另一个则是由于DNS服务器变慢。我建议缩小范围并优先解决更高级别的问题。
    - Seed的VDOM补丁算法在第一次尝试时是足够快的，然而由于DOM的调用大大降低了它的速度。我建议在你尝试去优化你的Rust代码之前，先去查看一下它的IO和外部依赖。

12. 仅编写跨平台代码，并且仅使用Rust工具链。

13. 不要害怕去写[异步](https://rust-lang.github.io/async-book/01_getting_started/01_chapter.html)代码。

14. 了解诸如此类受欢迎的库：

    - [serde](https://crates.io/crates/serde)
    - [rand](https://crates.io/crates/rand)
    - [futures](https://crates.io/crates/futures)
    - [strum](https://crates.io/crates/strum)
    - [once_cell](https://crates.io/crates/once_cell)
    - [itertools](https://crates.io/crates/itertools)
    - [indexmap](https://crates.io/crates/indexmap)
    - [chrono](https://crates.io/crates/chrono)
    - [uuid](https://crates.io/crates/uuid) + [ulid](https://crates.io/crates/ulid) + [cuid](https://crates.io/crates/cuid)
    - [rayon](https://crates.io/crates/rayon)

15. [Clippy](https://github.com/rust-lang/rust-clippy) 和 [rustfmt](https://github.com/rust-lang/rustfmt) 同样也是你的朋友。[cargo-make](https://sagiegurari.github.io/cargo-make/) 是你非官方的朋友。

    - 你可以使用`cargo make verify`命令在几乎所有的Seed/my projects。它将会使用`Clippy`格式化代码，并且测试它。可以在Rust quickstart看到[任务定义](https://github.com/seed-rs/seed-quickstart/blob/8c5807721e2e67d12e3f93533ebb75b871203800/Makefile.toml#L22-L24)

16. 编写[测试](https://doc.rust-lang.org/book/ch11-01-writing-tests.html)和基准(参见 [Criterion.rs](https://bheisler.github.io/criterion.rs/book/criterion_rs.html))。

17. 用下面的函数参数类型进行实验，并了解这些用处。（提示：在Seed的[repo](https://github.com/seed-rs/seed)里这些参数类型都有不同的用处，可以参考学习。）

    - `fn(text: impl AsRef<str>)` - [AsRef](https://doc.rust-lang.org/std/convert/trait.AsRef.html)
    - `fn(text: impl ToString)` - [ToString](https://doc.rust-lang.org/std/string/trait.ToString.html)
    - `fn(text: impl Into<Cow<'static, str>>)` - [Cow](https://doc.rust-lang.org/std/borrow/enum.Cow.html)
    - `fn<'a>(text: impl Into<Cow<'a, str>>)`

18. 偶尔需要做一下：

    - 运行`rustup update`去更新你的编译器和工具链
    - 每次更新编译器版本之后（或者更新依赖之后），删除你项目下占用你磁盘空间的`target`文件夹，他们是老版本编译器的产物。

19. 了解这些[内存](https://doc.rust-lang.org/std/mem/index.html)相关的函数：

    - [discriminant](https://doc.rust-lang.org/std/mem/fn.discriminant.html)
    - [drop](https://doc.rust-lang.org/std/mem/fn.drop.html)
    - [replace](https://doc.rust-lang.org/std/mem/fn.replace.html)
    - [swap](https://doc.rust-lang.org/std/mem/fn.swap.html)
    - [take](https://doc.rust-lang.org/std/mem/fn.take.html)

20. 只有在特殊情况下，或者当你觉得极致性能对于你的领域来说十分有必要的情况下（比如你写操作系统，或者写超快速低级别的库的时候），才去使用[unsafe](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html?unsafe-rust)。在Seed项目下只有安全的代码（包括Seed的核心部分）。

21. 只有在为了给你的用户改进你提供的API的时候，才去使用像[Any](https://doc.rust-lang.org/std/any/trait.Any.html)这种魔法。它通常只会让你的代码更糟糕。

22. 学会使用通道（channel），标准库的文档以及一些相关crate的文档：

    - [std::sync::mpsc::channel](https://doc.rust-lang.org/std/sync/mpsc/fn.channel.html)
    - [futures::channel](https://docs.rs/futures/0.3.5/futures/channel/index.html)
    - [tokio::sync::mpsc::channel](https://docs.rs/tokio/0.2.21/tokio/sync/mpsc/fn.channel.html)
    - [crossbeam::channel](https://docs.rs/crossbeam/0.7.3/crossbeam/channel/index.html)
    - [flume](https://docs.rs/flume/0.7.1/flume/)

23. 只有你真的觉得有必要的时候才去使用[宏](https://doc.rust-lang.org/book/ch19-06-macros.html#macros)，并且要给你的宏写上合适的注释。IDE对宏的支持很糟糕，比如自动补全通常不能工作。

    - 有一些例外的宏比如 `println`，`vec`，`include_str`，所有的[标准库的宏](https://doc.rust-lang.org/std/index.html#macros)
    - 是的，在Seed里面有很多的宏，但是这些宏的大部分被用来替代HTML并且他们十分短。还有我们修复了他们当中的很多bug，因此该规则仍然适用。我们在权衡了很多东西之后才决定去使用宏。
    - 然而宏在很多地方十分有用：
      - 缺少抽象概念的时候，比如`drop_zone`里面的[`stop_and_prevent`](https://github.com/seed-rs/seed/blob/3134d21c6fcb2383685885687fe2a7610fb2ff74/examples/drop_zone/src/lib.rs#L89-L97)宏。
      - 宏对于可读性很有帮助，比如`i18n`里面的[`create_t`](https://github.com/seed-rs/seed/blob/29666287eaf5e914c80e9fae7cc6736cd31ce087/examples/i18n/src/i18n.rs#L90-L131)宏。
      - 很难或不可能通过正确的Rust类型编码所有内容的时候，比如Seed里面像是`div!`之类的元素宏。
      - 它可以帮助你减少样板代码，比如Seed的`log!`宏，它格式化输入的参数然后在后台调用了JavaScript的`console.log`。


译者：虽然就Seed和yew而言我更喜欢yew，但是不妨碍这篇文章写的好啊，同时在本篇文章的后半段还有部分作者对rust的表白话语，看的我差点也想翻译了，手动狗头。。。
    翻译的不是很流畅，可能会出现不少错误，欢迎大家指正。

    