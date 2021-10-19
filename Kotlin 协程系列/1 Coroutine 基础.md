> 本系列为翻译和解读 `Kotlin` 协程的官方文档，对应官方文档版本1.5.3 时间是2021-10
>
> 约定：全部的正文均对应文档原文，个人解读以引用的形式插入
>
> 官方文档链接 https://kotlinlang.org/docs/coroutines-guide.html
>
> 如果对协程完全没任何概念，强烈推荐先看这篇文章 https://xie.infoq.cn/article/351ddc94200d03948c41fbabd
>
> 如果你想写代码实操，可以参考这个配置环境 https://openxu.blog.csdn.net/article/details/116999821

## 你的第一个协程程序

一个协程对象是一个可暂停的计算过程。它在概念上和线程很相似，也就是说协程也是让一个代码块和其他代码同时运行。然而，协程并不会和任何特定的线程绑定。它可能在某个线程上暂停运行，然后在另一个线程上恢复运行

协程可以被认为是轻量级的线程，但是务必注意二者有大量的重要的不同之处，这使协程的使用迥异于线程

> 这段话非常重要，因为中文翻译协程有“轻量级线程”的说法，很容易误导理解为一个线程上分出很多协程。但事实上协程和线程是没有任何这种“继承”关系的，它是一个独立的框架。官方文档也强调那只是个比喻
>
> 协程是运行于线程上的，一个线程可以运行多个（可以是几千上万个）协程。线程的调度行为是由 OS 来操纵的，而协程的调度行为是可以由开发者来指定并由编译器来实现的。

运行下面的代码来观察你的第一个协程

```kotlin
fun main() = runBlocking { // 该闭包内的this指向一个CoroutineScope类型的对象
    launch { // 启动一个子协程
        delay(1000L) // 不阻塞地延时1s (单位 ms)
        println("World!") // 延时后打印
    }
    println("Hello") // 主协程在子协程延时的同时在继续运行
}
```

你将看到结果

```plaintext
Hello
World!
```

让我们分析一下这段代码的功能

`lanuch` 是一个协程构造器。它能启动一个协程，同时使剩余的代码继续独立的运行。这是为什么 `Hello` 先被打印

`delay` 是一个特殊的挂起函数。它能在一个特定的时间点暂定协程的运行。挂起一个协程并不会阻塞该协程所在线程的运行，此时线程可以去运行其他的协程代码

`runBlocking` 也是一个协程构造器，它连接了非协程的代码如上述的`fun main()`和协程代码如`runBlocking{}`大括号中的内容。`IDE` 会在`runBlocking`后提示`this: CoroutineScope`

如果你删掉或者忘了加 `runBlocking` ，你会在`launch`函数上看到一个`error`。因为`lanuch`只能声明在 `CoroutineScope`中

```none
Unresolved reference: launch
```

> 这里的`CoroutineScope`是协程作用域，就先理解成协程特定的上下文环境，协程只能运行在这种环境里。而协程构造器就是提供这种环境

`runBlocking`这个名字意味着，当前运行的协程会被下面的代码阻塞（在这个例子中是主线程），直到代码块中的全部协程都运行结束。你将经常看到在应用程序的最顶层使用`runBlocking`，而在具体的逻辑代码中很少使用，因为线程是昂贵的资源，阻塞线程是低效且不推荐的

### 结构化并发

协程遵循结构化并发的原理，这意味着新的协程只能在特定的协程作用域中被启动，协程作用域就限制了协程的生命周期。上面的例子中`runBlocking`构建了协程作用域，这是为什么主函数一直等待到`World!`被打印后才结束运行

在真实的应用中，我们会启动非常多的协程。结构化并发能保证他们不会丢失和泄露。一个外部的作用域不会结束运行直到它的所有子协程运行结束。结构化并发也能保证任何代码错误都能正确无遗漏地被上报

> 这段话很抽象，但核心思想是之所以有协程作用域这种语法，是为了追踪所有协程对象，避免内存泄漏

## 提取函数与重构

