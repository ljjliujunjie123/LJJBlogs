> 本系列是我学习compose过程中，对官方文档的翻译和解读，以及实验性的Demo工程。主要参考官方文档和中文手册
>
> 全部的正文内容（Demo工程除外）源自Compose官方文档，个人解读以引用的形式插入。
>
> Compose 官方文档 https://developer.android.google.cn/jetpack/compose
>
> Compose 中文手册 https://compose.net.cn/
>
> 本文翻译内容 https://developer.android.google.cn/jetpack/compose/lifecycle

## 生命周期的概述

`Composition`（翻译成**绘制**）是运行可组合函数后生成的结果，它是一个树状的结构，描述着`app`的各个`UI`

当`Jetpack`第一次运行可组合函数时，也叫作`initial composition`，`Jetpack`会跟踪你所调用的所有可组合函数。然后，当`app`的状态改变时，`Jetpack`会触发一次`recomposition`（翻译成**重绘**）。重绘是指随着状态的改变，那些把该状态作为入参的可组合函数，会被`Jetpack`重新运行一遍。任何状态的改变都是通过重绘的方式反映到`UI`上的。

`Composition`只能通过`initial composition`和`recomposition`来生成。而改变一个已经存在的`composition`只能通过`recomposition`

<img src="https://developer.android.google.cn/images/jetpack/compose/lifecycle-composition.png" alt="Diagram showing the lifecycle of a composable" style="zoom: 33%;" />

如图，一个可组合函数的生命周期分三部分，分别是`initial composition`时进入一个`composition`，然后是随着状态的改变`recomposition`0次或多次，最后是离开这个`composition`

重绘往往是通过`State`对象的变化触发的。`Jetpack Compose`会跟踪这些`State`对象，当它们改变时，把它们作为入参的可组合函数们也被重新执行。

如果一个可组合函数被重复调用了多次，那么在该`Composition`中会创建多个实例，每一个实例有独立的生命周期

```kotlin
@Composable
fun MyComposable() {
    Column {
        Text("Hello")
        Text("World")
    }
}
```

<img src="https://developer.android.google.cn/images/jetpack/compose/lifecycle-hierarchy.png" alt="Diagram showing the hierarchical arrangement of the elements in the previous code snippet" style="zoom:18%;" />

> Compose中每一个UI组件的生命周期被大大简化了，而不是像传统View中的一系列过程
>
> 所以我们编码时，只需要关注三个点。一是UI的初始化时机，二是UI的重绘时机，三是UI的Remove时机

## 对Composition中一个Composable的解析

`Composition`对每一个`Composable`的实例分配一个独一无二的标识，这个标识由该`Composable`的调用点决定，称为`call site`。在不同的`call site`调用同一个可组合函数，会创建多个不同的`Composable`实例

> `call site`就是源代码中可组合函数的调用位置

如果一个可组合函数发生了重绘，且重绘前后的两次绘制中，调用了不同的子可组合函数，`Compose`会根据`call site`标记出哪些是两次都调用了的，哪些是只有第二次才调用的。对于前者，如果它们的入参没有改变，那就跳过。对于后者，则是`initial composition`。如下例

```kotlin
//假设第一次showError是False
//第二次showError是True
@Composable
fun LoginScreen(showError: Boolean) {
    if (showError) {
        LoginError()
    }
    LoginInput() 
}

@Composable
fun LoginInput() { /* ... */ }
```

`LoginScreen`发生重绘前后，都会调用`LoginInput`，但只有第二次才会调用`LoginError` 。`LoginInput`和`LoginError`有各自的`call site`作为在`composition`中的标识

<img src="https://developer.android.google.cn/images/jetpack/compose/lifecycle-showerror.png" alt="Diagram showing how the preceding code is recomposed if the showError flag is changed to true. The LoginError composable is added, but the other composables are not recomposed." style="zoom: 25%;" />

图中`LoginInput`颜色相同表示没有重绘。尽管第一次绘制时，`LoginInput`的调用顺序是首位，第二次绘制时，变成了第二位，但在`Composition`眼里，它的`call site`始终没变。同时，`LoginInput`的入参也没变，所以它就被跳过了重绘

### 通过附加信息实现更智能的重绘

多次调用同一个可组合函数会向`Composition`中添加多个实例。但如果多次调用时的`call site`都相同，`Compose`就无法通过`call site`来区分这些实例。因此`Compose`会默认引入执行顺序作为附加信息，来区分这些实例。这种默认行为有些时候符合预期，但有些时候不符合预期。

```kotlin
@Composable
fun MoviesScreen(movies: List<Movie>) {
    Column {
        for (movie in movies) {
            //这里的MovieOverview的call site是固定的
            //区分多个实例的方法是for循环的执行顺序
            MovieOverview(movie)
        }
    }
}
```

上例中`Compose`使用执行顺序作为附加信息来区分多个实例。如果一个新的`movie`被添加到`movies`的尾部，那么重绘时，`Compose`可以复用之前的实例，因为它们的执行顺序没有改变。

