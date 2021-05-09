# 入门ART虚拟机

url：https://www.jianshu.com/p/20dcfcf27004



# 1） 加载DEX文件

以Android-6.0.1为例，从BaseDexClassLoader开始跟踪一些源码，只关心感兴趣的部分。

![image-20210509184057424](images/image-20210509184057424.png)

在构造函数中，new了一个DexPathList对象，赋值给成员变量pathList。(BaseDexClassLoader的findClass方法就是调用pathList.findClass来查找类的)

![image-20210509184130437](images/image-20210509184130437.png)

在DexPathList的构造函数中，调用makePathElements生成一个Element数组，赋值给dexElements。

每一个Element对象中封装着一个DexFile：

![image-20210509184154380](images/image-20210509184154380.png)

这个数组里面每一个Element中的dexFile成员，是在makePathElements函数中调用loadDexFile加载的。

![image-20210509184211352](images/image-20210509184211352.png)

loadDexFile是DexPathList的一个静态成员方法。

![image-20210509184228029](images/image-20210509184228029.png)

在loadDexFile中调用DexFile类的loadDex静态方法加载DEX。

![image-20210509184244144](images/image-20210509184244144.png)

loadDex直接new了一个DexFile对象。

![image-20210509184259410](images/image-20210509184259410.png)

在DexFile的构造函数中，调用openDexFile来加载DEX。

![image-20210509184315863](images/image-20210509184315863.png)

openDexFile又调用了openDexFileNative。

![image-20210509184331551](images/image-20210509184331551.png)

这是一个native方法，对应art/runtime/native/dalivk_system_DexFile.cc中的DexFile_openDexFileNative静态方法。

下一篇笔记再继续看DexFile_openDexFileNative源码，预留一个问题：假设要动态加载DEX文件，一些核心class的方法指令被抽走加密或虚拟化，在运行时再还原或解释执行。DexFile_openDexFileNative在加载DEX时，会有一个dex2opt的过程，通常壳代码会HOOK execv函数，阻止dex2opt对指定DEX文件的优化，那么这会对DexFile_openDexFileNative的执行产生哪些影响？进一步的，如果没有生成对应的OAT文件，那么在ART加载一个类方法时，在LinkCode时会有什么影响？



# 2）加载DEX文件续

接上文，继续看art/runtime/native/dalivk_system_DexFile.cc中的DexFile_openDexFileNative静态方法。

![image-20210509214232189](images/image-20210509214232189.png)

在DexFile_openDexFileNative中调用了ClassLinker的openDexFilesFromOat来加载DEX。

![image-20210509214249244](images/image-20210509214249244.png)

openDexFilesFromOat的内容比较多，分段来看，先构造一个OatFileAssistant对象，用于辅助从OAT文件中加载DEX。

![image-20210509214401834](images/image-20210509214401834.png)

继续看openDexFilesFromOat，这里假设该DEX是第一次被加载，系统中不存在对应的OAT文件。

![image-20210509214427948](images/image-20210509214427948.png)

先后调用了OatFileAssistant的MakeUpToDate和GetBestOatFile方法。

![image-20210509214532122](images/image-20210509214532122.png)

再继续看OatFileAssistant的GetDexOptNeeded方法。

![image-20210509214545893](images/image-20210509214545893.png)

因为是第一次加载该DEX，所以返回kDex2OatNeeded，MakeUpToDate将会调用GenerateOatFile为DEX生成对应的OAT文件。

![image-20210509214608295](images/image-20210509214608295.png)

GenerateOatFile调用Dex2Oat(Dex2Oat最终是调用execv执行Android系统中的/system/bin/dex2oat)来为DEX生成OAT文件。

现在假设正在加载的DEX，是一个被加固程序处理过的DEX，并且DEX壳代码HOOK execv函数，阻止了OAT文件的生成。回到openDexFilesFromOat，继续看OatFileAssistant的GetBestOatFile方法。

![image-20210509214644388](images/image-20210509214644388.png)

因为没有对应的OAT文件，所以最终返回空指针。再回到openDexFilesFromOat，继续看剩余的代码。

![image-20210509214721644](images/image-20210509214721644.png)

