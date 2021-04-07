# Android平台下hook框架adbi的研究

url：

* https://blog.csdn.net/roland_sun/article/details/34109569
* https://blog.csdn.net/Roland_Sun/article/details/36049307

对于Android系统来说，底层本质上来说还是一个Linux系统，所以过往在Linux上常用的技巧，在Android平台上都可以实现。

比如，可以利用ptrace()函数来attach到一个进程上，从而可以修改对应该进程的内存内容和寄存器的值。

但是，有了这些功能，要想真正做到随意hook一个进程中的指定函数，还是有很多工作要做。而adbi（The Android Dynamic Binary Instrumentation Toolkit）就是一个通用的框架，使得hook变得异常简单。可以从这里获得其源代码：https://github.com/crmulliner/adbi。

它的基本实现原理是利用ptrace()函数attach到一个进程上，然后在其调用序列中插入一个调用dlopen()函数的步骤，将一个实现预备好的.so文件加载到要hook的进程中，最终由这个加载的.so文件在初始化函数中hook指定的函数。

整个adbi工具集由两个主要模块组成，分别是用于.so文件注入的进程劫持工具（hijack tool）和一个修改函数入口的基础库。接下来，我们还是通过阅读代码来分别了解这两个模块的实现原理，本篇我们先将重点放在劫持工具的实现上。

前面的介绍中已经提到过了，这个模块的作用是将一个调动dlopen()函数的步骤插入到指定进程的调用序列中。

要做到这点，需要解决几个问题，一是如何获得目标进程中dlopen()函数的调用地址，二是如何插入一个调用dlopen()函数的步骤。

好，我们先来看看第一个问题是如何解决的。在adbi中，具体是通过以下几行代码来找到目标进程中dlopen()函数的起始地址的：

```
void *ldl = dlopen("libdl.so", RTLD_LAZY);
if (ldl) {
    dlopenaddr = dlsym(ldl, "dlopen");
    dlclose(ldl);
}

unsigned long int lkaddr;
unsigned long int lkaddr2;
find_linker(getpid(), &lkaddr);
find_linker(pid, &lkaddr2);

dlopenaddr = lkaddr2 + (dlopenaddr - lkaddr);
```

首先用dlopen()在当前进程中加载libdl.so动态库，接着用dlsym()函数获得dlopen()函数的调用地址。感觉有点先有鸡还是先有蛋的问题了，这里稍微说明一下，libdl.so库肯定早就已经加载到进程中了，这里再加载一次其实并不会真的把这个动态库再在内存中的另一个位置加载一次，而是返回已经加载过的地址。
但是，获得了dlopen()函数在当前进程中的地址后并没有用，因为libdl.so是动态库，每次会被动态的加载到任意的一个地址上去，并不会固定。所以，当前进程中dlopen()函数的地址，一般情况下并不等于别的进程中dlopen()函数的地址。那要如何才能获得指定进程中的dlopen()函数的地址呢？代码中调用到了find_linker()函数，我们接下来看看它的实现：
```
static int find_linker(pid_t pid, unsigned long *addr)
{
    struct mm mm[1000];
    unsigned long libcaddr;
    int nmm;
    char libc[256];
    symtab_t s;

    if (0 > load_memmap(pid, mm, &nmm)) {
        printf("cannot read memory map\n");
        return -1;
    }
    if (0 > find_linker_mem(libc, sizeof(libc), &libcaddr, mm, nmm)) {
        printf("cannot find libc\n");
        return -1;
    }
    
    *addr = libcaddr;
    
    return 1;
}
```
在这个函数里，主要又调用了两个函数，一个是load_memmap()，另一个是find_linker_mem()。下面我们先来分析load_memmap()函数，这个函数主要由三个步骤组成：
```
static int
load_memmap(pid_t pid, struct mm *mm, int *nmmp)
{
    char raw[80000]; // this depends on the number of libraries an executable uses
    char name[MAX_NAME_LEN];
    char *p;
    unsigned long start, end;
    struct mm *m;
    int nmm = 0;
    int fd, rv;
    int i;

    sprintf(raw, "/proc/%d/maps", pid);
    fd = open(raw, O_RDONLY);
    if (0 > fd) {
        printf("Can't open %s for reading\n", raw);
        return -1;
    }
```
首先，会打开一个内存文件，获得其句柄。该文件路径是“/proc/<进程号>/maps”，作用就是读出指定进程的内存映射信息，其格式大概如下：
```
40096000-40098000 r-xp 00000000 b3:16 109        /system/bin/app_process
40098000-40099000 r--p 00001000 b3:16 109        /system/bin/app_process
40099000-4009a000 rw-p 00000000 00:00 0 
4009a000-400a9000 r-xp 00000000 b3:16 176        /system/bin/linker
400a9000-400aa000 r--p 0000e000 b3:16 176        /system/bin/linker
400aa000-400ab000 rw-p 0000f000 b3:16 176        /system/bin/linker
400ab000-400ae000 rw-p 00000000 00:00 0 
400ae000-400b0000 r--p 00000000 00:00 0 
400b0000-400b9000 r-xp 00000000 b3:16 855        /system/lib/libcutils.so
400b9000-400ba000 r--p 00008000 b3:16 855        /system/lib/libcutils.so
400ba000-400bb000 rw-p 00009000 b3:16 855        /system/lib/libcutils.so
400bb000-400be000 r-xp 00000000 b3:16 955        /system/lib/liblog.so
400be000-400bf000 r--p 00002000 b3:16 955        /system/lib/liblog.so
400bf000-400c0000 rw-p 00003000 b3:16 955        /system/lib/liblog.so
400c0000-40107000 r-xp 00000000 b3:16 832        /system/lib/libc.so
40107000-40108000 ---p 00000000 00:00 0 
```
好了，打开句柄之后，那接下来就是一行一行读取内存映射信息文件的内容：
```
    ......
    /* Zero to ensure data is null terminated */
    memset(raw, 0, sizeof(raw));

    p = raw;
    while (1) {
        rv = read(fd, p, sizeof(raw)-(p-raw));
        if (0 > rv) {
            return -1;
        }
        if (0 == rv)
            break;
        p += rv;
        if (p-raw >= sizeof(raw)) {
            printf("Too many memory mapping\n");
            return -1;
        }
    }
    close(fd);
    ......
```