让我们把 `launch { ... }` 中的代码提取出来形成一个函数。当你在上述例子中试图提取函数时，你需要在新函数前加上`suspend`修饰符。这是你的第一个挂起函数。挂起函数在协程中可以像普通函数一样使用，但是它额外的能力是可以调用其他挂起函数，来暂停当前协程的运行。比如`doWorld`中调用`delay`

```kotlin
fun main() = runBlocking { // this: CoroutineScope
    launch { doWorld() }
    println("Hello")
}

suspend fun doWorld() {
    delay(1000L)
    println("World!")
}
```

## 作用域构造器

除了使用官方库的构造器创建协程作用域，还可以用`coroutineScope`声明你自己的作用域。它能创建一个协程作用域，该作用域会等待其中的所有子协程运行完成后才结束运行

你会发现`runBlocking`和`coroutineScope`看起来很像，但注意它们主要的区别在于前者在等待子协程运行的同时会阻塞当前的线程，而后者在等待子协程运行的时候，会释放当前线程去运行其他代码。由于这个区别，前者只是个普通函数，后者则是个挂起函数

你可以从任意挂起函数中使用`coroutineScope`。例如：

```kotlin
fun main() = runBlocking {
    doWorld()
}

suspend fun doWorld() = coroutineScope {  // this: CoroutineScope
    launch {
        delay(1000L)
        println("World!")
    }
    println("Hello")
}
```

```plaintext
Hello
World!
```

> 注意这里，`runBlocking`创建了一个协程作用域记作A，`doWorld`是A中的一个挂起函数。这个挂起函数里用`coroutineScope`又创建了一个作用域记作B，函数体里用`lanuch`又创建了一个作用域记作C，`delay`是C中的挂起函数
>
> 所以`delay`挂起了协程C，`doWorld`挂起了协程A。因为`runBlocking`是阻塞挂起，所以在`doWorld`没有运行结束前不会结束主函数。因为`coroutineScope`是非阻塞挂起，所以在C被挂起后，继续运行B中的剩余代码

## 作用域构造器与并发

协程作用域构造器还可以用在任意挂起函数的内部，来执行并发的操作。让我们在`doWorld`函数内部启动两个协程

```kotlin
// 顺序执行：先是 doWorld ，然后是打印"Done"
fun main() = runBlocking {
    doWorld()
    println("Done")
}

// 并发执行：同时执行两个子协程
suspend fun doWorld() = coroutineScope { // this: CoroutineScope
    launch {
        delay(2000L)
        println("World 2")
    }
    launch {
        delay(1000L)
        println("World 1")
    }
    println("Hello")
}
```

两个由`launch { ... }`启动的子协程是并发执行的，所以从最开始开始计时，先打印出`Hello`，然后第1 s 后打印出`World 1`，第2 s 后打印出`World 2`，这两个子协程运行结束后，`doWorld`所在的协程作用域才运行结束，返回到 `runBlocking`中打印`Done`

```plaintext
Hello
World 1
World 2
Done
```

## 一个显式的Job

协程构造器`lanuch`会返回一个 `Job` 类型的对象，这是一个句柄，指向被启动的协程。它可以用来显式声明等待该协程的运行结束。例如，你可以显式声明等待子协程完成后，再打印`Done`

```kotlin
val job = launch { // 启动一个协程，并用job作为指向它的引用
    delay(1000L)
    println("World!")
}
println("Hello")
job.join() // 显式声明等待该协程运行结束，结束后才运行后面的代码
println("Done") 
```

```plaintext
Hello
World!
Done
```

## 协程是轻量的

运行如下代码

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    repeat(100_000) { // launch a lot of coroutines
        launch {
            delay(5000L)
            print(".")
        }
    }
}
```

它启动了10万个协程，5 s后每个协程打印一个点号

尝试用线程去实现，去掉`runBlocking`，用`thread`替代`launch` ，用`Thread.sleep`替代`delay`。大概率你的代码会报内存溢出错误

> 因为启动一个协程是创建一些对象，而启动一个线程是分配一大堆内存！