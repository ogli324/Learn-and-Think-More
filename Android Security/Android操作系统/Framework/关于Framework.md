# 关于Framework

如果出去面试，针对高一点的级别，Framework 基本上会纳为必问的知识点。



**你可能会困惑，Framework 我们平时开发过程中又用不到，为什么面试官喜欢问这样的问题呢？**



其实不然...



考察 Framework **更多的目的是考察大家对于 Binder 的掌握。**



例如说：



**对于再简单的App，我们都有 Activity 吧，那么启动 Activity 背后会有哪些逻辑呢？**



背后的原理至少涉及到两部分：



1. app 与 system 进程交互，通过 ActivityManagerProxy 与 ActivityManagerService 交互；

2. system 进程与 app 进程交互，通过ApplicationThread 的代理Proxy 与 ApplicationThread 交互。



这个背后都**离不开 Binder 在背后的默默支撑。**



又例如，在启动Activity 的时候，我们可以携带一些数据到目标 Activity，但是我们携带数据过大，会造成TransactionTooLargeException。



**你可能会困惑为什么呢？**



还是因为**涉及到跨进程 binder 通信，在这个通信过程中对数据量的大小是有限制的。**



如果你对 binder 这些细节深知，那么我们在写代码的时候就有意识可以规避掉这些细节。



当然还有一些场景，是确实要用到跨进程通信，例如一些硬件设备，我们需要跟一个内置的服务通信，比如音箱，我们需要跟系统内置语音模块通信，可能会用到 aidl，当然 aidl 帮我们省去了非常多跨进程通信的细节，**不过如果你真的想掌握好 aidl，翻看源码时，又会看到 Binder 的身影。**



所以，对于 Framework 的掌握是尤为必要的。



不过你可能会担心，我以前没有深入了解过 Framework，该如何下手呢？



其实在面试过程中，除了我上述的一些点，剩下的 Framework 相关的问题，基本离不开以下几个方面：



\1. Android 进程启动会，Zygote 相关流程；

\2. 四大组件相关逻辑，涉及到 AMS，WMS，常见的 Activity 启动，Activity 相关管理；

\3. 安装包相关，涉及到 PMS，apk 是如何安装到系统上的；

\4. Binder 的一些特点，例如跨进程方式那么多，为什么要用 Binder...