> 本系列是我学习compose过程中，对官方文档的翻译和解读，以及实验性的Demo工程。主要参考官方文档和中文手册
>
> 全部的正文内容（Demo工程除外）源自Compose官方文档，个人解读以引用的形式插入。
>
> Compose 官方文档 https://developer.android.google.cn/jetpack/compose
>
> Compose 中文手册 https://compose.net.cn/
>
> 本文翻译内容 https://developer.android.google.cn/jetpack/compose/state

`App`中的状态是指任何可以随着时间改变的值。这是个非常宽泛的概念，包含了几乎所有东西，大到一个`Room`数据库，小到某个类的一个成员变量

所有的Android应用都是在把状态展示给用户。下面是一些例子

- 无网络连接时展示的`snackBar`
- A blog post and associated comments.
- 用户点击按钮时触发的涟漪动画
- 用户在图像上绘制的贴纸

Jetpack Compose 能帮助你明确在哪里存放这些状态，以及如何使用它们。本文主要关心状态与可组合函数之间的关系，以及Jetpack Compose提供了一些帮助你管理状态的API。

## 状态与绘制

Compose是声明式的，所以想更改一个组件的状态，只能用新的状态值去调用该组件的可组合函数。这些状态值描述了这个`UI`组件的状态。状态的更新就对应着重绘的发生。所以，当`UI`的状态值改变时，基于XML的`UI`框架会自动刷新`UI`，但是在Compose中，必须显式地调用该`UI`的可组合函数，才能刷新该`UI`

```kotlin
@Composable
fun HelloContent() {
   Column(modifier = Modifier.padding(16.dp)) {
       Text(
           text = "Hello!",
           modifier = Modifier.padding(bottom = 8.dp),
           style = MaterialTheme.typography.h5
       )
       OutlinedTextField(
           value = "",
           onValueChange = { },
           label = { Text("Name") }
       )
   }
}
```

运行上述代码，会发现虽然向输入框输入了文字，但并没有渲染出来。这是因为Compose的渲染机制决定，必需显式重新调用OutlinedTextField这个可组合函数，但显然这里并没有

> 这里的OutlinedTextField暂时理解成Compose版本的EditText

<img src="E:\ljj的博客\pictures source\WeChat_20211024152758.gif" style="zoom:33%;" />

关键词解释：

> **Composition:** Jetpack Compose把可组合函数渲染成`UI`的这个过程，可以意译成绘制
>
> **Initial composition:** 首次运行可组合函数进行的绘制过程
>
> **Recomposition:** 当状态值改变后，再次调用可组合函数进行的绘制过程

## 可组合函数中的状态

通过`remember`函数，可组合函数可以在内存中保存一个单例。第一次绘制时，通过remeber记录的值会被保存到这个单例中，再次绘制时，这个值会被返回。`remember` 可以用来存储可变或不可变的对象

> **注意:** 当这个`UI`组件被从整个`UI`树上移除时，`remember`记录的这个单例会被回收

