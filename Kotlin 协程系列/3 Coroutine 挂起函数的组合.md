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
> 前文链接：
>
> [https://blog.csdn.net/ljjliujunjie123/article/details/120846681](https://blog.csdn.net/ljjliujunjie123/article/details/120846681)
>
> [https://blog.csdn.net/ljjliujunjie123/article/details/120847011](https://blog.csdn.net/ljjliujunjie123/article/details/120847011)

## 协程默认是顺序执行的﻿

假设我们有两个挂起函数做一些有用的操作，比如调用远端的系统服务，或者是计算些什么东西。比如下面这两个函数，我们假设它们是有意义的，尽管事实上只是延迟了1s

```kotlin
suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // 假装在做一些有用的操作
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // 同上
    return 29
}
```

如果我们希望顺序调用它们该怎么办呢？也就是先调用`doSomethingUsefulOne` 再调用`doSomethingUsefulTwo`，然后把它们的结果加起来。对应到真实开发的例子，就是如果我们需要根据第一个函数的结果，来决定是否调用或者如何调用第二个函数。

我们只需要使用默认的调用顺序就行了，因为协程的代码默认就是顺序执行的，看起来和普通代码完全一样。下面的例子，通过计算总的消耗时间可以证明它们确实是顺序执行的

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = doSomethingUsefulOne()
        val two = doSomethingUsefulTwo()
        println("The answer is ${one + two}")
    }
    println("Completed in $time ms")    
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) 
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L)
    return 29
}
```

```plaintext
The answer is 42
Completed in 2017 ms
```

> 这里看起来很反直觉，因为协程不就是用来做并发的吗，但为啥默认是顺序的呢
>
> 其实很简单，因为协程的并发并不是多线程那样依赖操作系统的调度，协程的调度由程序员来控制，如果你不进行任何调度的管理，它的执行当然就是顺序的，毕竟代码本质上就是顺序执行的，这才是符合直觉

## 使用async实现并发

还是上面的例子，如果两个函数之间没有依赖关系，我们该怎么让它们同时运行来更快地得到结果？这时候可以用`async`

从概念上讲，`async`很像`launch`。它会启动一个单独的协程，这个协程是和其他协程并行工作的轻量级协程。不同之处在于，`lanuch`会返回一个`Job`类型的引用，同时没有返回值。而`async`会返回一个`Deferred`类型的引用，这是个轻量级非阻塞的对象，能在函数运行结束时提供返回值。我们可以通过`Defferred.await()`来拿到这个返回值。同时，`Deffered`也实现了`Job`接口，所以必要时我们也可以取消这个协程

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = async { doSomethingUsefulOne() }
        val two = async { doSomethingUsefulTwo() }
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")    
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) 
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) 
    return 29
}
```

```plaintext
The answer is 42
Completed in 1017 ms
```

我们发现运行速度是原本的两倍，因为这两个协程在并发地运行。注意，协程的并发必须是显式的，因为默认是顺序执行

> 这是最基本的实现并发的方法，用async手动声明一个子协程，子协程里做你需要的异步操作，然后用await阻塞父协程，等待返回结果

## async的惰性启动﻿

`async`可以通过设置它的`start`参数为`CoroutineStart.LAZY`变成惰性模式。惰性模式下，只有程序员主动调用了`Defferred`的`await`方法，或者调用了`Job`的`start`方法后，该协程才会开始运行。例如

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = async(start = CoroutineStart.LAZY) { doSomethingUsefulOne() }
        val two = async(start = CoroutineStart.LAZY) { doSomethingUsefulTwo() }
        // 其他操作
        one.start() // 启动第一个协程
        two.start() // 启动第二个协程
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")    
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) 
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) 
    return 29
}
```

```plaintext
The answer is 42
Completed in 1017 ms
```

该例中，两个协程虽然被定义了，但并没有像之前的例子一样直接执行，而是把控制权给了程序员，由程序员用`start`显式启动协程。我们先启动了第一个，然后再启动第二个，最后用`await`等待结果

注意，如果我们没有手动启动协程，而仅仅是在`println`语句中调用`await`函数，那么结果这两个协程会顺序执行。因为`await`函数会启动一个协程并等待它结束，这不符合懒惰模式的预期。懒惰模式 `async(start = CoroutineStart.LAZY)`是kotlin标准库中的lazy函数，在协程领域下的替代品，用于值的计算涉及挂起函数的情况

> 这一节是上一节的补充，核心思想是最后一句话。普通kotlin代码提供lazy函数实现变量、对象的延迟初始化，协程中也可以通过配置参数实现一个协程的延迟启动

## *异步风格的函数

我们可以定义异步风格的函数，使用带有显式 GlobalScope 引用的异步协程生成器来调用 `doSomethingUsefulOne` 和 `doSomethingUsefulTwo` 函数。用 “…Async” 后缀来命名这些函数，以此来强调它们用来启动异步计算，并且需要通过其返回的延迟值来获取结果

> [GlobalScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html) 是一个脆弱的api，可能会导致意想不到的错误，下面有一个例子。所以使用它必须加上注解`@OptIn(DelicateCoroutinesApi::class)`.

```kotlin
// 返回值类型是 Deferred<Int>
@OptIn(DelicateCoroutinesApi::class)
fun somethingUsefulOneAsync() = GlobalScope.async {
    doSomethingUsefulOne()
}

