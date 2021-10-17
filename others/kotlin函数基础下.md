kotlin号称更好的java，不仅支持java的绝大部分语法，还新增了非常多语言特性。函数作为编程语言最重要的核心（我认为没有之一），kotlin的函数对于像我这样的初学者来说“面目狰狞”，本文记录了我学习过程中遇到的各种与函数有关的概念，并对各自的原理做一点点探索。

本文涉及概念：扩展函数、匿名函数、标准函数、构造函数、委托函数、覆盖函数、挂起函数、泛型函数、回调函数

本文还有一篇上文，介绍了其他类型函数 [kotlin函数基础 上](https://blog.csdn.net/ljjliujunjie123/article/details/118464790?spm=1001.2014.3001.5502)

### 扩展函数

顾名思义，扩展是对某个东西原有功能的增强。在kotlin中，所谓的“某个东西”就是类。而一个类中最常包含的就是属性和方法，所以扩展也分为扩展属性和扩展方法（扩展函数）。如果你听说过“装饰者模式”，会发现它的目的和扩展这种语法非常相似，只不过扩展更加简洁、开销也更低（大多数情况）。
语法规则是 `fun 接受者类型.函数名(参数列表): 返回值类型 {函数体}` 
 下面是一个简单的例子

```kotlin
 //demo.kt
 //扩展函数1
 private fun MutableList<Int>.swap1(index1: Int, index2: Int) {
   val tmp = this[index1] // “this”对应该列表
   this[index1] = this[index2]
   this[index2] = tmp
 }

class A {
   private val a = mutableListOf(1,2,3)
   private val b = mutableListOf(1,2,3)
   init {
     a.swap1(1,2)
     b.swap2(1,2)
     println(a.toString())//输出 [1,3,2]
     println(b.toString())//输出 [1,3,2]
   }
   //扩展函数2
   private fun MutableList<Int>.swap2(index1: Int, index2: Int) {
     val tmp = this[index1] // “this”对应该列表
     this[index1] = this[index2]
     this[index2] = tmp
   }
 }
 //main函数来验证效果
 fun main() {
   val aa = A()
 }
```

直观上和普通函数有两点区别

- 函数名前加了一个类名
-  函数体里调用了this

实现的功能也很简单，给MutabList这个类新增一个交换其中两个元素的方法。让我们看下编译后对应的java代码

```java
 public final class A {
   private final List a = CollectionsKt.mutableListOf(new Integer[]{1, 2, 3});
   private final List b = CollectionsKt.mutableListOf(new Integer[]{1, 2, 3});

  private final void swap2(List $this$swap2, int index1, int index2) {
    int tmp = ((Number)$this$swap2.get(index1)).intValue();
    $this$swap2.set(index1, $this$swap2.get(index2));
    $this$swap2.set(index2, tmp);
   }

  public A() {
    DemoKt.access$swap1(this.a, 1, 2);
    this.swap2(this.b, 1, 2);
    ......
   }
 }

public final class DemoKt {
   private static final void swap1(List $this$swap1, int index1, int index2) {
    int tmp = ((Number)$this$swap1.get(index1)).intValue();
    $this$swap1.set(index1, $this$swap1.get(index2));
    $this$swap1.set(index2, tmp);
   }
   ......
   public static final void access$swap1(List $this$access_u24swap1, int index1, int index2) {
    swap1($this$access_u24swap1, index1, index2);
   }
 }
```

 **发现函数名前加的类名，变成了真正函数的第一个参数，函数体里的this指针指向这个参数。** 所以扩展函数的本质并不是修改原类，而是提供一个第三方函数操作原类的实例。
 扩展函数的一些使用规则如下：

- 注意swap1和swap2声明的位置有所不同，swap1作为顶层函数直接声明在一个文件中，所以只要引用了这个文件，任意类或方法都可以调用swap1。而swap2作为成员函数声明在类A中，遵循成员函数调用规则的同时有一个附加限制，**不能在类A及其子类之外的任何地方调用**。
- 扩展方法是静态的，如果若干个类都增加了相同函数名的扩展方法，则真正调用的扩展函数仅由表达式编译时的类型决定，与运行时的类型无关。如例：

```kotlin
open class Shape

class Rectangle: Shape()
 //父类
 fun Shape.getName() = "Shape"
 //子类
 fun Rectangle.getName() = "Rectangle"
 //编译时s的类型是Shape，所以最终调用的方法是Shape.getName()
 fun printClassName(s: Shape) {
   println(s.getName())
 }   
 //运行时s的类型是Rectangle
 printClassName(Rectangle())//输出“Shape“
```

- 如果一个类中声明了两个同名函数，一个是成员函数，一个是扩展函数，则成员函数优先级高
- 扩展函数的接受者类型是可空的，不过需要自己在扩展函数体中进行空类型处理
- 一个类的伴生对象也可以作为扩展函数的接受者类型，但其外部类的实例无法调用该函数

```kotlin
 class MyClass {
   companion object Demo{ }  // 默认名"Companion"，显式声明为“Demo”
 }
 //为伴生对象声明扩展方法
 fun MyClass.Demo.printCompanion() { println("companion") }
 //调用时
 fun main() {
   MyClass.printCompanion()//输出 companion
   MyClass.Demo.printCompanion()//输出 companion
   MyClass().printCompanion()//编译错误：接受者类型不符
 }
```

#### 应用

最常见的应用其实是apply, let等标准函数，它们是kotlin自带的扩展函数，能非常方便地简化代码。
真实开发中，业务代码其实很少用到自定义扩展函数，因为用扩展实现的功能，也可以用继承或其他方法实现。所以扩展更多作为设计业务框架时的工具，或者作为优化代码逻辑时的辅助函数。比如下面这个例子，两种写法都能实现，但扩展看起来更易读

```kotlin
//单例实现
 object Utils {
    //某个工具函数，依赖activity实例
   fun demoUtil(activity: Activity): Boolean {
     return false
   }
 }
 //扩展实现
 fun Activity.demoUtil(): Boolean = false

//在某个activity调用时
 val resultA = Utils.demoUtil(this)
 val resultB = this.demoUtil()
```

### 匿名函数

这个概念在python、js、c++、c#等多种语言中都存在，是一种相当好用的语法，但对初学者非常不友好。因为使用它的前提是如何理解“表达式”这个概念。简单地讲，kotlin中分为语句和表达式，语句是可以单独执行的、能够产生实际效果的代码，比如`val a = 1`就是在内存里开了一个单元存一个变量(表意而已，事实不是这样)；而表达式则是包含在语句中，为语句提供一个返回值，然后由语句去判断和处理，比如`if(a == 1) { //do something}`这里的a==1就是一个表达式，返回一个boolean值供if使用。
所以回到匿名函数的概念，其就是一个比较复杂的表达式，能接受参数，最终给出一个返回值（没有显式返回值的函数其实返回的是unit
语法规则是`fun(参数列表): 返回值 {函数体}`下面是一个例子

```kotlin
var a = fun(param: Int): String {
   return param.toString() 
}
```

 匿名函数可以作为一个表达式返回一个函数对象，对象可以赋值给变量，或者作为参数任意传递。其余部分和普通函数完全一致。反编译后会发现a就是一个Function1类型的对象。需要注意的一点是，**匿名函数不支持泛型语法**

#### 应用

匿名函数的使用在kotlin中可以说是无处不在，但又很少有人提匿名函数的概念。因为kotlin对匿名函数做了进一步的简化，有lambda表达式这个好用到无敌的语法糖！比如替代java中声明一个interface实现回调的机制，kotlin中传入一个lambda可以优雅地实现回调(说是优雅，但回调本身就不优雅)。lambda的更多知识可以参考我的这篇博客[kotlin的lambda](https://blog.csdn.net/ljjliujunjie123/article/details/118421873)

###  标准函数

标准函数是kotlin独有的概念，仅仅是一个语法糖的统称而已，不要过分解读它。官方文档中又称标准函数为作用域函数。特指kotlin标准库提供的let、run、with、apply、also这5个函数。一些例子如下

```kotlin
//let
val str: String? = "Hello world" 
str?.let { println("this is a string: $it") }

//run
val str: String? = "Hello world" 
val strLength = str.run { length }

//apply
val adam = Person("Adam").apply {
    age = 32
    city = "London"        
}
```

这5个函数的使用方式基本相同，即一个对象通过点操作符调用，传入一个lambda表达式，在该表达式体中执行一些逻辑。它们的区别在于两点，一个是上下文的默认传递方式，一个是lambda表达式的返回值。

| 函数  | 上下文传递 | 返回值           |
| ----- | ---------- | ---------------- |
| let   | it         | lambda表达式结果 |
| run   | this       | lambda表达式结果 |
| with  | this       | lambda表达式结果 |
| apply | this       | 上下文对象       |
| also  | it         | 上下文对象       |

- 上下文传递是指在lambda表达式体中，可以用this/it代指调用该标准函数的那个对象。

- 表达式结果是指lambda表达式这个局部作用域中，最后一个表达式的返回值，如果最后一行是语句没有返回值，可以理解为返回了一个Unit对象。比如上面的例子中，`println("this is a string: $it")`返回一个Unit，`length`返回`str.length`。

- 上下文对象指的就是那个调用了标准函数的对象，而返回值是上下文对象，可以理解为**将lambda表达式中的操作应用于该对象**。

开发时会发现，很多情况下用哪个函数都行，在选择上并没有强制，官方文档也只是给出了一些建议，比如尽量不要出现`it.xxx`这种调用（但有时候很难避免），尽量不要嵌套标准函数等等。但其实只要符合部门的编码习惯，怎么写都行

#### 原理

```kotlin
@kotlin.internal.InlineOnly
public inline fun <R> run(block: () -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}

@kotlin.internal.InlineOnly
public inline fun <T, R> T.run(block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}

@kotlin.internal.InlineOnly
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return receiver.block()
}

@kotlin.internal.InlineOnly
public inline fun <T> T.apply(block: T.() -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block()
    return this
}

@kotlin.internal.InlineOnly
@SinceKotlin("1.1")
public inline fun <T> T.also(block: (T) -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block(this)
    return this
}

@kotlin.internal.InlineOnly
public inline fun <T, R> T.let(block: (T) -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block(this)
}
```

其中contract关键字是契约的意思，大致理解为对这个函数的执行做一些约束，帮助编译器进行优化。详见 

[kotlin contract]: https://www.jianshu.com/p/a35f99adf365	"kotlin contract"

可以发现，also apply let都是扩展函数，with不是扩展函数，run既有扩展实现，也有非扩展实现。

其中扩展的类型对象是泛型，这解释了为什么标准函数能在任意对象上调用。

run的扩展实现就是标准函数，而非扩展实现更好理解，接收一个lambda并执行它，这也是kotlin的一个语法糖。

观察这些函数参数会发现，上下文传递为this时，参数是带接收者的lambda，另外的则是普通lambda，这部分原理请见https://blog.csdn.net/ljjliujunjie123/article/details/118421873?spm=1001.2014.3001.5501

#### 应用

标准函数属于语法糖层次，所以应用范围非常广，任何地方只要满足调用规则，都可以用它来简化你的代码。比如

```kotlin
class mobHelper(val time: String, val name: String, val count: String) {
	fun processTime(time: String) {...}
	fun processName(name: String) {...}
	fun processCount(count: Stirng) {...}
}
//使用apply简化链式调用
val curMob = mobHelper("time", "name", "count").apply {
	processTime(time)
	processName(name)
	processCount(count)
}
```

### 构造函数

这个函数可能是所有coder最早接触的函数之一，几乎所有支持面向对象的语言都有类的概念，几乎所有的类必须有构造函数。顾名思义，这个函数是为了创造某种东西存在的，这个东西就是类的实例，也就是漫天遍野的对象。而函数可以接收参数，这些参数也被称为类的构造参数。

kotlin中的类的构造函数，分为主构造函数和次构造函数。其中次构造函数使用较少。主构造函数的语法如下

- 主构造函数必须写在类头处，跟在类名后

```kotlin
class Person constructor(firstName: String) {}
```

- 一般情况下，主构造函数没有任何注解和可见性修饰符，所以可以省略constructor

```kotlin
class Person(firstName: String) {}
```

- 主构造函数只负责声明参数列表，不能有任何代码。初始化逻辑放到 init 方法体里。init的执行顺序是按照声明顺序来的，和类体中属性的初始化混杂在一起的

```kotlin
class Person(name: String) {
    val sex = "male"
    init { println(sex + name) }
    
    val height = "2m"
    init { println(height + name) }
}
```

- 主构造函数的参数并不是类属性，但可以用于类体中属性的初始化和init语句块，但不能用于成员函数体中。主构造参数列表中添加val/var可以直接将参数声明为属性

```kotlin
class Person(private val name: String, val sex: String) {
	init { print(name + sex) } //正常
	
	fun doSomething() {
		print(sex) //无法访问
	}
}
```

次构造的语法如下

- 类体中用constructor显式声明，数量不定
- 如果有主构造函数，需要用this委托主构造函数，或者通过委托另一个次构造函数间接委托主构造
- 所有属性的初始化和init语句块都会编译到主构造中，而次构造需要先调用主构造，然后再是自己的逻辑。所以次构造的逻辑发生在初始化之后

```kotlin
class Person(val name: String) {
    var children: MutableList<Person> = mutableListOf()
    constructor(name: String, parent: Person) : this(name) {
        parent.children.add(this)
    }
}
```

构造函数在类继承时需要显式调用，其参数列表可以重写。在kotlin的特殊类下，构造函数还有些特性，比如伴生类就没有构造函数，这些准备放到kotlin类系列下再写~

### 委托函数

准确地说，kotlin中的委托不叫委托函数，而是分为类委托和委托属性。只不过感觉委托这个点和函数有点搭边，所以放在这。如果你不清楚委托模式，可以先翻阅一下相关博客，不过记住其核心理念就够了：

> 操作对象将某段逻辑的处理工作，交给另外一个辅助对象去做

##### 类委托

假如有一个coder名叫ljj，是美帝湾区大佬，每天需要做这些事，我们用一个接口收拢起来

```kotlin
interface toDoList {
	fun doWork()
	fun moYu()
}
```

但是ljj很懒惰，不想work，只想摸鱼。所以他有一个绝妙的想法，把工作外包给国内coder，名叫coder996。怎么搞呢

```kotlin
class ljj(coder: coder996): toDoList by coder {
	override fun moYu() { println("ljj is 摸鱼ing") }
}

class coder996: toDoList {
	override fun doWork() { println("修福报ing")}
	
	override fun moYu() { println("国内coder怎么能摸鱼呢") }
}

fun main() {
    val LJJToDoList: toDoList = ljj(coder996())
    LJJToDoList.doWork()// 输出 修福报ing
}
```

我们通过by关键字，将接口中的方法实现转移到另一个类里。对于外部调用而言，它只看到ljj正在努力work，却不知work的另有其人。而这里的coder996的实例，就是前面所说的辅助对象

##### 委托属性

基本思想和类委托完全一样，可以参考这篇博客https://blog.csdn.net/baidu_39589150/article/details/111908226

#### 应用

委托的应用较为广泛，最常见的是和lazy函数结合，实现懒加载来降低内存开销

lazy函数是一个原生的高阶函数，创建一个Delegate对象，并把一个lambda参数传入这个委托对象。Delegate类是kotlin专门为委托设计的类。

在lazyObject没有被使用之前，其不会进行初始化，其变量只记录类型信息。当首次调用lazyObject时，会触发lazy的lambda表达式走一遍初始化，然后执行逻辑。再之后调用lazyObject时，就和调普通属性没区别

```kotlin
class MyClass {
	fun moyu() {}
}
val lazyObject: MyClass by lazy {
    MyClass()
}
fun main() {
    if ("ljj" == "tired") {
    	lazyObject.moyu()
    }
}
```

### 覆盖函数

这个词也是我生造的，事实上应该叫作“函数覆盖”，是一种重写函数的语法，在大部分编程语言中都支持。与之类似的还有一种语法叫作 “函数重载”，放在这里一起讨论。

函数覆盖：发生在父类与子类之间，简而言之就是子类和父类实现了同名方法，子类对象调用该方法时，优先使用子类自己的实现。（事实上，属性的覆盖和函数覆盖几乎一模一样）

```kotlin
//抽象类的方法可以且必须被重写
abstract class A {
    abstract fun doA()
}
//接口中的方法如果没有方法体，则实现类中必须重写该方法。如果有方法体，不必重写
//接口中的方法默认是open的
interface B {
    fun doB()
    fun doSomething() {}
}
//只有声明了open的类才能被继承，只有声明了open的方法才能被重写
open class C {
    open fun doC() {}
    open fun doSomething() {}
}
//子类声明override来重写方法。重写后的方法默认是open的，可以被继续重写。
//但可以显式声明final来禁止继续重写
class D: A(),B {
    override fun doA() {}
    final override fun doB() {}
}
//如果父类或父接口中有同名方法，则子类必须重写该方法避免歧义
class E: C(),B {
    override fun doC() {}
    override fun doSomething {
        //通过super可以调用父类的方法。用<>可以指定父类
        super<C>.doSomething()
    }
}
```

上述代码中遇到的修饰符的含义如下

| 修饰符   | 作用                                       | 备注                                              |
| -------- | ------------------------------------------ | ------------------------------------------------- |
| final    | 声明类可以被继承，或方法不能被重写         | `kotlin`中的所有类和方法默认都是final的           |
| open     | 声明类可以被继承，或方法可以被重写         | 需要显式声明                                      |
| abstract | 声明类必须被继承，或方法必须被重写         | 只能在抽象类使用                                  |
| override | 重写父类或者接口中的成员（包括属性和方法） | 如果没有使用final表明，子类重写的成员默认是open的 |
| super    | 子类调用父类的方法                         | 常用                                              |

函数重载：指在同一个类或者父类与子类之间，若干个函数名相同，但参数列表不同，返回值类型可同可不同的函数，被称为重载函数。

```kotlin
//下面四个函数都是重载函数
interface BaseA {
    fun doS(tmp: Int): String
}

class A:BaseA {
    fun doS(): String {	return "1" }
    
    override fun doS(tmp:Int): String {	return tmp.toString() }
    
    fun doS(prop1:Int, prop2:String) { print(prop1.toString() + prop2) }
}
```

#### 应用

无论是覆盖还是重载，在真实开发中都是很常用的。以两点为例

- 自定义View/Layout或者其他组件。实现时肯定要继承原生的组件或者工程历史逻辑的组件，如果你想改改显示逻辑，大概率要重写一些方法和属性
- 多参数的工厂类。这个在`java`中更常见一些，由于`kotlin`的方法支持默认值，所以更偏向用参数列表+默认值实现。但如以下的情况仍可能遇到

```kotlin
//重载次构造函数
class MyView : View {
    constructor(ctx: Context) : super(ctx)

    constructor(ctx: Context, attrs: AttributeSet) : super(ctx, attrs)
}
```

### 挂起函数

这个是kotlin协程引入的概念，本人还没学会协程，所以此处只写下简单示意

**kotlin用suspend关键字标记一个函数，称这个函数是挂起函数**

想深入学习的请参考以下博客

- 扔物线大佬  https://rengwuxian.com/kotlin-coroutines-2/
- 官方文档 https://www.kotlincn.net/docs/reference/coroutines/composing-suspending-functions.html
- 简洁概括 https://blog.csdn.net/qq_39969226/article/details/101058033

### 泛型函数

泛型字面意思是泛化的类型，也就是类型不固定，而是一个范围。该概念在众多语言中都有，而以java为代码的jvm语言，由于其最终的编译都要基于jvm的类型规定，所以这些语言中泛型的概念比较互通。下面从类型开始介绍泛型

**类 != 类型**

<img src="https://upload-images.jianshu.io/upload_images/2613397-6986a322d2f21ded.png?imageMogr2/auto-orient/strip|imageView2/2/w/792/format/webp" alt="img" style="zoom:50%;" />

基本的泛型函数

```kotlin
//这里的T只是一个泛型符号，随便用什么都行，比如T、E、TMD、NB等等
fun doSomething<T> (prop: T) { print(prop.toString()) }
```

#### 型变

定义是类型转换后的继承关系。kotlin中把型变分为协变、逆变和不变。如例

- 协变：A是B的父类，且`C<A>`是`C<B>`的父类，称`C<T>`是一个协变类
- 逆变：A是B的父类，但`C<A>`是`C<B>`的子类，称`C<T>`是一个逆变类
- 不变：A是B的父类，但`C<A>`和`C<B>`没有任何关系。默认模式

假设没有协变和逆变的引入，所有泛型都是不变的，考虑下面这个例子

```kotlin
open class Person 
class Student: Person()
//一个包装类
class Handler<T> (val prop: T) {
    fun doSomething() { print(prop.toString()) }
}
//List<T>是一个泛型类
fun main() {
    var person: Handler<Person> = Handler<Person>(Person()) 
    val student: Handler<Student> = Handler<Student>(Student())
    person = student //Handler<Person> != Handler<Student> 编译报错
}
```

是不是感觉非常不合理，因为Student是Person的子类，Person的能力，Student全都有，所以把Student的包装类对象赋给Person的包装对象是安全的，但由于类型不变的特性，过不了编译。

所以有了协变的概念。kotlin通过修饰符out来实现协变。然后上述的赋值操作就能进行了

```kotlin
class Handler<out T> (val prop: T) { //do Something }
//<out T>在这里等价于java中的<? extend T>
//Handler<Student>是Handler<Person>的子类型
```

同理，考虑下面情况

```kotlin
open class Person 
class Student: Person()
//<in T>等价于java中的<? super T>
class Handler<in T> {
    fun doSomething(prop: T) {
        print(T.toString())
    }
}
//List<T>是一个泛型类
fun main() {
    //Handler<Person>是Handler<Student>的子类型
    var person: Handler<Person> = Handler<Person>()
    var student: Handler<Student> = person
    //合法，因为Student一定是<? super Student>的子类型
    student.doSomething(Student())
    //不合法，因为<in T>规定T可能是Student及其父类，当T是Student时，Person类型不是它的子类型
    student.doSomething(Person())
}
```

#### 泛型约束

顾名思义，就是给T加一些范围限制。比如

```kotlin
//用冒号限制上界，表示T为Animal和Aniaml的子集
class Monster<T : Animal>
//如果有多个上界，用where展开，并取交集
class Monster<T> where T : Animal, T : Food
```

kotlin还提供一个所谓星投影的东西。注意Nothing类是kotlin一个原生类，无法实例化，表示一个不存在的值。

- 对于 `Foo <out T : TUpper>`，其中 `T` 是一个具有上界 `TUpper` 的协变类型参数，`Foo <*>` 等价于 `Foo <out TUpper>`。 **这意味着当 `T` 未知时，从`Foo<*>`中取出的值都会被当作 `TUpper`类型**
- 对于 `Foo <in T>`，其中 `T` 是一个逆变类型参数，`Foo <*>` 等价于 `Foo <in Nothing>`。 **这意味着当 `T` 未知时，无法向`Foo<*>`中写入任何值**
- 对于 `Foo <T : TUpper>`，其中 `T` 是一个具有上界 `TUpper` 的不型变类型参数，`Foo<*>` 对于读取值时等价于 `Foo<out TUpper>` 而对于写值时等价于 `Foo<in Nothing>`

#### 类型擦除

类型擦除是kotlin和java实现泛型用到的技术之一。即泛型信息只存在于代码编译阶段，编译完成后都变成了默认的Any?。这个过程被称为类型擦除。具体请参考《Java编程思想》中泛型章节

#### 应用

可以说非常广泛了。贴一个博客 https://www.jianshu.com/p/b25966f1d699 介绍了几种。另外，一些工具函数也常常用到泛型

### 回调函数

个人理解，计算机中的回调可以追溯到中断的概念。当A程序需要等待B程序结束后才能执行，那么A程序需要每隔一段事件去询问B程序完成了没，为了保证A的响应灵敏性，这个询问的频率就要很高，显然这种做法很蠢。所以出现了中断，当B结束后，发一个中断信号给A，然后A开始执行。

回调是同样的道理，A对象传给B对象一些参数让B去执行某任务小c，等B执行完后返回一个message给A，A根据这个message去执行任务d。但如果B执行c的过程是耗时的，那么A就有两种选择：

- 持续空转等待。**称为同步回调模式**。这种写起来最简单，B只需要把A需要的值打包成message返回即可
- 继续向下执行。**称为异步回调模式**。这种是最常用写法。B需要通过一些编程技巧持有A的引用，在结束任务后，通过这个引用调用A的方法，去执行d

上述是基本思想，java中的回调请参考这篇博客  https://cloud.tencent.com/developer/article/1676582

- 反射
- 直接调用
- 接口调用：java中最常用
- Lambda表达式

由于kotlin对lambda强大的支持，kotlin中虽然仍可以用接口实现回调，但更推荐用lambda实现回调。下面是个例子

```kotlin
//对应上文中的B
class Student() {
    var homework: String? = null
    var checkHomework: ((String?) -> Unit)? = null
    //doHomework是个耗时操作
    fun doHomework() {
        print("doing homework...\n")
        print("$homework has been done.\n")
        homework = null
        //结束之后通过lambda调用A的方法
        checkHomework?.invoke(homework)
    }
}
//对应上文中的A
class Teacher {
    fun dispatchHomework(student: Student) {
        student.homework = "Coding"
        //通过一个lambda作为回调的处理函数
        student.checkHomework = { homework:String? ->
            if (homework == null) {
                print("Good")
            } else {
             	print("Bad")   
            }
        }
    }
}

fun main() {
    val LiHua = Student()
    Teacher().dispatchHomework(LiHua)
    LiHua.doHomework()
}
```

更多关于lambda的知识请见 https://blog.csdn.net/ljjliujunjie123/article/details/118421873

## 总结

kotlin函数这个系列，虽然只有两篇文章，但总计有12000多字，属实写了个小论文....

基本上总结了我从啥都不会的学生，勉强入门android开发的过程中，对函数的认知。其中引用了很多前辈大佬的博客和内容，如有侵权，私聊速删。如有错误，恳请斧正。

