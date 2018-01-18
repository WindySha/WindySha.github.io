---
title: Android AOP编程的四种策略探讨：Aspectj，cglib+dexmaker，Javassist，epic+dexposed
date: 2018-01-18 21:06:30
categories: 
- Android
- Aspectj
tags:
- Aspectj
- AOP
- cglib
- epic
- dexposed
- Xposed
---


---


## **前言**
AOP:面向切面编程(Aspect-Oriented Programming)。

它和我们平时接触到的OOP都是编程的不同思想，OOP，即『面向对象编程』，它提倡的是将功能模块化，对象化，而AOP的思想，则不太一样，它提倡的是针对同一类问题的统一处理。

利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，提高开发效率。

那么AOP这种编程思想有什么用呢，一般来说，主要用于不想侵入原有代码的场景中，例如SDK需要无侵入的在宿主中插入一些代码，做日志埋点、性能监控、动态权限控制、甚至是代码调试等等。

下面介绍Android开发中四种实践AOP编程的方案。

<!-- more -->

## **AspectJ**
AspectJ是一个基于Java语言的AOP框架，提供了强大的AOP功能，其他很多AOP框架都借鉴或采纳其中的一些思想。在Java编程中，AspectJ算是最有名的AOP编程工具，具体有以下几个原因：

 - 功能强大
 - 支持编译期和加载时代码注入
 - 易于使用
 