读完之后，再下来就是一行一行的解释：

```
    ......
    p = strtok(raw, "\n");
    m = mm;
    while (p) {
        /* parse current map line */
        rv = sscanf(p, "%08lx-%08lx %*s %*s %*s %*s %s\n",
    	    &start, &end, name);
     
        p = strtok(NULL, "\n");
     
        if (rv == 2) {
            m = &mm[nmm++];
            m->start = start;
            m->end = end;
            strcpy(m->name, MEMORY_ONLY);
            continue;
        }
     
        if (strstr(name, "stack") != 0) {
            stack_start = start;
            stack_end = end;
        }
     
        /* search backward for other mapping with same name */
        for (i = nmm-1; i >= 0; i--) {
            m = &mm[i];
            if (!strcmp(m->name, name))
                break;
        }
     
        if (i >= 0) {
            if (start < m->start)
                m->start = start;
            if (end > m->end)
                m->end = end;
        } else {
            /* new entry */
            m = &mm[nmm++];
            m->start = start;
            m->end = end;
            strcpy(m->name, name);
    }
    }
     
    *nmmp = nmm;
    return 0;
}
```
先是按照格式解析出每行的起始地址，结束地址，和名称。如果没有解析出名称，就会用一个自定义的名称补上（“[memory]”）
#define MEMORY_ONLY  "[memory]"
如果名字是“stack”，表明这段内存用于栈。从前面的格式中可以看出，会有几行都叫一个名字的情况，接下来的代码会将这些连续的并且名字相同的内存段合并一下。

好了，现在就已经得到了指定进程的内存映射情况，包括起始地址、结束地址和名称。

接下来，我们继续看find_linker_mem()函数做了些什么：
```
static int
find_linker_mem(char *name, int len, unsigned long *start,
	  struct mm *mm, int nmm)
{
    int i;
    struct mm *m;
    char *p;
    for (i = 0, m = mm; i < nmm; i++, m++) {
        if (!strcmp(m->name, MEMORY_ONLY))
            continue;
        p = strrchr(m->name, '/');
        if (!p)
            continue;
        p++;
        if (strncmp("linker", p, 6))
            continue;
	break;
    }
    if (i >= nmm)
	/* not found */
        return -1;

    *start = m->start;
    strncpy(name, m->name, len);
    if (strlen(m->name) >= len)
        name[len-1] = '\0';
    return 0;
}
```
这个函数的作用是在前面读取的指定进程内存映射中，找出名字最后以“linker”结尾的那段内存的起始地址。其实，就是找到“/system/bin/linker”加载到内存中的地址。
顺便提一句，linker是Android提供的动态链接器，不同于普通的Linux。dlopen()函数就是在linker里面定义的（bionic/linker/dlfcn.cpp）：
```
static Elf32_Sym gLibDlSymtab[] = {
  // Total length of libdl_info.strtab, including trailing 0.
  // This is actually the STH_UNDEF entry. Technically, it's
  // supposed to have st_name == 0, but instead, it points to an index
  // in the strtab with a \0 to make iterating through the symtab easier.
  ELF32_SYM_INITIALIZER(sizeof(ANDROID_LIBDL_STRTAB) - 1, NULL, 0),
  ELF32_SYM_INITIALIZER( 0, &dlopen, 1),
  ELF32_SYM_INITIALIZER( 7, &dlclose, 1),
  ELF32_SYM_INITIALIZER(15, &dlsym, 1),
  ELF32_SYM_INITIALIZER(21, &dlerror, 1),
  ELF32_SYM_INITIALIZER(29, &dladdr, 1),
  ELF32_SYM_INITIALIZER(36, &android_update_LD_LIBRARY_PATH, 1),
#if defined(ANDROID_ARM_LINKER)
  ELF32_SYM_INITIALIZER(67, &dl_unwind_find_exidx, 1),
#elif defined(ANDROID_X86_LINKER) || defined(ANDROID_MIPS_LINKER)
  ELF32_SYM_INITIALIZER(67, &dl_iterate_phdr, 1),
#endif
};

// This is used by the dynamic linker. Every process gets these symbols for free.
soinfo libdl_info = {
    "libdl.so",

    phdr: 0, phnum: 0,
    entry: 0, base: 0, size: 0,
    unused1: 0, dynamic: 0, unused2: 0, unused3: 0,
    next: 0,
     
    flags: FLAG_LINKED,
     
    strtab: ANDROID_LIBDL_STRTAB,
    symtab: gLibDlSymtab,
    ........
}
```
关于linker目前只要知道这些就够了。所以，分析到这里可以看出，find_linker()函数的真正目的是获得“/system/bin/linker”程序加载到内存中的起始地址。
因为要注入的进程也在本机上运行，肯定用的是同一个linker，所以其内部的dlopen()函数和linker头的偏移量是固定的，这样计算其它进程内dlopen()函数的地址就非常简单了。先在本进程内，计算出dlopen()相对于linker头的偏移量，再加上要注入进程的linker头地址就好了。事实上，adbi也是这么做的，大家可以回过头去分析一下本节开头的那段代码。

