> 本系列是我学习compose过程中，对官方文档的翻译和解读，以及实验性的Demo工程。主要参考官方文档和中文手册
>
> 全部的正文内容（Demo工程除外）源自Compose官方文档，个人解读以引用的形式插入。
>
> Compose 官方文档 https://developer.android.google.cn/jetpack/compose
>
> Compose 中文手册 https://compose.net.cn/
>
> 本文翻译内容 https://developer.android.google.cn/jetpack/compose/mental-model

`Jetpack Compose`是Android一个现代化的声明式`UI`工具包。Compose使编写和维护`UI`代码变得很容易，它提供了声明式的`Api`允许我们在不用绘制 view 的前提下渲染出想要的`UI`

## 声明式的编程模型

历史上，Android的`UI`结构是用view树的形式来组织的。当应用的状态随着用户的操作而变化时，这个树结构需要根据当前数据进行更新。最常见的更新方法是使用`findViewById()`遍历view树找到对应view节点，然后调用 `button.setText(String)`, `container.addChild(View)`, 或者`img.setImageBitmap(Bitmap)`这类函数更新节点。这些方法会改变这些组件的内部状态

手动操作view会增加犯错的可能。如果一个数据需要在多个view上展示，那在数据更新后很容易忘记更新其中某个view。另外当两个更新操作冲突时，会导致一些错误的更新结果。例如，某个更新操作需要对某个节点设置一个值，而这个节点刚刚被移出了view树。通常来说，软件维护的成本随着view数目的增加而增加

在过去的几年里，整个行业都在转向声明式的`UI`框架，它能大大简化`UI`相关的工程量。该技术的工作原理是从头开始重新生成整个屏幕，然后只应用必要的更改。这种方法避免了手动更新有状态的视图带来的复杂性。Compose是一个声明式的`UI`框架。

重新生成整个屏幕的一大难点在于昂贵的开销，包括运行时间、计算性能和电量消耗。为了降低成本，Compose在任意给定的时间会智能地选择需要重绘的`UI`部分。这种原理对如何设计你的`UI`结构会有影响，下文会继续讨论。

> 这一小节其实解释了声明式`UI`兴起的根本原因，原有的view tree不适合越来越复杂的应用

## 一个简单的可组合函数

你可以用一系列可组合函数（`composable function`）来构建你的`UI`视图，这些函数接收一些数据的输入，然后生成`UI`组件。一个简单的例子是`Greeting`组件，接收一个`String`类型输入，然后生成一个`Text`组件并展示一个欢迎的信息

<img src="E:\ljj的博客\pictures source\星愿浏览器截图20211021130926@2x.png" style="zoom:33%;" />

关于这个函数的一些注意事项:

- 所有的可组合函数都要有 `@Composable` 注解。这个注解会通知`Compose`编译器这个函数需要把数据转换成`UI`
- 可组合函数接收一些描述`UI`逻辑的参数。这个例子中接受一个`String`用来生成欢迎语句
- 可组合函数通过调用其他可组合函数来生成`UI`组件。这里调用了`Text()`生成一个文本
- 这个函数没有返回值。可组合函数生成`UI`都不需要返回值，因为它只是在描述需要一个什么样的`UI`而不是构建一个`UI`组件
- 可组合函数是快速的、幂等的、没有副作用的
  - 使用相同的参数调用这个函数，无论调多少次，它的结果都是完全相同的，因为它的执行并不依赖全局变量
  - 这个函数没有副作用是指它不会对外部的变量造成任何影响

理论上我们编写的所有可组合函数都要满足这些特点

> 这一小节讲了compose开发的核心工具：可组合函数。没有提原理，但是讲了几点约束。其中所谓幂等的和没有副作用的，其实就是设计模式中老生常谈的高内聚，低耦合，函数体里的全部逻辑应该只依赖于入参

## 声明式模型带来的变化

使用面向对象的`UI`工具包，你需要实例化一个view树来初始化整个`UI`结构。我们通常使用`xml`文件来做实现这点。每一个组件持有它内部的状态，通过暴露getter 和 setter方法允许应用控制它的逻辑

而在compose的声明式模型里，组件是没有状态的，也不暴露getter和setter方法。事实上组件并不被视为一个对象。你通过对相同的可组合函数传入不同的参数来控制一个`UI`的展示。这非常契合`ViewModel`提供的`observable`。可组合函数只要负责监听`observable`，在其变化时更新`UI`即可

<img src="E:\ljj的博客\pictures source\星愿浏览器截图20211021133200@2x.png" style="zoom: 33%;" />

当用户与`UI`交互时，`UI`需要上报这些事件比如`onClick`。这些事件需要通知应用的逻辑层，逻辑层会依据这些改变状态。当状态改变时，监听这些状态的可组合函数被调用，然后使用新的数据重绘`UI`。这个过程叫作`recomposition`

<img src="E:\ljj的博客\pictures source\星愿浏览器截图20211021133221@2x.png" style="zoom:33%;" />

