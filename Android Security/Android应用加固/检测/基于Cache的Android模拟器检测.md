# 基于Cache的Android模拟器检测

url：http://www.wireghost.cn/2018/03/10/%E5%9F%BA%E4%BA%8ECache%E7%9A%84Android%E6%A8%A1%E6%8B%9F%E5%99%A8%E6%A3%80%E6%B5%8B/



本文主要解决了ARM64位指令的兼容性问题，并通过进程间通信杜绝了崩溃现象，让这部分的检测代码更具有可操作性。。

## ARM和X86

目前，绝大部分手机都是基于ARM架构，其他CPU架构给忽略不计，模拟器全部运行在PC的X86架构上。因此，可以利用ARM与X86的区别来判断是否为真机。
[![ELF](images/1.png)](http://www.wireghost.cn/2018/03/10/基于Cache的Android模拟器检测/1.png)
从上图我们可以看出，在CPU和内存之间，可以存在几级Cache，这里是L1和L2。Cache的作用是加速，把特定地址的数据值缓存起来，这样就不用到低速的内存中去取了。其中，ARM采用的是将指令存储与数据存储分开的哈佛架构，L1 Cache（一级缓存）被分成了平行的两块，即I-Cache（指令缓存）和D-Cache（数据缓存），而X86采用的是将指令存储和数据存储合并在一起的冯•诺伊曼结构，L1 Cache是连续的一块缓存。
因此，如果我们通过读写地址指令的方式对一段可执行代码进行动态修改，那么在执行的时候，X86架构上的指令缓存会被同步修改，而对ARM架构而言，这种数据读写操作修改的只是D-Cahce中的内容，此时I-Cache中的指令并不会被更新。

## 设计思路

先看下思路的流程图：
[![ELF](images/2.png)](http://www.wireghost.cn/2018/03/10/基于Cache的Android模拟器检测/2.png)
左边的是真机上发生的情况，右边是模拟器发生的情况，下面详述一下操作过程。

> 1. 先执行一个地址上的指令，假设就是address这个地址。那么对于真机，指令将会写到I-Cache，而模拟器则会直接写到一整块Cache上;
> 2. 向address写入一个新指令。注意，这就有区别了，真机上的新指令会写入D-Cache，而在模拟器直接写到Cache;
> 3. 执行address的指令。此时在真机上，会从I-Cache读指令，也就是会执行第一步的指令。模拟器直接从Cache上读指令，会执行第二步的新指令。

实际操作中，我们发现存在真机上的指令Cache被洗掉的情况，但总的来说这种可能性还是比较低的，可以通过对市面上的大量机型做多次重复实验，并统计它的大盘分布..

## 具体实现

### 测试代码

#### ARM32

以下实现代码是测试代码的核心，主要就是将第8行地址的指令add r4, r4, #1，在运行中动态替换为第5行的指令add r6, r6, #1，这里的目标是ARM-V7架构的，要注意它采用的是三级流水，PC寄存器的值等于当前程序执行位置加8，代码如下：

```
#!cpp
__asm __volatile (
  STMFD  SP!,{R4-R7,LR}
  MOV  R6, #0                  //为r6赋初值
  MOV  R7, PC                  //PC指向第7行指令所在位置
  MOV  R4, #0      
  ADD  R6, R6, #1              //用来覆盖$address的“新指令”
  LDR  R5, [R7]      
  code:      
  ADD  R4, R4, #1              //这就是$address，是对r4加1
  MOV  R7, PC                  //10~13行的作用就是把第10行的指令写到第7行
  SUB  R7, R7, #0xC      
  STR  R5, [R7]
  CMP  R4, #2                  //控制循环次数
  BGE  out      
  CMP  R6, #2                  //控制循环次数
  BGE  out                     //不满足循环次数，则调回去
  B    code 
  out:
  MOV  R0, R4                  //把r4的值作为返回值
  LDMFD SP!,{R4-R7,PC}
);
```



#### ARM64

考虑到在64位的真机上可能会存在兼容性问题，需要针对arm64-v8a架构重新设计一段代码，原理同上。另外，由于arm64指令集中没有PC寄存器，这里选择用ADR指令做为替代方案。。

```
#!cpp
__asm __volatile (
  SUB SP, SP, #0x30       //开辟栈空间
  STR X9, [SP, #8]
  STR X10, [SP, #0x10]
  STR X11, [SP, #0x18]
  STR X12, [SP, #0x20]
  MOV X10, #0
  _start:
  ADR X11, _start        //ADR指令，自动取_start的地址（相对于PC的），并存放到x11寄存器中
  ADD X11, X11, #12      //x11加12，指向第13行指令
  MOV X12, #0            //为x12赋初值
  ADD X10, X10, #1       //用来覆盖$address的"新指令"
  LDR X9, [X11]
  code:
  ADD X12, X12, #1       //这就是$address，是对x12加1
  ADR X11, code          //adr伪指令，自动取code的地址（相对于PC的，即第16行指令)
  STR X9, [X11]
  CMP X12, #2            //控制循环次数
  BGE out                //跳出循环
  CMP X10, #2            //控制循环次数
  BGE out                //跳出循环
  B   code               //指定次数内的循环调回去
  out:
  MOV W0, W12            //取低32位值作为返回值
  LDR X9, [SP, #8]
  LDR X10, [SP, #0x10]
  LDR X11, [SP, #0x18]
  LDR X12, [SP, #0x20]
  ADD SP, SP, #0x30      //出栈，恢复栈空间
  RET
);
```



### 申请权限

这里会遇到个问题，就是我们是没有写代码段的权限的，所以需要将上面的汇编代码翻译成相应的机器码，再申请一块内存，将可执行代码段拷贝过去并执行。值得注意的是，如果用mmap映射会有bug，在真机上只能执行一次，第二次崩溃，可以通过如下方式解决：

```
#define PAGE_START(addr) (~(getpagesize() - 1) & (addr))
void *exec = malloc(0x1000);
char code[] = "\xF0\x40\x2D\xE9\x00\x60\xA0\xE3\x0F\x70\xA0\xE1\x00\x40\xA0\xE3"
              "\x01\x60\x86\xE2\x00\x50\x97\xE5\x01\x40\x84\xE2\x0F\x70\xA0\xE1"
              "\x0C\x70\x47\xE2\x00\x50\x87\xE5\x02\x00\x54\xE3\x02\x00\x00\xAA"
              "\x02\x00\x56\xE3\x00\x00\x00\xAA\xF6\xFF\xFF\xEA\x04\x00\xA0\xE1"
              "\xF0\x80\xBD\xE8";
void *page_start_addr = (void *)PAGE_START((uint32_t)exec);
memcpy(exec, code, sizeof(code)+1);
mprotect(page_start_addr, getpagesize(), PROT);
LOGI("magic_addr = %x", exec);
asmcheck = exec;
status = asmcheck();
```



### 验证方式

如果返回值等于2，说明执行的是旧指令，是arm架构；如果返回值等于1，说明执行的是新指令，是x86架构。最后，由于真机上存在I-Cache被洗掉的情况，也可能返回1，故需要通过多次循环执行对返回的结果进行统计，越靠近1~1000这个范围值的左侧，越可能是真机，反之应是模拟器。

## 备注

最后，为了防止在真机上出现崩溃，最好还是单独开一个进程服务，通过进程间通信实现模拟器鉴别的查询。。

## 参考链接

1. [基于cache的模拟器检测](https://bbs.pediy.com/thread-208471.htm)
2. [利用cache特性检测Android模拟器](http://www.vuln.cn/6766)