因为没有对应的OAT文件，所以最终会调用DexFile::Open方法来直接加载原始的DEX文件。

![image-20210509214748779](images/image-20210509214748779.png)



# 3）加载类和方法

还是以Android-6.0.1源码为例，从BaseDexClassLoader的findClass开始跟踪一些源码。

![image-20210509221752331](images/image-20210509221752331.png)

调用成员变量pathList的findClass方法来查找类。

![image-20210509222155836](images/image-20210509222155836.png)

在DexPathList类的findClass方法中，遍历dexElements数组的每一个成员（前面笔记提到过，每一个Element对象里面封装着一个DexFile），然后调用DexFile的loadClassBinaryName方法来加载类。



![image-20210509222248349](images/image-20210509222248349.png)

loadClassBinaryName又调用了defineClass。

![image-20210509222324482](images/image-20210509222324482.png)

defineClass又调用了native方法defineClassNative。

![image-20210509222451606](images/image-20210509222451606.png)

defineClassNative对应art/runtime/native/dalvik_system_DexFile.cc中的DexFile_defineClassNative方法。

![image-20210509222604689](images/image-20210509222604689.png)

这里还是只抓主干，不过多关注一些旁枝末节。在DexFile_defineClassNative方法中，调用class_linker对象的DefineClass来加载类。（在创建ART虚拟机的过程中，会创建一个ClassLinker对象，用于加载类和链接类方法）



![image-20210509222705151](images/image-20210509222705151.png)

ClassLinker方法的内容比较多，只关注其中最关键的几步。先调用InsertClass将新类添加到已加载类的列表中。再调用LoadClass和LinkClass来加载和链接类。

![image-20210509222732940](images/image-20210509222732940.png)

调用FindOatClass查找与被加载类对应的OatClass，然后调用LoadClassMembers加载类的成员。

![image-20210509222755050](images/image-20210509222755050.png)

前面笔记中做过一种假设，就是DEX加固壳阻止了指定DEX生成OAT文件的过程，这种情况下传给LoadClassMembers方法的oat_class参数为nullptr。

下篇笔记继续。



# 4）加载类和方法续



接上篇笔记。

![image-20210509223105296](images/image-20210509223105296.png)

在ClassLinker的LoadClass方法中，先调用DexFile的GetClassData方法从对应的DEX文件中得到class数据。

![image-20210509223148712](images/image-20210509223148712.png)

然后不管是否具有对应的oat_class，都是调用LoadClassMembers加载类的成员。前面笔记中做过一种假设，就是DEX加固壳阻止了指定DEX生成OAT文件的过程，这种情况下传给LoadClassMembers方法的oat_class参数为nullptr。

![image-20210509223251531](images/image-20210509223251531.png)

LoadClassMembers方法的前半部分用于加载class的静态域和实例域，主要看后面加载方法的部分。

![image-20210509223341647](images/image-20210509223341647.png)

先加载direct方法，再加载virtual方法。先调用LoadMethod方法初始化ArtMethod对象。

![image-20210509223450719](images/image-20210509223450719.png)

回到LoadClassMembers，再继续调用LinkCode链接类方法。先来看LinkCode的前半部分。

![image-20210509223516003](images/image-20210509223516003.png)

如果oat_class不为空，则调用其GetOatMethod方法得到相应的oat_method，再调用LinkMethod方法对method进行设置。

![image-20210509223529947](images/image-20210509223529947.png)

实际上就是设置ArtMethod对象的entry_point_from_quick_compiled_code_成员，作为编译代码入口点。

继续看LinkCode，调用NeedsInterpreter，看看是否需要解释执行。

![image-20210509223543483](images/image-20210509223543483.png)

第2个参数quick_code传入的是method->GetEntryPointFromQuickCompiledCode()，也就是LinkMethod里面设置的entry_point_from_quick_compiled_code_。如果前面oat_class为空，没有设置entry_point_from_quick_compiled_code_，那我觉得NeedsInterpreter传入的参数quick_code应该为空，不管ART是否运行在解释模式中，NeedsInterpreter都会返回true。