![Diagram showing how the preceding code is recomposed if a new element is added to the bottom of the list. The other items in the list have not changed position, and are not recomposed.](https://developer.android.google.cn/images/jetpack/compose/lifecycle-newelement-bottom.png)

但是，如果 `movies` 在其他位置插入或删除了`movie`，这会导致所有入参改变的`MovieOverview`发生重绘。这看起来问题不大，不过是多重绘了几个组件而已，但事实上某些情况下会导致严重的性能问题。

例如，`MovieOverview`需要使用`side effect`去请求一张图片。那么如果重绘发生时，`effect`正在进行，那么这个`effect`就会被重启。

> 关于side effect请见下一篇文章。这里姑且理解为开了一个后台线程，去请求网络资源，如果重绘发生时还没有请求成功，那么就会重启这个请求。众所周知，建立一个网络连接是耗时的，我们肯定不希望反复建立网络连接

```kotlin
@Composable
fun MovieOverview(movie: Movie) {
    Column {
        val image = loadNetworkImage(movie.url)
        MovieHeader(image)
    }
}
```

![Diagram showing how the preceding code is recomposed if a new element is added to the top of the list. Every other item in the list changes position and has to be recomposed.](https://developer.android.google.cn/images/jetpack/compose/lifecycle-newelement-top-all-recompose.png)

本次重绘时`MovieOverview` 无法被复用，不同的颜色表示它们发生了重绘

理论上，我们希望把`MovieOverview`的实例和传入的`movie`绑定起来，也就是说当插入一个新的`movie`时，我们希望仅仅是在`Composition`的树状结构中调整这些实例的位置，而不是重绘这些实例。`Compose`提供了一个`key composable`的函数，能允许我们实现这一想法

通过用`key`函数包裹住代码块，同时传入一个或多个参数作为标识，这些参数就会与代码块下的实例绑定起来。传入的参数没必要是全局下都不同的，只需要在该`call site`的局部作用域下是各不相同的就行。例如，每一个`movie`需要一个独一无二的`id`作为`key`的入参，只需要保证在`for`循环下的局部作用域中，这些`id`各不相同即可。

```kotlin
@Composable
fun MoviesScreen(movies: List<Movie>) {
    Column {
        for (movie in movies) {
            key(movie.id) { // Unique ID for this movie
                MovieOverview(movie)
            }
        }
    }
}
```

使用上述代码，当`movies`列表改变时，`Compose`能复用那些已经绘制了的实例

![Diagram showing how the preceding code is recomposed if a new element is added to the top of the list. Because the list items are identified by keys, Compose knows not to recompose them, even though their positions have changed.](https://developer.android.google.cn/images/jetpack/compose/lifecycle-newelement-top-keys.png)

有一些`UI`组件已经内置了这种功能，比如`LazyColumn`可以直接传入一个lambda表达式作为`key`参数

```kotlin
@Composable
fun MoviesScreen(movies: List<Movie>) {
    LazyColumn {
        items(movies, key = { movie -> movie.id }) { movie ->
            MovieOverview(movie)
        }
    }
}
```

> 这一部分的核心思想是，可组合函数与实际的UI实例之间，是一对多的关系
>
> Compose优先通过Call Site来区分这些实例，如果Call Site相同，就用执行顺序来区分
>
> 如果你希望UI实例能和数据一一对应起来，使用Key方法，能给UI实例绑定自定义的标识符

### 入参不变跳过重绘

这个概念提了很多遍了，但什么叫入参不变呢？`Compose`规定入参不变是指，入参的类型是稳定的，且值没有发生改变。

所谓类型是稳定的，需要满足下面三个准则：

- 对于两个相同的该类型的实例，`equals`的结果永远相等
- 该类型实例的公共属性发生改变时，`Compose`能接收到通知
- 该类型的所有公共属性的类型也是稳定的

`Compose`默认的稳定类型是`State`类型，如果你希望把自定义类型的实例传入可组合函数作为稳定的入参，那么需要声明`@State`注解。但有一些常用的类型，虽然没有声明`@State`注解，也被`Compose`视为稳定的类型

- 所有基本类型 `Boolean`, `Int`, `Long`, `Float`, `Char`, etc.
- Strings
- All Function types (lambdas)

这些类型之所以是稳定的，是因为它们都是不可变的。不可变的类型实例就不需要通知`Compose`发生了状态改变，所有它们当然是稳定的

有一种特殊类型是`Compose`的 `MutableState` 类型，虽然是可变的，但也被视为稳定类型。如果通过`MutableState`保存一个值，那么这个对象被视为稳定的，因为这个值的任何改变都会通知给`Compose`

如果一个可组合函数的所有入参都是稳定类型的，那么`Compose`在重绘时就会根据UI树中保存的数据进行值的比较，如果重绘前后的所有入参的值都相等，就跳过重绘。这里的比较其实就是调用`equals`方法

只有当类型能够证明它是稳定的时候，`Compose`才会认为它是稳定的。例如，接口通常被视为不稳定的，具有可变公共属性(即便其实例被声明为`val`不可变)的类型也被视为不稳定

如果`Compose`不能推断出一个类型是稳定的，但你想强迫它认同，可以使用注解 [`@Stable`](https://developer.android.google.cn/reference/kotlin/androidx/compose/runtime/Stable)

```kotlin
@Stable
interface UiState<T : Result<T>> {
    val value: T?
    val exception: Throwable?

    val hasError: Boolean
        get() = exception != null
}
```

这个例子中，因为`UiState` 是个接口类型，`Compose`默认认为所有接口类型都是不稳定的。但通过添加注解，你可以强行告诉`Compose`它是稳定的，之后`Compose`就会把`UIState`类型以及它的所有实现都视为稳定的。这种方法能实现自定义的智能重绘