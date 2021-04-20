# Android SO文件保护加固——加密篇

url：

https://blog.csdn.net/feibabeibei_beibei/article/details/51498285

https://blog.csdn.net/feibabeibei_beibei/article/details/52642288



## Section加密

>  这篇是一系列的关于SO文件保护的自我理解，SO文件保护分为加固，混淆以及最近炒的比较火的虚拟机，由于本人菜鸟，无力分析虚拟机，我相信以后会有机会。。。加固就是将真正的so代码保护起来，不让攻击者那么轻易的发现，至于混淆，由于ART机制的介入，使得O-LLVM越来越火，这以后有机会再分析，这次主要是基于有源码的so文件保护，下次介绍无源码的so文件保护，废话不多说，开搞

在这之前首先对elf文件结构有一定的了解，不一定完全了解，本菜鸟就不是完全懂，在文章开始之前有个知识点必须了解：

![img](images/20160525155024790)

这两个节头要有所了解：

.init:可执行指令，构成进程的初始化代码，发生在main函数调用之前。

.fini:进程终止指令，发生在main函数调用之后。

以上这么分析感觉有点像c++的构造函数和析构函数，的确构造和析构是由此实现的。

并且结合GGC的可扩展机制：

`__attribute__((section(".mytext")));`可以把相应的函数和要保护的代码放在自己所定义的节里面。

这就引入了我们今天的主题，可以把我们关键的so文件中的核心函数放在自己所定义的节里面，然后进行加密保护，在合适的时机构造解密函数，当然解密函数可以用这个`_attribute__((constructor))`进行定义;类似于C++构造函数发生在main函数之前。

OK这个就是这篇文章的核心思想。

流程安排：

1.编写一个Native程序，对里面的关键函数放在自己所定义的节中，并且编写解密函数（当然这个是在你已知加密函数的基础上）

2.对得到的.so文件进行加密

3.加密后的替换验证

接下来走流程：

1.编写一个简单的计算器，把核心的代码放在.so文件里面如图：

![img](images/20160525161404337)

这个比较简单很容易理解：

接下来是关键函数的自定义与解密函数：直接看代码：

```
#include "com_example_jni02_CallSo.h"
#include <jni.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <elf.h>
#include <sys/mman.h>
#include <Android/log.h>
//这里对 Java_com_example_jni02_CallSo_plus这个方法进行加密保护
jint JNICALL Java_com_example_jni02_CallSo_plus(JNIEnv* env, jobject obj, jint a, jint b)  __attribute__((section (".mytext")));
JNIEXPORT jstring JNICALL Java_com_example_jni02_CallSo_getString
  (JNIEnv* env, jobject obj){
	return (*env)->NewStringUTF(env,"Hello");
}
JNIEXPORT jint JNICALL Java_com_example_jni02_CallSo_plus
  (JNIEnv* env, jobject obj, jint a, jint b){
 
	return a+b;
}
//在调用so文件进行解密
void init_Java_com_example_jni02_CallSo_plus() __attribute__((constructor));
unsigned long getLibAddr();
 
void init_Java_com_example_jni02_CallSo_plus(){
	char name[15];
	unsigned int nblock;
	unsigned int nsize;
	unsigned long base;
	unsigned long text_addr;
	unsigned int i;
	Elf32_Ehdr *ehdr;
	Elf32_Shdr *shdr;
	base=getLibAddr();
	ehdr=(Elf32_Ehdr *)base;
	text_addr=ehdr->e_shoff+base;
	nblock=ehdr->e_entry >>16;
	nsize=ehdr->e_entry&0xffff;
	__android_log_print(ANDROID_LOG_INFO, "JNITag", "nblock =  0x%d,nsize:%d", nblock,nsize);
    __android_log_print(ANDROID_LOG_INFO, "JNITag", "base =  0x%x", text_addr);
	printf("nblock = %d\n", nblock);
   //修改内存权限
	 if(mprotect((void *) (text_addr / PAGE_SIZE * PAGE_SIZE), 4096 * nsize, PROT_READ | PROT_EXEC | PROT_WRITE) != 0){
	   puts("mem privilege change failed");
	    __android_log_print(ANDROID_LOG_INFO, "JNITag", "mem privilege change failed");
	 }
	 //进行解密，是针对加密算法的
	 for(i=0;i<nblock;i++){
		 char *addr=(char*)(text_addr+i);
		 *addr=~(*addr);
	 }
	  if(mprotect((void *) (text_addr / PAGE_SIZE * PAGE_SIZE), 4096 * nsize, PROT_READ | PROT_EXEC) != 0){
	    puts("mem privilege change failed");
	  }
	  puts("Decrypt success");
}
//获取到SO文件加载到内存中的起始地址，只有找到起始地址才能够进行解密；
unsigned long getLibAddr(){
	unsigned long ret=0;
	char name[]="libaddcomputer.so";
	char buf[4096];
	char *temp;
	int pid;
	FILE *fp;
	pid=getpid();
	sprintf(buf,"/proc/%d/maps",pid);
	fp=fopen(buf,"r");
	if(fp==NULL){
		puts("open failed");
		goto _error;
	}
	while (fgets(buf,sizeof(buf),fp)){
		if(strstr(buf,name)){
			temp = strtok(buf, "-");
			ret = strtoul(temp, NULL, 16);
			break;
		}
	}
	_error:
	  fclose(fp);
	  return ret;
}
```
在这里重点解释这个解密函数：

