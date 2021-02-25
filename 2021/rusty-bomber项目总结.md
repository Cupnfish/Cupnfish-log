### 目录
- [前言](#前言)
- [开发动机](#开发动机)
- [rust开发环境推介](#rust开发环境推介)
- [编译速度](#编译速度)
- [Query filter](#query-filter)
- [QuerySet](#queryset)
- [Event](#event)
- [Timer](#timer)
- [`system`的链接与代码复用](#-system的链接与代码复用)
- [如何实现游戏的不同状态](#如何实现游戏的不同状态)
- [Rapier简短笔记](#rapier简短笔记)
- [通过Rapier来实现碰撞过滤](#通过rapier来实现碰撞过滤)
- [多平台支持](#多平台支持)
- [日志](#日志)
- [碎碎念](#碎碎念)


### 前言

[Rusty BomberMan](https://github.com/rgripper/rusty-bomber)是著名的BomberMan小游戏的bevy复刻版。虽然说是复刻，但实际上和原本游戏长得完全不一样，原因是原版游戏的美术资源没搞到，所以另找了一些美术资源，十分感谢[opengameart.org](https://opengameart.org/)上[这些](https://github.com/rgripper/rusty-bomber#assets-and-attribution)美术资源。

> Changed: 1. 修正了之前刚体类型使用场景 2. 添加了目录，方便直接跳转想要阅读的内容。 3. 末尾加上了本人联系方式。 4. 原`Rapier`部分拆分成两个部分，更方便查阅。 5. 修正部分语句不通顺的地方。

### 开发动机

开发这个游戏的起因是当时我正在逛reddit，正好看到了[@rgripper](https://github.com/rgripper)发帖想找人一起写bevy项目，抱着学习、实践的心态，我和他联系之后一拍即合，随即开始了这个项目。

### rust开发环境推介

开发中使用最新版rust（建议nightly版本，bevy官网的快速开发迭代有推介用这个）。

开发环境推介 `vscode` + [`rust-analyzer`](https://github.com/rust-analyzer/rust-analyzer)（建议安装最新发布版，尽量别用nightly版本，我喜欢自己下载源码编译。） + [`Tabline`](https://www.tabnine.com/)（可选），或者`Clion` + [`IntelliJ Rust`](https://www.jetbrains.com/rust/)。
前者可能需要自己折腾，后者开箱即用，不过`Clion`不是免费的。
                

### 编译速度

bevy的官网中有提到其编译速度很快，其中0.4版本发布的时候，由于添加了动态链接的feature，增量编译的编译速度确实快了几倍，但是需要进行一系列的配置。

rust本身的编译速度实在不能说快，但在使用bevy进行开发迭代过程中，配置好快速编译的开发环境后，增量编译的速度令人十分满意。

我笔记本的配置是：
```
处理器	Intel(R) Core(TM) i5-8400 CPU @ 2.80GHz   2.81 GHz
机带 RAM	16.0 GB (15.9 GB 可用)
```
在开启动态链接的feature进行编译的情况下，每次增量编译的时间大概2.5秒左右，加入其它大型依赖之后，比如`bevy_rapier`，增量编译的速度会变长，但是仍然在可接受范围内，约3.5秒。在这次开发过程中，项目编译速度我很满意，开发体验十分良好。

那么如何搭建一个快速编译的开发环境呢？

官网里有详细的介绍了如何搭建一个快速开发环境：https://bevyengine.org/learn/book/getting-started/setup/ （在最后的`Enable Fast Compiles (Optional)`部分）

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

Bevy内部提供了不少查询过滤器，0.4版本更新之后也更好用，易读性得到了提高。

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
        Query<(&mut Transform, &Point)>,
        Query<&Transform, Or<(With<Point>, With<Head>)>>,
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
        Query<(&mut Transform, &Point)>,
        Query<&Transform, Or<(With<Point>, With<Head>)>>,
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
        Query<(&mut Transform, &Point)>,
        Query<&Transform, Or<(With<Point>, With<Head>)>>,
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

### Event
0.4版本的bevy的`event`有个十分不好用的地方，看以下示例：
```rust
pub fn game_events_handle(
    game_events: Res<Events<GameEvents>>,
    mut events_reader: Local<EventReader<GameEvents>>,
) -> Result<()> {
    ...
}
```
可能只看函数参数并不能感受到哪里不好用，可是如果你注意到这是一个事件处理系统，传递进来的参数居然同时需要`Events`和`EventReader`，并且使用的时候是这样的：
```rust
    for event in events_reader.iter(&game_events) {
        match event {
            ...
        }
    }
```
没错，`EventReader`不是一个真正的迭代器，在调用`iter()`的时候需要传递一个该事件的引用，这在使用的过程中感受到多余。

好在`EventReader`在即将要发布的0.5版本当中已经得到了改善，在这个[PR](https://github.com/bevyengine/bevy/pull/1244)合并之后，`EventReader`的调用已经变成了这样：
```rust
pub fn game_events_handle(
    // 不再需要多余的Events作为EventReader参数
    // game_events: Res<Events<GameEvents>>,
    // mut events_reader: Local<EventReader<GameEvents>>,
    // 不再需要指定Local，EventReader在Bevy中已经变成了更高级别的API
    mut events_reader: EventReader<GameEvents>,
) -> Result<()> {
    // 变得更像真实的迭代器
    for event in events_reader.iter() {
        match event {
            ...
        }
    }
}
```
需要注意的是，不仅仅是`EventReader`变成了更高级别的API（即成为真正的系统参数），`Events`也同样不再需要在其外部套一个`ResMut`了，写系统时直接写`Events<T>`作为参数。

> 可以这样改动的原因：之前的`Events`是作为`Resource`使用的，也就是说存在`Res`、`ResMut`两种状态。其中`Res<Events<T>>`只有给旧版的`EventReader`当作参数的存在意义，但是新版的`EventReader`已经不再需要这个参数，`Res`版本的`Events`失去了其存在意义，因此相对于`ResMut<Events<T>>`，索性改成了`Events<T>`，减少了用户API层面的复杂性。
### Timer
bevy现版本的`Timer`是个值得争议的地方，先来看看具体用法：
```rust
// 定义一个动画计时器组件：
pub struct Animation(pub Timer);
// 作为Player实体的组件添加到Player中：
#[derive(Bundle)]// 使用Bundel派生宏可以将多个组件打包到一块，bevy官方指南也推介这样做，性能上似乎也比直接使用with更好
pub struct PlayerBundle {
    player: Player,
    animation: Animation,
    ...//省略了其它组件
}
// 初始化PlayerBundle
impl Default for PlayerBundle {
    fn default() -> Self {
        Self {
            player: Player,
            animation: Animation(Timer::from_seconds(0.3, true)),
            ...//省略了其它组件
        }
    }
}
// Timer 在实例化的时候需要提供两个参数，一个是计时器计时的时间，另一个是该计时器是否重复计时。
// 查询计时器进行相关修改：
fn player_animation(
    time: Res<Time>,// 使用计时器时必须用到时间去tick计时器
    mut query: Query<(&mut Animation,&Player)>,
) {
    for (mut animation,player) in query.iter_mut(){
        animation.0.tick(time.delta_seconds());
        // animation.0是因为我们将Timer包裹在了Animation下
        if animation.0.just_finished() {
            ...// 相关操作
        }
    }
}
```
以上基本就是计时器在使用时的流程，现在来回答几个问题。

- 为什么要使用一个结构体去包裹已有的计时器？
> 大家应该注意到我们没有直接将计时器作为组件附加到`Player`上，而是通过一个结构体去包裹计时器之后再附加到`Player`上，这样做的其中一个原因是我们的`Player`实体可能需要不止一个计时器，所以我们需要给每个计时器不同的标识。
- 为什么在调用计时器的`finished()`等相关计时API之前需要先调用`tick(time.delta_seconds())`?
> bevy的计时器本身相当于一个保存有当前时间量的结构体，本身没有时间流动的概念，只有tick的时候告诉它已经过去了多少时间，它才会把过去了多少时间加到它本身保存的状态上。

`Timer`比较有争议的地方就是使用计时器时不能十分容易的给它添加标识，需要在计时器外部套一个结构体，目前有些[PR](https://github.com/bevyengine/bevy/pull/1151)提出了给`Timer`增加一个泛型的位置的想法，我个人不是很喜欢这种实现，理由很多，比如`@cart`大大的理由就是bvey内部有不少不需要特殊标识的计时器，如果添加泛型之后需要这样写：`Timer<()>`，相对于之前的`Timer`来说，实在是太丑了。

出了标识的问题，还有目前的计时器使用的`f32`类型，应该替换成时间更常用的`Duration`，刚刚提到的PR在这个方面就已经完成了。

### `system`的链接与代码复用

之前`Events`部分有个系统例子和其它常规例子不一样：
```rust
pub fn game_events_handle(
    mut events_reader: EventReader<GameEvents>,
) -> Result<()> {
    ...
}
```
它拥有一个`Result`返回值，如果直接将这个系统添加到`App`中，会被`rust-analyzer`直接报错，因为bevy不支持带有返回值的系统。

那如何让带有返回值的系统添加到`App`中去呢？当然是处理掉它的返回值，bevy给我们提供了一个`fn chain(self, system: SystemB)`函数，调用的时候大概像下面这样：
```rust
    .add_system(game_events_handle.system().chain(error_handler.system()))
```
它可以‘无限续杯’，只要你愿意，你可以无限`chain`下去。

那如何写一个可以`chain`的系统呢？考虑以下系统
```rust
pub fn head_translation(query: Query<&Transform, With<Head>>) -> Option<Vec3> {
    query.iter().map(|transform| transform.translation).next()
}
```
该系统返回一个`Option<Vec3>`，因此能够处理该返回值的系统应该要带有一个`In<Option<Vec3>>`的参数：
```rust
pub fn head_translation_handle(come_in: In<Option<Vec3>>) -> Option<Vec3> {
    if let In(Some(vec)) = come_in {
        Some(vec + Vec3::new(1.0, 1.0, 0.0) * 30.0)
    } else {
        None
    }
}
```
出于教学目的，这里没有直接处理本不需要再返回出去的`Option<Vec3>`，而是为了验证多次链接是否有用：
```rust
pub fn body_point_translation_handle(
    come_in: In<Option<Vec3>>,
    mut query: Query<&mut Transform, With<BodyPoint>>,
) {
    if let In(Some(vec)) = come_in {
        for mut transform in query.iter_mut() {
            transform.translation = vec;
        }
    }
}
```
没错，在每次链接的时候，你可以添加新的参数，这种设计大大增加代码的灵活性，同时也提高了代码复用率。

这是bevy中我很喜欢的一个功能，既实用又灵活。虽然在本次项目中用到的地方不多，基本都用来做错误处理了，但是我相信在一个大型项目中，这种功能够充分发挥出它的优势，大概就是bevy中各处都彰显着类似这样设计的人体工程学，因此大家才为之感到兴奋。

当然上面的代码可能有些地方让有强迫症的人感到不适，比如传出来的结果为啥是`Option`的，这样如果这个系统返回`None`的时候仍然一直在游戏中运行会不会很占资源？确实是会有这方面的考虑，所以现在已经有[PR](https://github.com/bevyengine/bevy/pull/1393)提出了异步系统的概念，如果真的实现出来的话，应该来大大减缓这种情况，编写出来的代码估计也会好看一些。

### 如何实现游戏的不同状态

我们的项目中实现了一个完整的游戏流程，包括开始游戏的菜单界面，游戏内部的暂停，玩家被炸弹炸死或者被生物触碰时的失败，以及玩家找到下一关的入口之后的胜利。如果有体验过我们的游戏，会发现关卡基本没有设计，仅仅只是实现了游戏中各种道具的效果，包括第一关与第二关的区别也仅仅是多了几只怪。作为游戏而言，我是对这部分的实现是很不满意的，但是作为体验、学习bevy而言，我觉得收获良多。我甚至还保留了一个随机的关卡实现接口，只不过没有真的去实现，roguelike的相关算法此前我都没有什么经验，只希望下一个项目能够在这方面得到提升。

回到正题，为了实现这样一个完整的游戏流程，我参考了[Kataster](https://github.com/Bobox214/Kataster)的相关代码，将游戏整体流程放在了`AppState`这个枚举体内：
```rust
pub enum AppState {
    StartMenu,
    Game,
    Temporary,
}
```

看上去我们的游戏有`StartMenu`、`Game`、`Temporary`三个状态，实际上只需要考虑前两个状态就好了，`Temporaty`这个状态只是为了方便修复游戏中的一个小bug而已。

通常构建一个游戏的状态需要以下四个步骤：

1.将我们的游戏状态以资源的方式添加到游戏中：
```rust
app.add_resource(State::new(AppState::StartMenu))
// 添加游戏状态资源时，需要特意指明初始化的状态，比如这里就指明了创建好的状态加载到游戏开始菜单的状态下
```
2.初始化StateStage
```rust
// 接上第一步的部分
    .add_stage_after(// 此处也很灵活，可以按照自己的喜好来
        stage::UPDATE,// target，你可以把你的状态放到你想放的任何已有状态下
        APP_STATE_STAGE,// name，名字也很灵活，可以自己取，这里是const APP_STATE_STAGE: &str = "app_state";
        StateStage::<AppState>::default(),// 这里就挺固定了，需要将你的游戏状态枚举作为StateStage的一个泛型，以便初始化。
    )
```
3.处理stage
```rust
// 紧接上一步
    .stage(APP_STATE_STAGE, |stage: &mut StateStage<AppState>| {
        // 通过这个闭包，可以给我们游戏的不同状态添加系统
        stage
            // start menu
            // on_state_enter用来设置进入该State时调用的系统，通常用来加载资源。
            .on_state_enter(AppState::StartMenu, start_menu.system())
            // on_state_update用来设置该State下游戏更新时调用的系统。
            .on_state_update(AppState::StartMenu, button_system.system())
            // on_state_exit用来设置退出该State时调用的系统，通常用来清楚屏幕，更新相关游戏数据之类的。
            .on_state_exit(AppState::StartMenu, exit_ui_despawn.system())
            // in game
            .on_state_enter(AppState::Game, setup_map.system()))
            // 类似于on_state_update，不过可以同时设置多个。
            .update_stage(AppState::Game, |stage: &mut SystemStage| {
                stage
                // 以下的方法都不是SystemStage自带的，而是在我们游戏项目的各个模块下通过自定义trait给SystemStage实现的，只是为了方便管理各个模块。
                // 这部分设计是有缺陷的，一般来说physics系统中的其中一部分是需要提前加载的，不然会造成现版本中出现查询错误的小bug
                    .physics_systems()
                    .player_systems()
                    .bomb_systems()
                    .buff_systems()
                    .creature_systems()
                    .portal_systems()
            })
            .on_state_exit(AppState::Game, exit_game_despawn.system())
            .on_state_enter(AppState::Temporary, jump_game.system())
    });
```
4.处理游戏状态跳转
```rust
// 另外构建一个处理游戏状态的跳转的系统
pub fn jump_state(
    mut app_state: ResMut<State<AppState>>,
    input: Res<Input<KeyCode>>,
    mut app_exit_events: ResMut<Events<AppExit>>,
) -> Result<()> {
    // 使用模式匹配能够很清晰的将我们游戏状态跳转进行处理
    match app_state.current() {
        AppState::StartMenu => {
            if input.just_pressed(KeyCode::Return) {
                // set_next这个方法就是从当前状态跳转到指定状态
                app_state.set_next(AppState::Game)?;
                // game_state是原来处理游戏状态下的各种状态的，比如暂停、胜利、失败等，和app_state大同小异，因此此处都省略了，如果感兴趣可以直接看这部分源码，放到了src/events下
                // game_state.set_next(GameState::Game)?;
            }
            if input.just_pressed(KeyCode::Escape) {
                // 这个事件是bevy内置的事件，用来退出应用
                app_exit_events.send(AppExit);
            }
        }
        AppState::Game => {
            if input.just_pressed(KeyCode::Back) {
                app_state.set_next(AppState::StartMenu)?;
                // game_state.set_next(GameState::Invalid)?;
                map.init();
            }
        }
        AppState::Temporary => {}
    }
    Ok(())
}
```
通过以上四个步骤，就能够为你的游戏添加上不同的状态，现在我们来谈一下第三步，其实这部分很有可能在之后的版本中被[新的调度器](https://github.com/bevyengine/bevy/pull/1144)取代，但那还是久远之后的事，到那时需要新的blog去探讨。

### Rapier简短笔记

`rapier`作为物理引擎，它的内容十分丰富，本项目所涉及的内容，仅仅是其中的一小部分，本文也只是从中挑出了一些有意义的进行记录。如果想要深入学习`rapier`，我的建议是先看[官方文档](https://rapier.rs/docs/user_guides/rust/getting_started)，然后再去[discord](https://discord.gg/VuvMUaxh)的`bevy_rapier`群组去交流学习。

rapier的常用组件有两个，一个是刚体（RigidBody），一个是碰撞体（Collider）。bevy中的每一个实体，只能有一个刚体，而碰撞体可以有多个，比如角色的头、胳膊、腿，这些部分都可以使用单独一个碰撞体来表示。

创建刚体的方法很简单：
```rust
// 创建一个运动学刚体，不受外部力影响，但是能单向影响动态刚体，需要通过专门设置其位置，常用于移动平台，如电梯
RigidBodyBuilder::new_kinematic()
.translation(translation_x, translation_y)
// 创建一个静态刚体，不受任何外部力的影响，常用于墙体等静态物体
RigidBodyBuilder::new_static()
.translation(translation_x, translation_y)        
// 创建一个动态刚体，受外部力的影响，常用于玩家控制的角色、游戏中的怪物等
RigidBodyBuilder::new_dynamic()        
.translation(translation_x, translation_y)        
.lock_rotations()// （可选）让刚体锁定旋转    
.lock_translations()// （可选）让刚体锁定位置
```
> 创建刚体时需要明确指定其位置，因为`bevy_rapier`内部有一个系统专门用于转换刚体的位置和实体的`Transform`，相当于我们不再需要去管理实体中的`Transform`，只需要通过刚体来管理该实体的速度、位置、旋转、受力等就可以。

创建碰撞体的方法也很简单：
```rust
// 碰撞体实际上就是定义参与碰撞计算的形状，rapier有多种选择，因为我们的游戏项目中只用到两种，所以只谈这两类
// 矩形，设置的时候需要提供它的半高和半宽
ColliderBuilder::cuboid(hx, hy)
// 圆形，设置的时候需要提供半径
ColliderBuilder::ball(radius)
```
> note：矩形碰撞体构建需要提供的参数是半高和半宽，而不是整高和整宽。

对于单一碰撞体的直接讲刚体和碰撞体作为组件插入到已有实体即可：
```rust
fn for_player_add_collision_detection(
    commands: &mut Commands,
    query: Query<
        (Entity, &Transform),
        (
            With<Player>,
            Without<RigidBodyBuilder>,
            Without<ColliderBuilder>,
            Without<RigidBodyHandleComponent>,
            Without<ColliderHandleComponent>,
        ),
    >,
) {
    for (entity, transform) in query.iter() {
        let translation = transform.translation;
        commands.insert(
            entity,
            (
                create_dyn_rigid_body(translation.x, translation.y),
                create_player_collider(entity),
            ),
        );
    }
}
```
> 如果只是单个碰撞体和刚体的组合，则用这种方法插入即可，但如果是多个碰撞体和单个刚体的组合，则稍微有所不同，详情可以看[这里](https://github.com/dimforge/bevy_rapier/blob/master/bevy_rapier2d/examples/multiple_colliders2.rs)。

我们的游戏当中使用的是动态加载，也就是在所有地图资源加载之后，再给没有加上刚体和碰撞体的实体插入相应的刚体和碰撞体。

比如上面给出的例子，可能大家会对查询的过滤器感到奇怪。因为我们是给没有刚体构建器和碰撞体构建器的实体插入刚体和碰撞体，所以再过滤器中有` Without<RigidBodyBuilder>`和`Without<ColliderBuilder>`并不让人奇怪。让人奇怪的地方是后两条过滤器`Without<RigidBodyHandleComponent>`和`Without<ColliderHandleComponent>`，这两条实际上是因为`bevy_rapier`内部有一个负责转换构建器（`Builder`）到句柄组件（`HandleComponent`）的系统，当我们给实体插入构建器之后，该系统就会通过一些内部的方法将其转换为句柄组件。所以为了防止我们查询到的结果当中存在已经插入过句柄组件的实体，所以需要再加入这条过滤。


仅仅添加这些并不足以让物理引擎在我们的游戏里面运行起来，主要原因是现在的`bevy_rapier`仍然是作为一个外部crate引入到我们的游戏项目中，在将来如果集成到了`bevy`主体的物理引擎中，则不再需要以下操作。
```rust
// 在app中添加物理引擎插件
    app
    ...// 初始化其它资源和添加其它插件
        .add_plugin(RapierPhysicsPlugin)
```
这样简单设置之后，我们的游戏中就成功的启用了物理引擎。

### 通过Rapier来实现碰撞过滤

还有一件事需要特别记录一下，在我们的游戏中，生物是可以互相碰撞的，那么如何实现这种效果呢？只需要在创建碰撞器的时候指明解算组或者碰撞组即可。

```rust
    ColliderBuilder::cuboid(HALF_TILE_WIDTH, HALF_TILE_WIDTH)
        // 用户数据，可以插入一些自定义的数据，但是只能以u128格式插入，通常用来插入实体，有了实体之后可以通过查询来获取该实体的其它组件
        .user_data(entity.to_bits() as u128)
        // 解算组，可以通过设定一个交互组（InteractionGroups）来让该碰撞器在该组规则下进行力的解算
        .solver_groups(InteractionGroups::new(WAY_GROUPS, NONE_GROUPS))
        // 碰撞组，同样设定交互组之后，让该碰撞器在该组规则下进行碰撞解算
        .collision_groups(InteractionGroups::new(WAY_GROUPS, NONE_GROUPS))
```
在更进一步谈论解算组和碰撞组的区别之前，我们需要了解交互组的构建规则，交互组`new`的时候需要提供两个参数，第一个参数是设定该碰撞体属于哪一组，需要的参数类型是一个`u16`，第二个参数是设定该碰撞体和哪些组的碰撞体会产生交互，参数同样是一个`u16`。

对于第二个参数，设定和单个碰撞体交互倒是挺好理解，但如果设定和多个碰撞体交互又该怎么设置呢？这正是参数的类型设定为`u16`的妙处，举个例子：
```rust
const CREATURE_GROUPS: u16 = 0b0010;
const PLAYER_GROUPS: u16 = 0b0001;
const WALL_GROUPS: u16 = 0b0100;
const WAY_GROUPS: u16 = 0b1000;
const NONE_GROUPS: u16 = 0b0000;
```
以上常量皆是我们这次游戏中用到的交互组变量，而`0b0011`表示的就是生物组和玩家组两个组，而这个数就是用`CREATURE_GROUPS`和`PLAYER_GROUPS`通过`&`运算出来的。

至于解算组和碰撞组的区别，解算组解算的就是受力状况，与之交互的组都会参与到受力解算中。而碰撞组是管理碰撞事件的，碰撞事件可以通过`Res<EventQueue>`进行接收处理。

还有`user_data`也是一个比较常用的，通常是在碰撞体插入的时候将该实体传入到碰撞体构建器当中，通过这个数据，可以使用以下命令获得实体：
```rust
let entity = Entity::from_bits(user_data as u64);
```
那`user_data`又从哪里来呢？从碰撞事件中我们会获得一个索引，该索引可以通过`Res<ColliderSet>`的get方法获取器`user_data`，这方面比较繁琐，也是我认为目前`bevy_rapier`当中最不好用的部分。

除此之外，如果你就此运行你的游戏，你会发现你的角色也好，画面中的其它动态刚体，除了你设定的之外，还会收到一个重力，这完全不符合你俯视2d游戏的初衷，所以我们需要将该重力给修改为零。

当前版本是通过添加这样一个系统来修改物理引擎的重力的：
```rust
fn setup(
    mut configuration: ResMut<RapierConfiguration>,
) {
    configuration.gravity = Vector::y() * 0.0;
}
```
将这个系统添加到`startup_system()`只需要在每次游戏启动之前运行一次就行。

### 多平台支持
我们的游戏这次除了支持正常的桌面端平台以外，还做了`wasm`的支持，其中因为`bevy`的声音在`wasm`没有得到支持继而没有实现声音以外，总算是没什么遗憾。做完游戏之后发给小伙伴们玩了一下，都在问我有没有手机版本的。`bevy`的支持计划里面是有移动端的，而且就从桌面端迁移到移动端上要做出的改变来说是很少的，再说我们尚未支持的移动端之前，来看看我们是如何支持`wasm`版本的。

`bevy`的渲染后端用的是`wgpu`，虽然原生的`wgpu`渲染后端已经支持编译到`wasm`了，但是由于某些原因居然没有给`bevy`实装上，我们能够参考的已有的`bevy`的`wasm`版本项目基本上都是基于`bevy_webgl2`这个crate。

添加`wasm`支持也十分方便，除了需要添加常规的html之类的文件，还需要做如下改动：
```rust
// 添加webgl2的插件，添加这个插件之前需要关闭bevy的wgpu的feature
    #[cfg(target_arch = "wasm32")]
    app.add_plugins(bevy_webgl2::DefaultPlugins);
    #[cfg(not(target_arch = "wasm32"))]
    app.add_plugins(DefaultPlugins);
```
关闭wgpu的feature：
```toml
[features]
# 这部分是native和wasm都会用到的bevy的feature
default = [
  "bevy/bevy_gltf",
  "bevy/bevy_winit",
  "bevy/bevy_gilrs",
  "bevy/render",
  "bevy/png",
]
# 这部分是native会用到的wgpu的feature
native = [
  "bevy/bevy_wgpu",
  "bevy/dynamic"# （可选，开发的时候提高增量编译速度，编译真的十分快！）
]
# 这部分是wasm支持会用到的webgl2的feature
web = [
  "bevy_webgl2"
]
```
基本上这样就设置好了，其余的设置是跟html有关的，需要稍微丢丢的wasm开发的知识。关于编译的时候用到的`cargo make`等工具链如何使用，同样是在那一丢丢的wams开发的知识里面学习。关于如何部署到github的page服务上，这个我是完全不会的，我们游戏的这部分部署是有我的搭档`@rgripper`完成的。

对于移动端的支持，以安卓为例，如果不考虑触屏啊，按钮之类的，官方其实给了示例的，在桌面端的基础上迁移起来也十分方便。除了基本的安卓开发环境的搭配（这部分可以详情看[`cargo mobile`](https://github.com/BrainiumLLC/cargo-mobile)的READEME里面讲的十分详情），只需要做出下面这种改动，即可支持移动端，甚至如果以后修复了wgpu对wasm端的支持，应该同样也只是需要下面这种修改，即可对多端支持：
```rust
// 对，就是添加这个过程宏之后，编译的时候使用对应平台的编译指令即可打包到相应平台
#[bevy_main]
fn main() {
    App::build()
        .insert_resource(Msaa { samples: 2 })
        .add_plugins(DefaultPlugins)
        .add_startup_system(setup.system())
        .run();
}
```
### 日志
bevy内建了日志系统，使用起来也十分方便，同时也能和rust生态中的其它日志crate配合在一起使用，对于后续测试和收集数据有很重要的作用。

这次项目中我们并没有深入使用日志功能，也没有和外部的日志crate深度结合使用，只是当作`println!`调试的时候用，所以这部分就不再探讨。

### 碎碎念

这是本文的最后一个部分，也是谈谈开发下来的一些感受，上面基本是干货居多，感受这种东西并不是每个人的愿意看，所以也不愿意放在前面叨扰大家。总得来说做完整个项目总结之后，发现自己之前走了不少弯路，甚至有些地方都用错了（比如前几个版本中的切换游戏状态，受参考的源代码影响也用了一堆if-else，当时自己看的时候也是一头雾水的，改成match之后清晰明了），在这个项目之前，rust对于我来说只是刷题、刷教程趁手的工具，虽然学到了不少的知识，但总觉得缺乏自己的实践。但这样一趟走下来，实践经验确实增长不少，最重要的是还交到了`@rgripper`这样的好朋友，果然github是个大型在线交友平台，哈哈哈。

使用`bevy`的开发体验在我这里被区分为两个部分，但总得来说是十分有趣的。

而这个分界点就是在游戏里加入[rapier](https://rapier.rs/)前后，加入之前和加入之后是两种完全不同的开发体验。

其中最主要原因还是因为自己之前没有使用过物理引擎，有不少生涩的词汇在开发中需要接触和学习，加上`bevy_rapier`当中不少接口放到`bevy`实际开发中体验并不良好，所以造成了使用`rapier`之后开发速率下降、开发心情糟糕等情况。

当然对于最终我们的游戏中使用了`rapeir`这件事，我觉得是很值得的，在这样一个小游戏中使用物理引擎这件事并不值得。但如果是为了学习这个物理引擎，那就是值得的，而且也确实涨了不少知识（在这部分真的十分感谢`rapier`的作者[@Sébastien Crozet](https://github.com/sebcrozet)，在他的discord群组里，基本上大家问的问题都得到了解决，也很感谢群组里帮助我们提出思路的各个网友）。

谈一下本次开发中的遗憾，游戏没有加入音频算一个遗憾，这部分的工作早先是由我的搭档去完成的，但是因为bevy的[一些原因](https://github.com/bevyengine/bevy/issues/88#issuecomment-753546363)，导致音频部分对wasm支持很差，所以我们放弃了。地图没有细致的去设计以及没有随机地图的支持这算两个遗憾。小怪的ai因为我们连个人此前都没写过游戏，因此对这方面不熟悉，导致有时候小怪会傻傻站着，和卡了bug一样，这也算一个。在游戏基本写完的时候[`bevy_tilemap`](https://github.com/joshuajbouw/bevy_tilemap)发布了，并且还有一个游戏动图，我们没能在一个网格游戏当中用到这种crate，也算是一个遗憾。游戏的资产加载没有专门做成一个状态，导致在网络差的情况下，网页版的游戏很有可能出现这个[issue](https://github.com/rgripper/rusty-bomber/issues/16)所说的游戏主体出现了但是游戏资产没有加载进来的诡异情况，这也算是一个遗憾。

> 关于我本人，目前青岛某大学大四在校生一名。大二的时候因为自己主力语言是python和C#（后面上课还学了Java，虽然很早之前就学过C，但不是很喜欢，刚接触指针的时候可懵逼了），所以很想学一门底层语言，当时看知乎不少关于Rust的讨论，对Rust产生了一些兴趣，恰好18年初张汉东老师的`Rust编程之道`正好上架，下单之后随即入坑Rust。2020年初疫情期间GAMES101课程在B站有录播，通过闫令琪老师的课程算是入门计算机图形学，同时期学了Wgpu，很想以后工作能从事 Rust 游戏开发，不过目前看来社区还得发展两三年。知乎上有不少人对Rust图形化编程方面呈悲观态势，起初只有Amethyst的时候我确实也很同意他们的观点，但是bevy给了rust社区中很多人希望，bevy不仅仅是想用Rust来做游戏引擎，同时也在鼓励使用Rust来编写游戏，这是区别于Amethyst等游戏引擎的，同时我想说，就目前bevy的ECS部分的Api来看，bevy做到了！这是梦想中的Rust，你几乎很少会用到生命期之类的Rust中一切繁琐的东西，bevy带给你的Rust开发体验是前所未有的，当然现在它仍然还很弱小，需要大家的呵护、照顾，它有很大的潜力，但同时也需要社区进行各方面的支持。

你可以通过以下方式联系到我，无论是进行技术讨论，还是项目合作，都可以直接和我联系：

邮箱：pointu@foxmail.com
QQ：760280519
微信：BXQC7610