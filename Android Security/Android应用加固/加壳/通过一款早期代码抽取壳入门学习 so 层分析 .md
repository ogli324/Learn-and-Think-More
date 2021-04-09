# 通过一款早期代码抽取壳入门学习 so 层分析 

url：https://bbs.pediy.com/thread-260251.htm



# 1. 前言

文章开始需要提下的就是，在如今看雪论坛的用户一发关于安卓加固的文章动辄就是有关脱壳机、vmp、函数级指令抽取或者各大厂商的加固等技术的情况下，为何我要发一个代码抽取壳的分析，并且还是早期的那种整体抽取、整体还原回去、没有混淆、代码风格十分民风淳朴的那种壳...

 

原因是我开始尝试复现论坛中一些很优秀的关于 so 层的分析帖子时候，尽管很多帖子都力所能及进行了十分详细的说明，可是有些步骤我复现起来还是觉得不太理解或者有点力不从心，很多时候也是因为一些知识点和操作对大佬来说是比较简单的事情，说起来比较琐碎大佬就给一笔带过略了。

 

然后自己也尝试着去找一些 App 的 so 文件或者厂商加固的 App 进行分析，但是移动安全毕竟发展了这么多年，很多保护对抗技术都很成熟了，结果就是很多时候分析计划首先卡在了混淆上了...

 

混淆是件很头疼的事，变量函数名混淆、汇编指令的混淆、ollvm 混淆等这些都很好地阻碍了入门分析者的脚步，很多时候我们只是刚入门的新手，so 层的分析还不太熟练，壳和加固原理也没有那么的清晰，去混淆也没有那么手到擒来，很多文章能浅显的读懂但是没有碰到过一个很好的分析案例来帮我们巩固等等...一言以蔽之，段位太低了。

 

然后问题就出现了，刚入门想提高，然后去找 App 练手分析，结果一看首先就是各种胡里花哨的混淆甚至连 JNI_Onload 函数都找不到，但是想着逆向分析要有耐心，更是想到了看没几天的四哥 scz 刚发的文章 **《有技术追求的人应该去挑战那些崇山峻岭》**，里面说到：

> **“对于某些志存高远的逆向工程爱好者来说，或者对于一些永远充满好奇心的人们来说，破解如果只是拼手速，显得没劲。我要是你们，我就去剁一下Burp企业版。”。**

 

然后给自己打了打鸡血就硬着皮头去怼壳怼加固怼混淆，过了几天因为没啥进展就心灰意冷地放弃了。然后过了些时间感觉自己又可以了，给自己打了鸡血又硬着皮头去怼壳怼加固怼混淆，然后又心灰意冷地放弃...

 

这样重复了几次后我开始意识到问题，进行了深刻的反思，想到自己并没有经历安卓加固从最初开始演变的这个过程，对加固和壳的原理还都还不太熟悉，然后作为入门者一分析就去搞加固很成熟的 App 当然会处处碰壁。但是放弃是不可能放弃了，想了想还是要分析体验一下早期的壳，并以此积累 so 层的分析技术，然后我就开始大量地搜索寻找早期的加固和壳的分析了。

 

啰里啰唆说了这么多，现在回头来看一下，这个决定还是很正确的，这篇文章的代码抽取壳就是找到的一个很好的入门练手案例，以前看过的很多文章但是浅显理解的，都在这次分析过程中给加深了很多理解。以及在分析过程中不理解的努力去搜索都是可以从前人的文章中找到答案，然后再进行深入的分析，整个过程下来自己还是十分有成就感的，同时在搜索过程中也是不断接触新的知识。

 

总的来说，对这个代码抽取壳的分析中学到了太多，这也是我写这篇文章最重要的原因，分享给和我一样 so 层分析入门的新手，用一个很好的案例当作 so 层分析的敲门砖来走出困境、迷惑。

# 2. 关于 App

App 来自一位大佬的博客（https://blog.csdn.net/m0_37344790/article/details/79102031），是加了腾讯御的壳，大佬在博客中进行了对反调试和脱壳的操作说明，我在找到了这篇博客和里面的 App 后进行了简单的分析后心情十分激动，因为十分适合练手分析！

 

