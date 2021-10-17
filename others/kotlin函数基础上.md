kotlin号称更好的java，不仅支持java的绝大部分语法，还新增了非常多语言特性。函数作为编程语言最重要的核心（我认为没有之一），kotlin的函数对于像我这样的初学者来说“面目狰狞”，本文记录了我学习过程中遇到的各种与函数有关的概念，并对各自的原理做一点点探索。

本文涉及概念：顶层函数、成员函数、抽象函数、lambda函数、高阶函数、嵌套函数、内联函数

剩余概念准备放到下一篇：扩展函数、匿名函数、标准函数、构造函数、委托函数、覆盖函数、挂起函数、泛型函数、回调函数

（如果全都很熟悉就别来csdn摸鱼了，你已经是大佬了，赶紧撸代码吧）

### 函数的概念

几乎所有编程语言都有函数，古老的x86汇编都支持LABEL-RET的结构实现函数。那么如何给函数下定义呢，这太难了，而且也没必要。因为不同的语言函数之间的语法相差甚远，但其思想万变不离其宗。所以借用java中的概念，**函数就是一段可能被多次执行的代码块**。

在kotlin和java中，函数更多被称为方法（function），这只是一种称呼上的差异，选择你喜欢的即可。kotlin中基本函数语法如下

```kotlin
fun demoOne(x: Int): String {
  	val res = x.toString()
  	return res
}
```

和其他语言别无两样，无非是函数关键字（fun）、函数名、参数列表、参数类型、返回类型、函数体、返回语句。kotlin在这些概念的基础上增加了非常多的语言特性，在下文中会一一涉及到。

但有一个比较特殊的点需要先注意下，kotlin不能返回null，而是返回Unit。表示该函数返回一个没有意义的值。注意是没有意义，不代表没有返回值。事实上Unit是一种类型。其源码如下。Unit用法和java的void相似，但原理不同。

```kotlin
public object Unit { 
  	//一个全局的单例，只实现了toString方法
  	//其父类是Any,是kotlin中一切类的根结点
    override fun toString() = "kotlin.Unit"
}
```

### 顶层函数

《kotlin核心编程》书中提到，kotlin中类和函数都是一等公民。这个比喻让我一度困惑，直到我反编译了下面的代码

```kotlin
//新建一个demo.kt文件
//这个a就是一个顶层属性
var a = "fuck code"
//这个demoOne就是一个顶层函数
fun demoOne() {
    print(a)
}
```

```java
public final class DemoKt {
   @NotNull
   private static String a = "fuck code";

   @NotNull
   public static final String getA() {
      return a;
   }

   public static final void setA(@NotNull String var0) {
      Intrinsics.checkParameterIsNotNull(var0, "<set-?>");
      a = var0;
   }

   public static final void demoOne() {
      String var0 = a;
     	//这里会多一个奇怪的bool变量，没研究过kotlin编译
      //有懂的大佬可以指教一下
      boolean var1 = false;
      System.out.print(var0);
   }
}
```

顶层函数就是一个直接声明中某文件中的函数。这个文件会被编译成一个不可被继承的类，类名就是文件名。而这个文件中的顶层函数被编译成该类的静态成员方法。顶层属性也是相同的。

所谓一等公民，就是能不能在一个文件里直接声明。

##### 应用

顶层函数的用法，目前我只接触过一种，代码如下

```kotlin
//创建一个 MyExtensions.kt 文件
//用来收集该package中的所有扩展函数

//这是个扩展函数，也是个顶层函数
fun String.mExtensionOne(): Unit {
		print("Extension one")
}

fun Int.mExtentionTwo(a: Int): Int {
  	return a+1
}
```

### 成员函数

这个概念就很简单，函数作为一个类的成员，称为成员函数。

作为初学者，我一度迷惑于对于一个类，函数和属性究竟有什么区别。《深入理解JVM》书中解释到，类被解析成class文件，其中存在字段表和方法表两种数据结构。字段表修饰成员变量，方法表修饰成员函数。二者都有一个属性叫做属性表，这也是个表结构，里面存在着描述这个属性或者方法的信息。而属性表里有一个属性叫做“Code”，这个属性还是个表结构，“Code”表里有一个属性叫做"code"，这个属性的value才是函数体被编译后真正的字节码。所以，一个类的函数和属性，都是一个个字段表或方法表，它们有一个成员是属性表结构，里面存放着描述信息。了解到这个程度足以应付开发了。