好了，获得了目标进程中dlopen()函数地址后，接下来就要解决第二个问题了，即如何将调用dlopen()函数的步骤插入到目标进程的中去。要做到这步，当然肯定是要请ptrace()出马了：
```
// Attach 
if (0 > ptrace(PTRACE_ATTACH, pid, 0, 0)) {
    printf("cannot attach to %d, error!\n", pid);
    exit(1);
}
waitpid(pid, NULL, 0);
```
调用ptrace()函数，并且第一个参数说明是通过附着的方式捕获，第二个参数是要捕获的那个目标进程。ptrace函数调用成功后，被捕获的进程将成为当前进程的子进程，并且会暂停执行。下面的 waitpid()函数会等待被捕获的进程，当其返回时，表示目标进程已经暂停运行了，接下来就要正式下“刀子”了。正式下手之前，当然还是要知道被捕获进程的当前状态，这还是可以通过ptrace()函数来完成：
ptrace(PTRACE_GETREGS, pid, 0, &regs);
此时，传递的第一个参数说明此次调用是想获得目标进程的所有寄存器的值。获得这些寄存器值的目的是为了将它们保存下来，然后修改它们，使得在当前正常程序的调用序列中加入一个队dlopen()函数的调用，调用完成后再回到原来的程序处继续执行。具体的奥秘在一个叫做“sc”的int型数组中：
```
unsigned int sc[] = {
    0xe59f0040, //        ldr     r0, [pc, #64]   ; 48 <.text+0x48>
    0xe3a01000, //        mov     r1, #0  ; 0x0
    0xe1a0e00f, //        mov     lr, pc
    0xe59ff038, //        ldr     pc, [pc, #56]   ; 4c <.text+0x4c>
    0xe59fd02c, //        ldr     sp, [pc, #44]   ; 44 <.text+0x44>
    0xe59f0010, //        ldr     r0, [pc, #20]   ; 30 <.text+0x30>
    0xe59f1010, //        ldr     r1, [pc, #20]   ; 34 <.text+0x34>
    0xe59f2010, //        ldr     r2, [pc, #20]   ; 38 <.text+0x38>
    0xe59f3010, //        ldr     r3, [pc, #20]   ; 3c <.text+0x3c>
    0xe59fe010, //        ldr     lr, [pc, #20]   ; 40 <.text+0x40>
    0xe59ff010, //        ldr     pc, [pc, #20]   ; 44 <.text+0x44>
    0xe1a00000, //        nop                     r0
    0xe1a00000, //        nop                     r1 
    0xe1a00000, //        nop                     r2 
    0xe1a00000, //        nop                     r3 
    0xe1a00000, //        nop                     lr 
    0xe1a00000, //        nop                     pc
    0xe1a00000, //        nop                     sp
    0xe1a00000, //        nop                     addr of libname
    0xe1a00000, //        nop                     dlopenaddr
};
```
初始化时，数组的后面都被设置成空指令，在获得了目标进程的寄存器值后，会对它们重新赋值：
```
sc[11] = regs.ARM_r0;
sc[12] = regs.ARM_r1;
sc[13] = regs.ARM_r2;
sc[14] = regs.ARM_r3;
sc[15] = regs.ARM_lr;
sc[16] = regs.ARM_pc;
sc[17] = regs.ARM_sp;
sc[19] = dlopenaddr;
```
是不是觉得漏了一个？不急，下面会对其赋值：
```
// push library name to stack
libaddr = regs.ARM_sp - n*4 - sizeof(sc);
sc[18] = libaddr;
```
变量“n”的值是：
```
case 'l':
    n = strlen(optarg)+1;
    n = n/4 + (n%4 ? 1 : 0);
    ......
    break;
```
参数-l之后跟的是要dlopen()函数加载进来的.so动态库的路径名。由于ptrace()写入目标进程是以4字节为单位的，所以这里要除4，如果不是4的倍数，在后面还要补上一个4字节。看了上面的代码是不是觉得有点晕？没关系，下面我们画个图来重点分析一下这块。

![img](images/20140630134924984)

大家知道，对于栈来说，是从高内存向低内存扩展的，但是程序的执行以及数据的读取正好相反，是从低内存到高内存的。

还有一点需要说明一下，对于ARM处理器来说，pc寄存器的值，指向的不是当前正在执行指令的地址，而是往下第二条指令的地址。

好，我们正式开始分析代码的含义，指令将从codeaddr指示的位置从低到高依次执行。

第一条指令将pc寄存器的值加上64，读出那个地方的内容（4个字节），然后放到寄存器r0中。刚才说过了，pc寄存器值指向的是当前指令位置加8个字节，也就是说这条指令实际读出的是当前指令位置向下72个字节。由于sc数组是int型的，就是数组当前元素位置向下18个元素处。数一数，刚好是libaddr的位置。所以这条指令是为了让r0寄存器指向.so共享库路径名字符串。

第二条指令很简单，是将0赋值给寄存器r1。