首先看到的是getLibAddr()这个函数：在介绍这个函数之前首先了解一个内存映射问题：

和Linux一样，Android提供了基于/proc的“伪文件”系统来作为查看用户进程内存映像的接口(cat /proc/pid/maps)。可以说，这是Android系统内核层开放给用户层关于进程内存信息的一扇窗户。通过它，我们可以查看到当前进程空间的内存映射情况，模块加载情况以及虚拟地址和内存读写执行（rwxp）属性等。

![img](images/20160526150148650)

接下来包括内存权限的修改以及函数的解密算法，最后包括内存权限的修改回去，应该都比较好理解。ok，以上编写完以后就编译生成.so文件。
2.对得到的.so文件进行加密：

这一块也是一个重点，大致上逻辑我们可以这么认为：先找到那个我们自己所定义的节，然后找到对应的offset和size，最后进行加密，加密完以后重新的写到另一个新的.so文件中，这块是需要建立在对ELF了解的基础上

这里重点了解一下这个加密函数，在自己写的时候可以在这个基础上进行改进。

首先看一下这个核心加密代码：

```
private static void encodeSection(byte[] fileByteArys){
		//读取String Section段
		System.out.println();
		
		int string_section_index = Utils.byte2Short(type_32.hdr.e_shstrndx);
		elf32_shdr shdr = type_32.shdrList.get(string_section_index);
		int size = Utils.byte2Int(shdr.sh_size);
		int offset = Utils.byte2Int(shdr.sh_offset);
 
		int mySectionOffset=0,mySectionSize=0;
		for(elf32_shdr temp : type_32.shdrList){
			int sectionNameOffset = offset+Utils.byte2Int(temp.sh_name);
			if(Utils.isEqualByteAry(fileByteArys, sectionNameOffset, encodeSectionName)){
				//这里需要读取section段然后进行数据加密
				mySectionOffset = Utils.byte2Int(temp.sh_offset);
				mySectionSize = Utils.byte2Int(temp.sh_size);
				byte[] sectionAry = Utils.copyBytes(fileByteArys, mySectionOffset, mySectionSize);
				for(int i=0;i<sectionAry.length;i++){
					//sectionAry[i] = (byte)(sectionAry[i] ^ 0xFF);
					sectionAry[i]=(byte) ~sectionAry[i];
				}
				Utils.replaceByteAry(fileByteArys, mySectionOffset, sectionAry);
			}
		}
 
		//修改Elf Header中的entry和offset值
		int nSize = mySectionSize/4096 + (mySectionSize%4096 == 0 ? 0 : 1);
		byte[] entry = new byte[4];
		entry = Utils.int2Byte((mySectionSize<<16) + nSize);
		Utils.replaceByteAry(fileByteArys, 24, entry);
		byte[] offsetAry = new byte[4];
		offsetAry = Utils.int2Byte(mySectionOffset);
		Utils.replaceByteAry(fileByteArys, 32, offsetAry);
	}
```