一直以来虽然我会不少整体脱壳的操作，但是我对壳原理实现还是很感兴趣的，一直苦于找不到合适的案例，这个 App 就是极佳的练手的对象！并且开始以为这是个一代壳，后面深入分析后发现是二代抽取壳，更是学到了不少。然后很重要的一点就是这个壳几乎没有啥混淆，代码十分民风淳朴，甚至在 so 层的注释都没去掉，很利于分析，让我们把重心放在原理实现上面。

 

然后就是因为是很早的 App 了，我在 Android 8.1 真机上没有跑起来，然后大佬博客里面是用 IDA 在
dvmDexFileOpenPartial 下断脱壳的，这个是 Dalvik 虚拟机的脱壳点，所以用的应该是 Adroid 4.x 的版本。我尝试了几款 Android 4.4 模拟器 App 都没有运行成功 ，最后是在逍遥模拟器 Android 5.1.1 版本成功运行了 App。所以此文的分析是基于 IDA 7.0 和逍遥模拟器 Android 5.1.1 版本。

# 3. 一些知识点

在开始分析壳之前让我们先了解学习一些基础知识点，虽然仅仅看着多少会觉得有点模糊，但是没关系，后面会结合具体实例分析来加深理解。

## 3.1 壳的代理 Application 机制

关于壳的一般性介绍就不多说了，我直接说下我在分析过程对壳加深理解的部分——壳的代理 Application 机制。

 

需要理解的是，壳的目的除了保护代码和对抗逆向分析，还有就是要加载目标 App 的功能 dex 文件或者指令。在这里就涉及到了一个过程，就是应用的运行环境的改变，当加壳的 App 启动后，首先运行起来的是壳的代码，整个应用的运行环境也是属于壳的，然后就引出来的一个很有趣的问题，如何进行改变应用的运行环境呢？

 

现在我们把改变应用运行环境的问题抽象一下，壳的 Application 我们称为 ProxyApplication，我们的目标应用的 Application 称为 DelegateApplication，现在问题就变成了 ProxyApplication 如何替换为 DelegateApplication，这也就是我们要说的壳的代理 Application 机制。

 

如何进行替换了就不详细分析了，在文章最后会给出前人们总结的优秀的相关学习文章和博客链接，以及在后面的分析过程也会结合壳的实现来验证这一部分，这里只简单说下替换的两大步骤。

 

首先要生成用户 Application 的实例，也就是 DelegateApplication，然后获得壳的 baseContext 进行 attach，这时候就是将控制权交给了 DelegateApplication。

 

然后就是要替换掉 API 层的所有 Application 引用，通过反射把 API 层所有保存的 ProxyApplication 对象，都替换成 DelegateApplication 对象，需要替换的部分如下：

- baseContext.mOuterContext
- baseContext.mPackageInfo.mApplication
- baseContext.mPackageInfo.mActivityThread.mInitialApplication
- baseContext.mPackageInfo.mActivityThread.mAllApplications

还有需要妥善处理的部分就是，当应用注册有 ContentProvider 时候，ContentProvider:onCreate()调用优先于Application:onCreate()。

 

当我们在学习了解这些的时候，切莫忘掉 ProxyApplication 替换为 DelegateApplication 过程中的重头戏——Classloader 和 mCookie 与内存加载 Dex 文件部分。

## 3.2 ClassLoader 和 Cookie

让我们先简单地说一下 Dex 文件和 ClassLoader 的关系。

 

我们知道 App 能够运行起来一定因为是我们写的代码编译成的 Apk、Dex 文件或者是 jar 文件被安卓系统加载起来了，知道这个有助于我们理解 ClassLoader 的存在，ClassLoder 负责加载这些文件，就包括我们要重点要说的 Dex 文件。

 

需要知道的是 DexClassLoader 和 PathClassLoader，前者能够加载 jar/apk/dex，而后者只能加载系统中已经安装的 Apk 中的 Dex。两者都有共同的父类 BaseDexClassLoader，在 BaseDexClassLoader 的构造方法中，有一个核心功能的类 DexPathList，负责解析加载文件的类。

 

在 DexPathList 的构造方法中，有一个方法 makeDexElements，makeDexElements 方法判断文件后缀名是否是“dex”，如果是就调用 LoadDexFile 方法加载文件，并且返回一个 DexFile 对象。

 

