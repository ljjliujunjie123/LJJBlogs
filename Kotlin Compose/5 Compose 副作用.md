> 本系列是我学习compose过程中，对官方文档的翻译和解读，以及实验性的Demo工程。主要参考官方文档和中文手册
>
> 全部的正文内容（Demo工程除外）源自Compose官方文档，个人解读以引用的形式插入。
>
> Compose 官方文档 https://developer.android.google.cn/jetpack/compose
>
> Compose 中文手册 https://compose.net.cn/
>
> 本文翻译内容 https://developer.android.google.cn/jetpack/compose/side-effects

Composable 应该是没有 side-effect 的。但是，如果确实有必要去更改app的状态，side-effect 的调用点应该被 composable的生命周期所能感知到。本文，你将学习Jetpack Compose提供的处理 side-effect 的 APIs

> 关键词：side-effect，中文翻译为 **副作用**
>
> 这个词在前面几篇文章也提到了，现在看下维基百科的定义：
>
> In computer science, an operation, function or expression is said to have a side effect if it modifies some state variable value(s) outside its local environment, that is to say has an observable effect besides returning a value (the main effect) to the invoker of the operation. -- wikipedia
>
> 通俗的讲就是，函数是程序的“仆人”，输入是给“仆人”的任务，输出是程序的预期结果。安分的仆人应该专注于手头的任务，不做僭越的事情，这种函数就是纯函数，除了返回一个运算结果，对其他东西一概不知。但不安分的仆人会偷拿主人的东西，虽然也完成了任务，但主人的某些东西减少了，这种函数就是有副作用的，它在执行任务时会依赖输入以外的东西，比如偷偷修改一个全局作用域的变量值。
>
> 回到compose中，理想的composable都应该是纯函数。但实际业务开发中，有些功能纯函数难以实现，或不可实现，必须引入副作用。所以compose提供了一些api，能帮助我们控制这些副作用，让它们不至于变得失控
>
> 这部分内容可以参考本文  https://juejin.cn/post/6930785944580653070

## 副作用示例

当在compose中使用副作用时，应当用以下这些副作用APIs，它们能使副作用的结果是可以预见的，也就是风险可控。这些APIs很容易被滥用，所以使用时确保不会破坏Compose的数据单向流动性

### LaunchedEffect 

如果想在一个composable中安全地调用挂起函数，需要使用可组合函数LaunchedEffect。当LaunchedEffect进入一个Composition时，它会启动一个协程，协程执行的代码块作为它的参数被传入。如果这个LaunchedEffect离开了Composition，它启动的协程会被取消。而如果发生了Recomposition，且传入LaunchedEffect的Key发生了变化，那么它所启动的协程会被取消，然后重新启动一个新协程

> 这里所说的"进入一个Composition"和"离开一个Composition"，是Compose 生命周期的内容，请参考本系列前文

例如，在一个Scaffold中展示一个Snackbar，具体的展示逻辑被封装在一个可组合函数中：SnackbarHostState.showSnackbar

```kotlin
@Composable
fun MyScreen(
    state: UiState<List<Movie>>,
    scaffoldState: ScaffoldState = rememberScaffoldState()
) {

    // 当UI的状态发生错误时，展示snackbar
    if (state.hasError) {
        //LaunchedEffect启动了一个协程
        //协程代码块作为最后一个参数以lambda的形式传入
        LaunchedEffect(scaffoldState.snackbarHostState) {
            //这里就是一个协程作用域
            //当该协程取消时，协程作用域中生成的所有UI也自动消失
            scaffoldState.snackbarHostState.showSnackbar(
                message = "Error message",
                actionLabel = "Retry message"
            )
        }
    }
}
```

上述代码中，当UI的状态出现错误时，LaunchedEfffect会启动一个协程，当没有错误时，该协程会被取消。之所以会自动取消，是因为LaunchedEffect的调用点是在一个if语句中，当if语句为真时，LaunchedEffect进入到一个Composition中。当state变量变化时，MyScreen发生Recomposition，此时if语句为假，LaunchedEffect没有调用点，所以已经存在于Composition中的LaunchedEffect会被移除，其启动的协程也随之被取消

