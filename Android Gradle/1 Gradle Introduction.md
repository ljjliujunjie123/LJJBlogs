> 本系列是学习Gradle技术中的一些总结，只关心与Android相关的部分，其余的应用暂不涉猎
>
> 主要参考《Android Gradle权威指南》一书，本书作者博客https://www.flysnow.org/
>
> 本文主要介绍Gradle的设计目的，基本思想，部分内容翻译自 https://docs.gradle.org/current/userguide/what_is_gradle.html 
>
> 正文是原文，个人解读以引用形式插入

Gradle 是一门 DSL , 基于 Groovy 语言设计（从Gradle 4.0开始，也支持使用kotlin编写），为了解决软件开发中的自动化构建问题

> DSL = Domain Specific Language  领域特定语言。参考 https://blog.csdn.net/u010278882/article/details/50554299
>
> 如果对自动化构建这个概念有疑问的，参考https://xie.infoq.cn/article/2e99f473f771fd189e0a82188
>
> 如果对Groovy语言不熟悉的（Gradle只用到它最简单的语法，同时它和kotlin也很像）可以参考 https://blog.csdn.net/u012165769/article/details/118904026
>
> 对kotlin不熟悉的，参考谷歌教程 https://developer.android.google.cn/kotlin

### Gradle是一个通用的构建工具

Gradle允许你构建几乎任意软件，因为它对你要构建什么以及如何构建只做了很少的约束。Gradle最显著的限制是目前的依赖管理，只支持Maven和lvy兼容的库

限制少并不意味着你需要为了构建执行一大堆操作。Gradle通过添加一层约定和插件的预构建，使得构建Java项目变得很轻松。你甚至可以创建和发布自己的插件，来实现个性化的构建功能

### Gradle的核心是Task

Gradle的构建模型是一个有向无环图（DAG），每个节点是一个Task，称为任务图。 这意味着构建一个项目需要先定义一个任务的集合，然后根据任务的依赖关系把它们连接成一个DAG。一旦任务图创建完成，Gradle就能决定每个任务的构建顺序并开始执行

下面展示了构建的概念图，左图是抽象层次，右图是以构建Java项目为例

![Example task graphs](https://docs.gradle.org/current/userguide/img/task-dag-examples.png)

几乎所有的构建流程都能被组织成一个DAG，这是Gradle如此灵活的原因之一。这个任务图可以用插件或者脚本来定义，任务之间由它们的依赖关系连接

任务由这三方面构成：

- 动作（Action） ：指做一些操作，比如复制文件或者编译代码
- 输入（Input）：动作需要的数值、文件、路径等等
- 输出 （Output）：动作修改或生成的文件、路径等等

事实上，上面这三个方面都是可选的，像标准生命周期任务甚至没有任何动作，它们仅仅将多个子任务聚合在一起而已。另外，Gradle的增量构建功能相当强大和方便，所以尽可能不要用`clean`命令，除非你真地需要构建一个干净的工程

> 这一小节个人觉得很重要。Gradle的构建思想，任务的有向无环图
>
> 关于文档最后说尽可能不要用clean命令，我保持怀疑。就个人经历来说，编译超大规模的app时，基本上需要经常执行 gradlew clean 命令，这并不是说gradle本身的增量构建有问题，而是工程太大，难免有一些诡异的错误。。所以这点上不要太死板

### Gradle有一些固定的构建阶段

Gradle运行编译脚本的三个阶段：

1. 初始化

   设置环境，决定哪些工程会参与这个脚本运行

2. 配置

   构建任务图，决定执行哪些任务，以及它们之间的执行顺序

3. 运行

   执行选定的任务

优秀的编译脚本应该是声明式的配置而非命令式的逻辑。也就是说在配置阶段就应该能解读所有的配置。但是，像`doLast {}` 和 `doFirst {}`代码块中的内容，只有到运行期才能被评估。.

关于配置阶段的另一个要点是，每次运行都会把配置阶段涉及的所有任务重新评估一遍。所以不要在配置阶段做一些耗时操作。Gradle提供了构建扫描工具来帮助你发现这种问题。

> 这一段先是讲了编译脚本的三个过程，再强调配置阶段的一些特点。至于它说的问题，实践时碰到了再理解

### Gradle是可扩展的

如果仅仅用Gradle提供的编译逻辑就能编译你的工程，那自然是很好的，但可惜这种情况很罕见。 大多数情况一些特殊需求需要你增加一些特有的编译逻辑。

Gradle提供了下列这些方法允许你扩展编译逻辑

- 自定义任务类型

  如果你希望执行某种任务，官方的任务类型无法做到，那你可以写一个自己的任务类型。通常最好将自定义任务类型的源文件放在buildSrc目录或打包好的插件中。然后你可以像用官方提供的任务类型一样使用自定义类型

- 自定义任务动作

  你可以通过 [Task.doFirst()](https://docs.gradle.org/current/dsl/org.gradle.api.Task.html#org.gradle.api.Task:doFirst(org.gradle.api.Action)) 和 [Task.doLast()](https://docs.gradle.org/current/dsl/org.gradle.api.Task.html#org.gradle.api.Task:doLast(org.gradle.api.Action)) 这两个方法，在官方任务之前或之后增加一些自定义的动作

- 为项目或任务扩展属性

  你可以为项目或任务扩展自定义的属性，然后在任意编译逻辑中调用它们

- 自定义约定

  约定是一种有效的工具来简化编译，帮助用户更容易理解和使用。你可以编写自己的插件，提供约定，它们只需要为构建的相关方面配置默认值

- 自定义编译模型

  除了任务、文件和依赖配置之外，Gradle还允许你在构建中引入新的概念。你可以在大多数语言插件中看到这一点，它们将源集的概念添加到构建中。适当的构建过程建模可以极大地提高构建的易用性和效率

> 这一段也挺重要，大概讲了作为开发者该如何自定义功能。这五种方法看起来前三种比较常用

### 把构建脚本当作API

我们很容易把Gradle的脚本认为是可执行的代码，因为它们确实是代码。但必须注意一点，一个优秀的Gradle脚本应该是在**声明编译该软件需要的步骤，而不是在描述这些步骤是如何工作的**。

所以Gradle的脚本虽然是代码，但Gradle强大的构建能力并不是因为它们是代码，这完全是一个误解。能力的来源是底层的任务图模型和Gradle脚本提供的api能力。正如我们推荐的最佳实践一样，尽可能不要在Gradle脚本中出现大量命令式代码

把Gradle脚本单纯看作代码也不是完全没有益处，在理解它的语法时很有帮助。另外，因为Gradle运行在JVM上，所以Gradle脚本可以使用标准的Java API。比如Groovy脚本可以调用Groovy APIs，而Kotlin可以调用Kotlin APIs

> 这一部分在反复强调声明式和命令式，这听起来非常抽象。
>
> 我的个人的简单理解是，脚本中的代码尽可能不要依赖外部的属性，而是仅仅依赖于传入脚本的参数，也就是前面task构成中的input。如果能做到这一点，这个脚本自然是一个高内聚的稳定的脚本，输入相同，输出就一定相同，那不就是个API吗。
>
> 而如何理解命令式，我觉得最常见的例子就是Android Activity的onCreate方法，我们在这个回调中进行一系列业务逻辑，这个onCreate中的语句就可以看作命令语句