而我们的 Dex 文件就是在生成这个 DexFile 对象调用它的构造方法时候，被更具体和底层地进行了加载。在 DexFile 的构造方法中调用了一个 native 方法 openDexFile。这个 native 方法返回了一个极为重要的值，让我们记住它并在不断地学习中加深对它的理解，这个值就是 cookie。

 

cookie 在 Java 层就是虚拟机的 cookie 值，在 so 层它是 pDexOrJar 指针，虚拟机在进行查找 Dex 文件中的类方法时候，都是需要对 cookie 进行操作的。

 

查找的过程是，BaseClassLoader类 的 findClass 方法，调用 DexPathList类 的 findClass 方法，然后调用刚才说到的返回的 DexFile 对象的 loadClassBinaryName 方法。在 loadClassBinaryName 方法中把查找的类方法名称、类加载器、cookie 值，作为三个参数传进了最后的 defineClass 方法，这个方法调用了 native 函数来返回一系列调用想要查找的 class。然后就是 loadClass 和 defineClass 的操作。

 

理解 cookie 有助于我们理解很多壳一系列操作的本质。

## 3.3 内存加载 Dex 文件

不知不觉已经说完了 ClassLoader 加载 Dex 文件得到 cookie 的部分，下面就开始进入到五花八门的壳各显神通的地方了，就是我们常看到的不落地加载 Dex 文件部分，也就是内存加载 Dex 文件，这一部分每种壳实现的多少都会不一样，有自己的方式，我说说我学习到和理解到的东西。

 

内存加载 Dex 文件方式的存在是因为不落地加载 Dex 文件方式的不安全性，如果我们想通过加壳来保护 Dex 文件，总会有一个时间点加载去 Dex 文件，如果直接使用 DexClassLoader 或者 PathClassLoader 去加载指定目录下解密好的 Dex 文件肯定是极度不安全的，同时也是相当于加载 Dex 文件到内存中两次，降低了效率。

 

我们当然期望用更安全和高效的方式来加载 Dex 文件，就是在内存中加载 Dex 文件。这样子首先加载到内存中的就是加密了的数据，然后在 Dex 文件加载之前进行解密即可，这样子就避免了不落地加载 Dex 文件的尴尬之处。然后需要知道的就是内存加载 Dex 文件主要是是分为 Dalvik 和 Art 虚拟机两种，两种虚拟机对应的 so 库 lidbvm.so 和 libart.so 底层实现函数是不一样的。《Android ART运行时无缝替换Dalvik虚拟机的过程分析》https://blog.csdn.net/luoshengyang/article/details/18006645?_t_t_t=0.4508641392433683这篇文章可以帮我们更好地理解两种虚拟机，了解两种虚拟机加载 Dex 文件的过程则有助于我们理解内存加载 Dex 文件，同样的，我会在文末放上学习了解到的一些关于这些的很好的文章博客链接。

 

先说下这篇文章我们要分析的 App 加的壳是怎么实现内存加载 Dex 文件，然后再提下我通过搜索了解到的大佬实现和使用的方式。

### 3.3.1 分析的壳的实现

在开始分析这个壳的时候，我注意到它是自己自定义了一个 CustomerClassLoader，并且是继承了 PathClassLoader。

 