关于成员函数，还有一点需要注意。下面这段代码中，有三个同名方法demo。只有第一个demo才是A类的成员。在kotlin中

- inner修饰的内部类，持有外部类的一个隐式的引用，注意不是this指针。在内部类里，可以直接调用外部类的属性和方法，但外部类无法直接调用内部类的属性和方法。内部类的this指针仍然是指向内部类的，遇到和外部类属性或方法同名时，也是内部类优先。
- 嵌套类可以粗略理解为外部类的平级类，外部类无法直接调到嵌套类的属性和方法，嵌套类也无法直接调到外部类的属性和方法。（是无法直接，并不是不能，直接new一个对象，那当然是随便调了）

```kotlin
class A {
    var one = "one"
  
    fun demo(x: String = ""){
        print("A $x\n")
    }
  
 	fun Aprint() {
        demo() //运行结果 A
        this.demo(one) // A one
    }

    inner class AA {
        fun demo(x: String = ""){
            print("AA $x\n")
        }

        fun AAprint() {
            demo() // AA
            this.demo(one) // AA one
            //这两个都是内部类的方法
            Aprint() //直接调用外部类的方法
            //运行结果
            //A
			//A one
        }
        
    }

    class AAA {
        fun demo(x: String = "") {
            print("AAA $x\n")
        }

        fun AAAprint() {
            demo() // AAA
            this.demo() // AAA
            demo(one)//报错
        }
    }
}
```

贴一下上面代码的反编译版本，只保留关键部分

```java
public final class A {
   private String one = "one";
	 ...
   public final void demo(@NotNull String x) {
      ...
   }

   // $FF: synthetic method
   public static void demo$default(A var0, String var1, int var2, Object var3) {
      ...
      var0.demo(var1);
   }

   public final void Aprint() {
      demo$default(this, (String)null, 1, (Object)null);
   }
	
	 //注意内部类不是static的，因为持有了外部类的指针，所以必须先等外部类实例化
   public final class AA {
      public final void demo(@NotNull String x) {
         ...
      }

      // $FF: synthetic method
      public static void demo$default(A.AA var0, String var1, int var2, Object var3) {
         ...
         var0.demo(var1);
      }

      public final void AAprint() {
         demo$default(this, (String)null, 1, (Object)null);
         //这里通过显式指定this指针，访问外部类的成员
         //同时没有走demo$default方法进行非空检查
         this.demo(A.this.getOne());
      }
   }

   public static final class AAA {
      public final void demo(@NotNull String x) {
         ...
      }

      // $FF: synthetic method
      public static void demo$default(A.AAA var0, String var1, int var2, Object var3) {
         ...
         var0.demo(var1);
      }

      public final void AAAprint() {
         demo$default(this, (String)null, 1, (Object)null);
         demo$default(this, (String)null, 1, (Object)null);
      }
   }
}
```

### 抽象函数

接触计算机的同学，入门数据结构或者任何一种编程语言，大概率都听过抽象这个词。这个概念含义太深，这篇文章把握不住。所以我们从更简单的角度理解。

抽象函数是普通函数的“领导”，领导决定一个类往哪个方向工作，也就是定义参数列表、返回值等概念性的东西。至于功能的实现，当然是交给打工人了！所以普通函数就需要用override重写抽象函数，实现方法体，而“领导”的方法体永远是空白

所以打工人们，来看一下抽象函数的语法

```kotlin
fun main() {
   var a = AImpl()
   print(a.doSomething(123))
  //输出 123
}

abstract interface IA {
    abstract fun doSomething(x: Int): String
}

interface IAA {
  	fun doSomething(x: Int): String
}

abstract class A {
    //定义抽象函数
    abstract fun doSomething(x: Int): String
}

class AImpl: A(),IA,IAA {
    //覆盖函数
    override fun doSomething(x: Int): String {
        return x.toString()
    }
}
```

