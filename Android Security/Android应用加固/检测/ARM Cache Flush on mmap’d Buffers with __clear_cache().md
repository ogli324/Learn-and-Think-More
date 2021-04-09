## ARM Cache Flush on mmap’d Buffers with __clear_cache()

url：https://minghuasweblog.wordpress.com/2013/03/29/arm-cache-flush-on-mmapd-buffers-with-clear-cache/

[Memory mapped operation from user space on devices is a powerful technique to improve runtime performance](https://minghuasweblog.wordpress.com/2013/03/25/mapping-dma-buffers-to-user-space-on-linux-with-mmap/). Some ARM processors use caches keyed to virtual addresses, instead of normally to physical addresses. The problem, is that if the kernel maps the region to one virtual address and the userspace the another, changes made on one side won’t reliably be flushed to the other side. At best the other side sees delayed content change. In the worst case, data will be lost.

From userspace, once the content is changed, it can call `__clear_cache()` to flush and invalidate the cache. This process works to write the content in the cache line into physical memory, and invalidate the cache line. The next time you read the date cache will re-read the memory and fill its content with real data in memory. It works if the mmap is opened with “PROT_EXEC” flag according to [this stackoverflow answer](http://stackoverflow.com/questions/6046716/how-clear-and-invalidate-arm-v7-processor-cache-from-user-mode-on-linux-2-6-35). Or you may follow the link on the so answer to read the [original article on arm.com about cache and self modifying code](http://blogs.arm.com/software-enablement/141-caches-and-self-modifying-code/).

If your compiler does not provide the builtin implementation you can implement it by yourself:

```
void clearcache(char* begin, char *end)
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
		:	"r0", "r1", "r7"
		);
}
```

The snippet of code is suggested by [an android lib implementation](http://code.google.com/p/android/issues/detail?id=1803).

The code behind the implementation is to make a system call 0xf0002. What’s behind that system call can be found following this steps:

- - Look for syscall and follow the definitions of entries in sys_call_table

```
1357 cd include
1358 find . -name 'syscall*'
1360 vi linux/syscalls.h
1363 vi asm-generic/syscalls.h
1364 vi asm-generic/syscall.h
1366 grep -r SYSCALL_DEFINEx *
1367 cd ..
1368 cd arch/arm/
1369 grep -r SYSCALL_DEFINEx *
1371 grep -r sys_call_table
1372 vi kernel/entry-common.S
1374 find . -name 'calls.S'
1375 vi kernel/calls.S
```

- Then look into calls.S, you find need to search for “CALL”:

```
arch/arm/include/asm/unistd.h:#define __NR_SYSCALL_BASE 0
arch/arm/include/asm/unistd.h:#define __NR_SYSCALL_BASE __NR_OABI_SYSCALL_BASE
arch/arm/include/asm/unistd.h:#define __ARM_NR_BASE     (__NR_SYSCALL_BASE+0x0f0000)
```

- Since you found “0x0f0000”, follow “__ARM_NR_”

```
[arch/arm]$ grep -r __ARM_NR_ *
include/asm/unistd.h:#define __ARM_NR_BASE           (__NR_SYSCALL_BASE+0x0f0000)
include/asm/unistd.h:#define __ARM_NR_cacheflush     (__ARM_NR_BASE+2)
kernel/traps.c:#define NR(x) ((__ARM_NR_##x) - __ARM_NR_BASE)
```

- The implementation is in the case for “cacheflush”, in traps.c

```
    case NR(cacheflush):
            do_cache_op(regs->ARM_r0, regs->ARM_r1, regs->ARM_r2);
            return 0;
```