![img](https://bbs.pediy.com/upload/attach/202006/802108_X3RQ4QKF5W8NQH2.png)

 

前面我们有说到 PathClassLoader 只能加载系统中已经安装的 Apk 中的 Dex，这是一个十分迷惑人的地方，这是不是意味着我们分析的壳是落地加载 Dex 文件呢？

 

带着这个疑问开始分析壳的 so 文件，不断的分析发现壳是在 so 层调用了 Java 层 MultiDex 类的 installDexes 方法来加载两个 Dex，关于MultiDex 类后面分析中我们再具体看下。

 

然后我尝试在 MultiDex 类的 installDexes 方法处设下断点，当断点停在这里的时候，我去相应的文件夹路径去查看这两个 Dex 文件，发现这两个文件居然是空的！

 

为什么自定义的 PathClassLoader 加载了两个空的 Dex 文件，App 还能正常运行呢，开始的时候被这个问题困惑了很久。

 

在后面持续不断分析和调试中，找到了问题的答案，原来在 so 层对 fstat、mmap 和 munmap 三个系统函数进行了 hook！这样在 libart.so 底层函数的代码中对文件进行内存映射的时候，返回的内存地址就是已经在内存中解密好的 Dex 文件的存放地址，而不是空的 Dex 文件。这样就通过 hook 系统函数做到了替换，ClassLoader 在底层函数中加载的不是空 Dex 文件，而是目标 Dex 文件。

 

![img](https://bbs.pediy.com/upload/attach/202006/802108_53WT64XUXBCJ9N2.png)

 

我们在三个 hook 替换的函数设下断点停下来的时候可以查看栈空间，发现返回地址在 libart.so 中的函数，从而可以验证我们的猜测。

### 3.3.2 别的实现方式

在大量的搜索中还了解到的一类方式是，通过调用 Dalvik 和 Art 虚拟机的底层函数加载 Dex 文件拿到 cookie，然后通过反射进行替换操作。

 

以 @寒号鸟二代 大佬一篇帖子中给出的项目（https://bbs.pediy.com/thread-225303.htm）为例。

 

mem_loadDex 函数中判断 Dalvik 和 Art 虚拟机调用相应的函数加载 Dex 文件获得 cookie，然后进行替换。

 

![img](https://bbs.pediy.com/upload/attach/202006/802108_JYH9F22WEC68JX8.png)

 

replace_cookie 函数具体实现了 cookie 替换。

 

![img](https://bbs.pediy.com/upload/attach/202006/802108_YJVG8TGP76ZZHY7.png)



# 5. 结语

写到这里其实已经写不动什么了，写文章总结纪录不是一件容易的事，十分地消耗精力和时间...

 

希望这篇文章能帮助像我一样入门学习 so 层分析的新手，如果觉得学习到了有用的东西，请务必点赞，也不枉我写了这么多。

# 6. 一些学习资料

这次分析中还有很大的收获就是找到了很多优秀的文章和博客，耐心分析学习着慢慢看懂了，再次感谢大佬们的无私分享精神！

 

Android的Proxy/Delegate Application框架
https://blogs.360.cn/post/proxydelegate-application.html#comment-77

 

性能优化 (八) APK 加固之动态替换 Application
https://juejin.im/post/5cf69d30f265da1b897abd53#heading-5

 

利用动态加载技术加固APK原理解析
https://juejin.im/post/5cf69d30f265da1b897abd53#heading-5

 

基于代理Application机制的Anddroid应用加壳方法
http://www.duorenwei.com/news/4539.html

 

第十二章 软件壳（四）（代码抽取型壳）
https://blog.csdn.net/zlmm741/article/details/106173778/

 

Android中apk加固完善篇之内存加载dex方案实现原理(不落地方式加载)
http://www.520monkey.com/archives/629

 

阿里早期Android加固代码的实现分析
https://blog.csdn.net/qq1084283172/article/details/78320445?utm_source=gold_browser_extension

 

android动态加载dex支持art
https://www.52pojie.cn/thread-590192-1-1.html

 

Android APK加固-内存加载dex
https://www.cnblogs.com/ltyandy/p/11642108.html

 

Android APK加固-完善内存dex
https://www.cnblogs.com/ltyandy/p/11644056.html

 

APK加壳【2】内存加载dex实现详解
https://segmentfault.com/a/1190000002578755

 

Android4.0内存Dex数据动态加载技术
https://blog.csdn.net/androidsecurity/article/details/9674251

 

Android安全分析挑战：运行时篡改Dalvik字节码
https://blog.csdn.net/androidsecurity/article/details/8833710

 

阿里早期加固代码还原4.4-6.0
https://bbs.pediy.com/thread-215078.htm

 

Android第二代加固（support 4.4-8.1）
https://bbs.pediy.com/thread-225303.htm

 

如何修复使用NOP指令抹去关键方法的DEX文件
https://www.anquanke.com/post/id/85886

 

DEX保护之指令抽取
https://www.freebuf.com/articles/terminal/205079.html

 

dvm，art模式下的dex文件加载流程
https://blog.csdn.net/m0_37344790/article/details/78523147

 

Android DEX加壳
https://yq.aliyun.com/articles/663294

 

Salsa20笔记
https://www.cnblogs.com/aquar/p/8437172.html

 

Bangcle
https://github.com/woxihuannisja/Bangcle

 

dexload
https://github.com/xiaobaiyey/dexload