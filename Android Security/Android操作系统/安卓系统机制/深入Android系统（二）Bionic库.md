# 深入Android系统（二）Bionic库

url：https://juejin.cn/post/6857508056620941320



> `Bionic库`是Android的基础库之一，也是连接Android和Linux的桥梁。`Bionic库`中包含了很多基本系统功能接口，这些功能大部分来自 Linux，但是和标准的 Linux 之间有很多细微差别。同时`Bionic库`中增加了一些新的模块（像`log`、`linker`），服务于Android的上层代码。

# 导读

> 为甚有导读呢，因为`Bionic库`这部分学起来太枯燥了，在实际开发过程中几乎没主动使用过，不过并不代表它不存在哈，这哥们无处不在。。。

通过`Bionic库`的学习呢，至少可以:

- 了解`Android`在一些系统库的实现与`Linux`实现的区别
- 了解`Android`是怎么执行`系统调用`的
- 了解`内存管理函数`并再一次认识`Doug Lea`老爷子（猜猜为什么要说`再`）
- 了解`管道`这种比较原始的进程间通信手段
- 了解`Android`在线程管理上和`Linux`的区别
- 了解一个叫`Futex`的同步机制（YY：感觉jvm的同步和这个很像）
- 了解我们常用的`Log.d()`的具体实现和Log系统的结构
- 了解`.so`、`.o`都是什么类型的文件（`可执行文件`）以及通用结构是啥样子的
- 了解`动态链接`的核心模块`linker`是怎样工作的
- 了解一个叫`ptrace`的系统调用，为以后`Hook API`做准备
- 了解到一些常用的开源协议
- 了解部分 Android 的变更和新特性
- 加深对Linux的认识

**咳咳，有木有发现这么多的`了解`字眼？因为从这几天本人大脑的表现来看，这种不常用的`姿势`大脑会习惯性的忘记，只能以`了解`来安慰自己了。。。。。**

-------导读到此结束-------

`Bionic库`到底是干啥用的呢？看下简介先。

# `Bionic`简介

> `Bionic`包含了系统中最基本的 lib 库，包括`libc`、`libm`、`libdl`、`libstdc++`、`libthread`，以及Android特有的链接器`linker`。

其实当时已经有成熟开源的`GNU Libc`库了，不过`GNU Libc`库遵守的是`GPL`开源协议。`GPL`有个特点就是`传染性`：一旦系统中有软件使用了GPL授权协议，那么该系统相关代码必须开源。