AspectJ官网为：**[https://eclipse.org/aspectj/][1]**

AspectJ是Java上的项目，并不能直接用在Android上。最早将AspectJ在Android项目的上使用的大神Jake Wharton的这个日志打印工具项目**[Jake Wharton’s Hugo Library][2]**。

AspectJ要想在项目中使用，就必须使用AspectJ的编译器（ajc，一个java编译器的扩展）, 对所有受 aspect影响的类进行织入。为了要在Android上使用，我们需要在Gradle的编译task中增加一些额外配置，使之能正确编译运行。

还好，在AS中集成Gradle编译插件的工作有人已经帮忙做好了，我们直接使用就行。
这个就是沪江网的开源项目：**[gradle_plugin_android_aspectjx][3]**
这个库的接入方法也非常简单，具体可以参考《Android群英传》一书作者徐宜生（在沪江网就职）的这个博客：**[看AspectJ在Android中的强势插入][4]**
他的这篇博客中也详细介绍了AspectJ的各种注解的用法，包括@Before，@After，@Around等等，使用起来并不会太复杂，非常适合在项目中接入使用。

AspectJ的原理其实很简单，就是在代码编译期间，将需要集成的代码插入到目标代码的前面或者后面，以实现代码的AOP。我们通过反编译集成了aspectJ的Apk可以很清楚的知道这一点，具体细节在**[看AspectJ在Android中的强势插入][5]**这个博客中有详细的介绍。

关于AspectJ在实际项目中的应用案例，有两篇博客值得一读。

第一篇是**[安卓AOP实战：面向切片编程][6]**
这篇博客通过几个小例子，讲解在Android开发中，如何运用AOP的方式，进行全局切片管理，达到简洁优雅，一劳永逸的效果。里面介绍了这几种全局切片：
>1. SingleClickAspect，防止View被连续点击出发多次事件
>2. CheckLoginAspect 拦截未登录用户的权限
>3. MemoryCacheAspect内存缓存切片
>4. TimeLogAspect 自动打印方法的耗时
>5. SysPermissionAspect运行时权限申请 


第二篇是**[AOP之@AspectJ技术原理详解][7]**
这篇博客对AspectJ的介绍更加详尽，从使用方法，到反编译源码分析原理，到项目中的应用方式，都有详细的介绍。具体在项目应用上，介绍了这个方面的应用：
>1. 日志打印
>2. 方法耗时监控
>3. 监控Activity页面的停留时间 
>4.  异常处理
>5. 降级替代方案——吐司 

假如不使用上面介绍的沪江网gradle插件，是否有其他方法在Android项目中实现AspectJ的编译时代码注入呢？
有的，国外有个开发者写了一篇文章对此有专门的介绍，主要是在build.gradle文件中添加了一些脚本，也能够在AS中使用，但是脚本的编写有点麻烦，需要熟悉Gradle脚本语言才能比较熟练地掌握好：
**[【翻译】Android中的AOP编程][8]**
针对这种方法，国内也有人专门实践过：**[Android 基于AOP监控之——AspectJ使用指南][9]**

## **cglib + dexmaker**

**[cglib][10]**是一个功能强大，高性能的代码生成包。在Java开发中，动态代码可以使用Proxy类来通过反射代理接口(interface)实现，但cglib不仅仅能够代理接口，它能够为类的非final方法提供代理，为JDK的动态代理提供了很好的补充。

cglib原理是通过字节码技术为一个类创建子类，并在子类中采用方法拦截的技术拦截所有父类方法的调用，顺势织入横切逻辑。由于是通过子类来代理父类，因此不能代理被final字段修饰的方法。

cglib如此强大，但是，它有一个很致命的缺点是：cglib底层是采用著名的ASM字节码生成框架，使用字节码技术生成代理类，也就是通过操作字节码来生成的新的.class文件，（关于class文件以及ASM的介绍可以参考我的博客**[深入理解JVM之Java字节码（.class）文件详解][11]**），而我们在android中加载的是优化后的.dex文件，也就是说我们需要可以动态生成.dex文件代理类，因此cglib在android中是不能使用的。不过，网上已经有人根据dexmaker框架（dex代码生成工具）来仿照cglib库动态生成.dex文件，实现了类似于cglib的AOP的功能。 

详细的用法可以参考这篇博客：**[将cglib动态代理思想带入Android开发][12]**
项目的源码为：**[MethodInterceptProxy][13]**

它的这个项目是在在**[dexmaker][14]**和**[cglib-for-android][15]**库的基础上，修改部分代码后形成的类似cglib框架。很有参考学习的价值！

## **Javassist For Android**
Javassist是什么？

Javassist是一个开源的分析、编辑和创建Java字节码的类库。是由东京工业大学的数学和计算机科学系的 Shigeru Chiba （千叶滋）所创建的。

关于java字节码的处理，目前有很多工具，如BCEL，ASM。不过这些都需要直接跟JVM的操作和指令打交道。相比而言，Javassist要简单的多，完全是基于Java的API，但其性能相比前二者要差一些。尽管如此，在性能要求相对低的场合，Javassist仍然十分有用。

Javassist的官网为：**[http://jboss-javassist.github.io/javassist/][16]**
Javassist的使用方法可以参考：**[用 Javassist 进行类转换][17]** &nbsp;&nbsp;&nbsp;**[Java动态编程之javassist][18]**

要想在Android中使用Javassist来实现AOP功能，需要自定义gradle plugin，在plugin中使用Javassist工具实现代码的插入，然后在项目中引用这个gradle plugin。这里，就涉及到自定义gradle plugin的知识了，这就需要对Gradle语法比较了解。

关于自定义gradle插件以及集成Javassist的方法，此处不作详细介绍，感兴趣的话，可以参阅以下博客，：
**[Android 中使用Javassist][19]**
**[通过自定义Gradle插件修改编译后的class文件][20]**
**[Android动态编译技术:Plugin Transform Javassist操作Class文件][21]**


## **epic + dexposed**
田维术的**[epic][22]**这个项目的AOP原理完全不同于以上几个方案，以上几个方案都是Java开发中已有的技术方案，只不过是移植到了Android中来。它们的根本原理是修改编译生成的字节码文件，对原本的代码进行替换，从而实现了方法的AOP。它们都是在编译时就对代码实现动态代理，而epic这个项目是在运行对代码实现动态代理的，是不是很牛逼！它的主要思想是在native层修改java method对应的native指针。

**[epic][23]**项目是基于阿里巴巴热更新项目**[Dexposed][24]**，而Dexposed是一个基于久负盛名的开源**[Xposed][25]**框架实现的Android平台上功能强大的无侵入式运行时AOP框架。

在Xposed这个框架下，我们可以加载很多插件App，这些插件App可以直接或间接操纵系统层面的东西，比如操纵一些本来只对系统厂商才open的功能。有了Xposed后，理论上我们的插件APP可以hook到系统任意一个Java进程（zygote，systemserver，systemui等等）。功能太强大，自然也有缺点。Xposed需要系统已经root，所以并不适合在项目中使用。

因此，Dexposed对其进行改进，将其中核心方法（处理java方法指针的native方法）抠出来，集成在APP开发中，实现对APP代码的AOP。Dexposed的AOP实现是完全非侵入式的，没有使用任何注解处理器，编织器或者字节码重写器，这一点跟以上几个框架完全不一样。Dexposed实现的hooking，不仅可以hook应用中的自定义方法，也可以hook应用中调用的Android框架的方法。

那既然Dexposed这么强大，使用Dexposed进行AOP就行了，为啥还出来一个epic？？？

Dexposed虽然很强大，但是但是但是，它有个致命的缺点，就是它只能支持Android Dalvik虚拟机！what? 现在的Android虚拟机都是Art虚拟机了，它居然只能支持Dalvik虚拟机，太不实用了！！

那它为啥Dexposed没能支持Art呢？

这是由于Art虚拟机和Dalvik虚拟机差别非常大，而且Art比Dalvik复杂太多，所以Dexposed项目一直没能够成功适配Art虚拟机。而xposed项目也是最近才成功适配上了Art虚拟机，而且适配的方法比较粗鲁，直接修改Art虚拟机的源代码，侵入性非常强。**[rovo89/android_art][26]**

技术的道路上，总有人愿意去披荆斩棘，冲破迷雾。

**[田维术][27]**在他的**[epic][28]**项目终于实现了在Art虚拟机上的运行时Method AOP功能（Dexposed的Art版本）！

他的实现方案主要借鉴了一篇国外的科技论文，这个论文就是Wißfeld, Marvin 的**[ArtHook: Callee-side Method Hook Injection on the New Android Runtime ART][29]**。这篇文章很精彩，讲述了各种Hook的原理，并且还在ART上实现了 dynamic callee-side rewriting 的Hook技术，而且提供了实现的源代码，代码在github上：**[ArtHook][30]**

关于epic的实现原细节，可以阅读田维术的博客：**[我为Dexposed续一秒——论ART上运行时Method AOP实现][31]**

讲的非常精彩，是一篇很有价值的文章！


---
## **总结**

以上就是Android开发中常见的AOP框架的介绍，通过分析对比，可知，最成熟最易用的应该是AsepctJ了，这个框架在Java中已经已经被广泛应用了，移植到android中可以直接上项目使用，并无风险。

cglib + dexmaker的方案也是比较稳定，其主要用来实现方法的动态代理，但功能性和易容性上比AsepctJ稍差一些，不建议在Android项目中使用。

Javassist For Android使用起来比较麻烦，而且效率可能并不会太高。

至于最后一个运行时AOP方案（epic），在这里提出来只是为了开拓一下思路，它暂时只适合在开发调试中来使用，并不适合在Android中直接上项目，毕竟涉及到了Art虚拟机的操作，各个手机厂商可能会对Art做修改，因此，在没有经过各种手机的严苛测试前提下，并不太适合上项目。


  [1]: https://eclipse.org/aspectj/
  [2]: https://github.com/JakeWharton/hugo
  [3]: https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx
  [4]: https://www.jianshu.com/p/5c9f1e8894ec
  [5]: https://www.jianshu.com/p/5c9f1e8894ec
  [6]: https://www.jianshu.com/p/b96a68ba50db
  [7]: http://blog.csdn.net/woshimalingyi/article/details/73252013
  [8]: https://www.jianshu.com/p/0fa8073fd144
  [9]: http://blog.csdn.net/woshimalingyi/article/details/51519851
  [10]: https://github.com/cglib/cglib
  [11]: http://blog.csdn.net/weelyy/article/details/78969412
  [12]: http://blog.csdn.net/zhangke3016/article/details/71437287
  [13]: https://github.com/zhangke3016/MethodInterceptProxy
  [14]: https://github.com/linkedin/dexmaker
  [15]: https://github.com/leo-ouyang/CGLib-for-Android
  [16]: http://jboss-javassist.github.io/javassist/
  [17]: https://www.ibm.com/developerworks/cn/java/j-dyn0916/
  [18]: http://blog.csdn.net/top_code/article/details/51708043
  [19]: http://blog.csdn.net/Deemons/article/details/78473874
  [20]: https://www.jianshu.com/p/417589a561da
  [21]: http://blog.csdn.net/yulong0809/article/details/77752098?locationNum=9&fps=1
  [22]: https://github.com/tiann/epic
  [23]: https://github.com/tiann/epic
  [24]: https://github.com/alibaba/dexposed
  [25]: https://github.com/rovo89/Xposed
  [26]: https://github.com/rovo89/android_art
  [27]: http://weishu.me/
  [28]: https://github.com/tiann/epic
  [29]: http://publications.cispa.saarland/143/
  [30]: https://github.com/mar-v-in/ArtHook
  [31]: http://weishu.me/2017/11/23/dexposed-on-art/