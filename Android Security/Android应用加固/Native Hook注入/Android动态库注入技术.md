# Android动态库注入技术

url：http://ele7enxxh.com/Android-Shared-Library-Injection.html



本文主要介绍了Android平台动态库注入技术。

关于这方面技术，网上已经有大把的实现。在此，我只是记录下自己的学习过程。

## 原理

所谓的SO注入就是将代码拷贝到目标进程中，并结合函数重定向等其他技术，最终达到监控或改变目标进程行为的目的。Android是基于Linux内核的操作系统，而在Linux下SO注入基本是基于调试API函数ptrace实现的，同样Android的SO注入也是基于ptrace函数，要完成注入还需获取root权限。

## 流程

注入过程如下：

- 获取目标进程的pid，关联目标进程；
- 获取并保存目标进程寄存器值；
- 获取目标进程的dlopen，dlsym函数的绝对地址；
- 获取并保存目标进程的堆栈，设置dlopen函数的相关参数，将要注入的SO的绝对路径压栈；
- 调用dlopen函数；
- 调用dlsym函数，获取SO中要执行的函数地址；
- 调用要执行的函数；
- 恢复目标进程的堆栈，恢复目标进程寄存器值，解除关联，完成SO动态库注入；

（注：实际上，0x06和0x07并不属于SO动态库注入的步骤，然而仅仅注入是完全没有意义的，通常我们需要执行SO中的函数。）

## 实现

- 获取目标进程的pid，关联目标进程：
  通过遍历查找/proc/pid/cmdline文件中是否含有目标进程名process_name,若有则进程名对应的进程号即为pid。接着，直接调用函数ptrace_attach(pid)即可完成关联。
- 获取并保存目标进程寄存器值：
  直接调用ptrace(PTRACE_GETREGS, pid, NULL, &saved_regs)，当然saved_regs要定义为全局变量。
- 获取目标进程的dlopen，dlsym函数的绝对地址：
  大概思路是这样的：首先通过遍历/proc/pid/maps文件分别得到本进程中dlopen函数所在动态库的基地址local_module_base和目标进程dlopen函数所在动态库的基地址remote_module_base，接着获取本进程dlopen函数的绝对地址local_addr = (void*)dlopen。需要明白的是，不同进程中相同的动态库中的同一个函数的偏移地址一定是一样的，所以目标进程dlopen函数的绝对地址为：local_addr - local_module_base + remote_module_base。dlsym同理，不再详述。
- 获取并保存目标进程的堆栈，设置dlopen函数的相关参数，将要注入的SO的绝对路径压栈：
  当我们的要执行的函数的某些参数需要压入堆栈的时候，就需要提前保存堆栈状态，调用ptrace_readdata(pid, (void *)regs.ARM_sp, (void* )sbuf, sizeof(sbuf))，其中sbuf为char数组，用来存放堆栈。调用ptrace_writedata(pid, (void *)regs.ARM_sp, (void* )so_path, strlen(so_path) + 1)，其中so_path为SO的绝对路径。函数传参规则：前四个参数分别由寄存器r0、r1、r2、r3存放，超过四个参数则压入堆栈。
- 调用dlopen函数：
  参数设置好后，设置ARM_pc = dlopen_addr, ARM_lr = 0。调用ptrace_setregs(pid, regs)写入修改后的寄存器值，调用ptrace_continue(pid)使目标进程继续运行。（注：dlopen_addr为获取到的目标进程dlopen函数的绝对地址，ARM_lr = 0的目的在于当目标进程执行完dlopen函数，使目标进程发生异常，从而让本进程重新获得控制权。）
- 调用dlsym函数，获取SO中要执行的函数地址：
  实现方式与调用dlopen函数类似，不再详述。
- 调用要执行的函数：
  实现方式与调用dlopen函数类似，不再详述。
- 恢复目标进程的堆栈，恢复目标进程寄存器值，解除关联，完成SO动态库注入：
  调用ptrace_writedata(pid, (uint8_t *)saved_regs.ARM_sp, (uint8_t* )sbuf, sizeof(sbuf))恢复堆栈，调用ptrace_setregs(pid, &saved_regs)恢复寄存器值，调用ptrace_detach(pid)解除关联，完成SO动态库注入。

## 代码

贴一下主要逻辑代码：

```
pid_t pid = find_pid_of("xxx");
ptrace_attach(pid);
uint32_t *inject_so_of(pid_t pid, const char *so_path) {
    int status = 0;
    struct pt_regs regs;
    memcpy(&regs, &saved_regs, sizeof(regs));
    ptrace_readdata(pid, (void *)regs.ARM_sp, (void *)sbuf, sizeof(sbuf));
    ptrace_writedata(pid, (void *)regs.ARM_sp, (void *)so_path, strlen(so_path) + 1);
    uint32_t parameters[2];
    parameters[0] = regs.ARM_sp;
    parameters[1] = RTLD_NOW;
    if ( ptrace_call(pid, find_dlopen_addr(pid), parameters, 2, &regs ) == -1 )
        DPRINTF("dlopen error\n");
    ptrace_getregs(pid, &regs);
    uint32_t r0 = regs.ARM_r0;
    DPRINTF("[+2]\t注入动态库成功，返回的句柄为: %x\n", r0);
    ptrace_setregs(pid, &saved_regs);
    ptrace_writedata(pid, (uint8_t *)saved_regs.ARM_sp, (uint8_t *)sbuf, sizeof(sbuf));
    ptrace_detach(pid);
    return (uint32_t *)r0;
}
```



## 参考

玩转ptrace：http://blog.csdn.net/sealyao/article/details/6710772
《转载》linux动态库注入：http://blog.chinaunix.net/uid-7247280-id-2060516.html
发个Android平台上的注入代码：http://bbs.pediy.com/showthread.php?t=141355