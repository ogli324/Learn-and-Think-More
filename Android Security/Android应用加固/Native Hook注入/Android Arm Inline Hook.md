# Android Arm Inline Hook

url：http://ele7enxxh.com/Android-Arm-Inline-Hook.html



本文将结合[本项目](https://github.com/ele7enxxh/Android-Inline-Hook)的源代码，详细阐述Android Arm Inline Hook的原理与实现过程。

## 什么是Inline Hook

Inline Hook即内部跳转Hook，通过替换函数开始处的指令为跳转指令，使得原函数跳转到自己的函数，通常还会保留原函数的调用接口。与GOT表Hook相比，Inline Hook具有更广泛的适用性，几乎可以Hook任何函数，不过其实现更为复杂，考虑的情况更多，并且无法对一些太短的函数Hook。
其基本原理请参阅网上其他资料。

## 需要解决的问题

> 1. Arm模式与Thumb模式的区别
> 2. 跳转指令的构造
> 3. PC相关指令的修正
> 4. 线程处理
> 5. 其他一些细节

下面我将结合源码对这几个问题进行解决。

## Arm模式与Thumb模式的区别

本文讨论的对象为基于32位的Arm架构的Inline Hook，在Arm版本7及以上的体系中，其指令集分为Arm指令集和Thumb指令集。Arm指令为4字节对齐，每条指令长度均为32位；Thumb指令为2字节对齐，又分为Thumb16、Thumb32，其中Thumb16指令长度为16位，Thumb32指令长度为32位。
在对一个函数进行Inline Hook时，首先需要判断当前函数指令是Arm指令还是Thumb指令，指令使用目标地址值的bit[0]来确定目标地址的指令类型。bit[0]的值为1时，目标程序为Thumb指令；bit[0]值为0时，目标程序为ARM指令。其相关实现代码为以下宏：

```
// 设置bit[0]的值为1
#define SET_BIT0(addr)		(addr | 1)
// 设置bit[0]的值为0
#define CLEAR_BIT0(addr)	(addr & 0xFFFFFFFE)
// 测试bit[0]的值，若为1则返回真，若为0则返回假
#define TEST_BIT0(addr)		(addr & 1)
```



## 跳转指令的构造

跳转指令主要分为以下两种：

> - B系列指令：B、BL、BX、BLX
> - 直接写PC寄存器

Arm的B系列指令跳转范围只有4M，Thumb的B系列指令跳转范围只有256字节，然而大多数情况下跳转范围都会大于4M，故我们采用`LDR PC, [PC, ?]`构造跳转指令。另外Thumb16指令中并没有合适的跳转指令，如果单独使用Thumb16指令构造跳转指令，需要使用更多的指令完成，并且在后续对PC相关指令的修正也更加繁琐，故综合考虑下，决定放弃对ARMv5的支持。
另外，Arm处理器采用3级流水线来增加处理器指令流的速度，也就是说程序计数器R15(PC)总是指向“正在取指”的指令，而不是指向“正在执行”的，即PC总是指向当前正在执行的指令地址再加2条指令的地址。比如当前指令地址是0×8000， 那么当前pc的值，在thumb下面是0×8000 + 2 *2， 在arm下面是0×8000 + 4* 2。
对于Arm指令集，跳转指令为：

```
LDR PC, [PC, #-4]
addr
```



`LDR PC, [PC, #-4]`对应的机器码为：0xE51FF004，`addr`为要跳转的地址。该跳转指令范围为32位，对于32位系统来说即为全地址跳转。
对于Thumb32指令集，跳转指令为：

```
LDR.W PC, [PC, #0]
addr
```



`LDR.W PC, [PC, #0]`对应的机器码为：0x00F0DFF8，`addr`为要跳转的地址。同样支持任意地址跳转。
其相关实现代码为：

```
// Arm Mode
if (TEST_BIT0(item->target_addr)) {
	int i;

	i = 0;
	if (CLEAR_BIT0(item->target_addr) % 4 != 0) {
		((uint16_t *) CLEAR_BIT0(item->target_addr))[i++] = 0xBF00;  // NOP
	}
	((uint16_t *) CLEAR_BIT0(item->target_addr))[i++] = 0xF8DF;
	((uint16_t *) CLEAR_BIT0(item->target_addr))[i++] = 0xF000;	// LDR.W PC, [PC]
	((uint16_t *) CLEAR_BIT0(item->target_addr))[i++] = item->new_addr & 0xFFFF;
	((uint16_t *) CLEAR_BIT0(item->target_addr))[i++] = item->new_addr >> 16;
}
// Thumb Mode
else {
	((uint32_t *) (item->target_addr))[0] = 0xe51ff004;	// LDR PC, [PC, #-4]
	((uint32_t *) (item->target_addr))[1] = item->new_addr;
}
```



首先通过TEST_BIT0宏判断目标函数的指令集类型，其中若为Thumb指令集，多了下面一个额外处理：

```
if (CLEAR_BIT0(item->target_addr) % 4 != 0) {
	((uint16_t *) CLEAR_BIT0(item->target_addr))[i++] = 0xBF00;  // NOP
}
```



对bit[0]的值清零，若其值4字节不对齐，则添加一个2字节的`NOP`指令，使得后续的指令4字节对齐。这是因为在Thumb32指令中，若该指令对PC寄存器的值进行了修改，则该指令必须是4字节对齐的，否则为非法指令。

## PC相关指令的修正

不论是Arm指令集还是Thumb指令集，都存在很多的与PC值相关的指令，例如：B系列指令、literal系列指令等。原有函数的前几个被跳转指令替换的指令将会被搬移到trampoline_instructions中，此时PC值已经变动，所以需要对PC相关指令进行修正（所谓修正即为计算出实际地址，并使用其他指令完成同样的功能）。相关修正代码位于relocate.c文件中。其中`INSTRUCTION_TYPE`描述了需要修正的指令，限于篇幅，这里仅阐述Arm指令的修正过程，对应的代码为`relocateInstructionInArm`函数。
函数原型如下：

```
/*
target_addr: 待Hook的目标函数地址，即为当前PC值，用于修正指令
orig_instructions：存放原有指令的首地址，用于修正指令和后续对原有指令的恢复
length：存放的原有指令的长度，Arm指令为8字节；Thumb指令为12字节
trampoline_instructions：存放修正后指令的首地址，用于调用原函数
orig_boundaries：存放原有指令的指令边界（所谓边界即为该条指令与起始地址的偏移量），用于后续线程处理中，对PC的迁移
trampoline_boundaries：存放修正后指令的指令边界，用途与上相同
count：处理的指令项数，用途与上相同
*/
static void relocateInstructionInArm(uint32_t target_addr, uint32_t *orig_instructions, int length, uint32_t *trampoline_instructions, int *orig_boundaries, int *trampoline_boundaries, int *count);
```



具体实现中，首先通过函数`getTypeInArm`判断当前指令的类型，本函数通过类型，共分为4个处理分支：

> 1. BLX_ARM、BL_ARM、B_ARM、BX_ARM
> 2. ADD_ARM
> 3. ADR1_ARM、ADR2_ARM、LDR_ARM、MOV_ARM
> 4. 其他指令

### BLX_ARM、BL_ARM、B_ARM、BX_ARM指令的修正

即为B系列指令（`BLX <label>`、`BL <label>`、`B <label>`、`BX PC`）的修正，其中`BLX_ARM`和`BL_ARM`需要修正LR寄存器的值，相关代码为：

```
if (type == BLX_ARM || type == BL_ARM) {
	trampoline_instructions[trampoline_pos++] = 0xE28FE004;	// ADD LR, PC, #4
}
```



接下来构造相应的跳转指令，即为：

```
trampoline_instructions[trampoline_pos++] = 0xE51FF004;  	// LDR PC, [PC, #-4]
```



最后解析指令，计算实际跳转地址`value`，并将其写入`trampoline_instructions`，相关代码为：

```
if (type == BLX_ARM) {
	x = ((instruction & 0xFFFFFF) << 2) | ((instruction & 0x1000000) >> 23);
}
else if (type == BL_ARM || type == B_ARM) {
	x = (instruction & 0xFFFFFF) << 2;
}
else {
	x = 0;
}

top_bit = x >> 25;
imm32 = top_bit ? (x | (0xFFFFFFFF << 26)) : x;
if (type == BLX_ARM) {
	value = pc + imm32 + 1;
}
else {
	value = pc + imm32;
}
trampoline_instructions[trampoline_pos++] = value;
```



如此便完成了B系列指令的修正，关于指令的字节结构请参考Arm指令手册。

### ADD_ARM指令的修正

`ADD_ARM`指的是`ADR Rd, <label>`格式的指令，其中`<label>`与PC相关。
首先通过循环遍历，得到Rd寄存器，代码如下：

```
int rd;
int rm;
int r;

// 解析指令得到rd、rm寄存器
rd = (instruction & 0xF000) >> 12;
rm = instruction & 0xF;

// 为避免冲突，排除rd、rm寄存器，选择一个临时寄存器Rr
for (r = 12; ; --r) {
	if (r != rd && r != rm) {
		break;
	}
}
```



接下来是构造修正指令：

```
// PUSH {Rr}，保护Rr寄存器值
trampoline_instructions[trampoline_pos++] = 0xE52D0004 | (r << 12);
// LDR Rr, [PC, #8]，将PC值存入Rr寄存器中
trampoline_instructions[trampoline_pos++] = 0xE59F0008 | (r << 12);
// 变换原指令`ADR Rd, <label>`为`ADR Rd, Rr, ?`
trampoline_instructions[trampoline_pos++] = (instruction & 0xFFF0FFFF) | (r << 16);
//POP {Rr}，恢复Rr寄存器值
trampoline_instructions[trampoline_pos++] = 0xE49D0004 | (r << 12);
// ADD PC, PC，跳过下一条指令
trampoline_instructions[trampoline_pos++] = 0xE28FF000;
trampoline_instructions[trampoline_pos++] = pc;
```



### ADR1_ARM、ADR2_ARM、LDR_ARM、MOV_ARM

分别为`ADR Rd, <label>`、`ADR Rd, <label>`、`LDR Rt, <label>`、`MOV Rd, PC`。
同样首先解析指令，得到`value`，相关代码如下：

```
int r;
uint32_t value;

r = (instruction & 0xF000) >> 12;

if (type == ADR1_ARM || type == ADR2_ARM || type == LDR_ARM) {
	uint32_t imm32;
	
	imm32 = instruction & 0xFFF;
	if (type == ADR1_ARM) {
		value = pc + imm32;
	}
	else if (type == ADR2_ARM) {
		value = pc - imm32;
	}
	else if (type == LDR_ARM) {
		int is_add;
		
		is_add = (instruction & 0x800000) >> 23;
		if (is_add) {
			value = ((uint32_t *) (pc + imm32))[0];
		}
		else {
			value = ((uint32_t *) (pc - imm32))[0];
		}
	}
}
else {
	value = pc;
}
```



最后构造修正指令，代码如下：

```
// LDR Rr, [PC]
trampoline_instructions[trampoline_pos++] = 0xE51F0000 | (r << 12);
// 跳过下一条指令
trampoline_instructions[trampoline_pos++] = 0xE28FF000;	// ADD PC, PC
trampoline_instructions[trampoline_pos++] = value;
```



### 其他指令

事实上，还有些指令格式需要修正，例如：`PUSH {PC}`、`PUSH {SP}`等，虽然这些指令被Arm指令手册标记为**deprecated**，但是仍然为合法指令，不过在实际汇编中并未发现此类指令，故未做处理，相关代码如下：

```
// 直接将指令存放到trampoline_instructions中
trampoline_instructions[trampoline_pos++] = instruction;
```



处理完所有待处理指令后，最后加入返回指令：

```
// LDR PC, [PC, #-4]
trampoline_instructions[trampoline_pos++] = 0xe51ff004;
trampoline_instructions[trampoline_pos++] = lr;
```



Thumb指令的修正，大家可以参考这里的思路，自行阅读源码。

## 线程处理

一个完善的Inline Hook方案必须要考虑多线程环境，即要考虑线程恰好执行到被修改指令的位置。在Window下，使用`GetThreadContext`和`SetThreadContext`枚举所有线程，迁移context到搬迁后的指令中。然而在Linux+Arm环境下，并没有直接提供相同功能的API，不过可以使用`ptrace`完成，主要流程如下：

> 1. 解析/proc/self/task目录，获取所有线程id
> 2. 创建子进程，父进程等待。子进程枚举所有线程，PTRACE_ATTACH线程，迁移线程PC寄存器，枚举完毕后，子进程给自己发SIGSTOP信号，等待父进程唤醒
> 3. 父进程检测到子进程已经SIGSTOP，完成Inline Hook工作，向子进程发送SIGCONT信号，同时等待子进程退出
> 4. 子进程枚举所有线程，PTRACE_DETACH线程，枚举完毕后，子进程退出
> 5. 父进程继续其他工作

这里使用子进程完成线程处理工作，实际上是迫不得已的。因为，如果直接使用本进程`PTRACE_ATTACH`线程，会出现**operation not permitted**，即使赋予root权限也是同样的错误，具体原因不得而知。
具体代码请参考`freeze`与`unFreeze`两个函数。

## 其他一些细节

1. 页保护
   页面大小为4096字节，使用`mprotect`函数修改页面属性，修改为`PROT_READ | PROT_WRITE | PROT_EXEC`。
2. 刷新缓存
   对于ARM处理器来说，缓存机制作用明显，内存中的指令已经改变，但是cache中的指令可能仍为原有指令，所以需要手动刷新cache中的内容。采用`cacheflush`即可实现。
3. 一个已知的BUG
   虽然本库已经把大部分工作放在了`registerInlineHook`函数中，但是在`inlineHook`、`inlineUnHook`函数中还是不可避免的使用了部分libc库的API函数，例如：`mprotect`、`memcpy`、`munmap`、`free`、`cacheflush`等。如果使用本库对上述API函数进行Hook，可能会失败甚至崩溃，这是因为此时原函数的指令已经被破坏，或者其逻辑已经改变。解决这个Bug有两个方案，第一是采用其他Hook技术；第二将本库中的这些API函数全部采用内部实现，即不依赖于libc库，可采用静态链接libc库，或者使用汇编直接调相应的系统调用号。