之所以说这些，是因为我发现有些DEX壳代码，在运行时想方设法的强制将ART设置为interpret-only，但单从加载类方法这块代码来看，如果壳代码之前阻止了指定DEX生成OAT的过程，这块在找不到方法对应的机器代码入口点的情况下，即使不设置interpret-only，也会以解释运行的方式设置各个Entry Point。（我对ART的了解还太少，后面随着学习的不断深入，再回过头来想想这个问题）

贴出LinkCode的后半部分代码。

![image-20210509223850181](images/image-20210509223850181.png)

主要就是根据各种情况，设置method的解释执行入口点，或者执行编译代码的入口点。



# 5）OAT文件

先贴老罗的一张图：

![image-20210509230303477](images/image-20210509230303477.png)

> 再摘一段老罗的描述：“作为Android私有的一种ELF文件，OAT文件包含有两个特殊的段oatdata和oatexec，前者包含有用来生成本地机器指令的dex文件内容，后者包含生成的本地机器指令，它们之间的关系通过储存在oatdata段前面的oat头部描述。此外，在OAT文件的dynamic段，导出了三个符号oatdata、oatexec和oatlastword，它们的值就是用来界定oatdata段和oatexec段的起止位置的。”

老罗的这段描述，有些地方稍微有点不太准确。

符号oatdata、oatexec和oatlastword是动态符号表(.dynsym)中导出的，不是dynamic段导出的。

![image-20210509230321377](images/image-20210509230321377.png)

另外，oatdata和oatexec并不是OAT(ELF格式)文件中两个独立的段，而是分别位于第1个和第2个LOAD段中，这一点对比上面的.dynsym和下面的Program Headers Table就可以看出来了。

![image-20210509230336219](images/image-20210509230336219.png)

如何在OAT文件中找到一个类方法的本地机器指令呢？还是贴老罗的一幅图，再结合上图和老罗的一段描述(感谢老罗的博客)就可以大概理解了。

![image-20210509230349361](images/image-20210509230349361.png)

> “首先根据类签名信息从包含在OAT文件里面的DEX文件中查找目标Class的编号，然后再根据这个编号在OAT文件中找到对应的OatClass。接下来再根据方法签名从包含在OAT文件里面的DEX文件中查找目标方法的编号，然后再根据这个编号在前面找到的OatClass中找到对应的OatMethod。有了这个OatMethod之后，我们就根据它的成员变量begin_和code_offset_找到目标类方法的本地机器指令了。”

下面以Android 6源码为例，看一下OatFile::Open函数(art/runtime/Oat_file.cc)：

![image-20210509230805295](images/image-20210509230805295.png)

> 可以看到，有两个函数可以加载OAT文件：OpenDlopen和OpenElfFile。这两个函数有什么区别？继续摘老罗的博客(非原文，简化了一下)：“ART运行时会为类方法生成相应的本地机器指令，这些本地机器指令可能会调用外部函数，这就涉及到模块依赖问题，就好像我们在编写程序时，需要依赖C库提供的接口一样。ART运行时支持两种类型的Backend：Portable和Quick。Portable类型的Backend通过静态链接器生成本地机器指令，通过重定位技术来处理模块依赖问题。这对熟悉linker动态加载过程的程序员来说很容易理解。而Quick类型的Backend生成的本地机器指令用另外一种方式来处理模块之间的依赖关系。简单的说，就是ART运行时会在每一个线程的TLS（线程本地区域）提供一个函数表，本地机器指令通过它来调用其它模块的函数。这使得生成的OAT文件在加载时不需要再处理模块之间的依赖关系，也就省去了重定位，不需要通过系统的动态链接器提供的dlopen来加载。这样OAT文件在加载时就会更快，这也是称其为Quick的缘由。”。

仔细看一下OatFile::Open，会发现参数executable为false时，不会执行OpenDlopen。什么情况下executable为false？如果是dex2oat过程中调用的OatFile::Open，参数executable就为false。

调用OpenDlopen加载非executable的OAT文件可能会失败，具体看函数注释：