> 这节讲的是compose的架构逻辑，其实思想很早就有了，就是`MVVM`，但是compose比传统view更契合这个思想。因为它的`UI`是不包含状态，和状态是完全解耦的。这样开发者只需要在`ViewModel`里注册好各种`Observable`，然后在Activity或者Fragment里监听它们，监听回调里调用对应的可组合函数。类似的，当点击等交互事件发生时，更改`ViewModel`里对应的`Observable`，然后就再次发生回调刷新`UI`

## 动态的内容

因为可组合函数是用`Kotlin`而非`xml`编写，它们可以像其他`Kotlin`代码一样是动态的。例如你试图对一系列用户表示欢迎：

```kotlin
@Composable
fun Greeting(names: List<String>) {
    for (name in names) {
        Text("Hello $name")
    }
}
```

可组合函数也可以变得非常复杂。比如使用if分支，和循环语句，再比如调用一些辅助函数。这种灵活和强大的特点是Compose的优势之一

> 这里的动态性我觉得它举的例子不够好，这样想更好：
>
> 比如你有一个自定义的文本框布局`MyTextView`是历史逻辑，你想展示一列文本，数目不确定，每一个文本就是一个`MyTextView`。那么在view树系统里，你可能会用`ListView`或者`RecyclerView`等等去封装它们。你需要写一个`adapter`、`ViewHolder`以及一系列样板代码。
>
> 像`RecyclerView`这样的组件虽然非常好用，功能强大，优化程度很高。但是你的本意仅仅是展示个列表，不需要那么多功能
>
> 如果用Compose你只需要一个循环语句就能解决这些问题（前提是用可组合函数重构`MyTextView`）

## 重绘

面向对象的`UI`模型中，改变一个组件需要调用该组件的`setter`方法改变它的内部状态。在`Compose`中，你可以用新的数据重新调用这个组件的可组合函数，来刷新这个组件。这个重新绘制的过程被叫作`recomposition`。`Compose`框架会智能地选择重绘那些改变了输入数据的组件

例如下面展示一个按钮的代码

```kotlin
@Composable
fun ClickCounter(clicks: Int, onClick: () -> Unit) {
    Button(onClick = onClick) {
        Text("I've been clicked $clicks times")
    }
}
```

每次点击会在`onClick`回调中更新 `clicks`的值。Compose 调用`Text`重绘组件内容，显示`clicks`的值。其他没有依赖`clicks`这个值的组件不会被重绘

如上所示，如果重绘全部的`UI`组件，开销是很昂贵的，包括对`CPU`计算资源和电量的消耗. Compose 通过`intelligent recomposition`，即智能重绘来解决这个问题

重绘发生在可组合函数的输入变化时。所以Compose在重绘时会检查调用链中，每一个可组合函数的入参有没有发生变化，如果没有就跳过这个函数的执行。通过跳过所有未改变入参的可组合函数，Compose重绘的过程相当高效

所以不要在可组合函数中写出带有副作用的代码，因为可组合函数的执行可能会被跳过。如果你强行这么做，用户可能会体验到一些诡异的效果。所谓副作用就是更改这个`UI`组件之外的值，例如:

- 给一个共享的对象赋值
- 更新`ViewModel`中`observable`的值
- 更新一些共享的配置

在运行一些动画时，可组合函数可能每一帧都会被重新调用一次，所以不要在里面写出耗时操作。如果你需要一些耗时逻辑，比如读取配置文件，那么把逻辑放在一个后台协程中，然后将结果作为可组合函数的一个参数

例如， 下面这个可组合函数更新了`SharedPreferences`的某个值。可组合函数中不应该直接读取或写入`SharedPreferences`，而是把这个逻辑移到`ViewModel`里的后台协程，然后把结果封装成一个`Observable`传入该可组合函数。那么每当读取完成，数据更新时会触发这个可组合函数的重绘。 

```kotlin
@Composable
fun SharedPrefsToggle(
    text: String,
    value: Boolean,
    onValueChanged: (Boolean) -> Unit
) {
    Row {
        Text(text)
        Checkbox(checked = value, onCheckedChange = onValueChanged)
    }
}
```

> 这部分讲述了Compose重绘`UI`的原理，以及为了降低功耗而进行入参检查
>
> 基于这种原理，开发者不应该做两件事：
>
> - 在可组合函数修改外部的共享资源
> - 在可组合函数中进行耗时操作

## Notes

剩下的部分是Compose的一些特点，针对每个特点给出一些“最佳实践”的指导，来帮助开发者写出更快、更好的代码

- 可组合函数的执行顺序是任意的
- 可组合函数可以并行执行
- 可组合函数尽可能多地跳过入参没有变化的函数
- 重绘可以被取消
- 重绘可能会很频繁

##### 可组合函数的执行顺序是任意的

如果一个可组合函数调用了多个可组合函数，这些函数的执行顺序可能是任意的。Compose会自行判断哪些`UI`的优先级更高，然后优先绘制它们。例如你试图在一个导航栏中绘制三个页面

```kotlin
@Composable
fun ButtonRow() {
    MyFancyNavigation {
        StartScreen()
        MiddleScreen()
        EndScreen()
    }
}
```

