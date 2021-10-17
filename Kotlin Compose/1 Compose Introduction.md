> Compose 官方文档 https://developer.android.google.cn/jetpack/compose
>
> Compose 中文手册 https://compose.net.cn/

本系列是我学习compose过程中，对官方文档的翻译和解读，以及实验性的Demo工程。主要参考官方文档和中文手册

本文翻译内容:  https://developer.android.google.cn/jetpack/compose/why-adopt?hl=en

### 环境配置

请参考中文手册教程搭建环境。https://compose.net.cn/start/

注意kotlin版本和compose版本的匹配关系，compose第一个稳定版于2021年发布，api随着版本迭代可能会有较大调整

### Compose 介绍

官方文档用 `Why Adopt Compose`作为引言，不知道此刻的读者是为了什么接触/学习/使用Compose，本人很诚实地回答是："内卷"。google 既然大力推出，业界又逐渐接受，这么多人开始学，那我也得开始啊，至于性能更好，设计更优雅什么的，和我一个打工人有啥关系

牢骚两句回归正题，既来之则安之，让我们好好研究一下compose到底好在哪

> ### Why Adopt Compose
>
> Jetpack Compose is Android’s modern toolkit for building native UI. It simplifies and accelerates UI development on Android bringing your apps to life with less code, powerful tools, and intuitive Kotlin APIs. It makes building Android UI faster and easier. While creating Compose we worked with different partners who experienced all of these benefits first hand and shared some of their takeaways with us.
>
> Compose是安卓为构建原生UI设计的现代工具库。它通过更少的代码，更强大的工具，和更符合直觉的api，能简化Android中开发UI的流程。它是构建UI变得更快更容易。在开发Compose过程中，我（Google）让一些合作伙伴（如Twitter）体验了下，他们说体验很好，并狠狠地夸了我们

开篇是商业互吹，跳过

> ### Less code 
>
> Writing less code affects all stages of development: as an author, you get to focus on the problem at hand, with less to test and debug and with less chances of bugs; as a reviewer or maintainer you have less code to read, understand, review and maintain.
>
> 更少的代码会影响整个app的开发流程。作为开发者，你可以把精力集中在手头的难题上，不必进行大量的测试和debug，因为compose降低了你写出bug的可能。作为代码审核者，你可以阅读、理解、检查、维护更少的代码
>
> Compose allows you to do more with less code, compared to using the Android View system: Buttons, lists or animation - whatever you need to build, now there’s less code to write. Here’s what some of our partners say:
>
> - “*For the same Button class it [the code] was 10x of magnitude smaller.*” (Twitter)
> - “*There’s also a significant reduction for any screen built with a RecyclerView, which the majority of our screens are.*” (Monzo)
> - “*We were very pleased to see how few lines were required to create lists or animations in our app. We’re writing fewer lines of code per-feature, which allows us to focus more on delivering value to our customers.*” (Cuvva)
>
> 相对于Android 以前的View系统，使用compose可以用更少的代码实现更多的功能，像按钮、列表和动画等等任何你需要的，都不必再写繁琐的代码，下面这些公司的话可以佐证。（商业互吹的就不翻译了）
>
> The code you’re writing is written only in Kotlin, rather than having it split between Kotlin and XML: “*It’s much easier to trace through code when it’s all written in the same language and often the same file, rather than jumping back and forth between Kotlin and XML*” (Monzo)
>
> Compose的代码只需要用kotlin写就行，不必像View系统那样拆成kotlin和xml两部分。相同编程语言的代码显然更容易传输和阅读，不必在代码文件和xml之间跳来跳去
>
> Code written with Compose is simple and easy to maintain whatever you’re building. “*The layout system of Compose is conceptually simpler so it’s easier to reason about. Reading the code for complicated components is easier as well.*” (Square)
>
> Compose的代码更容易看得懂，因为它的布局思路更清晰更简单，面对复杂的布局也更容易理解

这段在讲compose能用更少的代码开发更多的功能，确实是好东西，也是compose兴起的重要原因