[`mutableStateOf`](https://developer.android.google.cn/reference/kotlin/androidx/compose/runtime/package-summary#mutableStateOf(kotlin.Any,androidx.compose.runtime.SnapshotMutationPolicy)) 函数创建一个类型为 [`MutableState`](https://developer.android.google.cn/reference/kotlin/androidx/compose/runtime/MutableState)的可观察对象

```kotlin
interface MutableState<T> : State<T> {
    override var value: T
}
```

`value`的改变会通知所有订阅了该可观察对象的可组合函数，并触发它们的重绘。

可组合函数中有三种方式定义一个 `MutableState` 对象

- `val mutableState = remember { mutableStateOf(default) }`
- `var value by remember { mutableStateOf(default) }`
- `val (value, setValue) = remember { mutableStateOf(default) }`

这三种声明是等价的，选择一种对你来说最容易阅读的即可

注意使用`by`委托的写法，需要这两个依赖

```kotlin
import androidx.compose.runtime.getValue
import androidx.compose.runtime.setValue
```

通过`remember`保存的变量，不仅可以作为其他可组合函数的入参，还可以参与逻辑控制。例如下面对`name`的非空判断

```kotlin
@Composable
fun HelloContent() {
   Column(modifier = Modifier.padding(16.dp)) {
       var name by remember { mutableStateOf("") }
       if (name.isNotEmpty()) {
           Text(
               text = "Hello, $name!",
               modifier = Modifier.padding(bottom = 8.dp),
               style = MaterialTheme.typography.h5
           )
       }
       OutlinedTextField(
           value = name,
           onValueChange = { name = it },
           label = { Text("Name") }
       )
   }
}
```

> 运行这个代码，会看到如下效果。因为onValueChange函数体里，状态量name被改变，所以一旦文本框文字变化，name随之变化，触发Text函数和OutlinedTextField函数的重绘
>
> <img src="E:\ljj的博客\pictures source\WeChat_20211024152812.gif" style="zoom:33%;" />

`remember`只能帮助你在重绘期间保存状态变量，但是Configuration改变时无法保存状态。对于这种情况，你必须使用`rememberSaveable`，它会自动保存所有可以存放进`Bundle`的类型的变量。对于其他类型的，你可以用自定义的对象保存机制

> 这里的Configuration可以参考 https://www.jianshu.com/p/7c27aabc5978

## 其他支持的状态类型

Jetpack Compose并不强制开发者使用`MutableState<T>`来管理状态，它还支持其他类型的可观察对象。 但是在使用这些其他类型的对象时，必须转型为`State<T>`类型，好让Jetpack Compose能在它们变化时自动给重绘相关组件

常见的可观察对象如

- [LiveData](https://developer.android.google.cn/reference/kotlin/androidx/compose/runtime/livedata/package-summary)
- [Flow](https://developer.android.google.cn/reference/kotlin/androidx/compose/runtime/package-summary#(kotlinx.coroutines.flow.StateFlow).collectAsState(kotlin.coroutines.CoroutineContext))
- [RxJava2](https://developer.android.google.cn/reference/kotlin/androidx/compose/runtime/rxjava2/package-summary)

另外，如果你使用的是自定义的可观察类型，可以通过为Jetpack Compose新增一个扩展方法的形式来使用它们。具体实现可以参考相关内置函数。

> **注意:**  [`State`](https://developer.android.google.cn/reference/kotlin/androidx/compose/runtime/State) 类型的变量作为可组合函数的入参后，该变量的改变会自动触发该组件的重绘
>
> 如果用其他类型的可观察变量作为入参，比如`LiveData`，必须先转型为`State<T>`。你可以用扩展函数来实现这一点，比如`LiveData<T>.observeAsState()`
>
> **注意:** 可变对象不能作为Compose中的状态变量，比如 `ArrayList<T>` 或 `mutableListOf()`
>
> 可变对象并不是可观察的对象，不能触发Compose的重绘。你可以用`State<List<T>>`替代，或者干脆用不可变的对象如`listOf()`

##### 有状态的 vs 无状态的

Compose使用 `remember` 存储内部状态，使可组合函数变成有状态的。 前面的例子中，`HelloContent` 就是个有状态的可组合函数，其内部持有并自动修改`name`属性的值。这在调用者不需要控制组件的状态，并且组件本身也无需管理状态的情况下非常有用。但是这样的可组合函数不容易复用和测试

无状态的可组合函数不持有任何状态变量，一种简单的实现方式是**状态提升**

当你在开发可复用的可组合函数时，你总希望暴露两个版本的函数。一个是有状态的，这样调用者可以很方便地使用，无需关注里面发生了什么。一个是无状态的，调用者必须自己控制状态的输入

## 状态提升

Compose中的状态提升是一种编程范式，指把可组合函数的状态变量提升到它的调用者里，来使该可组合函数本身是无状态的。Compose通用的状态提升方法，是讲一个状态变量用两个参数替代：

- **`value: T`:** 当前要展示的状态变量值
- **`onValueChange: (T) -> Unit`:** 该状态变量发生改变的事件，`T`是建议的新值

当然，Compose并不限制你一定用`onValueChange`，可以用`lambda`表达式自定义你需要的事件形式

状态提升有一些重要的特征

- **单一数据源:** 通过提升状态而不是复制状态，可以确保状态来源是唯一的，有助于减少bug
- **封装性:** 只有有状态的可组合函数才能修改状态
- **可共享的:** 被提升的状态变量可以同时用于多个可组合函数。比如上例中，我们希望在多个可组合函数中使用`name`变量
- **可拦截的:** 可组合函数的调用者接收到状态变量的改变这一事件后，可以选择忽略本次事件，或者修改该事件
- **解耦的:** 被提升的状态变量可以存储在任意位置，比如`ViewModel`中

下例中，我们把`name`这个状态变量提升到一个更高的层级`HelloScreen`，并用`name`和`onNameChange`这两个参数替代它，作为原函数`HelloContent`的参数

```kotlin
@Composable
fun HelloScreen() {
    var name by rememberSaveable { mutableStateOf("") }

    HelloContent(name = name, onNameChange = { name = it })
}

@Composable
fun HelloContent(name: String, onNameChange: (String) -> Unit) {
    Column(modifier = Modifier.padding(16.dp)) {
        Text(
            text = "Hello, $name",
            modifier = Modifier.padding(bottom = 8.dp),
            style = MaterialTheme.typography.h5
        )
        OutlinedTextField(
            value = name,
            onValueChange = onNameChange,
            label = { Text("Name") }
        )
    }
}
```

通过状态提升，`HelloContent`更容易复用和测试，同时和它的状态变量是如何保存的解耦开来。这种解耦意味着当我们修改或者替换`HelloScreen`时，不用修改`HelloContent`的实现

<img src="https://developer.android.google.cn/images/jetpack/compose/udf-hello-screen.png" alt="img" style="zoom:33%;" />

这种状态向下传递，事件向上传递的方式被叫作单向数据流。上例中，状态量从`HelloScreen`流向`HelloContent`，而事件则反向传递。遵循这种编程模式，你可以将展示`UI`的可组合函数与存储状态变量解耦开来

> **注意:** 提升状态的三条原则
>
> 1. 状态变量至少被提升到所有使用了该状态变量的可组合函数的最近的调用者处
> 2. 状态变量应该被提升到它有可能被改变的最高层次
> 3. 如果两个状态变量会被同一个事件所更改，它们应该被一起提升
>
> 你可以把状态提升得更高，但最低也不能低于这三条准则，否则就很难维护单向数据流的概念

## 状态恢复

使用[`rememberSaveable`](https://developer.android.google.cn/reference/kotlin/androidx/compose/runtime/saveable/package-summary#rememberSaveable(kotlin.Array,androidx.compose.runtime.saveable.Saver,kotlin.String,kotlin.Function0))来存储`UI`的状态变量，可以在activity或进程重新创建、可组合函数的重绘过程中保存状态。

存储状态的方法：

所有能被添加到`Bundle`中的数据都会自动保存。如果你想保存一些无法被添加到`Bundle`中的数据，可以用下列方法：

##### Parcelize

最简单的方法是对该对象的类加上注解 [`@Parcelize`](https://github.com/Kotlin/KEEP/blob/master/proposals/extensions/android-parcelable.md)。例如下面的代码定义了一个`City`数据类型实现了`Parcelable`接口，就可以使用`rememberSaveable`保存状态

```kotlin
@Parcelize
data class City(val name: String, val country: String) : Parcelable

@Composable
fun CityScreen() {
    var selectedCity = rememberSaveable {
        mutableStateOf(City("Madrid", "Spain"))
    }
}
```

##### MapSaver

如果不方便使用`@Parcelize`，可以用`mapSaver`自定义转换规则，把一个对象转变成一系列键值对，这些键值对可以存入`Bundle`

```kotlin
data class City(val name: String, val country: String)

val CitySaver = run {
    val nameKey = "Name"
    val countryKey = "Country"
    mapSaver(
        save = { mapOf(nameKey to it.name, countryKey to it.country) },
        restore = { City(it[nameKey] as String, it[countryKey] as String) }
    )
}

@Composable
fun CityScreen() {
    var selectedCity = rememberSaveable(stateSaver = CitySaver) {
        mutableStateOf(City("Madrid", "Spain"))
    }
}
```

##### ListSaver

如果不想定义键值，可以用`listSaver`替代，它默认用索引作为键值

```kotlin
data class City(val name: String, val country: String)

val CitySaver = listSaver<City, Any>(
    save = { listOf(it.name, it.country) },
    restore = { City(it[0] as String, it[1] as String) }
)

@Composable
fun CityScreen() {
    var selectedCity = rememberSaveable(stateSaver = CitySaver) {
        mutableStateOf(City("Madrid", "Spain"))
    }
}
```

## 状态管理

简单的状态提升可以在可组合函数本身中进行管理。但是，如果要跟踪的状态数量增加了，或者出现了在可组合函数中执行的逻辑，那么最好将逻辑和状态责任委托给其他类:状态持有者（state holder）

> **注意:** **状态持有者** 管理可组合函数的状态的相关逻辑在某些材料中，它也被叫作状态提升对象

下面根据状态管理的复杂程度，Compose提供了不同的管理状态的选择

- 直接用可组合函数管理简单的`UI`状态逻辑
- 状态持有者用于复杂的`UI`状态逻辑。它们持有`UI`组件的状态并控制状态逻辑
- **[Architecture Components ViewModels](https://developer.android.google.cn/topic/libraries/architecture/viewmodel)** 是特殊的状态持有者，用来处理复杂逻辑

状态持有者的大小取决于它们管理的相应`UI`元素的范围，从底部应用导航栏这样的单个小部件到整个屏幕。状态持有者是可复合的，这意味着一个状态持有者可能作为另一个状态持有者的一个变量

下图显示了Compose状态管理中所涉及的概念之间的关系。本节的其余部分将详细讨论每个概念

- Composable: 一个可组合函数可以依赖于0个或多个状态持有者(可以是普通对象、ViewModels或两者都有)，这取决于其复杂性
- State holder: 如果需要访问业务逻辑或屏幕状态，普通状态持有者可能依赖于ViewModel
- ViewModel依赖于业务层或数据层。

![Diagram showing the dependencies in state management, as decribed in the preceding list.](https://developer.android.google.cn/images/jetpack/compose/state-dependencies.svg)

##### 状态和逻辑的类型

状态类型：

- `UI` 组件本身的状态。比如[`ScaffoldState`](https://developer.android.google.cn/reference/kotlin/androidx/compose/material/ScaffoldState)持有[`Scaffold`](https://developer.android.google.cn/reference/kotlin/androidx/compose/material/package-summary#Scaffold(androidx.compose.ui.Modifier,androidx.compose.material.ScaffoldState,kotlin.Function0,kotlin.Function0,kotlin.Function1,kotlin.Function0,androidx.compose.material.FabPosition,kotlin.Boolean,kotlin.Function1,kotlin.Boolean,androidx.compose.ui.graphics.Shape,androidx.compose.ui.unit.Dp,androidx.compose.ui.graphics.Color,androidx.compose.ui.graphics.Color,androidx.compose.ui.graphics.Color,androidx.compose.ui.graphics.Color,androidx.compose.ui.graphics.Color,kotlin.Function1))的状态
- `UI` 需要展示到屏幕上的内容数据。这些状态需要和其他业务层相连，因为涉及到数据的传递

相对应的逻辑类型：

- `UI` 逻辑。例如导航栏决定下一个页面渲染什么、用户的数据如何展示到屏幕上，是用toast还是`snackBar`。`UI` 的逻辑总是在 `UI` 树这个层次进行的
- 业务逻辑。例如把用户的行为记录保存下来作为他的偏好。这些逻辑不在 `UI` 层进行，而是数据层或业务层

##### 可组合函数作为单一数据源

如果状态和逻辑很简单，把它们直接放在可组合函数中是一个很好的方法。例如这里的`MyApp`函数控制`ScaffoldState`和一个`CoroutineScope`

```kotlin
@Composable
fun MyApp() {
    MyTheme {
        val scaffoldState = rememberScaffoldState()
        val coroutineScope = rememberCoroutineScope()

        Scaffold(scaffoldState = scaffoldState) {
            MyContent(
                showSnackbar = { message ->
                    coroutineScope.launch {
                        scaffoldState.snackbarHostState.showSnackbar(message)
                    }
                }
            )
        }
    }
}
```

因为`ScaffoldState`包含可变属性，所以与它的所有交互都应该发生在`MyApp`函数中。如果将它传递给其他可组合函数，那么它有可能会被其他函数改变，这就不符合单一可信数据源的准则，使跟踪bug变得困难

##### 状态持有者作为单一数据源

当一个可组合函数包含复杂 `UI` 逻辑或者涉及多个 `UI` 组件的逻辑时，应该把状态的管理委托给一个状态持有者。这种方法符合**关注点分离准则：可组合函数负责渲染 `UI` ，状态持有者负责控制 `UI` 逻辑**

状态持有者就是个普通的类，只不过它的实例化对象的生命周期，是跟随可组合函数的生命周期。

例如，假设上面例子中的可组合函数`MyApp`变得越来越庞大，我们可以定义一个`MyAppState`类来管理它的状态，然后`MyApp`函数就可以专注于渲染 `UI`

```kotlin
// 一个普通类，管理 MyApp 的状态和逻辑
class MyAppState(
    val scaffoldState: ScaffoldState,
    val navController: NavHostController,
    private val resources: Resources,
    /* ... */
) {
    val bottomBarTabs = /* 状态变量 */

    // 控制何时展示bottomBar
    val shouldShowBottomBar: Boolean
        @Composable get() = /* ... */

    // 导航栏的跳转逻辑
    fun navigateToBottomBarRoute(route: String) { /* ... */ }

    // 展示snackbar
    fun showSnackbar(message: String) { /* ... */ }
}

@Composable
fun rememberMyAppState(
    scaffoldState: ScaffoldState = rememberScaffoldState(),
    navController: NavHostController = rememberNavController(),
    resources: Resources = LocalContext.current.resources,
    /* ... */
) = remember(scaffoldState, navController, resources, /* ... */) {
    MyAppState(scaffoldState, navController, resources, /* ... */)
}
```

```kotlin
@Composable
fun MyApp() {
    MyTheme {
        val myAppState = rememberMyAppState()
        Scaffold(
            scaffoldState = myAppState.scaffoldState,
            bottomBar = {
                if (myAppState.shouldShowBottomBar) {
                    BottomBar(
                        tabs = myAppState.bottomBarTabs,
                        navigateToRoute = {
                            myAppState.navigateToBottomBarRoute(it)
                        }
                    )
                }
            }
        ) {
            NavHost(navController = myAppState.navController, "initial") { /* ... */ }
        }
    }
}
```

> **注意:** 如果你希望在activity或进程重新创建时仍能恢复状态，用`rememberSaveable`和自定义的`Saver`来实现

##### ViewModels 作为单一数据源

ViewModel是一个特殊的状态持有者，有以下功能

- 提供一个访问其他业务层的入口，比如数据层
- 为屏幕的`UI`渲染提供数据

ViewModels的生命周期比`UI`树更长。它们可以和可组合函数的宿主，比如activity或fragment的生命周期保持一致。正因为此，ViewModels不应该持有那些只在`UI`存活期间用到的状态变量。否则容易造成内存泄漏。

我们推荐对于那些，屏幕级别的可组合函数，用ViewModels来作为它们的单一数据源。例如

```kotlin
data class ExampleUiState(
    dataToDisplayOnScreen: List<Example> = emptyList(),
    userMessages: List<Message> = emptyList(),
    loading: Boolean = false
)

class ExampleViewModel(
    private val repository: MyRepository,
    private val savedState: SavedStateHandle
) : ViewModel() {

    var uiState by mutableStateOf<ExampleUiState>(...)
        private set

    // Business logic
    fun somethingRelatedToBusinessLogic() { ... }
}

@Composable
fun ExampleScreen(viewModel: ExampleViewModel = viewModel()) {

    val uiState = viewModel.uiState
    ...

    Button(onClick = { viewModel.somethingRelatedToBusinessLogic() }) {
        Text("Do something")
    }
}
```

> **注意:** 如果你希望在进程重新创建后仍能恢复状态，请在ViewModel中使用[`SavedStateHandle`](https://developer.android.google.cn/topic/libraries/architecture/viewmodel-savedstate)来存储

##### ViewModel 和状态持有者的关系

在Android开发中，ViewModels的优点使它们适合于提供对业务逻辑的访问，以及为在屏幕上显示应用程序数据做准备。包括：

- 由ViewModels触发的操作在Configuration更改后仍然有效
- 兼容Navigation
- 兼容其他Jetpack libraries 例如 [Hilt](https://developer.android.google.cn/training/dependency-injection/hilt-jetpack#compose).

> **注意:**如果发现ViewModel的优点并不适合于你的需求，可以用状态持有者去替代ViewModel

对于一个屏幕来说，可以同时使用ViewModel和状态持有者，把ViewModel看作对数据层的访问入口，状态持有者如果需要数据，就去依赖ViewModel。这种思路是可行的，因为状态持有者是可复合的。

```kotlin
private class ExampleState(
    val lazyListState: LazyListState,
    private val resources: Resources,
    private val expandedItems: List<Item> = emptyList()
) { ... }

@Composable
private fun rememberExampleState(...) { ... }

@Composable
fun ExampleScreen(viewModel: ExampleViewModel = viewModel()) {

    val uiState = viewModel.uiState
    val exampleState = rememberExampleState()

    LazyColumn(state = exampleState.lazyListState) {
        items(uiState.dataToDisplayOnScreen) { item ->
            if (exampleState.isExpandedItem(item) {
                ...
            }
            ...
        }
    }
}
```

## 总结

> 这一篇的概念相当抽象，难以理解。简单概括几点，其余细节在实践中慢慢体会
>
> - 每个`UI`组件都需要状态变量来控制渲染逻辑
> - Compose提供了`remember`机制记录状态，可以直接在可组合函数中使用
> - 有状态的`UI`用起来简单，维护难。无状态的`UI`用起来稍微麻烦点，但维护简单
> - `UI`逻辑复杂起来后，最好用无状态的`UI`，此时需要用 **状态提升** 的技巧
> - 状态提升只是个编程技巧，提供一种分离状态变量的思路而已。只要你能保证团队成员都能很清晰地理解层级划分，并且每隔`UI`组件都（尽量）做到了无状态，怎么提升都行
> - 使用`remeber`记录的状态，生命周期和`UI`绑定在一起。如果你希望状态变量的生命周期更长点，用`rememberSaveable`
> - 被提升的状态得需要一个管理者，根据业务复杂程度分为三档管理者，最简单的是可组合函数，其次是状态持有者，最后是ViewModel
> - 使用哪一个状态管理者都行，只要团队成员认可且维护起来方便。估计大多数还是用ViewModel比较方便