以上加密是没有问题的，但是对于最后so文件头的修改简单的说明一下：
修改so文件为什么不会报错的原因进行简单的说明：

我们在这考虑一个问题就是Section与Segment的区别，由于OS在映射ELF到内存时，每一个段会占用是页的整数倍，这样会产生浪费，在操作系统的层面来讲，可以吧相同权限的section放在一起成为一个Segment再进行映射，这样一来减少浪费，但是在映射的时候会有一部分信息不会映射到内存中，可以看这个图：

![img](images/20160526162933267)

因此来说修改这些不会报错。

3.对于文件替换后没有什么问题，运行结果为：

![img](images/20160526163354478)



该篇是在有源码的基础上进行对特定的section进行加密，但是试想一下，有多少情况下才能有源码，因此局限性比较大，



下一篇是基于二进制级别的特定函数的加密，链接为：[点击打开链接](http://blog.csdn.net/feibabeibei_beibei/article/details/52642288)
源码是：http://download.csdn.net/detail/feibabeibei_beibei/9532172



## 函数加密

已经很长一段时间没更新了，一方面本人技术一般，不知道能给技术网友分享点什么有价值的东西，一方面，有时候实验室事比较多，时间长了就分享的意识就单薄了，今天接着前面那个so文件重要函数段加密，接着更，接下来开始书写。

一、原理篇

在很多时候我们对一个.so文件中重要的函数段加密时，是无法拿到源码的，当然并不排除可能在以后随着Android开发与安全结合，出现逆向开发者，在开发的时候就进行一些重要的函数的保护，在上一篇中是对特定的section的加密，加密就可以根据section来进行查找加密不需要源码，而解密是利用linker在加载执行的时候利用__attribute__((constructor))的特性实行so加载时的自解密的特性； 

在这一篇中，我们不需要源码，我们拿到待需要保护的.so文件进行加密，怎么加密呢？是基于特定函数进行加密，具体的原理后面会详解；我们知道任何函数在加密过后，在你加载执行的时候都是需要解密的，如果不进行解密的话是一定会报错的，因此这里面比较重要的就是解密时机，只要在加密函数被执行前进行解密完就ok，可以另外写一个.so文件为解密文件，只要在执行函数前解密就可以，原理就是这样。

基本上大致的步骤为：

1.首先要给你进行保护的.so文件中的重要函数加密；
2.逆向开发，自己写针对以上待解密的.so文件；
3.然后修改smali层进行调用解密so文件；

二、详解篇

这里重点讲解对于一个.so文件的重要函数进行加密，逆向开发以及解密在下面实现篇进行介绍；

我们可以用IDA工具拿到要加密的函数名，比如本篇中：



![img](images/20160924095529567)



接下来最重要的就是怎么在这个.so文件中找到这个函数名，需要对ELF文件有足够的了解：有一篇北大的关于ELF篇的详细分析文档 ，当然我都会传到后面附件中。

在加密之前要明白一点就是在ELF文件中一些关于动态链接的时候的一些重要的节区，观察下面这个表格:

![img](images/20160924101326091)

以看出这几个节区的重要性，因此接着看下面这个加密的流程；这一块是借鉴网上大神的一个源码，当然后面会给出分享，我只是在此做出一个梳理有利于读者的学习和理解：

加密流程如下：

1.解析文件头，获取e_phoff、e_phentsize和e_phnum等字段的信息，后面根据这些信息进一步得到p_offset和p_filesz；

2.根据程序头部的结构中的p_type得到Dynamic段的偏移值和大小；



关键代码段为：


![img](images/20160924151524275)



3.遍历Dynamic段找到dynsym、.dynstr、.hash section文件中的偏移和.dynstr的大小；这块大家可能比较好奇为什么一个段中有这么多节，这是因为在执行时，把一些相同权限的节放在一起，以减少空间浪费；

大家或许会问为什么有.hash节？

因为别的与此函数名相关的section的type有可能会相同)

