# Android第二代加固（support 4.4-8.1）

url：https://bbs.pediy.com/thread-225303.htm

### 介绍

代码放在github上：https://github.com/woxihuannisja/Bangcle
第二代加固使用的是内存动态加载Dex，也就是不落地加载，可以将Dex加密放在Apk中，在内存中实现解密

### 兼容性

测试可以支持Andorid 4.4-8.1版本，目前还不能支持重写了Application类 的Apk

### 原理

Dalvik下的动态加载方法使用的是我这篇文章的方案 https://bbs.pediy.com/thread-215078.htm，
Art 下提供了2种方案，
方案一是 call libart下的OpenMemory函数，如何将java的mCookie和c层的cookie联系起来是一个难点，
方案二是 使用elf Hook来实现 ，由于在Nougat+上dlsym openMemory失败，还有dex_location和dex_cache_location的路径检查，使用方案一有些问题。

### 最后

Android二代加固尽管脱壳不是很难，但是这种技术还是一些加固厂商的基础，很多在这基础上添加保护so，以及类抽取等等功能。