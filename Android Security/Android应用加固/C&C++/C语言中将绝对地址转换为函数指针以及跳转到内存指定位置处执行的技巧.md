# C语言中将绝对地址转换为函数指针以及跳转到内存指定位置处执行的技巧

url：https://my.oschina.net/u/920274/blog/2354109

1、方法一
要对绝对地址0x100000赋值，我们可以用
   `(unsigned int  * ) 0x100000 = 1234;`
   那么要是想让程序跳转到绝对地址是0x100000去执行，应该怎么做？
  ` *((void (*)( ))0x100000 ) ( );`
  首先要将0x100000强制转换成函数指针,即:
   `(void (*)())0x100000`
   然后再调用它:
 ` *((void (*)())0x100000)();`
  用typedef可以看得更直观些:
  `typedef void(*)() voidFuncPtr;`
`  *((voidFuncPtr)0x100000)();`

又如
如果用 C 语言，可以像下列示例代码这样来调用内核：

```
void (*theKernel)(int zero, int arch, u32 params_addr)
= (void (*)(int, int, u32))KERNEL_RAM_BASE; 
…… 
theKernel(0, ARCH_NUMBER, (u32) kernel_params_start); 
```



KERNEL_RAM_BASE 是内核在系统内存中的第一条指令的地址。
2、方法二
C语言使用函数指针跳转到程序固定地址(0x8000)执行程序的方法

使用函数指针，把一个纯数据强制转换为函数指针类型。

```
int main(void)

{
void (* my_function)(void);
//int *my_address = 0x8000;
my_function =(void (*)())(0x8000);
my_function();
}
```



其实更简单，不适用中间变量，直接一步到位：

`(*(void(*)())0x8000)();`
\--------------------- 