比如看这个so文件
![img](images/20160924151108349)

这个时候我们看到在北大的ELF分析中讲到：（以下是对原内容的截取）

![img](images/20160924154206686)

因此我们可以来分析hash .dynamic段一般用于动态链接的，所以.dynsym和.dynstr，.hash肯定包含在这里。我们可以解析了程序头信息之后，通过type获取到.dynamic程序头信息，然后获取到这个segment的偏移地址和大小

![img](images/20160924152207910)

![img](images/20160924152457273)

4.根据函数的方法名，计算所对应的hash值，根据hash值，找到下标hash % nbuckets的

bucket；根据bucket中的值，读取.dynsym中的对应索引的Elf32_Sym符号；从符号的

st_name所以找到在.dynstr中对应的字符串与函数名进行比较。若不等，则根据

chain[hash % nbuckets]找下一个Elf32_Sym符号，直到找到或者chain终止为止 

5.找到函数方法后进行加密；



**三、实现篇**

Number01：首先我们自己写一个样本

样本很简答，就是在本地层写一个算法，**样本源码**会在后面给出链接大致上如下：

```
JNIEXPORT jstring JNICALL Java_com_example_zbb_test01_MainActivity_getStringFromNative
        (JNIEnv *env, jobject obj, jstring str)
{
   // jstring   CharTojstring(JNIEnv* env,   char* str);
    //首先将string类型的转化为char类型的字符串
    const char *strAry=(*env)->GetStringUTFChars(env,str,0);
    if(strAry==NULL){
        return NULL;
    }
    int len=strlen(strAry);
    char* last=(char*)malloc((len+1)* sizeof(char));
    memset(last,0,len+1);
    //char buf[]={'z','h','a','o','b','e','i','b','e','i'};
    char* buf ="beibei";
    int buf_len=strlen(buf);
    int i;
    for(i=0;i<len;i++){
        last[i]=strAry[i]|buf[i%buf_len];
        if(last[i]==0){
            last[i]=strAry[i];
        }
    }
    last[len]=0;
    return (*env)->NewStringUTF(env, last);
}
```

Number02:对形成的.so文件的重要函数名进行加密

接着对形成的.so文件中的"Java_com_example_zbb_test01_MainActivity_getStringFromNative"进行加密，当然这块的加密方法读者可以自己来定义，具体的看以下的代码：