第三条指令用来将pc寄存器值保存到lr寄存器中，这样做的目的是为了调用dlopen()函数返回后，跳转到指令“ldr sp, [pc, #56]”处。

第四条指令是将pc加上56处的数值加载到pc中，pc+56处是哪？当前指令位置往下64字节，16个元素，刚好是dlopen()函数的调用地址。所以，这条指令其实就是调用dlopen()函数，传入的参数一个是r0寄存器指向的共享库路径名，另一个是r1寄存器中的0。

调用dlopen()返回后将继续执行下面的所有指令，我就不一一分析了，作用就是恢复目标进程原来寄存器的值。先是sp，然后是r0、r1、r2、r3和lr，最后恢复原来pc的值，继续执行被暂停之前的指令，就像什么都没发生过一样。

好了，这部分就是hijack的全部核心思路，看似很神秘，其实很简单，是吧？当然，个人觉得要达到这个目的还是有很多种其它方法的，但ptrace()是必不可少的。

接下来，顺理成章，就是要把精心构造的数据写到目标进程的栈上：
```
// write library name to stack
if (0 > write_mem(pid, (unsigned long*)arg, n, libaddr)) {
    printf("cannot write library name (%s) to stack, error!\n", arg);
    exit(1);
}
	
// write code to stack
codeaddr = regs.ARM_sp - sizeof(sc);
if (0 > write_mem(pid, (unsigned long*)&sc, sizeof(sc)/sizeof(long), codeaddr)) {
    printf("cannot write code, error!\n");
    exit(1);
}
```
这个相对简单，直接从刚才读到的寄存器值中找出sp寄存器的值，这个值直接指示出当前栈顶的位置，接着往下写就好了。注意，栈是从高地址到低地址拓展的，所以计算地址时是减而不是加。write_mem()函数负责用ptrace()将一段内存值写到目标进程的指定位置：
```
/* Write NLONG 4 byte words from BUF into PID starting
   at address POS.  Calling process must be attached to PID. */
static int
write_mem(pid_t pid, unsigned long *buf, int nlong, unsigned long pos)
{
    unsigned long *p;
    int i;

    for (p = buf, i = 0; i < nlong; p++, i++)
        if (0 > ptrace(PTRACE_POKETEXT, pid, pos+(i*4), *p))
            return -1;
    return 0;
}
```
好了，有了可执行的代码在目标进程的栈上了，是不是就可以直接运行了呢？问题还没那么简单。我们来看看栈所在内存段的属性：
```
be846000-be867000 rw-p 00000000 00:00 0          [stack]
```
看到没有，栈并没有执行的属性，如果直接强行将pc指向栈所在内存段的话，有可能直接导致目标进程异常退出。
那接下来怎么办呢？能不能改一下栈内存段的属性，给其加上执行属性呢？mprotect()函数可以做到这点。

这里有同样一个问题，如何得到目标进程中mprotect()函数的调用地址呢？能不能用前面使用的在linker中找dlopen()函数调用地址一样的方法呢？笔者试过，其实是可以的，但是adbi采用了另一种方法，思路是通过直接分析ELF文件libc.so来获得其中符号mprotect的值，这个值其实就是表示mprotect()函数相对于文件头的偏移，再将这个值通libc.so文件在内存中映射的起始地址相加，就是mprotect()函数真正的调用地址了，具体代码如下：
```
static int
find_name(pid_t pid, char *name, unsigned long *addr)
{
    struct mm mm[1000];
    unsigned long libcaddr;
    int nmm;
    char libc[256];
    symtab_t s;

    if (0 > load_memmap(pid, mm, &nmm)) {
        printf("cannot read memory map\n");
        return -1;
    }
    if (0 > find_libc(libc, sizeof(libc), &libcaddr, mm, nmm)) {
        printf("cannot find libc\n");
        return -1;
    }
    s = load_symtab(libc);
    if (!s) {
        printf("cannot read symbol table\n");
        return -1;
    }
    if (0 > lookup_func_sym(s, name, addr)) {
        printf("cannot find %s\n", name);
        return -1;
    }
    *addr += libcaddr;
    return 0;
}
```
还是通过调用前面介绍的load_memmap()函数来获得目标进程的内存映射表，然后调用find_libc()函数从中找出名字末尾是“libc.so”的内存段，获得这个内存段的起始地址以及具体路径：
```
/* Find libc in MM, storing no more than LEN-1 chars of
   its name in NAME and set START to its starting
   address.  If libc cannot be found return -1 and
   leave NAME and START untouched.  Otherwise return 0
   and null-terminated NAME. */
static int
find_libc(char *name, int len, unsigned long *start,
	  struct mm *mm, int nmm)
{
    int i;
    struct mm *m;
    char *p;
    for (i = 0, m = mm; i < nmm; i++, m++) {
        if (!strcmp(m->name, MEMORY_ONLY))
            continue;
        p = strrchr(m->name, '/');
        if (!p)
            continue;
        p++;
        if (strncmp("libc", p, 4))
            continue;
        p += 4;

        /* here comes our crude test -> 'libc.so' or 'libc-[0-9]' */
        if (!strncmp(".so", p, 3) || (p[0] == '-' && isdigit(p[1])))
            break;
    }
    if (i >= nmm)
        /* not found */
        return -1;
     
    *start = m->start;
    strncpy(name, m->name, len);
    if (strlen(m->name) >= len)
        name[len-1] = '\0';
    return 0;
}
```
再下来调用load_symtab()函数，读取包含libc.so的ELF文件，分析并加载器符号表。最后，调用lookup_func_sym()函数，在符号表中查找指定名字的符号值。这部分代码牵涉到ELF文件的格式，就不重点分析了，如果有兴趣可以自己看。为什么不采用原来的方法找目标进程中mprotect()函数的调用地址呢？难道不可行吗？笔者写了如下一段函数：
```
static int
find_name2(pid_t pid, char *name, unsigned long *addr)
{
    struct mm mm[1000];
    unsigned long libcaddr1, libcaddr2;
    int nmm;
    char libc[256];
    symtab_t s;

    if (0 > load_memmap(pid, mm, &nmm)) {
        printf("cannot read memory map\n");
        return -1;
    }
    
    if (0 > find_libc(libc, sizeof(libc), &libcaddr1, mm, nmm)) {
        printf("cannot find libc\n");
        return -1;
    }
    
    if (0 > load_memmap(getpid(), mm, &nmm)) {
        printf("cannot read memory map\n");
        return -1;
    }
    
    if (0 > find_libc(libc, sizeof(libc), &libcaddr2, mm, nmm)) {
        printf("cannot find libc\n");
        return -1;
    }
    
    unsigned long mfunaddr;
    void *ldl = dlopen("libc.so", RTLD_LAZY);
    if (ldl) {
        mfunaddr = dlsym(ldl, name);
        dlclose(ldl);
    }
    
    *addr = mfunaddr - libcaddr2 + libcaddr1;
    return 0;
}
```
发现同样可以准确获得目标进程libc.so中指定函数的地址。这样也好，作者为我们展示了两种方法，给大家开开眼。
下面就简单了，修改目标进程的寄存器值，使其调用mprotect()函数，并且从mprotect()返回后直接调用栈上的代码就好了：
```
// calc stack pointer
regs.ARM_sp = regs.ARM_sp - n*4 - sizeof(sc);

// call mprotect() to make stack executable
regs.ARM_r0 = stack_start; // want to make stack executable
regs.ARM_r1 = stack_end - stack_start; // stack size
regs.ARM_r2 = PROT_READ|PROT_WRITE|PROT_EXEC; // protections

regs.ARM_lr = codeaddr; // points to loading and fixing code
regs.ARM_pc = mprotectaddr; // execute mprotect()
```
当然，栈寄存器要重新设置一下，指向库路径名的下面，免得在调用mprotect()函数的时候把前面精心写入的代码给冲掉。
r0寄存器设置为栈的起始地址，r1寄存器设置为栈的长度，r2保存要设置的属性（可读、可写且可执行）。

lr寄存器设置为栈上代码的起始地址，这样当调用mprotect()函数返回后就可以正常执行栈上代码了。

最后，将pc寄存器的值设置为前面查找到的目标进程中mprotect()函数的调用地址。

好了，万事具备，只欠东风，最后将寄存器值写到目标进程中，并且继续执行吧：
```
ptrace(PTRACE_SETREGS, pid, 0, &regs);
ptrace(PTRACE_DETACH, pid, 0, SIGCONT);
```
到目前为止，万里长征已经走完了第一步，即可以让目标集成调用dlopen()函数，自动加载一个任意指定的.so动态库。
具体如何hook，hook哪个函数，就要看.so怎么做了，下一篇我们会继续分析hook基础库的实现。



---

上篇中，我大致介绍了一下如何将一个dlopen()的调用插入到指定进程的执行序列中去。

但是，光插入这个没用，还没有具体解决如何hook进程中指定函数的问题。这个任务就要交给dlopen()函数加载进来的那个动态库来完成了。

但是具体要hook哪个进程内的，哪个动态库中的哪个函数，以及hook之后做什么，肯定是要使用者自己来指定的。adbi的作者写了一个简单的框架来帮助使用者，这个就是所谓的instruments，和前面的hijack同级目录。

有了前篇的铺垫，要分析这个instruments就简单的多了。同样的，要想hook一个函数，也要解决两个问题，一是找到这个函数在内存中的位置，二是如何修改函数偷梁换柱。

首先，我们来看看第一个问题是如何解决的。具体的代码位于base\util.c中，先从find_name()函数开始。
```
int find_name(pid_t pid, char *name, char *libn, unsigned long *addr)
{
	struct mm mm[1000];
	unsigned long libcaddr;
	int nmm;
	char libc[1024];
	symtab_t s;

	if (0 > load_memmap(pid, mm, &nmm)) {
		log("cannot read memory map\n")
		return -1;
	}
	if (0 > find_libname(libn, libc, sizeof(libc), &libcaddr, mm, nmm)) {
		log("cannot find lib: %s\n", libn)
		return -1;
	}
	s = load_symtab(libc);
	if (!s) {
		log("cannot read symbol table\n");
		return -1;
	}
	if (0 > lookup_func_sym(s, name, addr)) {
		log("cannot find function: %s\n", name);
		return -1;
	}
	*addr += libcaddr;
	return 0;
}
```
看着是不是有点眼熟？没错，这里也是从读取“/proc/<进程号>/maps”内存文件下手的，函数load_memmap()和在hijack里的实现一模一样，这里就不再详细解释了，不清楚的话可以参考 上篇。
好了，读取并解析完宿主进程的内存映射信息后，接下来调用了find_libname()函数，实现如下：
```
static int find_libname(char *libn, char *name, int len, unsigned long *start, struct mm *mm, int nmm)
{
	int i;
	struct mm *m;
	char *p;
	for (i = 0, m = mm; i < nmm; i++, m++) {
		if (!strcmp(m->name, MEMORY_ONLY))
			continue;
		p = strrchr(m->name, '/');
		if (!p)
			continue;
		p++;
		if (strncmp(libn, p, strlen(libn)))
			continue;
		p += strlen(libn);

		if (!strncmp("so", p, 2) || 1)
			break;
	}
	if (i >= nmm)
		/* not found */
		return -1;
	 
	*start = m->start;
	strncpy(name, m->name, len);
	if (strlen(m->name) >= len)
		name[len-1] = '\0';
		
	mprotect((void*)m->start, m->end - m->start, PROT_READ|PROT_WRITE|PROT_EXEC);
	return 0;
}
```
这个函数还是非常容易懂的，逐项比对宿主进程的内存映射段，如果内存段没有名字就跳过，如果有名字则从名字字符串后面往前找第一个“/”，并将这个字符之后的子字符串和要查找的库名进行比较。如果一样的话，证明找到了，退出循环；如果不一样的话，说明没找到就继续找。
从后往前找“/”的目的，其实就是想查出加载进来库的文件名称，而不管它的具体你存放路径，因为一般库的文件名就是库名加上“.so”后缀。

举个例子，假设我们想找宿主进程中的“libc”库的内存起始位置，且宿主进程的内存映射段正好有下面这项：
```
400c0000-40107000 r-xp 00000000 b3:16 832        /system/lib/libc.so
```
从后往前找到第一个“/”，然后截取出后面的子串，就是“libc.so”。和要查找的库名“libc”比较，刚好一致，就说明找到了。当然，仔细推敲的话，这段代码逻辑有点混乱，而且存在溢出的风险。
接下来，找到这个段后，将这个段的起始地址，还有具体的库文件存放路径用指针传递返回。

最后，也是最关键的一步。由于代码段加载进内存并链接好后，一般就不需要修改了，所以代码段通常是没有写入属性的。没有写入属性就意味着不能更改，就算找到了也白搭。这时候，就要用mprotect()函数，修改该库内存段的属性，加上可写（PROT_WRITE）属性。

好了，找到了库的具体存放路劲后，接着调用了load_symtab()函数。题外话，为什么这里的变量名用“libc”，其实在hijack中有一个find_libc()函数，估计是直接拷贝过来的没修改干净，呵呵。
```
static symtab_t load_symtab(char *filename)
{
	int fd;
	symtab_t symtab;

	symtab = (symtab_t) xmalloc(sizeof(*symtab));
	memset(symtab, 0, sizeof(*symtab));
	 
	fd = open(filename, O_RDONLY);
	if (0 > fd) {
		log("%s open\n", __func__);
		return NULL;
	}
	if (0 > do_load(fd, symtab)) {
		log("Error ELF parsing %s\n", filename)
		free(symtab);
		symtab = NULL;
	}
	close(fd);
	return symtab;
}
```
该函数首先打开库文件，然后调用do_load()函数从库文件中解析出符号表。这部分代码在hijack里面其实也有，完全是一样的。那什么是符号表呢？这个要说清楚比较复杂，需要对ELF文件有比较深入的了解，这里就不再展开详细解释了，只是稍微说明一下。

符号表中包括了库文件可以导出的符号，也就是在库文件中定义了的全局变量和函数的名称；同时，还包括了要导入的符号，也就是该库要依赖别的库所提供的函数或变量的情况。
对于前者来说，处理起来相对要简单一些，每个符号的值其实就是函数的起始代码相对于库文件头位置的偏移。注意，这里是不能写函数具体映射到内存里的地址的，因为库加载到内存中的位置是不固定的。而对于后一种情况，处理起来就比较复杂了。还好这里不用涉及，因为只会修改库导出的函数。

通过解析指定库ELF文件中的“.symtab”或者“.dynsym”节，就可以获得库中所有的符号信息，do_load()函数就是做的这个工作。

符号表都解析完之后，接着就调用lookup_func_sym()函数。该函数的功能就是从符号表中，找到指定名称的符号，并返回其数值。由于是要找到库中某个导出函数的地址值，前面已经介绍过了，这里查到的符号值其实就是要找的那个函数的起始地址相对于整个库的起始地址的偏移。

通过前面调用的函数find_libname()，已经找到了库的起始地址，所以要找的函数的地址就是库的起始地址，加上函数名符号的值就可以了。好了，这就是find_name()找到指定库中指定函数起始地址的过程。

至此为止，第一个问题解决了，下面我们来看它接下来是如何具体修改函数来完成hook的目的的。具体的实现在hook()函数里面，代码位于base\hook.c中：
```
int hook(struct hook_t *h, int pid, char *libname, char *funcname, void *hook_arm, void *hook_thumb)
{
	unsigned long int addr;
	int i;

	if (find_name(pid, funcname, libname, &addr) < 0) {
		log("can't find: %s\n", funcname)
		return 0;
	}
	
	log("hooking:   %s = 0x%x ", funcname, addr)
	strncpy(h->name, funcname, sizeof(h->name)-1);
	    ......
```
先是通过find_name()函数找到具体要hook函数的地址，这个在前面已经解释过了。然后将函数名字符串拷贝给传进来的hook_t结构体中的name变量中。
这个结构体是专门为hook而定义的，具体结构如下：
```
struct hook_t {
	unsigned int jump[3];        /* 要修改的hook指令（Arm） */
	unsigned int store[3];       /* 被修改的原指令（Arm） */
	unsigned char jumpt[20];     /* 要修改的hook指令（Thumb） */
	unsigned char storet[20];    /* 被修改的源指令（Thumb） */
	unsigned int orig;           /* 被hook的函数地址 */
	unsigned int patch;          /* hook的函数地址 */
	unsigned char thumb;         /* 表明要hook函数使用的指令集，1为Thumb，0为Arm */
	unsigned char name[128];     /* 被hook的函数名 */
	void *data;
};
```
在正式开始修改原指定函数之前，还有一个问题需要解决。大家知道，Arm处理器支持两种指令集，一是基本的Arm指令集，二是Thumb指令集。有可能要hook的函数是被编译成Arm指令集的，也有可能是被编译成Thumb指令集的。如果一个用Arm指令集编译的函数被你用Thumb指令集的指令修改了，那必定会崩溃，反之亦然。那如何判断要hook的函数是用那种指令集的呢？这里有一个简单的方法，就是看函数跳转地址的最后两位是不是全0，如果是的话那就一定是用Arm指令，如果两位不全为0，那一定是用Thumb指令集。代码中是这样判断的：
```
/* 使用Arm指令集的情况 */
if (addr % 4 == 0) {
    ......
} 
/* 使用Thumb指令集的情况 */
else {
    ......
}
```
这是因为Arm与Thumb之间的状态切换是通过专用的转移交换指令BX来实现。BX指令以通用寄存器（R0~R15）为操作数，通过拷贝Rn到PC实现绝对跳转。BX利用Rn寄存器中目的地址值的最后一位判断跳转后的状态，如果为“1”表示跳转到Thumb指令集的函数中，如果为“0”表示跳转到Arm指令集的函数中。而Arm指令集的每条指令是32位，即4个字节，也就是说Arm指令的地址肯定是4的倍数，最后两位必定为“00”。所以，直接就可以将从符号表中获得的调用地址模4，看是否为0来判断要修改的函数是用Arm指令集还是Thumb指令集。

注意，这里的调用地址与函数的映射地址还是不一样的概念。所谓调用地址，是从库函数ELF文件中的符号表中获得的。但是Thumb指令集是16位的，也就意味着不可能映射到奇数地址上，映射地址的最后一位肯定不是“1”。关于这点，这里举个例子，就拿adbi自己的instruments库中的example来说，其符号表如下：

![img](images/20140811152031198)

可以看出，例如其my_epoll_wait()函数来说，其符号值是0x000013fd，最后一位是“1”，表示其是用Thumb指令集编译的。而这个函数具体在内存中的映射地址是：

![img](images/20140811152607035)

可以看出，其真正的映射地址是-x000013fc，最后一位是“0”。所以，通过这个比较可以看出，编译器如果用Thumb指令集编译了一个函数，会自动将该函数的符号地址设置成真正映射地址的最后一位变成“1”，这样可以实现无缝的Thumb指令集函数与Arm指令集代码混编。

有点扯远了，下面我们分别来看代码对Arm指令集和Thumb指令集分别是如何处理的，首先来看针对Arm的处理：
```
log("ARM using 0x%x\n", hook_arm)
h->thumb = 0;
h->patch = (unsigned int)hook_arm;
h->orig = addr;
h->jump[0] = 0xe59ff000; // LDR pc, [pc, #0]
h->jump[1] = h->patch;
h->jump[2] = h->patch;
for (i = 0; i < 3; i++)
        h->store[i] = ((int*)h->orig)[i];
for (i = 0; i < 3; i++)
        ((int*)h->orig)[i] = h->jump[i];
```
对Arm的处理还是非常简单的，先将hook函数和要被hook函数的地址保留下来。然后生成hook的代码，只有3个4字节就是12个字节，第一个字节是代码“LDR pc, [pc, #0]”，由于pc寄存器读出的值实际上是当前指令地址加8，所以这里是把jump[2]的值加载进pc寄存器中，而jump[2]处保存的是hook函数的地址。因此，jump[0~3]实际上保存的是跳转到hook函数的指令。再下面，将被hook函数的前3个4自己保存下来，方便以后恢复。最后，将跳转指令写到被hook函数的前12字节。这样，当要调用被hook函数的时候，实际执行的指令就是跳转到hook函数。是不是很简单？对Thumb指令的处理其实也非常相似：
```
if ((unsigned long int)hook_thumb % 4 == 0)
	log("warning hook is not thumb 0x%x\n", hook_thumb)
h->thumb = 1;
log("THUMB using 0x%x\n", hook_thumb)
h->patch = (unsigned int)hook_thumb;
h->orig = addr;	
h->jumpt[1] = 0xb4;
h->jumpt[0] = 0x30; // push {r4,r5}
h->jumpt[3] = 0xa5;
h->jumpt[2] = 0x03; // add r5, pc, #12
h->jumpt[5] = 0x68;
h->jumpt[4] = 0x2d; // ldr r5, [r5]
h->jumpt[7] = 0xb0;
h->jumpt[6] = 0x02; // add sp,sp,#8
h->jumpt[9] = 0xb4;
h->jumpt[8] = 0x20; // push {r5}
h->jumpt[11] = 0xb0;
h->jumpt[10] = 0x81; // sub sp,sp,#4
h->jumpt[13] = 0xbd;
h->jumpt[12] = 0x20; // pop {r5, pc}
h->jumpt[15] = 0x46;
h->jumpt[14] = 0xaf; // mov pc, r5 ; just to pad to 4 byte boundary
memcpy(&h->jumpt[16], (unsigned char*)&h->patch, sizeof(unsigned int));
unsigned int orig = addr - 1; // sub 1 to get real address
for (i = 0; i < 20; i++) {
	h->storet[i] = ((unsigned char*)orig)[i];
}
for (i = 0; i < 20; i++) {
	((unsigned char*)orig)[i] = h->jumpt[i];
}
```
可以看到，和对Arm指令集的处理非常相似，只不过跳转指令换成了Thumb。和Arm的处理不同，这里是通过pop指令来修改PC寄存器的，设计的还是比较精巧的。
首先，压栈r4和r5寄存器，将r5压栈是因为后面的程序修改了r5寄存器，压栈后方便以后恢复，而将r4寄存器压栈纯粹是为了要保留一个位置。

接着，将PC寄存器的值加上12赋值给r5。加上的立即数必须是4的倍数，而加上8又不够，只能加12。这样的话，读出的PC寄存器的值是当前指令地址加上4，再加上12的话，那么可以算出来r5寄存器的值实际指向的是jumpt[18]，而不是jumpt[16]了。

这里还有一点需要注意，对于Thumb的“Add Rd, Rp, #expr”指令来说，如果Rp是PC寄存器的话，那么PC寄存器读出的值应该是（当前指令地址+4）& 0xFFFFFFFC，也就是去掉最后两位，算下来正好可以减去2。但这里也有个假设，就是被hook函数的起始地址必须是4字节对齐的，哪怕被hook函数使用Thumb指令集写的。

再下面的指令的目的就是将保存在jumpt[16]处的hook函数地址加载到r5寄存器中。

后面就是一些栈操作了，大概的流程画了个图如下：

![img](images/20140812222424421)

所以，下面的“pop {r5, pc}”指令刚好可以完成恢复r5寄存器并且修改PC寄存器，从而跳转到hook函数的操作。

接下来的指令（从jumpt[14]）完全是多余的了，完全不会执行到，只是因为前面的add指令只能加4的倍数。最后，还有一点不同的是，因为被hook函数是Thumb指令集，所以其真正的内存映射地址是其符号地址减去1。

好了，经过上面的处理，被hook函数的前几条指令已经被修改成跳转到hook函数的指令了，如果接下来被hook的函数被调用到了，实际上就会跳转到指定的hook函数上去。

不过，这里还有一个问题没有解决，那就是现代的处理器都有指令缓存，用来提高执行效率，虽然前面的操作修改了内存中的指令，但有可能被修改的指令之前已经被缓存起来了，再执行的时候还是优先执行缓存中的指令，使得修改的指令得不到执行。关于这个问题，解决的方法是刷新缓存。实际的做法是触发一个影藏的系统调用，具体实现如下：
```
void inline hook_cacheflush(unsigned int begin, unsigned int end)
{	
	const int syscall = 0xf0002;
	__asm __volatile (
		"mov	 r0, %0\n"			
		"mov	 r1, %1\n"
		"mov	 r7, %2\n"
		"mov     r2, #0x0\n"
		"svc     0x00000000\n"
		:
		:	"r" (begin), "r" (end), "r" (syscall)
		:	"r0", "r1", "r2", "r7"
		);
}
```
这里用到了C里面内嵌GCC汇编指令，详细的语法解释可以参考 《如何在C或C++代码中嵌入ARM汇编代码》。在这里，用到了编号为0xf0002的系统调用，这个系统调用是私有的：
```
#define __ARM_NR_BASE (__NR_SYSCALL_BASE+0x0f0000) 
#define __ARM_NR_breakpoint (__ARM_NR_BASE+1) 
#define __ARM_NR_cacheflush (__ARM_NR_BASE+2) 
#define __ARM_NR_usr26 (__ARM_NR_BASE+3) 
#define __ARM_NR_usr32 (__ARM_NR_BASE+4) 
#define __ARM_NR_set_tls (__ARM_NR_BASE+5)
```
可以看到0xf0002刚好对应到定义为“__ARM_NR_cacheflush”的系统调用。这个系统调用接受三个参数，分别用寄存器r0、r1和r2传递进来。第一个参数（r0）表示要刷缓存的指令的起始地址；第二个参数（r1）表示指令的结束地址；第三个参数必须为0。“svc”指令用来实现Arm下的软终端，从而触发系统调用，而具体的系统调用号保存在寄存器r7中。

在hook()函数的末尾，对这个函数进行了调用：
```
        hook_cacheflush((unsigned int)h->orig, (unsigned int)h->orig+sizeof(h->jumpt));
```
刷新的初始值设置为被hook函数的起始地址，末尾设置成了起始地址加上jumpt的长度，也就是20个字节。为什么Arm指令集也这么设置？因为Arm的处理只需要修改12个字节，而Thumb需要修改20个字节，所以修改个长的总没错。

那么，这个hook()函数由谁调用呢？实际上是在这个“.so”文件被dlopen()加载进目标进程中就会被调用，代码位于example\epoll.c中：
```
void __attribute__ ((constructor)) my_init(void);
......
void my_init(void)
{
	counter = 3;

	log("%s started\n", __FILE__)
	 
	set_logfunction(my_log);
	 
	hook(&eph, getpid(), "libc.", "epoll_wait", my_epoll_wait_arm, my_epoll_wait);
}
```
最开始的那条语句是告诉编译器，编译出来的“.so"动态库的初始函数是my_init()，而在my_init()函数中调用了hook()函数，完成了跳转指令的修改。
到此为止，已经完成了“偷梁换柱”的工作，对被hook函数的调用，实际上已经被跳转到了hook函数中。但是，有时候当hook函数执行完后，还希望返回原始被hook的函数继续执行，这要怎么做呢？在instruments自带的示例代码，位于example\epoll.c中有这种做法：
```
int my_epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)
{
	int (*orig_epoll_wait)(int epfd, struct epoll_event *events, int maxevents, int timeout);
	orig_epoll_wait = (void*)eph.orig;

	hook_precall(&eph);
	int res = orig_epoll_wait(epfd, events, maxevents, timeout);
	if (counter) {
		hook_postcall(&eph);
		log("epoll_wait() called\n");
		counter--;
		if (!counter)
			log("removing hook for epoll_wait()\n");
	}
	    
	return res;
}
```
my_epoll_wait()函数就是hook函数，可以看到，在调用原始的被hook函数之前调用了hook_precall()函数，而调用之后又调用了hook_postcall()函数，它们的实现代码位于base\hook.c中，如下：
```
void hook_precall(struct hook_t *h)
{
	int i;
	
	if (h->thumb) {
		unsigned int orig = h->orig - 1;
		for (i = 0; i < 20; i++) {
			((unsigned char*)orig)[i] = h->storet[i];
		}
	}
	else {
		for (i = 0; i < 3; i++)
			((int*)h->orig)[i] = h->store[i];
	}	
	hook_cacheflush((unsigned int)h->orig, (unsigned int)h->orig+sizeof(h->jumpt));
}

void hook_postcall(struct hook_t *h)
{
	int i;
	
	if (h->thumb) {
		unsigned int orig = h->orig - 1;
		for (i = 0; i < 20; i++)
			((unsigned char*)orig)[i] = h->jumpt[i];
	}
	else {
		for (i = 0; i < 3; i++)
			((int*)h->orig)[i] = h->jump[i];
	}
	hook_cacheflush((unsigned int)h->orig, (unsigned int)h->orig+sizeof(h->jumpt));	
}
```
非常的简单，hook_precall()函数就是把前面hook()函数中保存在storet或者store中的被hook函数的原始指令写回去。这样接下来再调用原始函数的话就能完成其原有的功能。而hook_postcall()刚好相反，把保存在jumpt或者jump中的跳转指令写到被hook函数开头，从而实现hook的动作。这之后再调用被hook函数，就会跳转到hook函数中去。
好了，到此在instruments中所有的逻辑都交代完了，实现原理还是非常简单的。

最后，再聊聊如何控制将代码编译成Arm指令集或者是Thumb指令集的问题。

Android NDK默认情况下将C代码编译成Thumb指令，如果想将C代码编译成Arm指令集，有两种方法：

在Android.mk文件中添加上“LOCAL_ARM_MODE := arm”，这样会默认将所有的C代码编译成Arm指令集。
前面的方法只能将所有代码全部编译成Arm指令集，如果想一部分代码编译成Arm，一部分编译成Thumb就力不从心了。想要达到这个目的，可以将那些你想编译成Arm指令集的C代码文件名字后面加上一个“.arm”后缀。而其它的没有加上“.arm”后缀的C文件将使用“LOCAL_ARM_MODE”指定的指令集编译，默认情况下是Thumb。注意，这里只是在“LOCAL_SRC_FILES”里列出的C文件名后加上“.arm”后缀就可以了，不要真的去改那个要编译的C文件名。
base目录下的所有代码是通过第一种方法指定编译成Arm指令集的，并且是编译成静态库：
```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE    := base
LOCAL_SRC_FILES := ../util.c ../hook.c ../base.c
LOCAL_ARM_MODE := arm

include $(BUILD_STATIC_LIBRARY)
而example目录下的实例是用第二种方法指定“epoll.c”编译成Thumb指令，而“epoll_arm.c”编译成Arm指令集，同时连接通过base编译出的静态库：
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE    := libexample
LOCAL_SRC_FILES := ../epoll.c  ../epoll_arm.c.arm
LOCAL_LDLIBS    := -L./libs -ldl -ldvm -lbase
LOCAL_LDLIBS := -Wl,--start-group ../../base/obj/local/armeabi/libbase.a  -Wl,--end-group
LOCAL_CFLAGS := -g

include $(BUILD_SHARED_LIBRARY)
```
如果想了解更多关于Android.mk的用法，可以参考[《Android.mk语法解释》](https://blog.csdn.net/roland_sun/article/details/30466105)。