### rememberCoroutineScope

LaunchedEffect是一个可组合函数，它提供的协程作用域是在其内部的。而rememberCoroutineScope能直接返回一个CoroutineScope，该协程作用域的生命周期与rememberCoroutineScope的调用点绑定。当这个调用点离开Composition时，该协程作用域自动被取消。

这个函数能允许开发者在可组合函数中自定义协程作用域，并手动管理它们的生命周期

例如，在用户点击按钮时弹出一个SnackBar

```kotlin
@Composable
fun MoviesScreen(scaffoldState: ScaffoldState = rememberScaffoldState()) {

    // 创建一个协程作用域，绑定到 MoviesScreen 的生命周期
    val scope = rememberCoroutineScope()

    Scaffold(scaffoldState = scaffoldState) {
        Column {
            Button(
                onClick = {
                    scope.launch {
                        scaffoldState.snackbarHostState
                            .showSnackbar("Something happened!")
                    }
                }
            ) {
                Text("Press me")
            }
        }
    }
}
```

### rememberUpdatedState

`LaunchedEffect` 会在入参改变时重启。但是有些情况下，某些入参改变时，我们不希望这个effect重启，但是我们又希望使用这个参数的最新值。这时候可以用`rememberUpdatedState`来创建一个引用，来跟踪这些入参，当它们改变时，应用这些改变，但不重启effect。对于那些effect中包含耗时操作的情况，这种方法很有用。

例如，假设你的app有一个`LandingScreen`需要在一段时间后自动消失。`LandingScreen`内部启动了一个effect来记录这个时间，那么即使LandingScreen发生了recomposition，这个effect也不应该被重启

```kotlin
@Composable
fun LandingScreen(onTimeout: () -> Unit) {

    val currentOnTimeout by rememberUpdatedState(onTimeout)

    LaunchedEffect(true) {
        delay(SplashWaitTimeMillis)
        currentOnTimeout()
    }
}
```

> 这个例子中，onTimeout是计时结束时的回调函数，delay是计时函数。
>
> 假设计时10分钟，初始化时，currentOnTimeout作为一个指向onTimeout的引用，而LaunchedEffect启动了一个协程，协程遇到挂起函数delay，开始一直等待。
>
> 假设到第5分钟时，用户做了一个操作，LandingScreen的入参onTimeout被更改了，那么LandingScreen发生recoposition，此时currentOnTimeout也随之被更新为最新的值。但是rememberUpdatedState函数的效果，使得虽然currenOnTimeout改变了，也就是LaunchedEffect的入参变化了，但LaunchedEffect却不会发生recomposition。所以delay函数也不会被取消，计时正常进行

### DisposableEffect

DisposableEffect也是一个可组合函数，当它的入参改变时，它会把内部产生的所有effect清除掉，重置为初始状态。

例如，如果你想监听一个组件的生命周期，并在特定时机发送一些events，可以使用`DisposableEffect`来注册和解除这个监听器

```kotlin
@Composable
fun HomeScreen(
  lifecycleOwner: LifecycleOwner = LocalLifecycleOwner.current,
  onStart: () -> Unit, 
  onStop: () -> Unit  
) {
    // 用rememberUpdatedState包裹一下，避免因为函数体的改变而重启effect
    val currentOnStart by rememberUpdatedState(onStart)
    val currentOnStop by rememberUpdatedState(onStop)

    // 当lifecycleOwner改变时，清除内部的effect，并重置状态
    DisposableEffect(lifecycleOwner) {
        // 创建监听器
        val observer = LifecycleEventObserver { _, event ->
            if (event == Lifecycle.Event.ON_START) {
                currentOnStart()
            } else if (event == Lifecycle.Event.ON_STOP) {
                currentOnStop()
            }
        }

        // 注册监听器
        lifecycleOwner.lifecycle.addObserver(observer)

        // 在离开composition时解注册
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }
}
```

注意一个`DisposableEffect`必须在函数体最后包含一个`onDispose` 闭包，否则IDE会报错