```
private static void encodeFunc(byte[] fileByteArys){
		//寻找Dynamic段的偏移值和大小
		int dy_offset = 0,dy_size = 0;
		for(elf32_phdr phdr : type_32.phdrList){
			if(Utils.byte2Int(phdr.p_type) == ElfType32.PT_DYNAMIC){
				dy_offset = Utils.byte2Int(phdr.p_offset);
				dy_size = Utils.byte2Int(phdr.p_filesz);
			}
		}
		System.out.println("dy_size:"+dy_size);
		int dynSize = 8;
		int size = dy_size / dynSize;
		System.out.println("size:"+size);
		byte[] dest = new byte[dynSize];
		for(int i=0;i<size;i++){
			System.arraycopy(fileByteArys, i*dynSize + dy_offset, dest, 0, dynSize);
			type_32.dynList.add(parseDynamic(dest));
		}
		
		//type_32.printDynList();
		
		byte[] symbolStr = null;
		int strSize=0,strOffset=0;
		int symbolOffset = 0;
		int dynHashOffset = 0;
		int funcIndex = 0;
		int symbolSize = 16;
		
		for(elf32_dyn dyn : type_32.dynList){
			if(Utils.byte2Int(dyn.d_tag) == ElfType32.DT_HASH){
				dynHashOffset = Utils.byte2Int(dyn.d_ptr);
			}else if(Utils.byte2Int(dyn.d_tag) == ElfType32.DT_STRTAB){
				System.out.println("strtab:"+dyn);
				strOffset = Utils.byte2Int(dyn.d_ptr);
			}else if(Utils.byte2Int(dyn.d_tag) == ElfType32.DT_SYMTAB){
				System.out.println("systab:"+dyn);
				symbolOffset = Utils.byte2Int(dyn.d_ptr);
			}else if(Utils.byte2Int(dyn.d_tag) == ElfType32.DT_STRSZ){
				System.out.println("strsz:"+dyn);
				strSize = Utils.byte2Int(dyn.d_val);
			}
		}
		
		symbolStr = Utils.copyBytes(fileByteArys, strOffset, strSize);
		//打印所有的Symbol Name,注意用0来进行分割，C中的字符串都是用0做结尾的
		/*String[] strAry = new String(symbolStr).split(new String(new byte[]{0}));
		for(String str : strAry){
			System.out.println(str);
		}*/
		
		for(elf32_dyn dyn : type_32.dynList){
			if(Utils.byte2Int(dyn.d_tag) == ElfType32.DT_HASH){
				//这里的逻辑有点绕
				/**
				 * 根据hash值，找到下标hash % nbuckets的bucket；根据bucket中的值，读取.dynsym中的对应索引的Elf32_Sym符号；
				 * 从符号的st_name所以找到在.dynstr中对应的字符串与函数名进行比较。若不等，则根据chain[hash % nbuckets]找下一个Elf32_Sym符号，
				 * 直到找到或者chain终止为止。这里叙述得有些复杂，直接上代码。
					for(i = bucket[funHash % nbucket]; i != 0; i = chain[i]){
					  if(strcmp(dynstr + (funSym + i)->st_name, funcName) == 0){
					    flag = 0;
					    break;
					  }
					}
				 */
				int nbucket = Utils.byte2Int(Utils.copyBytes(fileByteArys, dynHashOffset, 4));
				int nchian = Utils.byte2Int(Utils.copyBytes(fileByteArys, dynHashOffset+4, 4));
				int hash = (int)elfhash(funcName.getBytes());
				hash = (hash % nbucket);
				//这里的8是读取nbucket和nchian的两个值
				funcIndex = Utils.byte2Int(Utils.copyBytes(fileByteArys, dynHashOffset+hash*4 + 8, 4));
				System.out.println("nbucket:"+nbucket+",hash:"+hash+",funcIndex:"+funcIndex+",chian:"+nchian);
				System.out.println("sym:"+Utils.bytes2HexString(Utils.int2Byte(symbolOffset)));
				System.out.println("hash:"+Utils.bytes2HexString(Utils.int2Byte(dynHashOffset)));
				
				byte[] des = new byte[symbolSize];
				System.arraycopy(fileByteArys, symbolOffset+funcIndex*symbolSize, des, 0, symbolSize);
				Elf32_Sym sym = parseSymbolTable(des);
				System.out.println("sym:"+sym);
				boolean isFindFunc = Utils.isEqualByteAry(symbolStr, Utils.byte2Int(sym.st_name), funcName);
				if(isFindFunc){
					System.out.println("find func....");
					return;
				}
				
				while(true){
					/**
					 *  lseek(fd, dyn_hash + 4 * (2 + nbucket + funIndex), SEEK_SET);
						if(read(fd, &funIndex, 4) != 4){
						  puts("Read funIndex failed\n");
						  goto _error;
						}
					 */
					//System.out.println("dyHash:"+Utils.bytes2HexString(Utils.int2Byte(dynHashOffset))+",nbucket:"+nbucket+",funIndex:"+funcIndex);
					funcIndex = Utils.byte2Int(Utils.copyBytes(fileByteArys, dynHashOffset+4*(2+nbucket+funcIndex), 4));
					System.out.println("funcIndex:"+funcIndex);
					
					System.arraycopy(fileByteArys, symbolOffset+funcIndex*symbolSize, des, 0, symbolSize);
					sym = parseSymbolTable(des);
					
					isFindFunc = Utils.isEqualByteAry(symbolStr, Utils.byte2Int(sym.st_name), funcName);
					if(isFindFunc){
						System.out.println("find func...");
						int funcSize = Utils.byte2Int(sym.st_size);
						int funcOffset = Utils.byte2Int(sym.st_value);
						System.out.println("size:"+funcSize+",funcOffset:"+funcOffset);
						//进行目标函数代码部分进行加密
						//这里需要注意的是从funcOffset-1的位置开始
						byte[] funcAry = Utils.copyBytes(fileByteArys, funcOffset-1, funcSize);
						for(int i=0;i<funcAry.length-1;i++){
							funcAry[i] = (byte)(funcAry[i] ^ 0xFF);
						}
						Utils.replaceByteAry(fileByteArys, funcOffset-1, funcAry);
						break;
					}
				}
				break;
			}
			
		}
		
	}
```

