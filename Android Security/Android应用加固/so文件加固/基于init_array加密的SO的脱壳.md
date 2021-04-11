# 基于init_array加密的SO的脱壳

url：http://ele7enxxh.com/Unpack-Android-Shared-Library-Based-On-Init_Array-Encryption.html



看雪上已经有好几个帖子讲述了这方面的原理了，也已经现成的dump工具，不过都没有开源出来，秉着学习的态度自己研究了下，最终实现了基于init_array加密的SO的脱壳。

具体的原理已经有人分享了出来：

- ELF section修复的一些思考：http://bbs.pediy.com/showthread.php?t=192874
- 从零打造简单的SODUMP工具：http://bbs.pediy.com/showthread.php?t=194053

我这里给出每个节区addr，offset，size的详细计算方法：

**SHN_UNDEF：全部是0，没什么好说的**

**.dynsym，.hash，.rel.dyn，.rel.plt，.ARM.exidx，.fini_array，.init_array这几个节区直接通过soinfo结构体就能直接恢复：**

- .dynsym：addr = offset = si->symtab - si->base; size = si->nchain * 16;
- .hash：addr = offset = hash_shdr(注1); size = (2 + si->nbucket + si->nchain) * 4;
- .rel.dyn：addr = offset = si->rel - si->base; size = si->rel_count * sizeof(Elf32_Rel);
- .rel.plt：addr = offset = si->plt_rel - si->base; size = si->plt_rel_count * sizeof(Elf32_Rel);
- .ARM.exidx：addr = offset = si->ARM_exidx - si->base; size = si->ARM_exidx_count * 8;
- .fini_array：addr = si->fini_array - si->base; offset = si->fini_array - si->base - 0x1000; size = si->fini_array_count * sizeof(Elf32_Addr);
- .init_array：addr = si->init_array - si->base; offset = si->init_array - si->base - 0x1000; size = si->init_array_count * sizeof(Elf32_Addr);
- .dynamic：addr = si->dynamic - si->base; offset = si->dynamic - si->base - 0x1000; size = dynamic_count * sizeof(Elf32_Dyn);

**.dynstr，.hash，.dynamic：**

```
Elf32_Word strsz = 0;
Elf32_Addr hash_shdr = 0;
size_t dynamic_count = 0;
for (Elf32_Dyn* d = si->dynamic; d->d_tag != DT_NULL; ++d) {
    switch (d->d_tag) {
    case DT_STRSZ:
        strsz = d->d_un.d_val;
        break;
    case DT_HASH:
        hash_shdr = d->d_un.d_ptr;
        break;
    }
    dynamic_count++;
}
dynamic_count++;
```

- .dynstr：addr = offset = si->strtab - si->base; size = strsz;
- .hash：addr = offset = hash_shdr; size = (2 + si->nbucket + si->nchain) * 4;
- .dynamic：addr = si->dynamic - si->base; offset = si->dynamic - si->base - 0x1000; size = dynamic_count * sizeof(Elf32_Dyn);

**.plt，通过遍历plt头部的固定十六个字节确定起始位置：**

```
unsigned int plt_start[] = {0xe52de004, 0xe59fe004, 0xe08fe00e, 0xe5bef008 };
Elf32_Addr plt_shdr = 0;
for (int i = 0; i < dump_size - 16; i++) {
    if (memcmp(dump_correct_so + i, plt_start, 16) == 0) {
        plt_shdr = i;
        break;
    }
}
```

其中dump_size为dump且内存修正过后的SO的大小（实际上就是第二个load段的vaddr加上filesz的值），dump_correct_so为dump且内存修正后的SO的指针。

- .plt：addr = offset = plt_shdr; size = 20 + 12 * si->plt_rel_count;

**.got**

```
int type = 0;
int flag = 0;
Elf32_Addr got_addr = 0;
Elf32_Word gotsz = 0;
Elf32_Word global_offset_table = (Elf32_Word)si->plt_got - (Elf32_Word)si->base;
for (Elf32_Rel* rel = si->rel; (Elf32_Addr)rel < (Elf32_Addr)si->rel + si->rel_count * sizeof(Elf32_Rel); rel++) {
    if (rel->r_offset == global_offset_table - 4) {
        type = 1;
        break;
    }
}
if (type == 1) {
    for(Elf32_Word global_offset = global_offset_table - 4; global_offset > 0; global_offset -= 4) {
        flag = 0;
        for (Elf32_Rel* rel = si->rel; (Elf32_Addr)rel < (Elf32_Addr)si->rel + si->rel_count * sizeof(Elf32_Rel); rel++) {
            if (rel->r_offset == global_offset) {
                got_addr = global_offset;
                flag = 1;
                break;
            }
        }
        if(flag == 0) {
            break;
        }
    }
    gotsz = global_offset_table + 8 + si->plt_rel_count * 4 + 4 - got_addr;
    rebuildSectionHeader(".got", shdr_addr, 119, SHT_PROGBITS, SHF_ALLOC | SHF_EXECINSTR, got_addr,
            (Elf32_Off)got_addr - 0x1000, gotsz, 0, 0, 4, 0);
    shdr_addr += sizeof(Elf32_Shdr);
}
else {

}
```

根据开始贴的两篇帖子，可以知道__global_offset_table__之前和之后为不同的got结构，可能为：{.got, .got.plt}、{.got.plt, .got}中的一种，并且.got.plt与.rel.plt一一对应，.got中的项一定会出现在.rel.dyn中。
Step1: 读取 DT_PLTGOT,获取__global_offset_table__地址,记为:plt_got；
Step2: 读取 plt_got – 4 地址的数值,在.rel.dyn表中进行搜索。如果匹配,说明 GOT 结构式:{.got, .got.plt},转 3.1;否则为{.got.plt, .got}转 3.2；
Step3.1: 继续向前搜索(plt_got -= 4),直到没有在.rel.dyn匹配，从而确定.got起始地址；由于.got.plt与.rel.plt一一对应，所以plt_got之后的大小也可以确定，从而得到gotsz；
Step3.2: 同理由于.got.plt与.rel.plt一一对应，可以确定.got的起始位置，再对(plt_got += 4)地址的数值在.rel.dyn表中进行搜索，直到找到末尾位置。

- .got：addr = offset = got_addr; size = gotsz;

**.text，.ARM.extab，.rodata，.data，.bss，这几个节表均是通过节表之间的相对位置进行重建的，如果节表位置被diy过，则重建无效，两篇帖子已有介绍，不再详述**

**.shstrtab，这个节表也没什么好说的，按照上面重建的节表，直接重建即可**

完整代码就不上了，写的太丑了，真是毫无美感。