# Android逆向中的各种tracer

url：https://blog.csdn.net/qq_38851536/article/details/115082548

# JAVA层

* 正向开发中的trace工具：DDMS
  永远的神，从很久以前到现在都有人用，这个工具说不上好坏，就挺牛的。
*   XPOSED TRACER
  使用Xposed 编写插件，HOOK 所有JAVA调用，以前见过，比较容易崩溃，现在应该几乎没人用了。
*   Zentrace
  基于Frida构建的trace脚本，非常优雅，大部分时候很好用，现在也依然是选择之一。
*   Dwarf
  稳定性好像不太行，我也没摸过几次
*   Objection和r0trace
  基于Frida构建，应该是现在Java层分析和trace的主力工具。
*   Frida-trace
  Frida官方trace工具，支持Java方法的模糊匹配，挺好的。
#   Native层

* trace-natives
基于Frida trace写的小脚本，挺有用。
* IDA trace
IDA，yyds。
* Frida Stalker
指令级trace，yyds，但在Android上只支持arm64。
* Unicorn/Unidbg trace
万能的神。
