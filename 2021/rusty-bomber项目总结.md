### 前言

[Rusty BomberMan](https://github.com/rgripper/rusty-bomber)是著名的BomberMan小游戏的bevy复刻版。虽然说是复刻，但实际上和原本游戏长得完全不一样，原因是原版游戏的美术资源没搞到，所以另找了一些美术资源，十分感谢[opengameart.org](https://opengameart.org/)上[这些](https://github.com/rgripper/rusty-bomber#assets-and-attribution)美术资源。

### 开发动机

开发这个游戏的起因是当时我正在逛reddit，正好看到了[@rgripper](https://github.com/rgripper)发帖想找人一起写bevy项目，抱着学习、实践的心态，我和他联系之后一拍即合，随即开始了这个项目。

### rust开发环境推介

开发中使用最新版rust（建议nightly版本，bevy官网的快速开发迭代有推介用这个）。

开发环境推介 `vscode` + [`rust-analyzer`](https://github.com/rust-analyzer/rust-analyzer)（建议安装最新发布版，尽量别用nightly版本，我喜欢自己下载源码编译。） + [`Tabline`](https://www.tabnine.com/)（可选），或者`Clion` + [`IntelliJ Rust`](https://www.jetbrains.com/rust/)。
前者可能需要自己折腾，后者开箱即用，不过`Clion`不是免费的。
                

### 编译速度

bevy的官网中有提到编译速度十分快速，其中0.4版本发布的时候，由于添加的动态链接的feature，增量编译的编译速度确实快了不少，但是需要进行一系列的配置。

rust本身的编译速度实在不能说快，甚至有时候让人烦操，但bevy开发迭代过程中，如果配置好快速编译的开发环境，每次增量编译的时间，都在可接受范围之类。

我笔记本的配置是：
```
处理器	Intel(R) Core(TM) i5-8400 CPU @ 2.80GHz   2.81 GHz
机带 RAM	16.0 GB (15.9 GB 可用)
```
在开启动态链接的feature进行编译的情况下，每次增量编译的时间大概2.5秒左右，加入其它大型依赖之后，比如`bevy_rapier`，增量编译的速度会变长，但是仍然在可接受范围内，约3.5秒。5秒是我进行开发迭代的可接受范围，在这次开发过程中，就编译速度而言，体验十分良好。

那么如何搭建一个快速编译的开发环境呢？

官网里有详细的介绍了如何搭建快速开发环境：https://bevyengine.org/learn/book/getting-started/setup/ （在最后的`Enable Fast Compiles (Optional)`部分）

在搭建环境的过程中,可能会出现一些奇怪的问题，比如这个：
```
error: process didn't exit successfully: `target\debug\bevy_salamanders.exe` (exit code: 0xc0000139, STATUS_ENTRYPOINT_NOT_FOUND)
```
解决方法是把该游戏项目下的`.cargo/config.toml`文件中这行改了：
```toml
#before: 
[target.x86_64-pc-windows-msvc]
linker = "rust-lld.exe"
rustflags = ["-Zshare-generics=y"]
#after:
[target.x86_64-pc-windows-msvc]
linker = "rust-lld.exe"
rustflags = ["-Zshare-generics=off"]
```

改了之后如果还有类似的奇怪错误，可以试着把`.cargo`这个文件夹直接删除，只使用动态链接就行，动态链接对编译速度提升是远远大于切换linker的。还有其它奇怪的没法解决的错误的话，那可以去提issue了。

除此之外，每次运行的时候带一个`--features bevy/dynamic`也很麻烦，我喜欢在`cargo.toml`内部添加两个bevy，平时开发的时候注释掉另一个，直到要发布最终版本的时候才替换成另一个，大概像这样：

```toml
bevy = { version="0.4", features = ["dynamic"] }
# bevy = "0.4" 
```

下面的这个平时注释掉，只有当要发布最终版的时候，才把上面的注释掉，切换成下面的这个。平时开发过程中基本是直接`cargo run`就可以了。

### Query filter

Bevy内部提供了不少查询过滤器，0.4版本更新之后也更好用，更易读了。

大致用法如下：
```rust
fn movement_system(
    query:Query<(要查询的组件),(查询的过滤器)>，
    mut example_query:Query<&mut Transform,With<Player>>
){
    for item in query.iter(){
        // 对查询内容进行操作
    }
    for mut transform in example_query.iter_mut() {
        // 就和迭代器一样使用
    }
}
```
常见的过滤器有`With<T>`,`Without<T>`,`Added<T>`,`Changed<T>`,`Mutated<T>`，`Or<T>`，其中`Mutated`是`Added`和`Changed`的集合，也就是说新添加的和改变了的都可以用`Mutated`来查到，而`Added`只查询新添加的组件，`Changed`只查询已经存在的组件中更改过的组件，这里面`Or`又比较特殊，使用其它几个过滤器基本都是减小查询范围，而使用`Or`却可以扩大过滤的范围，比如查询玩家和生物的位置与速度，就可以这样定义查询：
```rust
    Query<(&Transform,&Speed),Or<(With<Player>,With<Creature>)>>
```
查询多于一个组件的时候需要用括号括起来，将多个组件作为一个元组进行参数传递,同样多个过滤器也以元组的形式传参。当然使用到Or，通常会和Option一起使用，比如既想查询玩家和生物的位置和速度，还想专门查询玩家专属的组件，玩家的力量，就可以这样写查询器：
```rust
    Query<(&Transform,&Speed,Option<&PlayerPower>),Or<(With<Player>,With<Creature>)>>
```
这样查询出来的结果带有`PlayerPower`的肯定是玩家，使用惯用的rust方式处理option就可以了。

### QuerySet

当一个`system`中的查询相互冲突时，编译后运行会触发一个`panic`：`xxx has conflicting queries`。这个时候就需要`QuerySet`来帮助我们了。

> 关于心智负担，我个人观点是写这部分代码时，完全不用带着审视的目光去查看所有的查询，只有在发生这种`panic`的时候，再去审视相关代码，将冲突的部分替换成`QuerySet`就可以了，正好对应rust中诸如所有权、生命期等情况。

> Note：关于哪些情况属于查询冲突，其实很好判断，在同一系统，多次可能查到同一结果的查询中，存在对组件的可变引用查询，那这个查询就是冲突的。

比如以下两个查询：
```rust
fn position(
    mut q0: Query<(&Transform, &mut Point)>,
    q1: Query<&Transform, Or<(With<Point>, With<Head>)>>,
){
    ...
}
```
同时查询了`Transform`和`Point`,并且，`q1`很有可能查到`q0`的结果，但是因为重复查询的组件`Transform`没有可变引用，所以这两个查询放在一个系统内，并不会发生冲突。

而以下两个查询：
```rust
fn position(
    mut q0: Query<(&mut Transform, &Point)>,
    q1: Query<&Transform, Or<(With<Point>, With<Head>)>>,
){
    ...
}
```
因为重复查询的组件`Transform`是有可变引用的，所以会发生冲突。

发生查询冲突之后，就是`QuerySet`大展身手的地方了。

考虑以下两个组件：
```rust
pub struct Head;

pub struct Point {
    pub pre: Entity,
}
```
假设我们需要写一个系统，让每一个点的位置根据前一个实体的位置而改变，可以有以下系统：
```rust
fn position(
    mut q0: Query<(&mut Transform, &Point)>,
    q1: Query<&Transform, Or<(With<Point>, With<Head>)>>,
) {
    ...
}
```
我们甚至没有给这个系统实现任何功能，直接添加到`App`中运行的话，就会直接触发查询冲突。

而使用`QuerySet`的话，也十分简单：
```rust
fn position(
    points_query: QuerySet<(
        Query<(&mut Transform, &BodyPoint)>,
        Query<&Transform, Or<(With<BodyPoint>, With<Head>)>>,
    )>,
) {
    ...
}
```
在没有实现任何内容的情况下添加到`App`中运行，能够正常运行。使用起来也十分方便，只需要将之前的查询以元组的形式当作泛型传到`QuerSet`中即可。

那实现具体的内容呢？
如果不使用`QuerySet`我们实现的内容看起来应该是这样的：
```rust
fn position(
    q0: Query<(&mut Transform, &Point)>,
    q1: Query<&Transform, Or<(With<Point>, With<Head>)>>,
) {
    for (mut transform, point) in q0.iter_mut() } {
        if let Ok(pre_transform) = q1.get(point.pre)  {
            *transform = Transform::from_translation(
                pre_transform.translation - Vec3::new(1.0, 1.0, 0.0)
            );
        }
    }
}
```
那么使用`QuerySet`之后，我们的内容应该是这样的：
```rust
fn position(
    mut points_query: QuerySet<(
        Query<(&mut Transform, &BodyPoint)>,
        Query<&Transform, Or<(With<BodyPoint>, With<Head>)>>,
    )>,
) {
    for (mut transform, point) in points_query.q0_mut().iter_mut() {
        if let Ok(pre_transform) = points_query.q1().get(point.pre) {
            *transform = Transform::from_translation(
                pre_transform.translation - Vec3::new(1.0, 1.0, 0.0) * 30.0,
            )
        } else {
            warn!("not find right transform!");
        }
    }  
}
```
我们还没有运行我们的代码，`rust-analyzer`就已经给我们报错了，我们在`q0_mut()`这里将`points_query`的`&mut`引用传了进去，按照借用规则，后续不能再把`points_query`的指针借用出去了，所以在这里我们就需要使用`unsafe`了。

添加`unsafe`之后我们的代码变成这样：
```rust
fn position(
    mut points_query: QuerySet<(
        Query<(&mut Transform, &BodyPoint)>,
        Query<&Transform, Or<(With<BodyPoint>, With<Head>)>>,
    )>,
) {
    // Safety: 一般调用unsafe时，情况复杂的需要写下相关注释
    for (mut transform, point) in unsafe { points_query.q0().iter_unsafe() } {
        if let Ok(pre_transform) = points_query.q1().get(point.pre)  {
            *transform = Transform::from_translation(
                pre_transform.translation - Vec3::new(1.0, 1.0, 0.0) * 30.0,
            )
        } else {
            warn!("not find right transform!");
        }
    }
}
```
bevy几乎所有的`unsafe`都贴心的写出了`Safety`，使用这部分api时的内存安全由使用者来保证，而使用者只需要判断自己的调用情况是否符合`Safety`的要求，就能判断这个调用是否满足内存安全。比如该处的`Safety`要求就是这样的：
> This allows aliased mutability. You must make sure this call does not result in multiple mutable references to the same component

我们已经能够明确，我们的两次查询，不会造成查询结果中，存在同一个组件的多个包含可变引用的引用，所以在这里调用该`unsafe`函数是`Safety`的！

当你把借用的问题处理好之后，再次运行我们的`App`，就一切如你所愿了。

谈谈`QuerySet`的体验，因为`rust-analyzer`对过程宏生成的Api支持不是很友好，对类似由宏生成的Api的代码补全体验可以说是很糟糕。而且出于减少总编译时间的考虑，这部分的过程宏只预备了五个参数的位置，也就说说除了`q0`到`q4`多出`q4`的部分，这个过程宏是没有预先生成相关函数的。当然我相信在实际应用的过程中，很少有出现这么极端的查询情况。总得来说掌握这个Api的使用并不难，而且在生产过程中也很实用。

### Bevy Event

### 如何实现游戏的不同状态

### 该项目中所涉及的值得记录的rapier笔记

### 开发过程中发现的bevy api缺陷

### wasm支持

### 安卓支持

### 使用bevy的开发体验

