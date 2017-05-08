
---
title: 理解mmap
tag: Linux

---

在接入日志组件xlog的工作中，对mmap内存映射加深了了解，分享一下学习心得。


## 1.一个Linux进程的虚拟内存
![](http://7viip0.com1.z0.glb.clouddn.com/17-2-25/50851685-file_1488011866599_1378c.jpg)
如图展示了一个Linux进程的虚拟内存。
虚拟的意思是进程以为自己有这么一大块内存，实际上物理内存可能还没有分配给它，等到缺页异常是系统才会分配，通过这种以时间换空间的方式提高了内存利用效率。从虚拟内存到物理内存的映射过程需要一个专门的硬件单元MMU来完成。
系统调用的代码和数据就在内核虚拟内存中，
因为在保护模式下，用户态进程无法访问到这里，必须要通过系统调用的方式陷入到内核态才行。

## 2.Linux是如何组织虚拟内存的
![](http://7viip0.com1.z0.glb.clouddn.com/17-2-25/44198324-file_1488012277201_ee2a.png)
内核为系统中的每个进程维护一个单独的任务结构`task_struct`，其中元素包含了内核运行该进程所需要的所有信息（PID、指向用户栈的指针、可执行目标文件的名字、虚拟内存状态、pc指针等）
`task_struct`中的`mm_struct`描述了虚拟内存的当前状态，其中mmap字段指向一个`vm_area_struct`（区域结构）的链表。顺序搜索区域结构的链表花销会很大，实际上Linux在链表中构建了一个树，并在这棵树中进行查找。
进程对某一虚拟内存区域的任何操作需要用要的信息，都可以从`vm_area_struct`中获得。mmap函数就是要创建一个新的`vm_area_struct`结构，并将其与文件的物理磁盘地址相连。

## 3.缺页处理
![](http://7viip0.com1.z0.glb.clouddn.com/17-2-25/30649400-file_1488012949934_13c4.jpg)
当MMU在试图翻译某个虚拟地址A时，触发了一个缺页。缺页异常处理程序会做如下检查：
- 1）虚拟地址A是否合法？即是否在链表`mm_struct`所描述的区域内。
- 2）试图进行的内存访问是否合法？即检查指令的权限是否与vm_prot字段所描述的页读写许可权限相匹配。
- 3）正常缺页。系统会负责把该虚拟内存区域对应的文件加载到内存中。

## 4.内存映射
Linux通过将一个虚拟内存区域与一个磁盘上的对象关联起来，以初始化这个虚拟内存区域的内容，这个过程称为内存映射。
Linux进程可以使用mmap函数来创建新的虚拟内存区域，并将对象映射到这些区域中。
mmap函数定义在libc中：
```cpp
 #include <sys/mman.h>

     void *
     mmap(void *start, size_t length, int prot, int flags, int fd, off_t offset);
```
具体内容可以通过命令`man 2 mmap`查看。
mmap参数的可视化解释：
![](http://7viip0.com1.z0.glb.clouddn.com/17-2-25/21318677-file_1488018309700_943f.png)

### mmap原理
在调用mmap实现这样的映射关系后，它只是在进程的虚拟空间中分配了一段空间，真实的物理地址还不会分配的，当进程第一次访问这段空间（当作内存一样），CPU陷入OS内核执行异常处理，然后异常处理会在这个时间分配物理内存，并用文件的内容填充这片内存，然后才返回进程的上下文，这时进程才会感知到这片内存里有数据。
之后进程即可对这片主存进行读或者写的操作，如果写操作改变了其内容，一定时间后系统会自动回写脏页面到对应磁盘地址，也即完成了写入到文件的过程。

mmap 的回写时机：
* 内存不足
* 进程退出
* 调用 msync 或者 munmap
* 不设置 MAP_NOSYNC 情况下 30s-60s(仅限FreeBSD)

### 程序的加载
Linux执行一个ELF格式的程序，这个程序在磁盘上，为了执行这个程序，需要把程序加载到内存中，这时采用的就是mmap，mmap让虚拟空间和文件的内容组成的空间（文件空间）对应。因为ELF格式是区分代码、数据段的，这里的就不是简单的整个文件的映射了，需要将文件的分段区域映射到内存的不同位置。OS加载ELF文件的过程非常复杂这里就不展开了，具体内容可以看《程序员的自我修养》。
当CPU真的在这个地址上发起读写执行等操作时，因为文件的内容在磁盘上是不能被CPU访问的，所以OS会进入异常，系统的缺页处理程序会调用文件系统把一页或者多页的文件内容加载到物理内存中。

可以通过 `cat /proc/<pid>/maps`看到某个进程的mmap状态，其实就是通过遍历`vm_area_struct`链表得到的，有关maps的解释可以看[这里](http://askubuntu.com/questions/93509/how-to-interpret-proc-pid-maps-for-pidgin-application)。
下面是使用xlog的Android程序进程的内存状态（截取一小部分）：

```python
shell@shamu:# cat /proc/9032/maps
address           perms offset  dev   inode   pathname
//...
0804d000-0806e000 rwxp 0804d000 00:00 0          [heap]
b7e88000-b7e89000 rwxp b7e88000 00:00 0 
x-x r--p 00000000 fe:00 791721     /data/dalvik-cache/arm/data@app@com.x-1@base.apk@classes.dex
x-x r-xp 00a6f000 fe:00 791721     /data/dalvik-cache/arm/data@app@com.x-1@base.apk@classes.dex
x-x rw-p 0155d000 fe:00 791721     /data/dalvik-cache/arm/data@app@com.x-1@base.apk@classes.dex
x-x r--p 00000000 103:09 630       /system/fonts/CarroisGothicSC-Regular.ttf
b35b7000-b35dd000 rw-s 00000000 00:14 3082       /storage/emulated/0/log.mmap2
b6fb6000-b6fc3000 r-xp 00000000 103:09 206       /system/bin/linker
b6fc3000-b6fc4000 r-xp 00000000 00:00 0          [sigpage]
b6fc4000-b6fc5000 r--p 0000d000 103:09 206       /system/bin/linker
b6fc5000-b6fc6000 rw-p 0000e000 103:09 206       /system/bin/linker
b6fc6000-b6fc7000 rw-p 00000000 00:00 0 
b6fc7000-b6fca000 r-xp 00000000 103:09 136       /system/bin/app_process32
b6fca000-b6fcb000 r--p 00002000 103:09 136       /system/bin/app_process32
be246000-bea45000 rw-p 00000000 00:00 0          [stack]
ffff0000-ffff1000 r-xp 00000000 00:00 0          [vectors]
```
这些分段空间后面的那些，就是每个虚拟空间分段对应的文件。这些文件，称为这片虚拟空间的backlog文件，它的作用是当这些内存需要被使用的时候，从磁盘中把对应的文件内容加载到物理内存中。
这里同一个文件`/system/bin/linker`在虚拟内存中有不同的内存映射区域，就是因为其文件中有不同的分段，从`offset`可以看出来。
`/storage/emulated/0/log.mmap2`就是xlog用作mmap的backlog文件了，它被映射到`b35b7000-b35dd000`这段内存区域。


## 5.为什么mmap()可以节约IO读写时间
常规文件操作为了提高读写效率和保护磁盘，使用了页缓存机制，这是由OS控制的。这样造成读文件时需要先将文件页从磁盘拷贝到页缓存中，由于页缓存处在内核空间，不能被用户进程直接寻址，所以还需要将页缓存中数据页再次拷贝到内存对应的用户空间中。这样，通过了两次数据拷贝过程，才能完成进程对文件内容的获取任务。写操作也是一样，待写入的buffer在内核空间不能直接访问，必须要先拷贝至内核空间对应的主存，再写回磁盘中（延迟写回），也是需要两次数据拷贝。

而使用mmap操作文件中，由于不需要经过内核空间的数据缓存，只使用一次数据拷贝，就从磁盘中将数据传入内存的用户空间中，供进程使用。
mmap的关键点是实现了用户空间和内核空间的数据直接交互而省去了空间不同数据不通的繁琐过程。因此mmap效率更高。

### xlog对mmap的效率做了验证
为了验证 mmap 是否真的有直接写内存的效率，通过一个简单的测试用例进行验证：把512 Byte的数据分别写入150 kb大小的内存和 mmap，以及磁盘文件100w次并统计耗时
![](http://7viip0.com1.z0.glb.clouddn.com/17-2-25/57475458-file_1488014455835_4ad2.jpg)
从上图看出mmap几乎和直接写内存一样的性能，而且 mmap 既不会丢日志，回写时机又基本可控。 


---
## 参考
- 《深入理解计算机操作系统》
- 《程序员的自我修养》
-  [认真分析mmap：是什么 为什么 怎么用 ](http://www.cnblogs.com/huxiao-tee/p/4660352.html)
- [微信mars 的高性能日志模块 xlog](http://dev.qq.com/topic/581c2c46bef1702a2db3ae53)
- [Linux 中 mmap() 函数的内存映射问题理解](https://www.zhihu.com/question/48161206/answer/110418693)
- [proc(5) - Linux man page](https://linux.die.net/man/5/proc)
- [How to interpret /proc/pid/maps](http://askubuntu.com/questions/93509/how-to-interpret-proc-pid-maps-for-pidgin-application)