> ### Intuitive 
>
> Compose uses a declarative API, which means that all you need to do is describe your UI - Compose takes care of the rest. The APIs are intuitive - easy to discover and use: “*Our theming layer is vastly more intuitive and legible. We've been able to accomplish within a single Kotlin file what otherwise extended across multiple XML files that were responsible for attribute definitions and assignments via multiple layered theme overlays.*” (Twitter)
>
> Compose的声明式API意味着，开发者只需要声明这个UI需要的内容，构建的部分交给Compose。这些API设计得非常符合直觉。Twitter 说它尝试在一个kt文件替代了之前多个xml文件嵌套才能实现的功能
>
> With Compose, you build small, stateless components that are not tied to a specific activity or fragment. That makes them easy to reuse and test: “*We set a goal for ourselves to deliver a new set of UI components that were stateless, easy to use and maintain, and intuitive to implement/extend/customize. Compose really provided a solid answer for us in this.*” (Twitter)
>
> 使用Compose能构建出更小更稳定的，且和activity或fragment解耦的UI组件，这有利于复用和测试。Twitter在吧啦吧啦。
>
> In Compose, state is explicit and passed to the composable. That way there’s one single source of truth for the state, making it encapsulated and decoupled. Then, as app state changes, your UI automatically updates. “*There’s less to hold in your head while you’re reasoning about something and less behavior that is outside of your control or poorly understood*” (Cuvva)
>
> Compose中，UI的状态显示地传入UI组件，状态只有一个来源，这符合封装和解耦的原则。当状态改变，UI自动跟着改变

这段说明Compose的UI结构更清晰简明，解耦性好。至于所说的状态单一来源，UI随状态自动刷新，其实用View系统完全能做到，只不过囿于开发者水平参差不齐，以及业务复杂多变，协同开发时总会有人写出耦合混乱的代码，破坏状态结构，所以Compose干脆直接实现好，从源头上解决问题

> ### Accelerate Development 
>
> Compose is compatible with all your existing code: you can call Compose code from Views and Views from Compose. Most common libraries like Navigation, ViewModel and Kotlin coroutines work with Compose, so you can start adopting when and where you want. “*Interoperability was where we started integrating Compose, and what we found was it 'just works'. We found that we didn’t have to think about things like light and dark mode, and the whole experience was incredibly seamless.*” (Cuvva)
>
> Compose能和现存的View代码兼容，像Navigation，ViewModel和kotlin协程这些库也都支持Compose，所以开发者可以方便接入Compose。Cuvva 吧啦吧啦
>
> Using the full Android Studio support, with features like live previews, you get to iterate and ship code faster: “*Previews in Android Studio have been a big time saver. Being able to build out multiple previews also saves us time. Often we need to check a UI component in different states or with different settings -- like error states, or with a different font size, etc. With the ability to create multiple previews we can easily check this.*” (Square)
>
> 通过AndroidStudio的支持，Compose的代码支持预览功能，这能帮助开发者加速代码调试

这段说明Compose的兼容性好，以及支持预览能力。这个预览能力确实要夸一夸！毕竟开发大型工程，编译一次都十几分钟，只能被迫摸鱼（不是），遇到要发版的紧急日子，等编译堪比坐牢。。就是不知道这个预览能力和真机的兼容如何，能不能替代真机测试（估计是不行....）

> ### Powerful 
>
> Compose enables you to create beautiful apps with direct access to the Android platform APIs and built-in support for Material Design, Dark theme, animations, and more: “*Compose has also solved more than declarative UI -- accessibility apis, layout, all kinds of stuff have been improved. There are fewer steps between the thing you want to make and actually making it*” (Square).
>
> Compose允许你通过官方api和预设的设计图，开发出漂亮的app
>
> With Compose, bringing movement and life to your apps through animations is quick and easy to implement: “*animations are so easy to add in Compose that there’s very little reason not to animate things like color/size/elevation changes*” (Monzo), “*you can make animations without requiring anything special -- it’s not different than showing a static screen*” (Square).
>
> 使用Compose能快速方便地向app中注入运动和生命的元素，其实就是动画啦
>
> Whether you’re building with Material Design or your own design system, Compose gives you the flexibility to implement the design you want: “*Having Material Design separated from the foundation has been really useful to us as we’re building our own design system that often requires differing design requirements from Material.*” (Square)
>
> 不管你是用Material Design还是自己的设计系统，Compose都预留了足够的灵活性来支持

这段对于大型app开发者其实没太大意义，因为设计的部分有单独的设计师岗位，原生的设计几乎用不到。

### 快速入门的体验性工程

参考中文手册复现的demo  https://compose.net.cn/tutorial/，这是一个相当不错的入门教程！

工程链接	https://github.com/ljjliujunjie123/myASdemos/tree/main/ComposeDemo

效果

<img src="E:\ljj的博客\pictures source\微信图片_20211017145606.jpg" style="zoom:33%;" />