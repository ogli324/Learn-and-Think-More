# Learn-and-Think-More
Live long and prosper.

一个无情的网络安全文章搜集整合机器，目前主攻安卓逆向。

```
$ date
Sat May  8 14:56:04 CST 2021
$ tree -I "images|files"
.
├── Android Security
│   ├── Android刷机
│   │   ├── Android Root 教程.md
│   │   ├── Android 玩机终极指南.md
│   │   ├── Magisk Manager详解.md
│   │   ├── Magisk 学习.md
│   │   ├── Magisk学习之刷入vbmeta.img及关闭avb2.0校验.md
│   │   ├── Magisk的root隐藏功能.md
│   │   ├── 【TWRP】使用adb sideload线刷ROM的方法.md
│   │   ├── 免root实现 Android改机（一键新机）技术解密.md
│   │   ├── 关于 Android 7.1 的 Xposed，你想知道的都在这.md
│   │   ├── 双龙战记：Andrax vs. Nethunter.md
│   │   ├── 在 2019 年，Root 是否还有必要？.md
│   │   └── 搞机的神器们——Xposed，Magisk，TaiChi(太极)的安装使用.md
│   ├── Android应用加固
│   │   ├── Native Hook注入
│   │   │   ├── ARM64下的Android Native Hook工具实践.md
│   │   │   ├── ARM平台backtrace与inlineHook多线程安全浅析.md
│   │   │   ├── Android Arm Inline Hook.md
│   │   │   ├── Android Hooking.md
│   │   │   ├── Android Inline Hook中的指令修复详解.md
│   │   │   ├── Android Inline Hook详解.md
│   │   │   ├── Android Native Hook工具实践.md
│   │   │   ├── Android Native Hook技术路线概述.md
│   │   │   ├── Android10 aarch64 dlopen Hook.md
│   │   │   ├── Android9.0 hook dlopen问题如何hook dlopen相关函数 .md
│   │   │   ├── Android利用ptrace实现Hook API.md
│   │   │   ├── Android动态库注入技术.md
│   │   │   ├── Android平台下hook框架adbi的研究.md
│   │   │   ├── ELF Hook总结与代码实现.md
│   │   │   ├── 使用LIEF打造类似王者X耀的静态代码注入 .md
│   │   │   ├── 如何写一个Android inline hook框架 .md
│   │   │   └── 用伪造so的方式注入 .md
│   │   ├── VMP-x86
│   │   │   ├── VMP3逆向分析虚拟机指令 浅析 .md
│   │   │   ├── VMProtect1.2x总结（完）.md
│   │   │   ├── VMProtect2.04加壳程序从入门到精通.md
│   │   │   ├── VMProtect加密程序解析 .md
│   │   │   ├── VMP虚拟机加壳的原理学习.md
│   │   │   ├── 人肉跟踪VMProtect入口至出口运行全过程(1) .md
│   │   │   ├── 人肉跟踪VMProtect入口至出口运行全过程(2)迷你代码还原 .md
│   │   │   ├── 动手实现代码虚拟机.md
│   │   │   ├── 想学习下VM技术,求资料.md
│   │   │   └── 虚拟机保护技术浅谈 .md
│   │   ├── ollvm混淆
│   │   │   └── Android之JNI混淆技术--OLLVM（趟坑记录）.md
│   │   ├── so文件加固
│   │   │   ├── Android SO文件保护加固——加密篇.md
│   │   │   ├── Android so加固的简单脱壳.md
│   │   │   ├── Android so库加固加壳方案.md
│   │   │   ├── Android so库加密Section 内存解密.md
│   │   │   ├── Android加壳过程中mprotect调用失败的原因及解决方案.md
│   │   │   ├── ELF section修复的一些思考.md
│   │   │   ├── ELF中可以被修改又不影响执行的区域.md
│   │   │   ├── ELF文件格式修复.md
│   │   │   ├── IDA动态调试-没啥卵用的静态加固.md
│   │   │   ├── android so加固之section加密.md
│   │   │   ├── so加固-加密特定section中的内容.md
│   │   │   ├── so文件的简单加密.md
│   │   │   ├── 从零打造简单的SODUMP工具 .md
│   │   │   ├── 基于init_array加密的SO的脱壳.md
│   │   │   ├── 基于对so中的section加密技术实现so加固.md
│   │   │   ├── 基于对so中的函数加密技术实现so加固.md
│   │   │   ├── 安卓加固之so文件加固.md
│   │   │   ├── 简单的so脱壳器.md
│   │   │   └── 简单粗暴的so加解密实现 .md
│   │   ├── 代码自解密
│   │   │   └── SMC自解码总结.md
│   │   ├── 加壳
│   │   │   ├── Android Vmp加固实现流程图.md
│   │   │   ├── Android第二代加固（support 4.4-8.1）.md
│   │   │   ├── VMP指令虚拟化原理.md
│   │   │   ├── dex vmp虚拟化 .md
│   │   │   ├── dex动态自修改demo题解.md
│   │   │   ├── dex壳简单分析与实现过程 .md
│   │   │   ├── 运行时加载隐藏dex.md
│   │   │   └── 通过一款早期代码抽取壳入门学习 so 层分析 .md
│   │   └── 检测
│   │       ├── ANDROID调试检测技术汇编.md
│   │       ├── ARM Cache Flush on mmap’d Buffers with __clear_cache().md
│   │       ├── Android 常用反调试方法.md
│   │       ├── Android安全防护检查root检查Xposed反调试应用多开模拟器检测（持续更新1.1.0）.md
│   │       ├── Android模拟器检测体系梳理.md
│   │       ├── Anti-debugging Skills in APK.md
│   │       ├── 一行代码帮你检测Android模拟器.md
│   │       └── 基于Cache的Android模拟器检测.md
│   ├── Android开发
│   │   ├── Android Misc
│   │   │   ├── Android SDK的下载与安装.md
│   │   │   ├── Android 对APK进行v1+v2+v3签名.md
│   │   │   └── base64加号问题.md
│   │   ├── Git
│   │   │   ├── Git pull 强制拉取并覆盖本地代码.md
│   │   │   ├── Windows下设置git代理.md
│   │   │   ├── Windows为cmdpowershell设置代理.md
│   │   │   ├── git error invalid path问题解决（win下）.md
│   │   │   ├── github在git push之后不记录Contributions.md
│   │   │   ├── git同时绑定两个账号gitlab、github.md
│   │   │   ├── 执行ssh-add时出现Could not open a connection to your authentication agent.md
│   │   │   └── 是否必须每次添加ssh-add.md
│   │   ├── Maven
│   │   │   └── Mac配置Maven以及IntelliJ IDEA Maven配置.md
│   │   ├── NDK开发
│   │   │   ├── Android JNI入门.md
│   │   │   ├── Android NDK开发  JNI包含多个cpp文件.md
│   │   │   ├── Android Studio NDK CMake 指定so输出路径以及生成多个so的案例与总结.md
│   │   │   ├── Android ndk abiFilters 无效 解决方案.md
│   │   │   ├── Android中JNI调用第三方so以及头文件方式.md
│   │   │   ├── NDK学习笔记.md
│   │   │   └── jni开发中 接口为什么要冠extern C呢.md
│   │   ├── 安卓UI
│   │   │   ├── 使用OkHttp发送网络请求并将结果更新至UI的几种方式.md
│   │   │   └── 详解Android中OkHttp3的例子和在子线程更新UI线程的方法.md
│   │   ├── 热修复
│   │   │   └── android热修复.md
│   │   ├── 网络通信
│   │   │   ├── OkHttp3架构分析.md
│   │   │   ├── Okhttp3基本使用.md
│   │   │   └── Okhttp3源码分析.md
│   │   └── 设计模式
│   │       └── Java设计模式之单例模式(SingleInstance).md
│   ├── Android操作系统
│   │   ├── Binder
│   │   │   └── 为什么 Android 要采用 Binder 作为 IPC 机制？.md
│   │   ├── ELF文件
│   │   │   ├── Android oat文件格式分析笔记 .md
│   │   │   ├── ELF Format.md
│   │   │   ├── ELF 文件格式.md
│   │   │   ├── ELF文件结构详解.md
│   │   │   └── 深入浅出ELF.md
│   │   ├── Framework
│   │   │   ├── Android SDK是什么？.md
│   │   │   └── 关于Framework.md
│   │   ├── 安卓系统机制
│   │   │   ├── Android ELF系列.md
│   │   │   ├── Android so加载深入分析：从载入到链接.md
│   │   │   ├── Android 操作系统架构开篇.md
│   │   │   ├── Android中unlink利用练习 .md
│   │   │   ├── Android系统启动-zygote篇.md
│   │   │   ├── CPU体系架构-ARM MIPS X86.md
│   │   │   ├── ELF格式及动态加载过程.md
│   │   │   ├── ROM安全梳理.md
│   │   │   ├── 从 Android 源码看 SO 的加载 .md
│   │   │   ├── 从dlsym()源码看android 动态链接过程.md
│   │   │   ├── 浅谈ELF可执行文件的启动流程.md
│   │   │   ├── 深入了解GOT,PLT和动态链接.md
│   │   │   └── 细说So动态库的加载流程.md
│   │   ├── 移动基带系统
│   │   │   ├── 深度揭密高通45G移动基带消息系统和状态机.md
│   │   │   └── 移动基带安全研究系列之一 概念和系统篇.md
│   │   ├── 脱壳机制
│   │   │   ├── AUPK：基于Art虚拟机的脱壳机.md
│   │   │   ├── Android的ProxyDelegate Application框架.md
│   │   │   ├── Android通用脱壳机FUPK3.md
│   │   │   ├── Dalvik解释器源码到VMP分析 .md
│   │   │   ├── FART：ART环境下基于主动调用的自动化脱壳方案 .md
│   │   │   ├── Youpk 又一款基于ART的主动调用的脱壳机.md
│   │   │   ├── fart的理解和分析过程.md
│   │   │   ├── frida跟踪应用中所有运行在解释模式的java函数 .md
│   │   │   ├── vmp入门.md
│   │   │   ├── 使用frida hook解释器Interpreter.md
│   │   │   ├── 使用frida来hook artmethod的RegisterNative.md
│   │   │   ├── 对类抽取加固的一点尝试与遇到的问题.md
│   │   │   ├── 将FART和Youpk结合来做一次针对函数抽取壳的全面提升.md
│   │   │   ├── 拨云见日：安卓APP脱壳的本质以及如何快速发现ART下的脱壳点.md
│   │   │   ├── 菜鸟学8.1版本dex加载流程笔记--第二篇DexFileOpen流程与简单脱壳原理.md
│   │   │   └── 记录向：单纯使用Frida书写类抽取脱壳工具的一些心路历程和实践 .md
│   │   └── 虚拟机
│   │       ├── Android ART执行类方法的过程.md
│   │       ├── Art 虚拟机代码加载简析.md
│   │       ├── Dalvik 虚拟机代码加载简析.md
│   │       ├── Dalvik解释器源码到VMP分析.md
│   │       ├── android ART hook.md
│   │       ├── 深入理解 Android（一）：Gradle 详解.md
│   │       ├── 深入理解 Android（三）：Xposed 详解.md
│   │       └── 深入理解 Android（二）：Java 虚拟机 Dalvik.md
│   ├── Android逆向
│   │   ├── Frida
│   │   │   ├── FRIDA 使用经验交流分享.md
│   │   │   ├── FRIDA-API使用篇 -- r0ysue.md
│   │   │   ├── Frida Android hook -- sakura.md
│   │   │   └── jnitrace 启动命令行.md
│   │   ├── Rom
│   │   │   ├── Android odex,oat文件的反编译，回编译.md
│   │   │   ├── 修改Nexus5的boot.img - 打开系统调试.md
│   │   │   ├── 绕过rom校验.md
│   │   │   ├── 编译AOSP.md
│   │   │   └── 自编译ROM：两行代码搞定钉钉打卡 免ROOT永久防封.md
│   │   ├── Xposed
│   │   │   ├── Xposed ART原理分析笔记 .md
│   │   │   ├── Xposed hook系统方法实践.md
│   │   │   ├── Xposed项目搭建.md
│   │   │   ├── nexus5刷机、root及安装xposed.md
│   │   │   └── 教你编译Xposed.md
│   │   ├── 功能分析
│   │   │   └── LBE主动防御逆向.md
│   │   ├── 加密
│   │   │   └── 逆向中常见的Hash算法和对称加密算法的分析.md
│   │   ├── 安全风控
│   │   │   ├── Anti-Bot安全SDK SGAVMP浅析.md
│   │   │   ├── 从黑产对抗角度谈设备指纹技术.md
│   │   │   ├── 以攻击者角度学习某风控设备指纹产品.md
│   │   │   ├── 基于设备指纹零感验证系统.md
│   │   │   ├── 对国内某盟ID厂商产品的技术分析（Android版）.md
│   │   │   ├── 数美科技：为业务互联网化保驾护航的幕后英雄.md
│   │   │   ├── 某移动端防作弊产品技术原理浅析与个人方案构想.md
│   │   │   ├── 比心数美——某陪玩软件协议加密算法分析（so层分析）.md
│   │   │   ├── 设备风控攻防新挑战：定制ROM改机.md
│   │   │   └── 黑产对抗系列之 多开检测 .md
│   │   ├── 改机
│   │   │   ├── Android svc获取设备信息.md
│   │   │   ├── Android 动态修改Linker实现LD_PRELOAD全局库PLT Hook.md
│   │   │   ├── AndroidSyscallLogger.md
│   │   │   ├── FakeXposed原理分析.md
│   │   │   ├── Nexus 5X过反调试修改Android内核实践笔记 .md
│   │   │   ├── strace命令详解.md
│   │   │   ├── 利用xhook安卓系统底层抹机原理 .md
│   │   │   ├── 改机 - 从源码着手任意修改GPS地理位置.md
│   │   │   └── 来自高纬的对抗：定制ART解释器脱所有一二代壳.md
│   │   ├── 模拟执行
│   │   │   ├── AndroidNativeEmu使用指南.md
│   │   │   ├── AndroidNativeEmu模拟执行大厂so实操教程.md
│   │   │   ├── Unicorn 在 Android 的应用.md
│   │   │   ├── Unicorn用法示例.md
│   │   │   ├── Unidbg使用指南.md
│   │   │   ├── Unidbg模拟执行大厂so实操教程.md
│   │   │   ├── 抖音数据采集教程，Unicorn 模拟 CPU 指令并 Hook CPU 执行状态.md
│   │   │   └── 挑战4个任务：迅速上手Unicorn Engine.md
│   │   ├── 混淆反混淆
│   │   │   ├── ARM64 OLLVM反混淆（续）.md
│   │   │   ├── Unicorn Trace还原Ollvm算法.md
│   │   │   ├── Unicorn反混淆：恢复被OLLVM保护的程序(一).md
│   │   │   ├── 一种通过后端编译优化脱虚拟机壳的方法.md
│   │   │   ├── 利用编译器优化干掉虚假控制流.md
│   │   │   ├── 基于LLVM Pass实现控制流平坦化.md
│   │   │   └── 基于LLVM的控制流平坦化的魔改和混淆Pass实战.md
│   │   ├── 理论知识点
│   │   │   ├── Android逆向中的各种tracer.md
│   │   │   ├── 《看雪安卓安全人才标准认证》技术要求细则发布
│   │   │   └── 漫谈逆向工程.md
│   │   ├── 签名算法
│   │   │   ├── B站
│   │   │   │   └── 某站App签名算法解析(一).md
│   │   │   ├── 京东
│   │   │   │   ├── AndroidNativeEmu模拟执行计算出某电商App sign.md
│   │   │   │   ├── Unidbg模拟执行大厂so实操教程.md
│   │   │   │   ├── 京东 APP SIGN算法分析.md
│   │   │   │   ├── 手机京东app sign st sv 签名 9.3.4版本签名算法分析.md
│   │   │   │   ├── 抓包实时打印_某东直播弹幕实时抓取.md
│   │   │   │   ├── 某电商App反反调试.md
│   │   │   │   ├── 某电商App签名算法解析(一).md
│   │   │   │   ├── 某电商App签名算法解析(二).md
│   │   │   │   └── 算法还原的助手(一) 先让时间停下来.md
│   │   │   ├── 京东到家
│   │   │   │   └── 某东到家app 签名算法分析.md
│   │   │   ├── 小红书
│   │   │   │   ├── xhs 6.73+ XYshield算法 .md
│   │   │   │   └── xhs新版shield算法 .md
│   │   │   ├── 快手
│   │   │   │   └── 某小视频App v8.x 签名计算方法.md
│   │   │   ├── 抖音
│   │   │   │   └── 发个某音生成请求校验数据的算法.md
│   │   │   ├── 最右
│   │   │   │   └── 某段子App签名计算方法.md
│   │   │   └── 盒马生鲜
│   │   │       └── 某生鲜App 签名计算方法 脱个壳试试
│   │   └── 脱壳分析
│   │       ├── 360加固
│   │       │   ├── 2017年10月30日360最新虚拟壳脱壳后完全修复.md
│   │       │   ├── libjiagu.so 解密，还原.md
│   │       │   ├── so文件动态加解密的CrackMe.md
│   │       │   ├── 某vmp壳原理分析笔记.md
│   │       │   └── 根据”so劫持”过360加固详细分析 .md
│   │       ├── 乐固加固
│   │       │   ├── 乐固加固（17年1月）逆向分析 .md
│   │       │   ├── 分析一下乐固.md
│   │       │   └── 分析一个有趣的so双重壳.md
│   │       ├── 梆梆加固
│   │       │   ├── 分析一下梆x加固 .md
│   │       │   ├── 某企业级加固[四代壳]VMP解释执行+指令还原.md
│   │       │   ├── 梆梆APP加固产品方案浅析.md
│   │       │   ├── 梆梆加固之防内存dump分析.md
│   │       │   └── 脱壳成长之路(3)-初识VMP.md
│   │       ├── 爱加密
│   │       │   ├── CISCN2018—爱加密过反调试的尝试.md
│   │       │   ├── X加密-反调试-DumpDex-修复指令-重打包.md
│   │       │   ├── 使用Frida简单实现函数粒度脱壳.md
│   │       │   ├── 某抽取壳的原理简析.md
│   │       │   ├── 某特殊抽取壳的原理简析.md
│   │       │   ├── 爱加密加固分析.md
│   │       │   └── 看雪3W4月第三题frida辅助尝试脱壳.md
│   │       ├── 百度加固
│   │       │   ├── 百度加固免费版分析及VMP修复.md
│   │       │   ├── 百度加固逆向分析.md
│   │       │   ├── 百度加固逆向分析—dex还原 .md
│   │       │   └── 脱壳成长之路(3)-初识VMP.md
│   │       ├── 脱壳常用知识点
│   │       │   ├── dex整体dump.md
│   │       │   ├── fart脱壳机.md
│   │       │   ├── frida常用脱壳脚本.md
│   │       │   └── 类方法dump.md
│   │       ├── 腾讯壳
│   │       │   └── 某类抽取加固APP的脱壳与修复.md
│   │       ├── 迦娜加固
│   │       │   └── APK加固之类抽取分析与修复.md
│   │       └── 阿里加固
│   │           └── ali 加固（17年1月）逆向分析 .md
│   └── Arm汇编
│       ├── ARM 汇编基础速成.md
│       ├── B与BL跳转指令目标地址计算方法.md
│       ├── CC++ 中嵌入 arm 汇编.md
│       ├── arm汇编学习（一）.md
│       ├── 从 Arm 汇编看 Android C++虚函数实现原理 .md
│       ├── 在android手机上运行C程序 - hello arm!.md
│       ├── 汇编写花指令完全手册 汇编花指令.md
│       ├── 花指令总结.md
│       └── 逆向学习笔记之花指令.md
├── Binary Exploitation
│   ├── Exploit工具
│   │   ├── Pwntools
│   │   │   ├── Exploit利器——Pwntools.md
│   │   │   ├── Handling SO Hell in CTF Pwn.md
│   │   │   ├── Pwntools 高级应用.md
│   │   │   └── ctf-all-in-one-pwntools.md
│   │   └── msfvenom
│   │       └── Shellcode生成器——msfvenom.md
│   ├── Fuzzing
│   │   └── Fuzzingbook学习指南 Lv1.md
│   ├── 二进制动态插桩框架
│   │   ├── Intel Pin
│   │   │   ├── Intel Pin II - 编译与运行时环境.md
│   │   │   ├── Intel Pin基本用法.md
│   │   │   └── Pin 动态二进制插桩.md
│   │   └── QBDI
│   │       └── 0x00  Introduction.md
│   ├── 动态分析工具
│   │   ├── A64dbg
│   │   │   ├── A64Dbg-ADCpp VS Frida及原理说明.md
│   │   │   ├── A64Dbg-v1.8.2可以愉快的编写Dex脱壳机了.md
│   │   │   ├── A64Dbg-开源重构完毕，继续前行.md
│   │   │   ├── A64Dbg-来瞧瞧ADCpp最潮流的Dump姿势.md
│   │   │   ├── ADCpp脚本.md
│   │   │   ├── ADCpp脚本系统简介.md
│   │   │   ├── UnicornVM-全平台全架构正式发布.md
│   │   │   ├── UnicornVM-回调事件添加指令获取操作码.md
│   │   │   ├── UraniumPacker-一个可以多合一的MachOELF免费压缩壳.md
│   │   │   ├── UraniumVCPU-10年青春化作这一个头文件了.md
│   │   │   ├── UraniumVCPU-这个虚拟CPU的故事长达10年.md
│   │   │   ├── UraniumVM JIT.md
│   │   │   ├── UraniumVM-armarm64x86x64指令级函数虚拟机.md
│   │   │   ├── UraniumVM-适用于AndroidiOSmacOS的指令级函数虚拟机 .md
│   │   │   ├── X64Dbg-构建属于你的逆向工程操作平台 .md
│   │   │   ├── YuYoo.md
│   │   │   └── 使用命令.md
│   │   ├── GDB
│   │   │   ├── GDB的那些奇淫技巧.md
│   │   │   ├── GDB调试笔记.md
│   │   │   └── 一窥GDB原理.md
│   │   ├── JDB
│   │   │   └── Android远程调试的探索与实现.md
│   │   └── LLDB
│   │       ├── Debugging Krita on Android.md
│   │       ├── GDB to LLDB command map.md
│   │       ├── LLDB image.md
│   │       ├── LLDB常用命令.md
│   │       ├── Untitled.md
│   │       └── 在linux环境下使用lldb调试，以及 dump 安卓内存.md
│   ├── 漏洞分析
│   │   ├── CVE标准及CVE工作机制的官方权威解读.md
│   │   ├── Sakuraのdiary.md
│   │   ├── 二进制漏洞分析技能脑图.md
│   │   ├── 二进制漏洞研究入门之上篇.md
│   │   ├── 如何进行CVE复现？.md
│   │   └── 白话二进制漏洞攻击方式.md
│   ├── 编译器
│   │   ├── GCC
│   │   │   └── GCC在CTF中的应用.md
│   │   └── LLVM
│   │       ├── LLVM入门笔记.md
│   │       ├── LLVM基本概念入门.md
│   │       ├── llvm中的pass例子.md
│   │       ├── llvm简单的使用.md
│   │       ├── o-llvm Control Flow Flattening _.md
│   │       ├── ollvm自定义string加密pass.md
│   │       ├── 基于 llvm 实现活跃变量分析.md
│   │       └── 跟随一条指令来看LLVM的基本结构.md
│   ├── 软件虚拟化
│   │   ├── Qemu
│   │   │   ├── 插桩之路——为QEMU TCG添加helper.md
│   │   │   └── 虚拟化逃逸的攻击面及交互方式.md
│   │   ├── Unicorn
│   │   │   └── Micro Unicorn-Engine API Documentation.md
│   │   ├── Unidbg
│   │   │   ├── AndroidNativeEmu和unidbg对抗ollvm的字符串混淆.md
│   │   │   ├── Unidbg使用指南.md
│   │   │   ├── unidbg学习笔记.md
│   │   │   ├── 分析unidbg(unidbgMutil)多线程机制.md
│   │   │   ├── 合并Unidbg更新.md
│   │   │   └── 逆向工具之unidbg.md
│   │   └── capstone
│   │       └── Micro Capstone-Engine API Documentation.md
│   └── 静态分析工具
│       ├── Ghidra
│       │   └── 安卓逆向之自动化JNI静态分析.md
│       ├── IDA
│       │   ├── Hex-Rays 十步杀一人，两步秒OLLVM-BCF.md
│       │   ├── IDA dump脚本.md
│       │   ├── IDApython脚本需要注意的问题.md
│       │   ├── IDA调试 android so文件的10个技巧.md
│       │   ├── IDA调试so文件.md
│       │   ├── Porting from IDAPython 6.x-7.3, to 7.4.md
│       │   └── 某右 so层ollvm字符串混淆.md
│       └── Radare2
│           ├── Radare 2之旅：通过crackme实例讲解Radare 2在逆向中的应用.md
│           ├── Radare2从入门到放弃.md
│           ├── ctf-all-in-one-radare2.md
│           └── radare2逆向笔记.md
├── CTF
│   └── python
│       └── 如何破解一个Python虚拟机壳并拿走12300元ETH.md
├── Crypto
│   ├── AES
│   │   ├── AES128加密-S盒和逆S盒构造推导及代码实现.md
│   │   ├── AES中S盒被修改过了，如何推导出逆S盒.md
│   │   ├── AES加密CBC模式的c语言实现.md
│   │   ├── AES密码算法的实现细节.md
│   │   ├── AES的S盒生成.md
│   │   ├── AES算法之理论与编程结合篇.md
│   │   ├── 中南大学现代密码学实验报告.md
│   │   ├── 分享个逆X梆vmp遇到变种aes源码.md
│   │   ├── 密码学基础：AES加密算法.md
│   │   └── 手动推导计算AES中的s盒的输出.md
│   ├── DES
│   │   └── des为什么有不同类型的sbox.md
│   └── RSA
│       └── RSA加密、解密、签名、验签的原理及方法
├── Linux
│   ├── Linux内核
│   │   ├── Linux内核工程导论——网络：Filter（LSF、BPF、eBPF）.md
│   │   └── 实现一个基于XDP_eBPF的学习型网桥.md
│   ├── Linux服务器配置
│   │   ├── GoAccess日志分析配置文件详解.md
│   │   ├── Linux查看内存和cpu利用率的命令.md
│   │   ├── Linux查看实时网卡流量的几种方式.md
│   │   ├── Nginx的启动、停止与重启.md
│   │   └── iptables 最多能阻挡多少个 ip？.md
│   └── WSL2
│       ├── WSL2配置V2ray代理.md
│       ├── 为 WSL2 一键设置代理.md
│       └── 记录一次WSL2的网络代理配置.md
├── Misc
│   ├── 命令行使用
│   │   ├── Linux
│   │   │   └── tree.md
│   │   └── Vim
│   │       └── Vim入门基础.md
│   └── 开源许可证
│       └── GPL
│           └── 为什么GPL是更好的开源许可证.md
├── Network
│   ├── DDoS攻击
│   │   ├── DDOS攻击方式总结.md
│   │   ├── DDOS核弹攻击--Memcached放大攻击复现.md
│   │   ├── DDoS反射放大之SSDP攻击.md
│   │   ├── DDos DoS工具集.md
│   │   ├── NTP反射攻击复现.md
│   │   ├── ddos、http协议、TCP协议攻击工具及使用方法.md
│   │   ├── 【ZoomEye专题报告】DDoS 反射放大攻击全球探测分析.md
│   │   ├── 傻瓜式的DDoS协议（SSDP）实现100 Gbps DDoS.md
│   │   ├── 利用Memcached的反射型DDOS攻击.md
│   │   ├── 反射DDoS攻击的一点小想法.md
│   │   ├── 基于Memcached分布式系统DRDoS拒绝服务攻击技术研究.md
│   │   ├── 基于snmp的反射攻击的理论及其实现.md
│   │   ├── 拒绝服务（DoS）理解、防御与实现.md
│   │   ├── 服务器通过NTP服务，疯狂向外发包.md
│   │   ├── 浅谈DDos攻击与防御.md
│   │   ├── 浅谈“慢速HTTP攻击Slow HTTP Attack”.md
│   │   ├── 游戏业务DDoS攻防对抗案例分享.md
│   │   ├── 盘点：常见UDP反射放大攻击的类型与防护措施.md
│   │   ├── 绿盟科技“黑洞”ADS.md
│   │   ├── 被骗几十万总结出来的Ddos攻击防护经验.md
│   │   ├── 记一次被当成ddos发包机.md
│   │   └── 隐秘的角落：基于某款游戏利用的反射攻击分析.md
│   ├── DDoS防御
│   │   ├── 20步打造最安全的Nginx Web服务器.md
│   │   ├── 9个常用iptables配置实例.md
│   │   ├── CentOS — 简单处理CC攻击的shell脚本.md
│   │   ├── Linux判断CC攻击命令详解.md
│   │   ├── Linux桌面系统iptables安全配置.md
│   │   ├── Linux简单处理CC攻击shell脚本.md
│   │   ├── Nginx 限制访问 - 限制对代理TCP资源的访问.md
│   │   ├── Nginx使用limit_req_zone对同一IP访问进行限流.md
│   │   ├── Nginx简单防御CC攻击.md
│   │   ├── 优化LINUX内核阻挡SYN洪水攻击.md
│   │   ├── 动态检测secure日志文件，iptables拒绝恶意IP.md
│   │   └── 通过shell脚本防止端口扫描.md
│   ├── 网络爬虫
│   │   ├── Python实战异步爬虫(协程)+分布式爬虫(多进程).md
│   │   ├── requests模块高级操作、代理、模拟登录.md
│   │   ├── 什么是代理ip极其检测方法.md
│   │   ├── 使用 aiohttp 和 asyncio 进行异步请求.md
│   │   ├── 反爬与反反爬.md
│   │   └── 爬虫工程师学习养成路径.md
│   └── 网络通信
│       ├── SSTap.md
│       ├── Shadowsocks(Sock5代理)的PAC模式与全局模式.md
│       ├── TCP 的那些事儿.md
│       ├── TCP协议疑难杂症全景解析.md
│       ├── TCP的3次握手和4次挥手过程.md
│       ├── TCP连接的TIME_WAIT和CLOSE_WAIT 状态解说.md
│       ├── books
│       │   ├── TCP-IP详解卷1：协议.pdf
│       │   └── The C10K problem
│       │       └── The C10K problem.html
│       ├── 一个TCP FIN_WAIT2状态细节引发的感慨.md
│       ├── 关于关闭 Socket 的一些坑.md
│       ├── 单台服务器支持的最大连接数-困扰多年的问题.md
│       ├── 测试Linux下tcp最大连接数限制.md
│       ├── 科学上网1：shadowsocks 全系列配置总结.md
│       ├── 科学上网2：shadowsocks+v2ray-plugin+TLS.md
│       ├── 科学上网3：macos使用shadowsocks做透明代理.md
│       └── 聊聊C10K问题及解决方案.md
├── README.md
└── Web Exploitation
    ├── ATT&CK
    │   ├── ATT&CK矩阵Linux系统安全实践.md
    │   ├── 入侵检测之syscall监控.md
    │   └── 安全防御：Linux入侵检测之文件监控.md
    ├── Tools
    │   ├── Burpsuit
    │   │   ├── 渗透神器：burpsuit教程 （入门）.md
    │   │   ├── 渗透神器：burpsuit教程 （汉化+Repeater）.md
    │   │   └── 渗透神器：burpsuit教程之intruder模块.md
    │   ├── Metasploit
    │   │   └── 渗透工具：Metasploit教程-windows渗透.md
    │   ├── SQLMap
    │   │   └── 渗透工具教程：SQLMap教程-入门.md
    │   ├── 蚁剑
    │   │   └── 渗透工具：蚁剑(AntSword)教学.md
    │   └── 靶场搭建
    │       └── WEB靶场搭建教程（PHPstudy+SQLllib+DVWA+upload-labs）.md
    ├── WebShell
    │   ├── GetShell
    │   │   ├── GetShell的姿势总结.md
    │   │   ├── 反弹Shell，看这一篇就够了.md
    │   │   └── 如何优雅的隐藏你的 Webshell.md
    │   ├── 免杀
    │   │   └── Webshell免杀的思考与学习.md
    │   └── 检测
    │       └── Webshell入侵检测初探.md
    ├── XSS
    │   ├── XSS教程-基础入门.md
    │   └── 浅谈XSS.md
    ├── 信息收集
    │   └── 内网渗透基石篇--内网信息收集.md
    └── 网络渗透
        └── 记一次 Linux 被入侵全过程.md
```
