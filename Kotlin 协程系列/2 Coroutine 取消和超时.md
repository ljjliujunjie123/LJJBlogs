> 本系列为翻译和解读 `Kotlin` 协程的官方文档，对应官方文档版本1.5.3 时间是2021-10
>
> 约定：全部的正文均对应文档原文，个人解读以引用的形式插入
>
> 官方文档链接 [https://kotlinlang.org/docs/coroutines-guide.html](https://kotlinlang.org/docs/coroutines-guide.html)
>
> 如果对协程完全没任何概念，强烈推荐先看这篇文章 [https://xie.infoq.cn/article/351ddc94200d03948c41fbabd](https://xie.infoq.cn/article/351ddc94200d03948c41fbabd)
>
> 如果你想写代码实操，可以参考这个配置环境 [https://openxu.blog.csdn.net/article/details/116999821](https://openxu.blog.csdn.net/article/details/116999821)
>
> 前文链接：[https://blog.csdn.net/ljjliujunjie123/article/details/120846681](https://blog.csdn.net/ljjliujunjie123/article/details/120846681)

## 取消协程的运行

在需要长时间运行的应用中，你可能需要对运行在后台的协程进行细粒度的控制。例如，用户可能关闭了一个页面，那么由这个页面启动的一个协程和它返回的结果可能就不再需要了，那么这个协程就应该被取消。前文提到`lanuch`函数返回一个句柄`Job`对象可以用来取消正在运行的协程

```kotlin
val job = launch {
    repeat(1000) { i ->
        println("job: I'm sleeping $i ...")
        delay(500L)
    }
}
delay(1300L) // 延时一会
println("main: I'm tired of waiting!")
job.cancel() // 取消这个协程
job.join() // 等待这个协程的完成
println("main: Now I can quit.")
```

```plaintext
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit.
```

一旦主函数中执行了`job.cancel()`，我们将看不到协程中的剩余输出，因为这个协程已经被取消了。官方库中有一个`Job`类的扩展函数`cancelAndJoin`封装了`cancel`和`join`

> cancel() 函数用于取消协程，join() 函数用于阻塞等待协程执行结束。之所以连续调用这两个方法，是因为 cancel() 函数调用后会马上返回而不是等待协程结束后再返回，所以此时协程不一定是马上就停止了，为了确保协程执行结束后再执行后续代码，此时就需要调用 join() 方法来阻塞等待。
>
> `public suspend fun Job.cancelAndJoin() {
>     cancel()
>     return join()
> }`

## 取消是协作完成的

协程的取消是需要协作完成的。协程的代码必须协作才能被取消。`kotlinx.coroutines` 中的所有挂起函数都是可取消的，它们在运行时会检查协程是否被取消了，并在取消时抛出 `CancellationException` 。但是，如果一个协程在执行计算任务，并且没有检查当前是否处于取消状态，那么它就无法被取消。例如

```kotlin
val startTime = System.currentTimeMillis()
val job = launch(Dispatchers.Default) {
    var nextPrintTime = startTime
    var i = 0
    while (i < 5) { // 计算任务，让CPU空转
        // 每隔0.5s输出一个日志
        if (System.currentTimeMillis() >= nextPrintTime) {
            println("job: I'm sleeping ${i++} ...")
            nextPrintTime += 500L
        }
    }
}
delay(1300L) 
println("main: I'm tired of waiting!")
job.cancelAndJoin() // 取消协程，然后等待其运行完成
println("main: Now I can quit.")
```

```
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
job: I'm sleeping 3 ...
job: I'm sleeping 4 ...
main: Now I can quit.
```

运行发现，它会不断地打印`I'm sleeping`直到 i = 5 循环跳出才结束协程运行，即使我们在中途取消了这个协程

> 这就解释了前面为什么取消协程后还需要手动用 `join` 阻塞等待，因为取消并不代表协程真得结束运行了
>
> 至于原理，见后面的系列

## 使计算任务的代码也可以取消

有两种办法使计算代码可以取消。第一种是周期性调用一个挂起函数，去检查是否处于取消状态。官方库的`yield`函数是实现这个功能很好的选择。第二种是显式地检查是否处于取消状态。让我们用第二种方法试一试：

用`while (isActive)`替换前面例子的 `while (i < 5)`

```kotlin
val startTime = System.currentTimeMillis()
val job = launch(Dispatchers.Default) {
    var nextPrintTime = startTime
    var i = 0
    while (isActive) { // 可取消的计算任务
        if (System.currentTimeMillis() >= nextPrintTime) {
            println("job: I'm sleeping ${i++} ...")
            nextPrintTime += 500L
        }
    }
}
delay(1300L) 
println("main: I'm tired of waiting!")
job.cancelAndJoin() 
println("main: Now I can quit.")
```

```
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit.
```

可以看到，这个循环是可取消的。`isActive`是`CoroutineScope`的一个扩展属性，用来在作用域中检查是否处于取消状态

> 这点启发我们，不仅是计算任务，任何协程中的耗时操作都应该加上对取消状态的监控

## 用finally关闭资源

**可取消的挂起函数通常情况下，会在被取消时抛出`CancellationException`**。例如，`try {...} finally {...}` 表达式或者 `kotlin` 中的 `use` 函数都可以在协程被取消时，执行回收操作

```kotlin
val job = launch {
    try {
        repeat(1000) { i ->
            println("job: I'm sleeping $i ...")
            delay(500L)
        }
    } finally {
        println("job: I'm running finally")
    }
}
delay(1300L) 
println("main: I'm tired of waiting!")
job.cancelAndJoin() 
println("main: Now I can quit.")
```

`join`和`cancelAndJoin`都会等待回收操作完成后再执行后面的代码，所以输出结果是

```plaintext
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
job: I'm running finally
main: Now I can quit.
```

> 某种意义上，可以把`finally`理解成协程结束的回调。只不过这个回调里最好不要执行耗时操作，也尽量不要调用其他挂起函数

## 运行不可取消的代码块

在`finally`代码块中尝试调用任何挂起函数都会抛出`CancellationException`异常，因为此时该协程已经被取消了。通常这并没啥问题，因为合理的关闭操作，比如关闭一个文件，取消一个任务，或者关闭任意一种通信通道，都是非阻塞的，也不会调用挂起函数。然而，在极端情况下如果你必须要在一个已经取消的协程中调用挂起函数，你可以使用 `withContext` 函数和 `NonCancellable` 上下文将相应的代码包装在 ```withContext(NonCancellable) {...}``` 代码块中。例如

```kotlin
val job = launch {
    try {
        repeat(1000) { i ->
            println("job: I'm sleeping $i ...")
            delay(500L)
        }
    } finally {
        withContext(NonCancellable) {
            println("job: I'm running finally")
            delay(1000L)
            println("job: And I've just delayed for 1 sec because I'm non-cancellable")
        }
    }
}
delay(1300L) 
println("main: I'm tired of waiting!")
job.cancelAndJoin() 
println("main: Now I can quit.")
```

```
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
job: I'm running finally
job: And I've just delayed for 1 sec because I'm non-cancellable
main: Now I can quit.
```

## 超时

我们主动取消协程的最显然的原因是该协程的执行时间已经超过了阈值。尽管我们可以通过`Job`引用去追踪这个协程，并在超时后取消它，但官方提供了函数`withTimeout`来完成这个操作。例如

```kotlin
withTimeout(1300L) {
    repeat(1000) { i ->
        println("I'm sleeping $i ...")
        delay(500L)
    }
}
```

```plaintext
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Exception in thread "main" kotlinx.coroutines.TimeoutCancellationException: Timed out waiting for 1300 ms
```

`withContext`抛出的`TimeoutCancellationException`异常是`CancellationException`的子类**。前面我们之所以没看到编译器抛出`CancellationExcetion`，是因为这个异常被认为是协程结束的正常原因。**但是这个例子中，我们在主函数中使用`withTimeout`，它会主动抛出`TimeoutCancellationException`

因为取消操作就是个异常，所以你可以在用`try {...} catch (e: TimeoutCancellationException) {...}` 捕获超时异常，然后附加一些你需要的逻辑。或者用`withTimeoutOrNull`作为替代，它会在超时后返回 null，而不是抛异常

```kotlin
val result = withTimeoutOrNull(1300L) {
    repeat(1000) { i ->
        println("I'm sleeping $i ...")
        delay(500L)
    }
    "Done" // 正常应该返回这个，但超时了，所以返回 null
}
println("Result is $result")
```

```plaintext
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Result is null
```

## 异步超时和资源泄漏

`withTimeout`中的超时事件是异步的，意味着抛出异常的时刻可以是代码块中的任意位置，比如在返回语句之前。谨记这一点，如果你在代码块中访问了外部的资源，需要在超时后释放这些资源。例如

```kotlin
var acquired = 0 //记录模拟资源被获取的次数

class Resource { //模拟资源类
    init { acquired++ } // 初始化资源
    fun close() { acquired-- } // 释放资源
}

fun main() {
    runBlocking {
        repeat(100000) { // 启动10万个协程
            launch { //每隔协程去获取一份资源
                val resource = withTimeout(60) { // 超时限制 60ms
                    delay(50) // 延时50ms
                    Resource() // 获取一份资源
                    //[标记点]
                }
                resource.close() // 释放该资源
            }
        }
    }
    // 等待全部协程运行结束
    println(acquired) // 打印此时被占有的资源数
}
```

运行会发现，结果有可能是正数（取决于你机器的性能）。因为超时时刻可能发生在【标记点】位置，此时还没来得及释放资源，协程就抛出异常退出了

解决这个问题的方法是可以用变量存储对资源的引用，而不是直接构造资源对象。这样强行捕捉异常，无论超时发生在何处，总会兜底到`finally`中释放资源。例如

```kotlin
runBlocking {
    repeat(100_000) {
        launch { 
            var resource: Resource? = null // 存一个外部引用
            try { //捕获超时异常
                withTimeout(60) { 
                    delay(50) 
                    resource = Resource()    
                }
                //对资源的操作
            } finally {  
                resource?.close() // 释放资源
            }
        }
    }
}
println(acquired)
```

> 这一小节是最近新增的，应该是有用户反馈内存泄漏问题吧。
>
> 个人感觉对我们的启发是，在使用超时监控时，务必加上异常处理，保证协程里的逻辑不能干扰外部逻辑