`StartScreen`, `MiddleScreen`, 和`EndScreen` 可能以任意顺序被执行。 这意味着你不能在`StartScreen()` 设置一个全局的属性，然后在 `MiddleScreen()` 依赖这个属性执行逻辑。每一个可组合函数都要是**自我完备的**

> 这里其实还是在强调“高内聚”这点

##### 可组合函数可以并行执行

Compose可以通过并发地执行可组合函数来优化重绘的过程。这使得Compose充分利用多个核心，并以低优先级运行那些`UI`组件不在屏幕上的可组合函数

这意味同一个可组合函数可能运行在后台的线程池中，如果这个函数调用了`ViewModel`中的某个函数A，那么函数A在同一时间可能会被多个线程调用

当一个可组合函数被执行时，所在的线程可能和调用这个可组合函数时的线程不相同。这意味着在**不要在可组合函数的lambda表达式中修改变量的值，这种行为被禁止因为它不是线程安全的**

下面有两份代码，第一个是线程安全的，因为它的渲染仅仅依赖可组合函数的输入。

```kotlin
@Composable
fun ListComposable(myList: List<String>) {
    Row(horizontalArrangement = Arrangement.SpaceBetween) {
        Column {
            for (item in myList) {
                Text("Item: $item")
            }
        }
        Text("Count: ${myList.size}")
    }
}
```

第二个不是线程安全的，因为每次重组`items`都会被修改，`UI`显示的值也不正确

```kotlin
@Composable
@Deprecated("Example with bug")
fun ListWithBug(myList: List<String>) {
    var items = 0

    Row(horizontalArrangement = Arrangement.SpaceBetween) {
        Column {
            for (item in myList) {
                Text("Item: $item")
                items++ // Avoid! Side-effect of the column recomposing.
            }
        }
        Text("Count: $items")
    }
}
```

> 这里所说的并发执行，其实是为了兼容协程。协程的并发本质上就是对线程框架的封装，协程提供了一套方便的工具来切换线程。所以Compose为了兼容，就要保证可组合函数在线程切换时也要正常运行，必须保证变量的线程安全
>
> 但是请注意，上面这个有Bug的例子，并不是运行不了，显示结果也是对的。它的问题在切换线程时才会暴露，所以这种错误很隐蔽，值得注意。（下图是我运行的demo效果，因为都是默认一直在主线程，所以结果是对）
>
> <img src="E:\ljj的博客\pictures source\微信图片_20211021195950.jpg"  />

##### 可组合函数尽可能多地跳过入参没有变化的函数

这种机制意味着只有需要重绘的组件才会真正被重绘。比如你可以单独重绘一个`UI`树的一个`Button`，而不用执行它的上层或下层的其他组件的代码。

下面这个例子演示了在渲染一个列表时，如何跳过其中的部分`item`的重绘

```kotlin
/**
 * 展示一系列用户可以点击的名字列表，顶部是一个header
 */
@Composable
fun NamePicker(
    header: String,
    names: List<String>,
    onNameClicked: (String) -> Unit
) {
    Column {
        // header只有在header参数变化时才重绘，names变化时不应该重绘
        Text(header, style = MaterialTheme.typography.h5)
        Divider()

        // LazyColumn 是Compose版本的RecyclerView
        // 传入items()的lambda可以理解为RecyclerView.ViewHolder
        LazyColumn {
            items(names) { name ->
                //当names列表中的某几项改变时，这改变的几项对应的item会重绘
                //当header改变时，任何item都不会重绘
                NamePickerItem(name, onNameClicked)
            }
        }
    }
}

/**
 * 展示一个可以点击的姓名
 */
@Composable
private fun NamePickerItem(name: String, onClicked: (String) -> Unit) {
    Text(name, Modifier.clickable(onClick = { onClicked(name) }))
}
```

##### 重绘可以被取消

当入参改变时，Compose会尽快地重绘这个可组合函数。但如果在重绘完成前，这个入参又改变了，那么Compose会取消重绘，然后用最新的值再次重绘。

当重绘被取消时，该`UI`组件会被从`UI`树上移除，所以如果你有一些对外部变量的操作是在`UI`被移除时触发的，那么可能会导致异常

> 这里告诫我们不要在`UI`移除这个事件上，做一些side-effects。
>
> 举个例子，如果你希望在某个UI消失上报一个埋点，那么不要把这个时机放在它被`UI`树移除的时刻，因为这个移除事件不一定是`UI`真的消失引起的，可能是快速的入参变化，带来快速的重绘，然后导致取消重绘，最后引起移除事件

##### 重绘可能很频繁

频繁的重绘导致在可组合函数中进行耗时操作，是对资源极大的消耗。比如你在一个`UI`组件中读取系统的某个配置，那么1s中随着反复重绘，可能会读取几百次而导致应用卡顿

如果你真的需要读取某些数据，把这些操作移到其他线程，写在可组合函数之外，然后用 `mutableStateOf` or `LiveData`把结果封装起来，作为参数传入可组合函数