// 返回值类型是 Deferred<Int>
@OptIn(DelicateCoroutinesApi::class)
fun somethingUsefulTwoAsync() = GlobalScope.async {
    doSomethingUsefulTwo()
}
```

注意到这些 `xxxAsync` 函数不是挂起函数。它们可以在任意地方被调用。但是调用它们就意味着在用异步的方式执行代码。

下面的例子展示在协程之外如何调用它们

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

// 注意没有用`runBlocking`封装整个主函数
fun main() {
    val time = measureTimeMillis {
        // 在协程之外也可以调用异步的函数
        val one = somethingUsefulOneAsync()
        val two = somethingUsefulTwoAsync()
        // 但是必须要在协程作用域中等待返回结果
        // 这里用`runBlocking { ... }`提供协程作用域，并阻塞主线程
        runBlocking {
            println("The answer is ${one.await() + two.await()}")
        }
    }
    println("Completed in $time ms")
}

@OptIn(DelicateCoroutinesApi::class)
fun somethingUsefulOneAsync() = GlobalScope.async {
    doSomethingUsefulOne()
}

@OptIn(DelicateCoroutinesApi::class)
fun somethingUsefulTwoAsync() = GlobalScope.async {
    doSomethingUsefulTwo()
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) 
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) 
    return 29
}
```

```
The answer is 42
Completed in 1114 ms
```

> 这里演示的异步风格代码仅仅是用来说明，因为这种代码在其他语言中很常见。但是在Kotlin协程中强烈建议不要这么写，原因如下

考虑这种情况，如果在`val one = somethingUsefulOneAsync()`和`one.await()` 这两行代码之间，发生了一些逻辑错误导致程序抛出异常，正在执行的操作会被终止。通常情况下，全局的错误处理器会捕获这个异常，然后以日志上报给开发者，但是程序本身是有可能在继续做一些其他操作的。这里的`somethingUsefulOneAsync`就会始终运行在后台，尽管它的调用者已经被终止了。这种问题在下一节的结构化并发中就不会再出现

> 这一节是个bad case，一个是谨慎使用`GlobalScope`，一个是不要试图用多线程并发的编码风格来使用协程

## 使用async实现结构化并发﻿

让我们回到本文的第二节"使用async实现并发"的例子，抽象出一个函数能并发地执行`doSomethingUsefulOne` 和`doSomethingUsefulTwo` 并返回它们的结果之和。因为`async`是`CoroutineScope`的一个扩展函数，所以我们需要为它提供一个协程作用域，这正是`coroutineScope`函数的功能

```kotlin
suspend fun concurrentSum(): Int = coroutineScope {
    val one = async { doSomethingUsefulOne() }
    val two = async { doSomethingUsefulTwo() }
    one.await() + two.await()
}
```

使用这种方法，如果在`concurrentSum()`函数体中出现错误并抛出异常，该函数启动的所有子协程（`one`和`two`）都会被取消

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        println("The answer is ${concurrentSum()}")
    }
    println("Completed in $time ms")    
}

suspend fun concurrentSum(): Int = coroutineScope {
    val one = async { doSomethingUsefulOne() }
    val two = async { doSomethingUsefulTwo() }
    one.await() + two.await()
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}
```

可以看到结果是对的，确实是并发执行

```plaintext
The answer is 42
Completed in 1017 ms
```

取消操作总是沿着协程的层次关系来传播

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
    try {
        failedConcurrentSum()
    } catch(e: ArithmeticException) {
        println("Computation failed with ArithmeticException")
    }
}

suspend fun failedConcurrentSum(): Int = coroutineScope {
    val one = async<Int> { 
        try {
            delay(Long.MAX_VALUE) //延时超长的时间，等待two抛出异常
            42
        } finally {
            println("First child was cancelled")
        }
    }
    val two = async<Int> { 
        println("Second child throws an exception")
        throw ArithmeticException()
    }
    one.await() + two.await()
}
```

```plaintext
Second child throws an exception
First child was cancelled
Computation failed with ArithmeticException
```

注意当第二个子协程抛出异常后，第一个子协程和父协程都被取消了

> 这一节是更进一步解释本系列第一篇文章中说的结构化并发，核心思想还是为了防内存泄露