### SideEffect

To share Compose state with objects not managed by compose, use the [`SideEffect`](https://developer.android.google.cn/reference/kotlin/androidx/compose/runtime/package-summary#SideEffect(kotlin.Function0)) composable, as it's invoked on every successful recomposition.

For example, your analytics library might allow you to segment your user population by attaching custom metadata ("user properties" in this example) to all subsequent analytics events. To communicate the user type of the current user to your analytics library, use `SideEffect` to update its value.

> 这一段官方文档属于看傻眼了，实在没理解为啥
>
> 参考大佬的解释https://juejin.cn/post/6930785944580653070
>
> SideEffect是简化版的DisposableEffect，无需参数控制，每次recomposition都执行一次

```kotlin
@Composable
fun rememberAnalytics(user: User): FirebaseAnalytics {
    val analytics: FirebaseAnalytics = remember {
        /* ... */
    }

    // On every successful composition, update the analytics library with
    // the userType from the current User, ensuring that future analytics
    // events have this metadata attached
    SideEffect {
        analytics.setUserProperty("userType", user.userType)
    }
    return analytics
}
```

> 下面的3个API，实在难以理解，强行翻译误人子弟。只做一点意译。之后学会了再覆写
>
> 有需要的可移步参考这篇文章 https://blog.csdn.net/Mr_Tony/article/details/118941147

### produceState

[`produceState`](https://developer.android.google.cn/reference/kotlin/androidx/compose/runtime/package-summary#produceState(kotlin.Any,kotlin.coroutines.SuspendFunction1)) launches a coroutine scoped to the Composition that can push values into a returned [`State`](https://developer.android.google.cn/reference/kotlin/androidx/compose/runtime/State). Use it to convert non-Compose state into Compose state, for example bringing external subscription-driven state such as `Flow`, `LiveData`, or `RxJava` into the Composition.

The producer is launched when `produceState` enters the Composition, and will be cancelled when it leaves the Composition. The returned `State` conflates; setting the same value won't trigger a recomposition.

Even though `produceState` creates a coroutine, it can also be used to observe non-suspending sources of data. To remove the subscription to that source, use the [`awaitDispose`](https://developer.android.google.cn/reference/kotlin/androidx/compose/runtime/ProduceStateScope#awaitDispose(kotlin.Function0)) function.

The following example shows how to use `produceState` to load an image from the network. The `loadNetworkImage` composable function returns a `State` that can be used in other composables.

> produceState 的入参可以是State类型变量，也可以不是。它接收一个lambda表达式作为函数体，能将这些入参经过一些操作后生成一个State类型变量并返回

```kotlin
@Composable
fun loadNetworkImage(
    url: String,
    imageRepository: ImageRepository
): State<Result<Image>> {

    // Creates a State<T> with Result.Loading as initial value
    // If either `url` or `imageRepository` changes, the running producer
    // will cancel and will be re-launched with the new keys.
    return produceState(initialValue = Result.Loading, url, imageRepository) {

        // In a coroutine, can make suspend calls
        val image = imageRepository.load(url)

        // Update State with either an Error or Success result.
        // This will trigger a recomposition where this State is read
        value = if (image == null) {
            Result.Error
        } else {
            Result.Success(image)
        }
    }
}
```

### derivedStateOf

Use [`derivedStateOf`](https://developer.android.google.cn/reference/kotlin/androidx/compose/runtime/package-summary#derivedStateOf(kotlin.Function0)) when a certain state is calculated or derived from other state objects. Using this function guarantees that the calculation will only occur whenever one of the states used in the calculation changes.

> derivedStateOf 接收一个lambda表达式作为函数体，并返回一个State类型变量
>
> 它和produceState的区别在于，它是根据现有的State变量派生新的State变量，而后者是从任意类型变量，直接创建State变量

The following example shows a basic *To Do* list whose tasks with user-defined high priority keywords appear first:

```kotlin
@Composable
fun TodoList(
    highPriorityKeywords: List<String> = listOf("Review", "Unblock", "Compose")
) {
    val todoTasks = remember { mutableStateListOf<String>() }

    // Calculate high priority tasks only when the todoTasks or
    // highPriorityKeywords change, not on every recomposition
    val highPriorityTasks by remember {
        derivedStateOf {
            todoTasks.filter { it.containsWord(highPriorityKeywords) }
        }
    }

    Box(Modifier.fillMaxSize()) {
        LazyColumn {
            items(highPriorityTasks) { /* ... */ }
            items(todoTasks) { /* ... */ }
        }
        /* Rest of the UI where users can add elements to the list */
    }
}
```

In the code above, `derivedStateOf` guarantees that whenever `todoTasks` or `highPriorityKeywords` changes, the `highPriorityTasks` calculation will occur and the UI will be updated accordingly. As the filtering to calculate `highPriorityTasks` can be expensive, it should only be executed when any of the lists change, not on every recomposition.

Furthermore, an update to the state produced by `derivedStateOf` doesn't cause the composable where it's declared to recompose, Compose only recomposes those composables where its returned state is read, inside `LazyColumn` in the example.

### snapshotFlow

> 将State类型变量转为Flow对象

Use [`snapshotFlow`](https://developer.android.google.cn/reference/kotlin/androidx/compose/runtime/package-summary#snapshotFlow(kotlin.Function0)) to convert [`State`](https://developer.android.google.cn/reference/kotlin/androidx/compose/runtime/State) objects into a cold Flow. `snapshotFlow` runs its block when collected and emits the result of the `State` objects read in it. When one of the `State` objects read inside the `snapshotFlow` block mutates, the Flow will emit the new value to its collector if the new value is not [equal to](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-any/equals.html) the previous emitted value (this behavior is similar to that of [`Flow.distinctUntilChanged`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/distinct-until-changed.html)).

The following example shows a side effect that records when the user scrolls past the first item in a list to analytics:

```kotlin
val listState = rememberLazyListState()

LazyColumn(state = listState) {
    // ...
}

LaunchedEffect(listState) {
    snapshotFlow { listState.firstVisibleItemIndex }
        .map { index -> index > 0 }
        .distinctUntilChanged()
        .filter { it == true }
        .collect {
            MyAnalyticsService.sendScrolledPastFirstItemEvent()
        }
}
```

In the code above, `listState.firstVisibleItemIndex` is converted to a Flow that can benefit from the power of Flow's operators.

## 重启效应

前面提到的`LaunchedEffect`, `produceState`, or `DisposableEffect` 入参都是不固定的，入参更新时会取消正在运行的 effect并重启一个新的

这些API的形式可以抽象为

```kotlin
EffectName(restartIfThisKeyChanges, orThisKey, orThisKey, ...) { block }
```

这种行为有可能带来以下问题：

- 重启次数少于预期
- 重启次数多余预期

大致来说，effect中用到的所有变量，无论是不可变的，还是可变的，尽量都作为该effect composable的入参。这些参数中，如果某个参数即使改变，你也不希望重启effect，那么用rememberUpdatedState来包裹这个参数。这些参数中，如果某个参数用remember包裹，且没有任何key，说明该变量永远不会改变，那么可以不作为effect composable的入参

下例中，lifecycleOwner作为effect的入参

> 这个重启效应讲得也非常抽象
>
> 不过核心观点就是使用effect时，仔细考虑effect入参的设计。不合理的设计可能会导致effect的触发次数与预期不符合。

```kotlin
@Composable
fun HomeScreen(
  lifecycleOwner: LifecycleOwner = LocalLifecycleOwner.current,
  onStart: () -> Unit, // Send the 'started' analytics event
  onStop: () -> Unit   // Send the 'stopped' analytics event
) {
    // These values never change in Composition
    val currentOnStart by rememberUpdatedState(onStart)
    val currentOnStop by rememberUpdatedState(onStop)

    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            /* --- */
        }

        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }
}
```

### 常量作为Key

你可以用常量作为effect的入参，来迫使一个effect遵循调用点的生命周期。这种用法在前面的`LaunchedEffect`有所展示。但是注意，使用这个特点之前，三思后行