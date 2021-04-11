# 简单的so脱壳器 

url：https://bbs.pediy.com/thread-192688.htm



自己总结了一下壳的三个发展流程。
1.文件本地加密，导入内存解密，壳加载器跑完不再做其他事情。
2.在程序正常运行时，壳可以重新接管控制权。 
3.虚拟机保护。

而当前众多android保护厂商，在so方面都停留在1阶段，还有很多没有。有so保护的不免费。
我们的so保护是免费放出。
可访问 www.nagapt.com <- 这里打个广告，免费。想加直接上传so就好。![img](images/16.gif)

这个简单的脱壳器，就是针对第一阶段而言。对于当前强度不是很高android保护而言。直接从内存
中dump就好了。修复其实就是对重定位表以及got表的修复。 这份代码中，没搞那么复杂。我太
懒了，有需求的朋友自己修改吧。

这个简单的脱壳器就是linker代码的简单应用。 用法直接看option.cpp就可以了。
下来之后直接 make DEBUG=1 all 即可编译调试版本。 make all 可以编辑发布版本。
配置ndk在最外层的 Makevars.global中配置ndk的路径。
从github下载时，切换到dev版本即可。 代码很乱，随意修改了一下linker而已。
以下是下载地址:
https://github.com/devilogic/udog