> 网上找到一篇讲解[Google和Linux 内核的GPL约束](https://www.ifanr.com/92261)的文章，可以了解下历史。而关于开源协议，文章最后罗列了一些常见的[开源协议](#开源协议)。

我们继续了解`Bionic`。

`Bionic`是 Google 在`BSD`开源协议的C库基础上加入 `Linux` 的特性而生成的。`Bionic`名字就是`BSD`和`Linux` 的混合。`BSD`协议是一种几乎不受限制的开源方式，比较受商业公司的喜爱。（普遍爱`加密`和`混淆`不是）

除了版权问题，性能也是Google重新开发libc库的原因。`Bionic`针对移动设备上有限的CPU周期和可用内存进行了裁剪（去掉了很多和进程、线程、同步相关的高级功能），在减少库大小的同时也提高了工作效率。

## `Bionic`中的模块简介

我们再看下`Bionic`的目录结构（简化版本）：

```
├── Android.mk
├── libc
├── libdl
├── libfdtrack  #5.0对比新增的，而且libthread_db不见了
├── libm
├── libstdc++
├── linker
```

对比9.0和5.0，这部分的变化挺大的，差异部分不先深究了（主要是跟`Linux`不怎么熟。。。），所以`Bionic`部分暂时以了解为主吧，迫不及待要看`Binder`了哈哈哈

### Libc 库

> Libc 库是C语言最基础的库文件，它提供了所有系统的基本功能，这些功能主要来源于它对系统调用的封装，Libc 库是应用和Linux内核交互的桥梁。

Libc 库提供的主要功能如下：

- 进程管理：包括进程的创建、调度策略的设置、优先级的调整等
- 线程管理：包括线程的创建和退出、线程的同步和互斥、线程本地储存等。
- 内存管理：包括内存的分配和释放
- 时间管理：包括获取和保存系统时间，获取当前系统运行时长等
- 时区管理：包括时区的设置和调整。
- 定时器管理：提供系统的定时服务。
- 文件系统管理：提供文件系统的挂载和移除功能。
- 文件管理：包括文件或者目录的创建、删除、属性修改等。
- Socket：创建、监听Socket；发送接收数据。
- DNS解析：帮助解析网络地址。
- 信号：用于进程间通信。
- 环境变量：设置和获取系统的环境变量。
- Android Log：提供和Android Log驱动进行交互的功能。
- Android 属性：管理一个共享区域来设置和读取Android的属性。
- 标准IO：提供格式化的输入输出功能
- 字符串：提供字符串的移动、复制、比较等功能
- 宽字符：提供对宽字符（应该就是是`UNICODE字符`）的支持

### Libm 库

> 数学函数库，提供了常见的数学函数和浮点运算功能。不过Android中的浮点运算是通过软件实现的，相比硬件支持的浮点运算运行速度慢，最好避免使用。

### libdl 库

> libdl 库原本是用于动态库的装载，但是Android的libdl 库中`dlopen`、`dlclose`、`dlsym`等函数的实现只是一个空壳。应用进程使用的`dlopen`等函数实际上是在`linker`模块中实现的。

### libstdc++ 库

> libstdc++ 是标准的c++库，但是在Android中的实现非常简单，只是`new`、`delete`等少数几个操作符的实现。

### Linker

> 首先`linker`不是编译程序时用的`链接器`，编译程序用到的链接器是`arm-elf-ld`。（此乃编译期）

> `linker`的功能是装载动态库以及用于库的重定位，相当于Linux中的`ld.so`。（此乃运行时）

Android没有使用Linux的`ld.so`，而是自己开发了这个`linker`程序

# `Bionic` C 库中的系统调用

> 有点偏底层了，忍住、忍住，继续学习

## 系统调用简介

> Linux 的系统调用就是 Linux 内核提供给用户进程使用的一套接口。用户进程可以通过这些接口来获得Linux内核提供的服务，例如：打开和关闭文件、启动退出一个进程、分配和释放内存等。

> 但是Linux的系统调用并不等同于普通的API调用，api是函数的定义，规定了这个函数的功能，跟内核无直接关系。而系统调用是通过中断向内核发请求，实现内核提供的某些服务。

现代CPU一般都实现了`特权等级`(x86 CPU)或`工作模式`(arm CPU)

x86 CPU包含4个特权级别`Ring0`~`Ring3`。

- `Ring0`的权限最高，可以使用任何CPU指令，Linux的内核代码就是运行在这个级别下。
- `Ring3`的权限最低，很多CPU指令都被限制使用，普通用户进程的代码就运行在`Ring3`下。

arm CPU则有7中工作模式：

- 用户模式（usr）：普通用户进程工作的模式，权限比较受限
- 快速中断模式（fiq）：用于高速数据传输或通道处理
- 外部中断模式（irq）：用于通用的中断处理
- 管理模式（svc）：操作系统使用的保护模式（高权限），复位和软件中断进入
- 数据访问终止模式（abt）：当数据或指令预取终止时进入该模式，可用于虚拟内存及存储保护
- 系统模式（sys）：运行均有特权的操作系统任务
- 定义指令终止模式（und）：用于支持硬件协处理器的软件仿真（浮点、微量运算）

无论是x86的特权等级还是arm的工作模式，目的都是将系统内核和用户进程分开，防止用户进程对系统内核进行破坏。同时，系统内核也能必须为用户进程提供服务。

### 应用程序如何使用系统调用呢？

> 首先，我们要了解的是，操作系统一般是通过中断来从用户态切换到内核态的。

那我们下面了解下中断的相关姿势

#### 中断的相关知识

> 中断一般分为三类：
>
> - 由计算机硬件异常或故障引起的中断，称为`内部异常中断`；
> - 由程序中执行了引起中断的指令而造成的中断，称为`软中断`（这也是和我们将要说明的系统调用相关的中断，`软中断`通常是一条指令，使用这条指令用户可以手动触发某个中断。）；
> - 由外部设备请求引起的中断，称为`外部中断`。

> 简单来说，中断的理解就是中止当前运行去处理一些特殊事情。

中断一般有两个属性，`中断号`和`中断处理程序`。

不同的中断有不同的`中断号`，每个`中断号`都对应了一个`中断处理程序`。

内核中有一个叫`中断向量表`的数组来映射这个关系。当中断到来时，cpu会暂停正在执行的代码，根据`中断号`去`中断向量表`找出对应的`中断处理程序`并调用执行。执行完成后，会继续执行之前的代码。

> `中断号`是有限的，所有不会用一个中断来对应一个系统调用（系统调用有很多）。**Linux下用`int 0x80`触发所有的系统调用**。

**那如何区分不同的调用呢**？对于每个系统调用都有一个系统调用号，在触发中断之前，会将系统调用号放入到一个固定的寄存器，`0x80`对应的`中断处理程序`会读取该寄存器的值，然后决定执行哪个系统调用的代码。

#### 不同平台的系统调用

> 在x86平台下，应用程序使用软中断`0x80`来调用系统功能；arm平台则使用swi软中断。同时Linux为每个系统调用都进行了编号（0~NR_syscall），并在内核中保存了一张系统调用表，这张表保存了系统调用编号和它对应的服务程序。

在x86上，通过`eax`寄存器来传递系统调用号。例如：

```
movl    $__NR_brk, %eax
 int    $0x80,     120
复制代码
```

除了使用`eax`传递系统调用号外，许多系统调用还需要传递一些参数到内核。x86平台按顺序使用寄存器`ebx`、`ecx`、`edx`、`esi`和`edi`来传递参数。例如：

```
mov    16(%esp),    %ebx
mov    20(%esp),    %ecx
mov    24(%esp),    %edx
movl   $__NR_write,  %eax
int    $0x80
复制代码
```

在arm平台中，Android中的系统调用是通过`swi`的0号软中断来实现的，例如：

```
mov    ip,  r7
ldr    r7,  =__NR_brk
swi    0
复制代码
```

**==好的，简单了解了中断的内容，不过指令示例看的有点晕，下面单独对这部分说明一下==**

#### 中断相关的指令说明

寄存器分类([详情请看这位大神的文章](https://blog.csdn.net/chenlycly/article/details/37912683))

- 4个数据寄存器：`EAX`、`EBX`、`ECX`、`EDX`
- 2个变址寄存器：`ESI`、`EDI`
- 2个指针寄存器：`ESP`、`EBP`
- 6个段寄存器：`ES`、`CS`、`SS`、`DS`、`FS`、`GS`
- 1个指令指针寄存器：`EIP`
- 1个标志寄存器：`EFlags`

中断指令：

- swi：arm 软中断指令（Software Interrupt, SWI）

  > 指令格式如下：`swi immed_24`

  - 其中：`immed_24` 24位立即数，值为从0――16777215之间的整数。内核程序通过该软中断立即数来区分用户不同操作，执行不同内核函数。

- int：x86 软中断指令

  > 指令格式如下：`int op_num`

  - 其中:`op_num`表示对应的中断号

上面用到的相关`ARM`指令：

- `mov` ：寄存器间数据移动指令
- `ldr`：内存和CPU间数据移动指令

上面用到x86架构指令：

- x86中没有`ldr`指令，因为x86的`mov`指令可以将数据从内存中移动到寄存器中。

- ```
  mov
  ```

  x：其中 x 可以是下面的字符：

  - `l`用于32位的长字值
  - `w`用于16位的字值
  - `b`用于8位的字节值

**==有了这部分上面的指令就比较好理解了。不管怎样到这里先暂停了百度了，越查资料问题越多。。。。==**

## 系统调用的实现方法

在路径`bionic/libc/arch-x86/syscalls`下存放的是系统调用的汇编代码（arm、mips的实现代码在`arch-arm`和`arch-mips`目录下）。

每个系统调用放在一个文件中，每个文件中只有一小段的汇编代码实现，称为`syscall stub`，以`mount.S`文件为例，我们看下文件内容：

```
/* Generated by gensyscalls.py. Do not edit. */
#include <private/bionic_asm.h>
ENTRY(mount)
    pushl   %ebx
    .cfi_def_cfa_offset 8
    .cfi_rel_offset ebx, 0
    pushl   %ecx
    .cfi_adjust_cfa_offset 4
    .cfi_rel_offset ecx, 0
    pushl   %edx
    .cfi_adjust_cfa_offset 4
    .cfi_rel_offset edx, 0
    pushl   %esi
    .cfi_adjust_cfa_offset 4
    .cfi_rel_offset esi, 0
    pushl   %edi
    .cfi_adjust_cfa_offset 4
    .cfi_rel_offset edi, 0

    call    __kernel_syscall
    pushl   %eax
    .cfi_adjust_cfa_offset 4
    .cfi_rel_offset eax, 0

    mov     28(%esp), %ebx
    mov     32(%esp), %ecx
    mov     36(%esp), %edx
    mov     40(%esp), %esi
    mov     44(%esp), %edi
    movl    $__NR_mount, %eax
    call    *(%esp)
    addl    $4, %esp

    cmpl    $-MAX_ERRNO, %eax
    jb      1f
    negl    %eax
    pushl   %eax
    call    __set_errno_internal
    addl    $4, %esp
1:
    popl    %edi
    popl    %esi
    popl    %edx
    popl    %ecx
    popl    %ebx
    ret
END(mount)
复制代码
```

全是一些汇编指令，指令的具体内容就不详解了，可以参照[中断相关的指令说明](#中断相关的指令说明)

还有一点就是这些代码不是人工手写的，是通过`gensyscalls.py`脚本根据`bionic/libc/SYSCALLS.TXT`文件生成的。

我们看下`SYSCALLS.TXT`文件部分内容格式：

```
# signals
int     __sigaction:sigaction(int, const struct sigaction*, struct sigaction*)  arm,mips,x86
int     __rt_sigaction:rt_sigaction(int, const struct sigaction*, struct sigaction*, size_t)  all
复制代码
```

标准的格式（这部分9.0和5.0还是不太一样的，列出的是9.0格式）：

```
return_type    func_name[:syscall_name[:call_id]]([parameter_list])    platform
复制代码
```

分为三部分：

- 第一部分是函数的返回类型。

- 第二部分是函数名和参数。

  - `func_name`指的是函数名称，也就是C程序调用时的名称。

  - ```
    syscall_name
    ```

    指的是系统调用的名称，这是一个内部名称，主要用来生成系统编号的宏定义。生成宏定义的方法是在

    ```
    syscall_name
    ```

    前面加上字符串

    ```
    __NR_
    ```

    。以

    ```
    _exit
    ```

    函数为例，它在

    ```
    SYSCALLS.TXT
    ```

    文件中的定义为

    ```
    void    _exit:exit_group(int)    all
    复制代码
    ```

    对应生成的系统编号的宏定义为

    ```
    __NR_exit_group
    ```

    。这些宏是在 Linux kernel 代码中定义的。

  - `call_id`指的是系统调用号，一般不用列出。通过`syscall_name`生成的宏定义可以得到系统调用编号。

  - `parameter_list` 指的是参数列表

- 第三部分是用来指定目标平台，包括：`all`、`arm`、`mips`、`x86`

# `Bionic`中的内存管理函数

> 暂不细谈哈，再细谈要偏离我在本书的学习路线了。。

## 关于内存的一些说明

> 对于传统32位处理器来说，寻址空间最大为4GB。其中`0~3 GB`的`地址空间`分配给用户进程使用，`3~4 GB`由内核使用。

用户进程并不是在启动时就获得了对所有`0~3 GB`地址空间的访问权限，而是要事先向内核申请对某块地址的读写权限。而且申请的只是`地址空间`而已，并没有分配实际的`物理内存`。

只有当进程访问某个内存地址时，如果该地址对应的物理页面不存在，则由内核产生`缺页中断`，在中断中才会分配物理内存并建立页表。如果用户进程不需要某块`地址空间`了，可以通过内核释放掉他们，对应的物理内存也同时被释放掉。

由于`缺页中断`运行缓慢，如果频繁的由内核来分配释放内存会降低整个系统的性能。因此，一般操作系统都会在用户进程提供地址空间的分配和回收机制，也就是`内存管理器`:

- `内存管理器`会预先向内核申请一块大的地址空间，称为`堆`。
- 当用户进程需要分配内存时，由`内存管理器`从`堆`中寻找一块空闲的内存分配给用户进程使用。
- 当用户进程释放某块内存时，`内存管理器`并不会立即交给内核释放，而是放到空闲列表中，留待下次分配使用。

`内存管理器`也会动态调整堆的大小：

- 当堆空间使用完了会继续向内核申请
- 当堆中内存空闲太多也会返还一部分给内核

## Linux 的内存管理方法

> Linux 有两种方式来申请和释放内存空间，一种是使用系统调用`brk`；另一种是使用系统调用`mmap`和`munmap`。

Linux 内核通常会将用户地址空间划分为一些大的区域，如`代码区`、`数据区`、`堆`、`栈`等。

先看个示意图： ![image](images/e015787268674dd3848af5ff1a7c11e0~tplv-k3u1fbpfcp-zoom-1.image)

### `brk`

> 系统调用`brk`的作用是调整`堆`的`高地址边界`。分配内存时把边界推高，释放内存时把边界拉低。

`brk`的优点是分配内存快，缺点是可能分配不到大块的内存空间。因为`堆区`和`栈区`之间的区域**不是完全的空白区域**，可能有部分内存已经被分配出去了。

`堆区`的`高地址边界`如果遇到了已分配的区域就会导致分配失败。因此`brk`通常用来分配比较小的内存空间，比如小于`256KB`的内存块。

### `mmap`

> 系统调用`mmap`用来分配大块的内存空间。`mmap`会在`堆区`和`栈区`之间寻找一块合适的空间分配给用户进程使用。

- 如果内存大小不合适，还可以通过系统调用`mremap`来改变大小。
- 使用完成后可以通过`munmap`释放掉内存空间。

## Bionic 的内存管理器

> Android 7.0 以前是可以指定`dlmalloc`或者`jemalloc`来作为内存管理器的；后来  7.0 增加了一个`Project Svelte`；再后来，我在9.0的项目中找不到`dlmalloc`的独立源码了。。

`jemalloc`的知乎[传送门](https://zhuanlan.zhihu.com/p/48957114)

而关于`dlmalloc`，这部分来自百度百科哈。

> `dlmalloc` 是目前一个十分流行的内存分配器，其由`Doug Lea`从 1987 年开始编写，到目前为止，最新版本为2.8.3，由于其高效率等特点被广泛的使用和研究。

`dlmalloc的`实现只有一个源文件（还有一个头文件），大概5000行，其内注释占了大量篇幅，由于有这么多注释存在的情况下，表面上看上去很容易懂，的确如此，在不追求细节的情况，对其大致思想的确很容易了解（没错，就只是了解而已），但是`dlmalloc`作为一个高品质的佳作，实现上使用了非常多的技巧，**在实现细节上不花费一定的精力**是没有办法深入理解其为什么这么做，这么做的好处在哪，只有当真正读懂后回味起来才发现它是如此美妙。

**这是继`Java 多线程`后又一次听说`Doug Lea`了，多线程核心几乎是老爷子一个人手撸出来的（尤其是`AQS`），真滴大神哇**

尽管时至今日，`dlmalloc`中的技术在一些地方已然落后于时代，已经很多优秀的`allocator`：像 google的`tcmalloc`,  freeBSD的`jemalloc`等在某些情况下性能可以达到`dlmalloc`的数十甚至上百倍。但`dlmalloc`的很多思想和基本算法对后来者产生了深远的影响。

### `dlmalloc` 的简单了解

[`dlmalloc`源码下载链接](http://cdn.hualee.top/malloc.c)

> `dlmalloc`以链表的方式将`堆区`的空闲空间根据尺寸组合在一起。分配内存时通过这些链表能快速找到合适大小的内存。如果不能找到满足要求的空闲空间，`dlmalloc`会使用系统调用扩大`堆`空间

先看下分块示意图： ![image](images/7592fb5c75d04fabad3f13300192430d~tplv-k3u1fbpfcp-zoom-1.image)

`dlmalloc`的内存块被称为`trunk`，每块大小要求按地址对齐（默认为8字节），因此`trunk`块的大小必须是8的倍数。

`dlmalloc`使用三种不同的链表结构来组织不同大小的空闲内存块：

- 尺寸小于 256 Byte的块使用结构`malloc_chunk`按尺寸组织在一起。由于空间小于256字节，因此，最多使用32个`malloc_chunk`结构的环形链表来组织小于256字节的块
- 大于 256 Byte的块由结构`malloc_tree_chunk`组成的链表管理，这些块根据尺寸组织成二叉树
- 当超过某个更大的尺寸（`DEFAULT_MMAP_THRESHOLD`指定的默认阈值为256 KB），则由系统通过`mmap`的方式单独分配一块内存空间，并通过结构`malloc_segment`组成的链表进行管理

#### `dlmalloc`分配内存流程

通过查找这些链表来快速找到一块和要求尺寸最匹配的空闲内存块（这样可以尽可能的避免内存碎片）。如果没有合适大小的内存块，则将一块大的分成2部分，一块分配出去，另一部分根据大小再加入对应的空闲列表中。

#### `dlmalloc`释放内存流程

会将相邻的空闲块合并成一个大块来减少内存碎片。如果空闲块过多，超过了`dlmalloc`内部设定的一个阈值，`dlmalloc`就开始向系统返回内存（应该就是向内核释放）。

#### `dlmalloc`的函数的简单说明

> 书中还介绍了一些具体函数暂不扩展哈

`dlmalloc`除了能管理进程的`堆`空间外，还可以提供`私有堆`管理。所谓的`私有堆`，是指在`堆`外单独分配的一块地址空间，由`dlmalloc`按同样的方式进行管理。 可以通过以下特征区分是否为`私有堆`函数：

- `dlmalloc`中用来管理进程`堆`空间的函数都带有`dl`前缀，如`dlmalloc`、`dlfree`等
- 私有堆的管理函数则带有`mspace_`前缀，如`mspace_malloc`、`mspace_free`等

# 管道

> 管道是从Unix系统出现的一种进程间通信的手段，分为`匿名管道(PIPE)`和`命名管道(FIFO)`两种。

历史上管道是半双工的,即数据同一时刻只能在一个方向上流动。现在一些系统提供全双工的管道。但是为了可移植性，我们假定系统无此功能。

**如果需要`IPC`，最佳做法是使用`Binder`；如果只需要线程间通信，可以使用`匿名管道`**

## `匿名管道`一些知识

> `匿名管道`主要用于父子进程间的通信。**Android 支持**

创建`匿名管道`会在内核建立一个`内存文件`（这个文件在文件系统中不可见）和两个`文件描述符`。管道使用者通过这两个`文件描述符`来读写`内存文件`中的数据，从而达到信息交换的目的。

通常情况下，`文件描述符`不能在进程间传递，只有`父子进程`或`兄弟进程`间可以通过继承的方式来共享`文件描述符`。因此，`匿名管道`主要用于父子进程间的通信。

`匿名管道`属于`半双工`的数据只能从一端到另一端。示意图如下： ![image](images/e48d46d93d00481d998c284465a41d17~tplv-k3u1fbpfcp-zoom-1.image) 管道两端的进程将管道看成同一个文件，一个进程负责向管道中写，另一个进程则从管道中读取。如果进程间需要双向通信，则不需建立起两条管道。

我们看下`匿名管道`的特征：

- 只提供单向通信，也就是说，两个进程都能访问这个文件，假设进程1往文件内写东西，那么进程2 就只能读取文件的内容。
- 只能用于具有血缘关系的进程间通信，通常用于父子进程建通信
- 管道是基于字节流来通信的
- 依赖于文件系统，它的生命周期随进程的结束结束（随进程）
- 其本身自带同步互斥效果

`匿名管道`最大的好处是简单、灵活，但是只能用于父子进程间通信限制了它的使用。因此，后来出现了`命名管道`。

## `命名管道`的特点

> `命名管道`又被称为先进先出队列(`FIFO`)，是一种特殊的管道，通过建立一个`inode`节点存在于文件系统中。**Android 不支持（貌似与`mkfifo`和`FAT32`文件系统格式有关）**

`命名管道`与`匿名管道`非常类似，但是又有自身的显著特征：

- `命名管道`可以用于任何两个进程间的通信，而不限于同源的两个进程。
- `命名管道`作为一种特殊的文件存放在文件系统中，而不是像`匿名管道`那样存放在内核中。当进程对`命名管道`的使用结束后，`命名管道`依然存在于文件系统中，除非对其进行删除操作，否则该命名管道不会自行消失。

和`匿名管道`一样，`命名管道`也只能用于数据的单向传输，如果要用`命名管道`实现两个进程间数据的双向传输，建议使用两个单向的命名管道。

# `Bionic`中的线程管理函数

> `Bionic`中的线程管理函数和统一 Linux 版本的实现有很多差异，Android 根据自己的需要做了很多裁剪工作。

## `Bionic`线程函数的特性

> Android 线程管理`pthread`相关的源码实现位于`bionic/libc/bionic/pthread*`

Android中的`pthread`基于`Futex`实现，并同时使用更简短的代码来实现通用操作，简单记录下部分特性：

- `pthread_mutex_t`,`pthread_cond_t`定义的类型只有4个字节
- 支持`normal`、`recursive`、`error-check`互斥量属性。
  - `PTHREAD_MUTEX_NORMAL`：这种类型的互斥锁不会自动检测死锁。如果一个线程试图对一个互斥锁重复锁定，将会引起这个线程的死锁。如果试图解锁一个由别的线程锁定的互斥锁会引发不可预料的结果。如果一个线程试图解锁已经被解锁的互斥锁也会引发不可预料的结果。
  - `PTHREAD_MUTEX_ERRORCHECK`：这种类型的互斥锁会自动检测死锁。 如果一个线程试图对一个互斥锁重复锁定，将会返回一个错误代码。 如果试图解锁一个由别的线程锁定的互斥锁将会返回一个错误代码。如果一个线程试图解锁已经被解锁的互斥锁也将会返回一个错误代码。
  - `PTHREAD_MUTEX_RECURSIVE`：如果一个线程对这种类型的互斥锁重复上锁，不会引起死锁。一个线程对这类互斥锁的多次重复上锁必须由这个线程来重复相同数量的解锁，这样才能解开这个互斥锁，别的线程才能得到这个互斥锁。如果试图解锁一个由别的线程锁定的互斥锁将会返回一个错误代码。如果一个线程试图解锁已经被解锁的互斥锁也将会返回一个错误代码
- 不支持`pthread_cancel`函数。与之替代的是`pthread_cleanup_push`、`pthread_cleanup_pop`以及`pthread_exit`函数。

> 这部分其实还包括`pthread`的线程操作函数、`TLS`线程本地储存、互斥量`Mutex`、条件量`Condition`的介绍，感觉有些深入。所以先简单了解到这里吧，等需要的时候再来细看。（PS：偷个懒）

# `Futex`同步机制

> `Futex`是`fast userspace mutex`的缩写。`Futex`是Linux的一个基础组件，可以用来构建各种更高级别的同步机制，比如锁或者信号量等等。Android中不但线程函数使用了`Futex`，甚至一些模块中也直接使用了`Futex`作为进程间同步的手段。

Linux 从 2.5.7 开始支持`Futex`。在类Unix系统中，传统的进程间同步机制都是通过对内核对象进行操作来完成的，这个内核对象在需要同步的进程中都是可见的。这种方式因为涉及到内核态与用户态的切换，效率比较低。

`Futex`的解决思路是：在无竞争的情况下操作完全在`user space`进行，不需要`系统调用`，仅在发生竞争的时候进入内核去完成相应的处理(`wait` 或者 `wake up`)。所以说，futex是一种`user mode`和`kernel mode`混合的同步机制，需要两种模式合作才能完成，`Futex`变量必须位于`user space`，而不是`内核对象`，`Futex`的代码也分为`user mode`和`kernel mode`两部分，无竞争的情况下在`user mode`，发生竞争时则通过`sys_futex`系统调用进入`kernel mode`进行处理。

## `Futex`的系统调用

在Linux中，`Futex`的系统调用如下：

```c
#define __NR_futex      240
复制代码
```

对应的`Futex`系统调用的原型是：

```c
#include <linux/futex.h>
#include <sys/time.h>
int futex (int *uaddr, int op, int val, const struct timespec *timeout,int *uaddr2, int val3);
复制代码
```

其中：

- `*uaddr`：就是用户态下共享内存的地址，里面存放的是一个对齐的整型计数器

- ```
  op
  ```

  ：表示操作类型，有五种预定义值，在

  ```
  Bionic
  ```

  中只使用了下面两种：

  - `FUTEX_WAIT`: 原子性的检查`uaddr`中计数器的值是否为`val`,如果是则让进程休眠，直到`FUTEX_WAKE`或者超时(`timeout`)。也就是把进程挂到`uaddr`相对应的等待队列上去。
  - `FUTEX_WAKE`: 最多唤醒`val`个等待在`uaddr`上进程。

在`Bionic`中，提供了两个函数来包装`Futex`系统调用（位于`bionic/libc/private/bionic_futex.h`）：

```
static inline int __futex_wait(volatile void* ftx, int value, const timespec* timeout) {
  return __futex(ftx, FUTEX_WAIT, value, timeout, 0);
}
static inline int __futex_wake(volatile void* ftx, int count) {
  return __futex(ftx, FUTEX_WAKE, count, NULL, 0);
}
复制代码
```

还有两个类似的函数：

```
static inline int __futex_wake_ex(volatile void* ftx, bool shared, int count) {
  return __futex(ftx, shared ? FUTEX_WAKE : FUTEX_WAKE_PRIVATE, count, NULL, 0);
}
static inline int __futex_wait_ex(volatile void* ftx, bool shared, int value) {
  return __futex(ftx, (shared ? FUTEX_WAIT_BITSET : FUTEX_WAIT_BITSET_PRIVATE), value, nullptr,
                 FUTEX_BITSET_MATCH_ANY);
}
复制代码
```

`_ex`后缀的函数对比前两个函数多了一个`shared`参数：

- 当`shared`的值为`true`时，表示`wait`和`wake`操作是用于**进程间**的挂起和唤醒
- 当`shared`的值为`false`时，表示`wait`和`wake`操作是用于**进程内线程间**的挂起和唤醒

## `Futex`的同步逻辑

> 首先明确下`Futex`变量值的状态：
>
> - 0 表示无锁状态；
> - 1 表示有锁无竞争状态；
> - 2 表示有竞争状态。

流程如下：

- 创建一个全局的整型变量作为`Futex`变量（一个整型计数器），初始值为0。如果用于进程间同步，这个变量必须位于共享内存。

- 当进程或线程持有锁的时候，检查

  ```
  Futex
  ```

  变量是否为 0。

  - 如果为0，将`Futex`变量设置为 1 然后继续执行。
  - 如果不为0，将`Futex`变量设置为 2 以后，执行`FUTEX_WAIT`系统调用进入挂起等待状态。

- 当进程或线程释放锁的时候

  - 如果`Futex`变量的值为 1，说明没有其他线程在等待锁，直接将`Futex`变量设置为 0 就结束了。
  - 如果`Futex`变量的值为 2，说明还有线程在等待锁，此时将`Futex`变量设置为 0，同时执行`FUTEX_WAKE`系统调用来唤醒等待的进程。

对于`Futex`变量的操作，需要保证`比较`和`赋值`操作是原子的。

## `Mutex`类

> Glibc库中实现有`pthread_mutex_lock()`/`pthread_mutex_unlock()`等用户态锁接口，以提供快速的futex机制。

> `Bionic` 的`pthread`实现中也提供了标准的`pthread_mutex_lock()`/`pthread_mutex_unlock()`接口。

`Mutex`类封装了`pthread_mutex_lock()`和`pthread_mutex_unlock()`接口。

不过书中的源码路径已经不好使了，在9.0上的路径是：`system/core/libutils/include/utils/Mutex.h`，可以参考下。

# Android中的`Log`模块

> 由于Android的开发是`Host-Target`模式，解决问题的主要手段就是分析log，

我们看下Log系统的架构： ![image](images/5e7d10f6591b430e9a3c6b002657f126~tplv-k3u1fbpfcp-zoom-1.image)

Log系统的输出分为5级：

- `ERROR`：用来输出错误信息
- `WARN`：用来输出警告信息
- `INFO`：用来输出一般性的提示信息
- `DEBUG`：用来输出调试信息
- `VERBOSE`：用来输出价值比较低的信息

**Log分级不是强制的，但是正确使用能让调试更加方便**

Android 的Log输出量巨大，特别是通信系统的Log很多，因此Android把Log输出到了不同的缓冲区。目前定义的缓冲区包括：

```
public static final int LOG_ID_MAIN = 0; //Java层的log
public static final int LOG_ID_RADIO = 1; //通信系统的log
public static final int LOG_ID_EVENTS = 2; //event模块的log
public static final int LOG_ID_SYSTEM = 3; //系统组件的log
public static final int LOG_ID_CRASH = 4; //crash信息
复制代码
```

缓冲区的定义主要是给系统组件用的。Java层的`Log.*`打印都会输出到`LOG_ID_MAIN`中。

## Log 系统的接口和用法

### Java 层的接口调用

`android.util.Log`类中常用的`Log.*(String tag, String msg)`就不介绍了，很常用的方法。我们看下几个特殊的：

- `Log.*(String tag, String msg, Throwable tr)`：增加的`Throwable tr`是为了在出现异常时更方便的打印堆栈信息。
- `Log.wtf()系列`：配合`setWtfHandler(TerribleFailureHandler handler)`使用，通过`setWtfHandler`设置带有回调函数的`handler`，这样在使用`Log.wtf()`时，可以统一处理这种严重情况。

我们再看下`Log.java`类中`Log.*(String tag, String msg)`具体的调用：

```
public static int d(String tag, String msg) {
    return println_native(LOG_ID_MAIN, DEBUG, tag, msg);
}
复制代码
```

`println_native`是一个JNI调用，位于`frameworks/base/core/jni/android_util_Log.cpp`，我们看下部分内容：

```
static const JNINativeMethod gMethods[] = {
    /* name, signature, funcPtr */
    { "isLoggable",      "(Ljava/lang/String;I)Z", (void*) android_util_Log_isLoggable },
    { "println_native",  "(IILjava/lang/String;Ljava/lang/String;)I", (void*) android_util_Log_println_native },
    { "logger_entry_max_payload_native",  "()I", (void*) android_util_Log_logger_entry_max_payload_native },
};

static jint android_util_Log_println_native(JNIEnv* env, jobject clazz,
        jint bufID, jint priority, jstring tagObj, jstring msgObj)
{
    // 部分省略
    
    int res = __android_log_buf_write(bufID, (android_LogPriority)priority, tag, msg);
    
    // 部分省略
}
复制代码
```

`println_native`最后调用到了位于`system/core/liblog`模块定义的`__android_log_buf_write`函数。

### native 层的调用

native 层使用的其实是宏定义，常用的形式包括：

- `ALOGV`：相当于`Log.v()`
- `ALOGD`：相当于`Log.d()`
- `ALOGW`：相当于`Log.w()`
- `ALOGI`：相当于`Log.i()`
- `ALOGE`：相当于`Log.e()`

上面这几个指令都会输出到`LOG_ID_MAIN`的缓冲区。以`ALOGD`为例，我们看下定义文件（`system/core/liblog/include/log/log_main.h`）的部分内容：

```c
#ifndef ALOGD
#define ALOGD(...) ((void)ALOG(LOG_DEBUG, LOG_TAG, __VA_ARGS__))
#endif

#ifndef ALOG
#define ALOG(priority, tag, ...) LOG_PRI(ANDROID_##priority, tag, __VA_ARGS__)
#endif

#ifndef LOG_PRI
#define LOG_PRI(priority, tag, ...) android_printLog(priority, tag, __VA_ARGS__)
#endif

#define android_printLog(prio, tag, ...) \
  __android_log_print(prio, tag, __VA_ARGS__)
复制代码
```

最后其实是调用的`__android_log_print`函数，我们再来看下函数的实现`system/core/liblog/logger_write.c`文件：

```
LIBLOG_ABI_PUBLIC int __android_log_print(int prio, const char* tag,
                                          const char* fmt, ...) {
  //省略部分内容
  return __android_log_write(prio, tag, buf);
}
LIBLOG_ABI_PUBLIC int __android_log_write(int prio, const char* tag,
                                          const char* msg) {
  return __android_log_buf_write(LOG_ID_MAIN, prio, tag, msg);
}
复制代码
```

最后也指向了`__android_log_buf_write`方法，等下我们仔细看下这个方法

此外，还有两组指令分别是：

- `SLOG*`：输出到`LOG_ID_SYSTEM`的缓冲区，定义文件为`system/core/liblog/include/log_system.h`
- `RLOG*`：输出到`LOG_ID_RADIO`的缓冲区，定义文件为`system/core/liblog/include/log_radio.h`

### `Java` 和 `native` 调用跟进

跟进上面的分析，我们发现native层和Java层最后调用到了`__android_log_buf_write`，我们看下内容：

```c
LIBLOG_ABI_PUBLIC int __android_log_buf_write(int bufID, int prio,
                                              const char* tag, const char* msg) {
  //省略部分内容
  return write_to_log(bufID, vec, 3);
}
static int __write_to_log_init(log_id_t, struct iovec* vec, size_t nr);
static int (*write_to_log)(log_id_t, struct iovec* vec,
                           size_t nr) = __write_to_log_init;
复制代码
```

`__android_log_buf_write`调用了`write_to_log(bufID, vec, 3)`，而`write_to_log`是一个指针，指向了函数`__write_to_log_init`（源码中充分利用了`write_to_log`函数指针，变换指针指向的函数来完成log的输出工作）。

根据书中的资料显示，最后会指向系统调用`write()`通过kernel层的log驱动来打印输出，这部分在9.0上并不是很清晰，跟踪源码只找到了如下部分信息：

```c
static int __write_to_log_daemon(log_id_t log_id, struct iovec* vec, size_t nr) {
 //省略部分内容
  write_transport_for_each(node, &__android_log_transport_write) {
    if (node->logMask & i) {
      ssize_t retval;
      retval = (*node->write)(log_id, &ts, vec, nr);
      if (ret >= 0) {
        ret = retval;
      }
    }
  }
//省略部分内容
  write_transport_for_each(node, &__android_log_persist_write) {
    if (node->logMask & i) {
      (void)(*node->write)(log_id, &ts, vec, nr);
    }
  }

}
复制代码
```

感觉也就是此处的`write()`最后会执行到系统调用`write()`那里吧。

**就先到这里吧，先往下进行了，在`Bionic`花费的时间有点长了**

# 可执行文件格式分析

> 在分析`Bionic`的`linker`之前，先介绍下Android的可执行文件格式。`linker`本身不是很复杂，但是对可执行文件不了解的话，就不太容易理解程序的逻辑。

Android 的可执行文件和动态库就是Linux的 `ELF` 文件格式，但是，由于Android使用了自己的`linker`，因此和普通的Linux系统不完全兼容。

## `ELF` 文件格式简介

> `ELF`是`Executable and Linkable Format`的缩写，最初由Unix实验室发布，它是`ABI`的一部分。`ELF`标准的目的是为软件开发人员提供一组二进制接口定义，这些接口可以在多种操作系统环境下生效，从而减少二次开发的工作量

`ELF`文件以`节(section)`的方式组织在一起的，`节(section)`描述了文件的各项信息，例如：代码、数据、符号表、重定位表、全局编译表等。

可执行文件被装载进内存时，并不是被完整的映射进内存，而是根据`ELF`文件中格式的定义，一段一段的装载进去，因此，可执行文件的格式和内存的映像并不完全相同，文件装载进内存后是以`段(segment)`的方式来组织，如：代码段、数据段、动态段等。

`ELF` 格式的文件结构和内存结构对比图： ![image](images/3c00b4eeca1a41d5a0a3a2e2b4f19cd7~tplv-k3u1fbpfcp-zoom-1.image)

`ELF` 格式的文件有三种：

- 可执行文件
- 动态库文件(`.so`文件)
- 重定位文件(`.o`文件)

这三种都有一个`ELF`文件头，描述了整个可执行文件的基本信息，如目标代码的格式、体系结构、各种段或节的偏移和大小等。

可执行文件和动态库会有`程序头部表(Program Header Table)`，但是重定位文件中没有。

`ELF`文件中还有一个`节区头部表(Section Header Table)`，描述文件中各个`节区`的内容。这个表和`程序头部表`的内容有些重复，这是因为这两张表的用途不一样：

- 在编译和链接阶段，也就是`ELF`文件的生成阶段，需要`节区头部表(Section Header Table)`
- `程序头部表(Program Header Table)`是在在`ELF`文件的装载阶段

分析`ELF`文件格式的目的，是为了了解可执行文件的装载过程，因此会重点学习`程序头部表`哈

## `ELF`文件头

> 书中`ELF`文件格式的定义和Android 9 中定义位置有些不同，9.0源码文件位于`bionic/libc/kernel/uapi/linux/elf.h`

`ELF`文件头的定义如下：

```c
typedef struct elf32_hdr {
  unsigned char e_ident[EI_NIDENT];     //目标文件标识
  Elf32_Half e_type;                    //目标文件类型
  Elf32_Half e_machine;                 //目标运行平台的体系结构
  Elf32_Word e_version;                 //目标文件版本
  Elf32_Addr e_entry;                   //程序的入口地址
  Elf32_Off e_phoff;                    //程序头部表的偏移量
  Elf32_Off e_shoff;                    //节区头部表的偏移量
  Elf32_Word e_flags;                   //文件相关的，特定于处理器的标志
  Elf32_Half e_ehsize;                  //ELF 头部字节的大小
  Elf32_Half e_phentsize;               //程序头部表的表项的字节大小
  Elf32_Half e_phnum;                   //程序头部表的表项数目
  Elf32_Half e_shentsize;               //节区头部表的表项的字节大小
  Elf32_Half e_shnum;                   //节区头部表的表项数目
  Elf32_Half e_shstrndx;                //节区头部表中字符串的索引表
} Elf32_Ehdr;
复制代码
```

在程序头部表里，最重要的是记录`程序头部表`和`节区头部表`的位置，表示表项数目和表项大小的字段。可以通过`readelf`和`objdump`指令查看。

以`readelf -h linker`为例，我们看下头部信息：

```shell
ELF 头：
  Magic：   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
  类别:                              ELF32
  数据:                              2 补码，小端序 (little endian)
  版本:                              1 (current)
  OS/ABI:                            UNIX - System V
  ABI 版本:                          0
  类型:                              DYN (共享目标文件)
  系统架构:                          ARM
  版本:                              0x1
  入口点地址：               0x1a640
  程序头起点：          52 (bytes into file)
  Start of section headers:          1162348 (bytes into file)
  标志：             0x5000200, Version5 EABI, soft-float ABI
  本头的大小：       52 (字节)
  程序头大小：       32 (字节)
  Number of program headers:         9
  节头大小：         40 (字节)
  节头数量：         25
  字符串表索引节头： 22
复制代码
```

## 程序头部表 (Program Header Table)

> `程序头部表 (Program Header Table)`的作用是记录文件中各种段的地址、大小等信息，在程序装载、链接时都需要它。

`程序头部表`是一个结构`Elf32_Phdr`的数组，每个结构中记录了装入内存中的各个`段`的信息，包括类型、地址、大小等。

结构`Elf32_Phdr`定义如下：

```c
typedef struct elf32_phdr {
  Elf32_Word p_type;        //段的类型
  Elf32_Off p_offset;       //段在文件中的偏移
  Elf32_Addr p_vaddr;       //段装入内存后的虚拟地址
  Elf32_Addr p_paddr;       //段装入内存后的物理地址
  Elf32_Word p_filesz;      //段在文件中的大小
  Elf32_Word p_memsz;       //段装入内存后的大小
  Elf32_Word p_flags;       //段的标志
  Elf32_Word p_align;       //内存对齐方式
} Elf32_Phdr;
复制代码
```

我们看下`p_type`字段定义的段类型：

| 名字    | 数值 | 说明                                                         |
| ------- | ---- | ------------------------------------------------------------ |
| NULL    | 0    | 表示此数组项未使用                                           |
| LOAD    | 1    | 表示此数组项描述了一个可加载的`段`，`段`的大小由`p_memsz`和`p_filesz`指定。一个`可执行文件`中可以有多个`LOAD`段 |
| DYNAMIC | 2    | 表示此数组项描述了动态链接信息。关于动态链接的所有区都在此段中描述。数组的长度并没有明确指定，而是将数组的最后一项值为NULL来表示数组的结束。 |
| INTERP  | 3    | 表示此数组项描述的动态装载器的信息，在Android中就是`Linker`。此类型仅对`可执行文件`有意义，在一个文件中只能有一个。如果必须存在该字段，其必须位于`LOAD`字段前 |
| NOTE    | 4    | 表示此数组项描述了附加信息的位置和大小                       |
| SHLIB   | 5    | 语义未指定。包含此种类型段的程序与ABI不符                    |
| PHDR    | 6    | 表示了此数组项描述了`程序头部表`自身在文件中及内存中的大小和位置。此类型的`段`在文件中只能有一个。如果存在此类型的`段`，则必须在所有可加载段项目的前面，包括`INTERP` |
| TLS     | 7    | 表示此数组项描述了线程局部存储模板信息                       |

我们通过`readelf -l linker`查看头部表的相关信息：

```
Elf 文件类型为 DYN (共享目标文件)
入口点 0x1a640
共有 9 个程序头，开始于偏移量 52

程序头：
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  PHDR           0x000034 0x00000034 0x00000034 0x00120 0x00120 R   0x4
  LOAD           0x000000 0x00000000 0x00000000 0xbeaf8 0xbeaf8 R E 0x1000
  LOAD           0x0bf730 0x000c0730 0x000c0730 0x05e48 0x0f494 RW  0x1000
  DYNAMIC        0x0c4b28 0x000c5b28 0x000c5b28 0x000b0 0x000b0 RW  0x4
  NOTE           0x000154 0x00000154 0x00000154 0x00020 0x00020 R   0x4
  GNU_EH_FRAME   0x0be814 0x000be814 0x000be814 0x002e4 0x002e4 R   0x4
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x10
  EXIDX          0x0a20e8 0x000a20e8 0x000a20e8 0x03b78 0x03b78 R   0x4
  GNU_RELRO      0x0bf730 0x000c0730 0x000c0730 0x058d0 0x058d0 RW  0x10

 Section to Segment mapping:
  段节...
   00     
   01     .note.gnu.build-id .dynsym .dynstr .gnu.hash .rel.dyn .text .ARM.exidx .rodata .ARM.extab .eh_frame .eh_frame_hdr 
   02     .data.rel.ro .init_array .dynamic .got .data .bss 
   03     .dynamic 
   04     .note.gnu.build-id 
   05     .eh_frame_hdr 
   06     
   07     .ARM.exidx 
   08     .data.rel.ro .init_array .dynamic .got
复制代码
```

虽然`程序头部表`包含很多个段，但是只有类型`LOAD`的`段`才会从文件映射到内存中。其余类型的`段`如果有实际的`节区`，这些`节区`也会出现在`LOAD`类型的`段`中。

从上面的打印信息分析：

- `linker`的`程序头`有9个`段`，其中包含2个`LOAD`类型，在装载这个文件时，实际`mmap`进内存的也只有这两个`段`，它们也就是所谓的`代码段`和`数据段`。从属性上也可以分辨一个是只读的`R`-`代码段`，一个是可读写的`RW`-`数据段`

- ```
  程序头
  ```

  下面是这9个

  ```
  段
  ```

  分别包含的

  ```
  节区
  ```

  。

  - `代码段`和`数据段`分别对应了节区的`01`和`02`项。
  - `DYNAMIC(03项)`只包含了一个`.dynamic`节区，这个`.dynamic`节区和`02`项中的`.dynamic`节区是同一个。只不过`.dynamic`节区的起始位置和大小等数据保存在`DYNAMIC(03项)`中。所以只能通过`DYNAMIC`段找到`.dynamic`节区。

所以，虽然`LOAD`段的地址空间覆盖了`.dynamic`节区，但是无法通过它来找到`.dynamic`节区，必须通过`DYNAMIC`段。这样设计的目的是，**当系统装载可执行文件时只需要将`LOAD`类型的`段`完整的映射进内存就完成了，而访问各个`节区`还是要通过相应的`段`所记录的地址来完成**

`.dynamic`节区一般定义了下列节区的起始地址、大小等内容。

- `.plt`节区：包含过程链接表
- `.got`节区：包含全局偏移表
- `rel.plt`节区：包含函数符号的重定位表
- `rel.dyn`节区：包含非函数符号的重定位表
- `.dynsym`节区：包含符号表
- `.dynstr`节区：包含字符串表
- `.hash`节区：包含符号的hash表

我们可以通过`readelf -d libjni_projector.so` 来查看`.dynamic`节区的结构。

```
Dynamic section at offset 0x5d60 contains 33 entries:
  标记        类型                         名称/值
 0x00000003 (PLTGOT)                     0x6eb4
 0x00000002 (PLTRELSZ)                   640 (bytes)
 0x00000017 (JMPREL)                     0x1660
 0x00000014 (PLTREL)                     REL
 0x00000011 (REL)                        0x1330
 0x00000012 (RELSZ)                      816 (bytes)
 0x00000013 (RELENT)                     8 (bytes)
 0x6ffffffa (RELCOUNT)                   68
 0x00000006 (SYMTAB)                     0x16c
 0x0000000b (SYMENT)                     16 (bytes)
 0x00000005 (STRTAB)                     0x74c
 0x0000000a (STRSZ)                      2489 (bytes)
 0x6ffffef5 (GNU_HASH)                   0x1108
 0x00000001 (NEEDED)                     共享库：[liblog.so]
 0x00000001 (NEEDED)                     共享库：[libandroid_runtime.so]
 0x00000001 (NEEDED)                     共享库：[libcutils.so]
 0x00000001 (NEEDED)                     共享库：[libnativehelper.so]
 0x00000001 (NEEDED)                     共享库：[libutils.so]
 0x00000001 (NEEDED)                     共享库：[libc++.so]
 0x00000001 (NEEDED)                     共享库：[libc.so]
 0x00000001 (NEEDED)                     共享库：[libm.so]
 0x00000001 (NEEDED)                     共享库：[libdl.so]
 0x0000000e (SONAME)                     Library soname: [libjni_projector.so]
 0x0000001a (FINI_ARRAY)                 0x6c30
 0x0000001c (FINI_ARRAYSZ)               4 (bytes)
 0x0000001e (FLAGS)                      BIND_NOW
 0x6ffffffb (FLAGS_1)                    标志： NOW
 0x6ffffff0 (VERSYM)                     0x1208
 0x6ffffffc (VERDEF)                     0x12c4
 0x6ffffffd (VERDEFNUM)                  1
 0x6ffffffe (VERNEED)                    0x12e0
 0x6fffffff (VERNEEDNUM)                 2
 0x00000000 (NULL)                       0x0

复制代码
```

`NEEDED`项标明的为需要动态装载的库。

我们看下`linker`中`段`和`节区`的对应关系： ![image](images/5f077e711ef94ea1b2437b64975e2c5f~tplv-k3u1fbpfcp-zoom-1.image)

## 函数重定位的一些姿势

Linux 为了解决外部引用的问题，特地设定了一个全局变量偏移表`.got`，表中每一项储存的都是外部引用函数的地址。这样在程序代码中只需要间接引用全局表表项的地址就可以了。

当Linux需要对一个引用符号重定位时，首先要装载这个库，然后在库中查表寻找函数的相对地址，最后在库装载的基地址上加上函数的相对地址得到函数的虚拟地址。

# `Bionic`中的`Linker`模块

## 可执行文件的创建

我们先简单了解下可执行文件的创建流程：

- 首先，C代码（`.c`） 经过`编译器`预处理，编译成汇编代码（`.asm`）
- 然后，经汇编器处理，生成目标代码（`.o`）
- 然后，通过链接器，链接相应的库生成可执行文件（`.out`）
- 最后`OS`将可执行文件加载到内存里执行。

如图: ![img](images/1bdebac00ffd42edad6b42ec0196dc1c~tplv-k3u1fbpfcp-zoom-1.image)

## 可执行文件的装载

Linux 系统上有两种并不完全相同的可执行程序：

- 一种是静态链接的可执行程序：包含了运行所需要的所有函数，可以不依赖任何外部库来运行。
- 一种是动态链接的可执行程序：不会包含所依赖的库文件，文件大小相对会小很多。

`静态可执行程序`用在一写特殊的场合，例如系统初始化时，这是整个系统还未准备好，`动态链接程序`还无法使用。系统的启动程序Init就是一个`静态可执行程序`。

在Android中，生成一个静态可执行程序的方法是在编译脚本中增加如下配置：

```mk
LOCAL_FORCE_STATIC_EXECUTABLE := true
复制代码
```

Linux 执行一个可执行文件的过程是：

> 父进程执行`fork`后，在`fork`出的子进程中执行`execve`函数，这个函数会将可执行文件装载进内存，准备好运行环境后就跳到可执行文件入口开始执行。通常可执行程序的入口是`_start_main()`函数。

### 静态链接

在静态链接时，Android会给程序自动加上2个`.o`重定位文件：~~`crtbegin_static.o`和`crtend_android.o`~~（**这部分从9.0的编译输出`out`中并没有找到对应的文件，但是找到了`crtbegin_so.o`和`crtend_so.o`，不清楚这部分是否有所变更**）。与之对应的源文件位置在`bionic/libc/arch-common/bionic`目录下：`crtbegin.c`和`crtend.S`。`_start_main()`函数就位于`crtbegin.c`中：

```c
__used static void _start_main(void* raw_args) {
  structors_array_t array;
  array.preinit_array = &__PREINIT_ARRAY__;
  array.init_array = &__INIT_ARRAY__;
  array.fini_array = &__FINI_ARRAY__;

  __libc_init(raw_args, NULL, &main, &array);
}
复制代码
```

最后调用了` __libc_init`函数，其中第一个参数是Linux内核加载器传递过来的`raw`类型数据，第三个参数是`main`的函数指针，第四个参数是几个段数组的地址。` __libc_init`执行完`libc`库的初始化后，就会调用`main`函数。

> 发现一本比较好的书，叫`程序员的自我修养：链接、装载与库`，对这部分的内容讲解比较有意思，值得一看。**咳咳，想要电子版评论我吧**

### 动态链接

在动态链接时，`execve`系统调用会分析`可执行文件`的`文件头`来寻找链接器。Linux 文件中是`ld.so`，而 Android 则是`linker`。

`execve`会将`linker`装载进可执行文件的空间，然后执行`linker`的`_start`函数。`linker`完成动态库的装载和符号的重定位后再去运行真正的可执行文件的代码。

`linker`使用的`_start`函数位于`bionic/linker/arch/arm/begin.S`（每个架构实现目录下都有）。

```
#include <private/bionic_asm.h>
ENTRY(_start)
  // Force unwinds to end in this function.
  .cfi_undefined r14

  mov r0, sp
  bl __linker_init      //执行__linker_init函数

  /* linker init returns the _entry address in the main image */
  bx r0                 //__linker_init函数返回可执行程序的入口地址
END(_start)
复制代码
```

这部分内容与书中稍有区别，不过流程都是一样的：`_start`函数跳转到`__linker_init`函数去执行，`__linker_init`执行完程序的初始化后会返回可执行文件的入口到r0寄存器，然后通过`bx r0`跳转到应用入口函数。

## 可执行程序的初始化

> 可执行程序的初始化是通过`__linker_init`函数完成的，9.0的具体实现在`bionic/linker/linker_main.cpp`。

`__linker_init`中有一个`soinfo`的结构。在Android中`soinfo`是一个非常重要的数据结构，这部分定义在`bionic/linker/linker_soinfo.h`。不管是可执行文件还是动态库，Android都会为其构造一个`soinfo`的结构，`soinfo`中保存了程序所有的`节区`信息。

我们看下`__linker_init`源码片段，严重删减并添加注释的那种：

```
extern "C" ElfW(Addr) __linker_init(void* raw_args) {

  soinfo linker_so(nullptr, nullptr, nullptr, 0, 0);

  //1 对linker_so的部分属性进行初始化，有木有发现soinfo就是在可执行文件的结构
  linker_so.base = linker_addr;
  linker_so.size = phdr_table_get_load_size(phdr, elf_hdr->e_phnum);
  linker_so.load_bias = get_elf_exec_load_bias(elf_hdr);
  linker_so.dynamic = nullptr;
  linker_so.phdr = phdr;
  linker_so.phnum = elf_hdr->e_phnum;
  linker_so.set_linker_flag();

  //2 预链接，此方法执行完，soinfo的信息基本上就被填充完了。如果失败则退出
  if (!linker_so.prelink_image()) __linker_cannot_link(args.argv[0]);

  //3 装载linker所有的依赖库并进行重定位，如果失则败退出
  if (!linker_so.link_image(g_empty_list, g_empty_list, nullptr)) __linker_cannot_link(args.argv[0]);

  //4 初始化主线程 (包括 TLS 表).
  __libc_init_main_thread(args);

  //5 初始化linker的静态libc库的全局变量
  __libc_init_globals(args);

  //6 初始化linker自身的全局变量
  linker_so.call_constructors();
  
  //7 获取libdl对应的soinfo并添加到列表中
  solist = get_libdl_info(kLinkerPath, linker_so, linker_link_map);
  g_default_namespace.add_soinfo(solist);

  //8 可执行程序重定位未出现异常，此时再去初始化（安全地引用外部数据和其他非本地数据），并得到跳转地址
  ElfW(Addr) start_address = __linker_init_post_relocation(args);

  return start_address;
}
复制代码
```

9.0在实现上和书中的差距有些大了，不过整体流程还是没变，只是细节上更丰富了下。

> `Bionic`学习到这里内心已经有些抗拒了。。。。。不过再坚持一下下啦

## `linker`如何替换掉`libdl.so`

> Linux 装载一个动态库时需要使用`dlopen`函数。`dlopen`原本位于`libdl.so`中，前面说过，`libdl.so`中的函数,如`dlopen`,`dlclose`,`dlsys`等Google并没有直接实现，真正的实现在`linker`中，那么`linker`是如何实现替换的呢？

在可执行文件的装载过程中，所有装载进来的动态库对应的`soinfo`结构都会放到一个链表中，当新装载一个动态库时，会首先检查它是否已经存在于链表中，如果不存在才会继续装载。

而`linker`伪造了一个`libdl.so`的soinfo结构，并放在了链表第一个元素的位置，因此程序中链接的`libdl.so`并不会真正的装载。

请留意上面`__linker_init`源码片段中的第7步就是在做这个事情了。不过9.0和书中的源码已经变化很大了，增加了`namespace`的逻辑。具体可以参考下[Android Linker简介](https://www.cnblogs.com/ntiger/p/12122308.html)

# 调试器-`ptrace`和`Hook API`

> `ptrace`系统调用通常用在调试器软件中，调试器利用`ptrace`函数达到控制目标进程运行的目的。一些Android的安全管家就是通过`ptrace`函数把自带的动态库插入到系统或者别的进程中，从而达到监控系统运行的目的。

## `ptrace`系统调用简介

看下`Bionic`中`ptrace`的定义：

```h
long ptrace(int __request, ...);
//具体实现中调用的是__ptrace
long __ptrace(int req, pid_t pid, void* addr, void* data);
复制代码
```

`__ptrace`的参数：

- `req`：请求执行的操作类型
- `pid`：目标进程ID
- `addr`：目标进程的地址
- `data`：操作相关的数据根据请求的操作不同而变化。如果写入操作，`data`存放的是需要写入的数据；如果是读取操作，`data`将存放返回的数据

我们看下`req`的操作类型：

- `PTRACE_TRACEME`：指示父进程跟踪某个子进程的执行。任何传给子进程的信号将导致其停止执行，同时父进程调用wait()时会得到通告。之后，子进程调用exec()时，核心会给它传送SIGTRAP信号，在新程序开始执行前，给予父进程控制的机会。pid, addr, 和 data参数被忽略。

  > 这是唯一由子进程使用的请求，剩下部分将由父进程使用的请求。

- `PTRACE_PEEKTEXT`：从目标进程的代码段中读取一个长整型，内存地址由参数`addr`决定

- `PTRACE_PEEKDATA`：从目标进程的数据段中读取一个长整型，内存地址由参数`addr`决定。

- `PTRACE_PEEKUSR` : 从子进程的用户区`addr`指向的位置读取一个`long int`，并作为调用的结果返回。

- `PTRACE_POKETEXT`,`PTRACE_POKEDATA` : 将`data`指向的`long int`拷贝到子进程内存空间由`addr`指向的位置。

- `PTRACE_POKEUSR` : 将`data`指向的`long int`拷贝到子进程用户区由`addr`指向的位置。

- `PTRACE_GETREGS`, `PTRACE_GETFPREGS`： 将子进程通用和浮点寄存器的值拷贝到父进程内由`data`指向的位置。`addr`参数被忽略。

- `PTRACE_SETREGS`, `PTRACE_SETFPREGS`： 从父进程内将`data`指向的数据拷贝到子进程的通用和浮点寄存器。`addr`参数被忽略。

- `PTRACE_SETSIGINFO`：将父进程内由`data`指向的数据作为`siginfo_t`结构体拷贝到子进程。`addr`参数被忽略。

- `PTRACE_SETOPTIONS`： 将父进程内由`data`指向的值设定为`ptrace`选项，`data`作为位掩码来解释，由下面的标志指定

  - `PTRACE_O_TRACESYSGOOD` : 当转发`syscall`陷阱(traps)时，在信号编码中设置位7，即第一个字节的最高位。例如：`SIGTRAP | 0x80`。这有利于追踪者识别一般的陷阱和那些由`syscall`引起的陷阱。
  - `PTRACE_O_TRACEFORK` : 通过 (`SIGTRAP | PTRACE_EVENT_FORK << 8`) 使子进程下次调用`fork()`时停止其执行，并自动跟踪开始执行时就已设置SIGSTOP信号的新进程。新进程的PID可以通过`PTRACE_GETEVENTMSG`获取。
  - `PTRACE_O_TRACEVFORK` : 通过 (`SIGTRAP | PTRACE_EVENT_VFORK << 8`) 使子进程下次调用vfork()时停止其执行，并自动跟踪开始执行时就已设置SIGSTOP信号的新进程。新进程的PID可以通过PTRACE_GETEVENTMSG获取。
  - `PTRACE_O_TRACECLONE` : 通过 (`SIGTRAP | PTRACE_EVENT_CLONE << 8`) 使子进程下次调用clone()时停止其执行，并自动跟踪开始执行时就已设置`SIGSTOP`信号的新进程。新进程的PID可以通过`PTRACE_GETEVENTMSG`获取。
  - `PTRACE_O_TRACEEXEC` : 通过 (`IGTRAP | PTRACE_EVENT_EXEC << 8`) 使子进程下次调用`exec()`时停止其执行。
  - `PTRACE_O_TRACEVFORKDONE` : 通过 (`SIGTRAP | PTRACE_EVENT_VFORK_DONE << 8`) 使子进程下次调用`exec()`并完成时停止其执行。
  - `PTRACE_O_TRACEEXIT` : 通过 (`SIGTRAP | PTRACE_EVENT_EXIT << 8`) 使子进程退出时停止其执行。子进程的退出状态可通过`PTRACE_GETEVENTMSG`。

- `PTRACE_GETEVENTMSG` : 获取刚发生的ptrace事件消息，并存放在父进程内由`data`指向的位置。`addr`参数被忽略。

- `PTRACE_CONT` ：重启动已停止的进程。如果data指向的数据并非0，同时也不是SIGSTOP信号，将会作为传递给子进程的信号来解释。那样，父进程可以控制是否将一个信号发送给子进程。 addr参数被忽略。

- `PTRACE_SYSCALL`, `PTRACE_SINGLESTEP` : 如同PTRACE_CONT一样重启子进程的执行，但指定子进程在下个入口或从系统调用退出时，或者执行单个指令后停止执行，这可用于实现单步调试。`addr`参数被忽略。

- `PTRACE_SYSEMU`, `PTRACE_SYSEMU_SINGLESTEP` : 用于用户模式的程序仿真子进程的所有系统调用。

- `PTRACE_KILL` : 给子进程发送`SIGKILL`信号，从而终止其执行。`data`，`addr`参数被忽略。

- `PTRACE_ATTACH` : 衔接到`pid`指定的进程，从而使其成为当前进程的追踪目标。

- `PTRACE_DETACH` : `PTRACE_ATTACH`的反向操作。

## `Hook API` 的一些内容

> `Hook API`技术由来已久，在操作系统未能提供所需功能的情况下，利用`Hook API`手段来实现某种有用的功能也算是一种不得已的方法。

书中讲到最早的`Hook API`是为了实现Windows上电子词典的光标取词功能，把系统的字符串输出函数换成电子词典中的函数，从而能得到屏幕上任何位置的字符串。厉害了~~~~

Linux由于安全性高，通常是采用`ptrace`函数来实现`Hook API` 的目的。不过调用`ptrace`函数需要root权限。

`Hook API`的原理是利用`ptrace`函数把一小段代码注入目标程序中，这一小段代码的任务是：装载自己开发的动态库到目标进程中，然后查找目标进程中特定函数在全局偏移表中的位置，替换成自己动态库的函数地址。

> 随着 Android 安全性的提高，这部分的实现越来越有难度了。不过搞破坏是人的天性，值得好好研究。亲切的附上知乎大神文章：[Android Native Hook知多少](https://zhuanlan.zhihu.com/p/132699875)供大家品尝

# 开源协议

`开源协议`规定了你在使用开源软件时的权利和责任，也就是规定了你可以做什么，不可以做什么。

`开源协议`虽然不一定具备法律效力，但是当涉及软件版权纠纷时，开源协议也是非常重要的证据之一。

我们简单介绍下比较常见的几种

## GNU GPL（General Public License）

> 遵循 GPL 协议的开源软件数量极其庞大，包括 Linux 系统在内的大多数的开源软件都是基于这个协议的。

特点是：只要软件中包含了遵循 GPL 协议的产品或代码，该软件就必须也遵循 GPL 许可协议，也就是必须开源免费，不能闭源收费，因此这个协议并不适合商用软件。

## BSD（Berkeley Software Distribution）

> GPL的出发点是代码的开源/免费使用和引用/修改/衍生代码的开源/免费使用，不允许修改后和衍生的代码做为闭源的商业软件发布和销售。

这也就是为什么我们能用免费的各种linux，包括商业公司的linux和linux上各种各样的由个人，组织，以及商业软件公司开发的免费软件了。

GPL协议的主要内容是只要在一个软件中使用（”使用”指类库引用，修改后的代码或者衍生代码）GPL协议的产品，则该软件产品必须也采用GPL协议，既必须也是开源和免费。这就是所谓的`传染性`。

GPL协议的产品作为一个单独的产品使用没有任何问题，还可以享受免费的优势。

由于GPL严格要求使用了GPL类库的软件产品必须使用GPL协议，对于使用GPL协议的开源代码，商业软件或者对代码有保密要求的部门就不适合集成/采用作为类库和二次开发的基础。

## Apache License Version

> Apache Licence是著名的非盈利开源组织Apache采用的协议。该协议和BSD类似，同样鼓励代码共享和尊重原作者的著作权，同样允许代码修改，再发布（作为开源或商业软件）。

需要满足的条件也和BSD类似：

- 需要给代码的用户一份Apache Licence
- 如果你修改了代码，需要在被修改的文件中说明。
- 在延伸的代码中（修改和有源代码衍生的代码中）需要带有原来代码中的协议，商标，专利声明和其他原来作者规定需要包含的说明。
- 如果再发布的产品中包含一个Notice文件，则在Notice文件中需要带有Apache Licence。你可以在Notice中增加自己的许可，但不可以表现为对Apache Licence构成更改。

Apache Licence也是对商业应用友好的许可。使用者也可以在需要的时候修改代码来满足需要并作为开源或商业产品发布/销售。

## 开源协议概述图

> 来自百度百科

![image](images/b420c4e0a1bf4e3f9f0ebdf10572659d~tplv-k3u1fbpfcp-zoom-1.image)

# 结语

`Bionic`章节到这里算是结束了，学起来真滴困啊。最近赶上出差再加上这部分章节的知识真滴陌生，在完成时间上有所推迟，不过收益匪浅，哈哈哈！

下一篇到了梦寐以求的章节了《进程间通信-Android的`Binder`》


作者：apigfly
链接：https://juejin.cn/post/6857508056620941320
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。