![img](https:////upload-images.jianshu.io/upload_images/20238-a1ff312226144982.png?imageMogr2/auto-orient/strip|imageView2/2/w/571/format/webp)

主要看OpenElfFile函数：

![image-20210509230838019](images/image-20210509230838019.png)

继续跟OatFile::ElfFileOpen（省去了一些出错处理代码）：

![image-20210509230910026](images/image-20210509230910026.png)

先Open文件，再Load加载，然后调用FindDynamicSymbolAddress找到OAT文件中的oatdata、oatlastword、oatbss、oatbsslastword四个符号的地址。

其中Open和Load的过程类似于linker的dlopen过程的，但没有重定位，仅仅是将OAT文件的LOAD段映射到内存，并解析出字符串表、符号表等重要信息的地址等等。

这里摘录Roland_Sun博客对这个过程的一点总结：

> 1）如果elf文件中包含了虚拟地址为0的PT_LOAD段，则证明它不是Boot Oat，而是一个普通的应用程序的oat，这时候该elf文件无所谓被映射到内存中的任何地方，其虚拟地址（p_vaddr）中记录的是该段相对于文件头的偏移；
>
> 2）如果elf文件中没有包含任何虚拟地址为0的PT_LOAD段，则证明它是一个Boot Oat，必须被加载到一个指定位置（实际是紧接在Image之后），其虚拟地址（p_vaddr）中记录的就是实际要被加载的绝对地址。

Boot OAT的Program Header：

![image-20210509230927648](images/image-20210509230927648.png)

一个普通应用程序的OAT的Program Header：

![image-20210509230938384](images/image-20210509230938384.png)

回到OatFile::ElfFileOpen，继续看Setup函数。下篇笔记继续。





# 6）OAT文件续

接上篇笔记。

先贴一下OatFile::Setup函数（去掉了所有的错误检查代码**，**这样更清晰）





在**oat** data的开始**位置是**OatHeader**，****Begin**()**返回的就是**上篇笔记中解析出的oatdata符号的地址。

OatHeader中的主要内容如下：





以一个实际的**OAT**文件为例：





可以看出**，**该**OAT**文件中包含一个DEX文件**，**该**OAT**文件就是由该DEX文件经过dex2**oat**生成的。

**oat** exec的偏移是2000h**，**该偏移是相对OatHeader（偏移为1000h）来说的**，**所以**oat** exec的文件偏移是3000h。

验证一下：





接下来是一系列的bridge和trampoline代码偏移**，**以后学习ART执行类方法时再说。

继续看Setup函数**，**跳过OatHeader**，**再跳过key_value_store**，**再后面就是DEX文件的相关内容了。

什么是key_value_store？dex2**oat**在将DEX转换为**OAT**的过程中**，**会在OatHeader后面填充一些key-value对**，**内容诸如“dex2**oat**-cmdline”等等：







继续看Setup函数**，**接着是一个for循环**，**依次解析每一个DEX文件的信息。





还是以实际的**OAT**文件内容对比着来看代码。





先是一个记录DEX文件位置的char数组**，**以及这个char数组的大小。然后是DEX文件的校验和。

再然后是DEX文件的偏移**，**这里的E8h是相对OatHeader来说的。OatHeader的偏移是1000h**，**所以DEX文件的偏移是10E8h。





再然后是一个偏移数组class_offs**，**因为相应的DEX文件中共有13个class（相应DEX文件的**header**中**，**class_defs_size的值为13）**，**所以数组的大小是13。而数组中的每一个偏移值指向一个数据结构**，**这个数据结构记录了相应的class的信息**，**尤其是记录了其methods偏移的code_offset等等。

下图是该DEX文件中包含的13个class。





下面看一下这个偏移数组class_offs的具体的值：





以class_offs[1]为例**，**偏移是15BCh**，**加上OatHeader的偏移1000h**，**所以总的偏移是25BCh。





那么这块内容就记录了**oat**_class[1]的信息。





状态值为9**，**表示“Class init in progress”。





该类包含两个方法**，**这两个方法对应的native代码的偏移分别是206Dh和20E5h（相对OatHeader的偏移）。

再回去看Setup函数的代码**，**class_offs数组的偏移位置赋值给了methods_offsets_pointer**，**然后在创建OatDexFile对象时作为参数传入。

在OatDexFile的构造函数中**，**将该值赋值给了成员变量**oat**_class_offsets_pointer_。





该成员变量在后续ART查找类和方法的过程中会用到。



完。