这段代码是不是感觉特别扭，我也这么觉得。因为kotlin中的抽象函数是用关键字abstract修饰，同时抽象函数必须写在抽象的“类”里。注意，我是说抽象的类，不仅仅是抽象类，abstract完全可以修饰interface以及其内部的方法。原因？反编译下，会发现这tm不一样吗。

```java
public abstract interface IAA {
  public abstract doSomething(I)Ljava/lang/String;
}

public abstract interface IA {
  public abstract doSomething(I)Ljava/lang/String;
}
```

这个点有啥意义呢？可以用来回答一个经典的面试题：如何用抽象类实现接口。因为接口的本质就是抽象，所以只需要把所有方法都定义成没有方法体的抽象方法即可（这个点还有很多业务上的妙用，可以搜搜看）

##### 应用

抽象函数最大的用途其实就是刚刚提的，实现接口这个点。更准确的说是增强接口。因为抽象类可以有属性以及属性的初始化，还可以有非抽象方法。而kotlin的接口只能有不带初始化的属性和抽象方法。所以抽象类更灵活一些。但真正业务开发里，稳定、可维护其实比灵活的能力更重要，所以反而是接口应用更广泛。

### lambda函数

这块就太复杂了，三言两语不可能说清楚的，可以看看我的另一篇文章。[kotlin lambda](https://blog.csdn.net/ljjliujunjie123/article/details/118421873?spm=1001.2014.3001.5502)

### 高阶函数

##### 函数类型

前文提过kotlin中函数是一等公民，所以函数是可以被存储在变量与数据结构中的、或作为参数传递给高阶函数、或从高阶函数返回。这句话是官方文档原话。既然函数能被视为变量，那它一定有类型。kotlin提供了Function0-Function22作为函数的类型，数字标号代表参数个数。取其中一个的源码如下

```kotlin
//P是param，R是return
public interface Function2<in P1, in P2, out R> : Function<R> {
    public operator fun invoke(p1: P1, p2: P2): R
}
```

kotlin提供两种声明函数的方法，一个自然是fun关键字，另一个是如下语法

```kotlin
val demoOne: (Int) -> Unit = {}
```

详细语法细节见官方文档 [kotlin官方文档 函数的类型](https://hltj.gitbooks.io/kotlin-reference-chinese/content/txt/lambdas.html#%E5%87%BD%E6%95%B0%E7%B1%BB%E5%9E%8B)

除了上述的直接声明函数实例，还有如下方法可以

- lambda表达式：见上文
- 匿名函数：见下文
- 顶层、局部、成员、构造、扩展函数

```kotlin
fun main() {
   demoOne()
   print(demoTwo(AImpl(),2))
}

fun playOne() {
    print("1\n")
}

interface IA {
   fun doSomething(x: Int): String
}

class AImpl: IA {
    override fun doSomething(x: Int): String {
        return x.toString()
    }
}
//顶层函数，直接用::操作符
val demoOne: () -> Unit = ::playOne
//成员函数，用类名加::操作符。
//至于这里的参数列表为啥有个AImpl，可以研究下jvm类加载
val demoTwo: (AImpl,Int) -> String = AImpl::doSomething
//其他几类见官方文档
```

- 实现函数类型接口的类的实例

```kotlin
//类之所以能实现这么个奇怪玩意，是因为所有的Function类型都是接口
class IntTransformer: (Int) -> Int {
    override operator fun invoke(x: Int): Int = TODO()
}
val intFunction: (Int) -> Int = IntTransformer()
```

##### 高阶的含义

高阶函数是将函数用作参数或返回值的函数。

如果直接看这个概念，可能不明所以，但了解下函数类型就很简单了。function是接口，一个函数就是实现了function接口的对象，把一个对象送进一个函数，不是很合理吗？事实上高阶函数的知识点，就是理解函数类型，剩下的就是个简单的调用方法，贴一下官方文档的调用例子

```kotlin
fun main() {
    val stringPlus: (String, String) -> String = String::plus
    //带接受者的函数类型，请见本文的lambda函数部分
    val intPlus: Int.(Int) -> Int = Int::plus

    println(stringPlus.invoke("<-", "->"))//手动invoke
    println(stringPlus("Hello, ", "world!"))//小括号重载

    println(intPlus.invoke(1, 1))
    println(intPlus(1, 2))
    println(2.intPlus(3)) 
}
```

##### 应用

应用太多了，以至于都成独立的知识点了，比如lambda函数、扩展函数、标准函数都是高阶函数的使用。总结一句，就是高阶函数让代码更抽象，更灵活，更强大。（也更难读）

### 嵌套函数

嵌套这个概念也是编程界的常见词，远古cpu 8086都支持嵌套中断的能力（中断的嵌套和嵌套函数的原理还不一样，只是举个例子）。高级语言下，不仅有嵌套函数，还有嵌套类。如果想搞懂为啥kotlin、java能支持嵌套，去研究jvm吧，一切的知识都在那里。

回归正题，嵌套函数语法如下

```kotlin
class A {  
   fun play() {
       print("A paly\n")
   }
    
   fun demoOne() {
       fun play() {
           print("one play\n")
       }
       play() // one play
       this.play() // A play
   }
   
   var s = "A\n"
   fun demoTwo() {
       var s = "One\n"
   		 fun play() {
         	 print(s) // One
           print(this.s) // A
       }
       play()
   }
}
```

很简单不是吗，唯一需要注意的不过是作用域的问题。局部作用域总是最优先的，除非用指针强制指定。关于同名函数或属性的优先级，在嵌套类、内部类、覆写函数、是否静态等情况下，表现各异。本文就不写了，我相信各位大佬们用这些语法时，不会丧心病狂地写一堆同名方法（大概吧...）

回想一下前文的高阶函数中，对函数类型的分析。fun关键字是隐式地创建function类型对象，我们还可以显式地创建啊。那这样写算嵌套吗

```kotlin
fun demoOne() {
    val demoOne: () -> Unit = {
        print("1")
    }
    demoOne() // 1 注意同名函数下，局部作用域最高优
    fun demoOne() = { //注意这里的=号不是错误，原因见lambda函数部分
        print("2")
    }
    demoOne() // 2 局部作用域下，变量名是被覆写的
}
```

我觉得完全可以理解为嵌套。

### 内联函数

官方解释：使用高阶函数会带来一些运行时的效率损失：每一个函数都是一个对象，并且会捕获一个闭包。 即那些在函数体内会访问到的变量。 内存分配（对于函数对象和类）和虚拟调用会引入运行时间开销。许多情况下通过内联化 lambda 表达式可以消除这类的开销。

上面这个说法看似很好理解，其实难得一批。大意都能看懂，高阶函数引入更多的对象和更深的调用栈，所以跑得慢。但如何理解这一过程中的内存分配和虚拟调用，够研究jvm很久的。（至少我不懂）先来看下内联的语法

```kotlin
inline fun hello() {//关键字inline修饰函数
  print("hello")
}
fun main() {
	hello()
}
```

很简单，按照定义，hello的方法体在编译期间会直接替换掉main里的hello，使运行期间减少函数调用。但真得是这样吗？函数被创建出来就是用来调用的，如果多调用几次都会引起性能问题，那函数这个概念根本就活不了这么多年。这个问题在《扔物线》的文章中解释得很好，[kotlin 内联](https://juejin.cn/post/6869954460634841101)。

限于对他人知识成果的尊重，不展开写了。只总结一下关键思想

- inline减少了函数调用，但膨胀了字节码总量。只有在高频调用的高阶函数处才能有所优化

上面链接的文章，不仅解释了inline，还有noinline与crossinline这两个相关的概念。

##### 应用

因为内联函数我用得很少，所以找了一篇还不错的文章。其中给了一个很好的例子

```kotlin
//reified是与泛型相关的关键字，见下文泛型函数部分
//Context.newFragment是扩展函数。见下文扩展函数部分
//本函数为context扩展了一个创建fragment的泛型方法，允许传递参数创建不同的fragment
inline fun <reified F : Fragment> Context.newFragment(vararg args: Pair<String, String>): F {
    val bundle = Bundle()
    args.let {
        for (arg in args) {
            bundle.putString(arg.first, arg.second)
        }
    }
    return Fragment.instantiate(this, F::class.java.name, bundle) as F
}
```

[kotlin内联的应用](https://blog.csdn.net/villa_mou/article/details/81910586)