核心部分代码已经在上面介绍；

加密过后再IDA中表现为：

![img](images/20160924160145732)





Number03:接下来我们进行逆向开发解密文件的分析：

解密流程为加密逆过程，大体相同，只有一些细微的区别，具体如下：

1)  找到so文件在内存中的起始地址
2)  也是通过so文件头找到Phdr；从Phdr找到PT_DYNAMIC后，需取p_vaddr和p_filesz字段，并非p_offset，这里需要注意。
3)  后续操作就加密类似，就不赘述。对内存区域数据的解密，也需要注意读写权限问题。

```
#include <jni.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <elf.h>
#include <sys/mman.h>
#include <android/log.h>
//解密函数
 
void init_getStringFromNative() __attribute__((constructor));
unsigned long getLibAddr();
 
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
 
void init_getStringFromNative(){
  char name[15];
  unsigned int nblock;
  unsigned int nsize;
  unsigned long base;
  unsigned long text_addr;
  unsigned int i;
  Elf32_Ehdr *ehdr;
  Elf32_Shdr *shdr;
 
  base = getLibAddr();
 
  ehdr = (Elf32_Ehdr *)base;
  text_addr = ehdr->e_shoff + base;
 
  nblock = ehdr->e_entry >> 16;
  nsize = ehdr->e_entry & 0xffff;
 
  if(mprotect((void *) base, 4096 * nsize, PROT_READ | PROT_EXEC | PROT_WRITE) != 0){
	  __android_log_print(ANDROID_LOG_INFO, "JNITag", "mem privilege change failed");
  }
 
  for(i=0;i< nblock; i++){
    char *addr = (char*)(text_addr + i);
    *addr = ~(*addr);
  }
 
  if(mprotect((void *) base, 4096 * nsize, PROT_READ | PROT_EXEC) != 0){
	  __android_log_print(ANDROID_LOG_INFO, "JNITag", "mem privilege change failed");
  }
 
  clearcache((char*)text_addr, (char*)(text_addr + nblock -1));
 
  __android_log_print(ANDROID_LOG_INFO, "JNITag", "Decrypt success");
}
 
unsigned long getLibAddr(){
  unsigned long ret = 0;
  char name[] = "libegg.so";
  char buf[4096], *temp;
  int pid;
  FILE *fp;
  pid = getpid();
  sprintf(buf, "/proc/%d/maps", pid);
  fp = fopen(buf, "r");
  if(fp == NULL)
  {
    puts("open failed");
    goto _error;
  }
  while(fgets(buf, sizeof(buf), fp)){
    if(strstr(buf, name)){
      temp = strtok(buf, "-");
      ret = strtoul(temp, NULL, 16);
      break;
    }
  }
_error:
  fclose(fp);
  return ret;
}
```

Number04:然后修改smali层进行调用解密so文件：

![img](images/20160924162941627)

然后在AK中重打包，在手机上运行，结果是OK的，但是在模拟器上出问题了，具体的原因还需进一步分析，如果网友有知道的望告知。



四、总结篇

通过这两篇的有无源码的so文件中特定setion还是无源码特定函数的加密，在一定程度上都能够防一些静态分析，但是防不了动态分析，只要加密无论是多么复杂的加密方法，但是就一定在执行的时候以解密后的形式在内存中完整的出现，因此只要在IDA中找到开始和结束的libegg.so的起始地址，dump出来就可以，因此下一步就是反dump和反调试来防止动态分析，以后有机会进一步分析。

最后附件是所有相关的代码和文件如图所示：


![img](images/20160924165431444)

附件代码:[点击打开链接](http://download.csdn.net/detail/feibabeibei_beibei/9643012)