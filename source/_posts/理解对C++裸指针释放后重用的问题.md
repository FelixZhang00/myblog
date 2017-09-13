# 理解对C++裸指针释放后重用的问题

title:  理解对C++裸指针释放后重用的问题
tag: c++ 安全

---

本文将以Android 2.2-2.3上的一个zergRush漏洞为例，分析指针释放后重用的问题。

[zergRush](https://nvd.nist.gov/vuln/detail/CVE-2011-3874)是Android 2.2-2.3上的一个漏洞，主要问题就在于指针的释放后重用。
zergRush利用了libsysutils库提供的Framework套接字的通用接口。
程序从套接字收到的消息中出抽取出的文本命令会导致栈缓冲区溢出，进而造成释放后重用问题。
具体地，是vold后台程序调用了libsysutils.so，bug出在FrameworkListener.cpp的dispatchCommand方法。

---
## 什么是释放后重用
释放后重用(Use After Free)问题是指，程序使用指针访问了一个已经通过free函数或者delete操作符释放过的对象，并且这个指针没有置空，攻击者在这块释放后的内存中写入了恶意的数据shellcode，当程序第2次使用这个指针的时候，控制流就转向了攻击者构造的恶意数据中了。

## FrameworkListener中的bug
FrameworkListener.cpp中有bug的关键代码如下：
```cpp
//参数cli为与用户进程连接的socket链接；参数data为用户进程的命令参数
void FrameworkListener::dispatchCommand(SocketClient *cli, char *data){
	FrameworkCommandCollection::iterator i;
    int argc = 0;
    //在栈上临时分配的局部缓冲区，用来存放从socket中解析命令参数指针
    char *argv[16];
    //栈上分配的缓冲区，存放从socket中解析命令参数数据
    char tmp[255];
    char *p = data; //p指向用户数据
    char *q = tmp; //q指向tmp数组
	//...	
	//下面的循环遍历输入中的所有字符，直到遇到一个结尾\0
	while(*p) {
		//...
		//将用户输入复制到缓冲区，参数放入tmp数组，但是没有检查边界
		*q = *p++;
		//如果引用的字符串外面还有一个空格，则将q重置到tmp的起始位置
        if (!quote && *q == ' ') {
            *q = '\0';
            //strdup会在堆上分配空间，返回这块堆内存的指针
            argv[argc++] = strdup(tmp);
            memset(tmp, 0, sizeof(tmp));
            q = tmp;
            continue;
        }
        q++;
	}
	argv[argc++] = strdup(tmp);
    
    for (i = mCommands->begin(); i != mCommands->end(); ++i) {
        FrameworkCommand *c = *i;
        if (!strcmp(argv[0], c->getCommand())) {
		    //调用FrameworkCommand的虚函数
            if (c->runCommand(cli, argc, argv)) {
            }
        }
    }

    int j;
    for (j = 0; j < argc; j++){
	   //因为是strdup动态分配出来的，所以需要主动释放
       free(argv[j]);
    }
    return;
}
```
下图是第1次调用`dispatchCommand`的内存布局：
![](https://ws1.sinaimg.cn/large/8b331ee1gy1fhq0hg4cbtj20x60w2424.jpg)

假设其中一个FrameworkCommand对象所在的内存地址是0x12345678，这个地址值，用户进程可以在参数中以字符串的形式提供，即`\x78\x56\x34\x12`，这里要考虑到字节序，内存低地址将存放小端的字节。

假设参数data的数据为`“cmd p1 p2 p3 p4 p5 p6 p7 p8 p9 p10 p11 p12 p13 p14 p15 p16 \x78\x56\x34\x12”`。
前15个参数的处理过程中，argv数组中的元素都是正常的从strdup返回的指向堆的指针值，即指向参数字符串的指针。
当p指针指向p16这个参数值，argv[16]=strdup("p16")，这时argv[16]已经超出了argv数组的范围，此时`&argv[16]=&tmp[0]`，这个参数值将覆盖tmp数组的头4字节。之后tmp清空，q指针重新指向tmp数组的开头，继续读入最后一个参数。

继续调用`*q = *p++`,此时tmp开头4字节即为`\x78\x56\x34\x12`,同时也是argv[16]元素的值，注意到这个值有别于argv数组中其它的元素的值，其它元素的值都是strdup动态分配返回的堆指针，而argv[16]是攻击者恶意构造的地址值。

此时argv[16]的头4字节，也就是tmp头4字节的数据是0x78,0x56,0x34,0x12，
free(argv[16])调用的是free(0x12345678),即释放掉了FrameworkCommand所在内存，即这块内存被内存分配器添加到类似freelist这样的数据结构中，供下一次动态分配使用。

这里需要说明下strdup这个函数。`char* strdup(const char *s1)`函数会为s1指针指向的字符串数据分配等大小的内存，并返回指向这块内存的指针。因为是动态分配的，这块内存在堆上，实际使用Android系统中Bionic lib库内置的dlmalloc分配器来动态分配的。dlmalloc分配器在某些情况下内存被free后不会马上释放回内核，而是保留给应用程序重新申请。

下图是第2次调用`dispatchCommand`的内存布局：
![](https://ws1.sinaimg.cn/large/8b331ee1gy1fhq0aprp4zj219m0wadko.jpg)
当用户进程第2次调用`dispatchCommand`，走到`argv[0] = strdup(tmp)`处时，strdup分配的内存就是上次释放掉的FrameworkCommand所在内存，并把tmp的字节数据拷贝到这块内存中。这时可以构造恶意数据覆盖vtable指针，让它指向shellcode的内存地址，这样当函数主动调用runCommand时，控制流就会跑到shellcode中了。比如第二次传给dispatchCommand的命令是"AAAA param"，vtable指针会被覆盖成0x41414141，EIP将被指向 `[0x41414141+runCommand虚函数在虚表中的偏移]`。剩下的问题就是如何巧妙的构造shellcode和放在哪块内存区域了。

## 修复方法
补丁[libsysutils: Fix potential overwrites in FrameworkListener](https://cloud.seu.edu.cn/gitlab/frederickjoe/android-aosp-sdcard/commit/c6b0def5f039dc3bbe1d4b7dc1666c24316eb020)
给出了一个修复方法,增加了数组越界检查。
```cpp
+            if (argc >= CMD_ARGS_MAX)
+                goto overflow;
```

---
## 参考
[zergRush (CVE-2011-3874) 提权漏洞分析](http://www.cnblogs.com/daishuo/p/4002963.html)
[从zergRush深入理解Use After Free](http://huntcve.github.io/2015/06/14/uaf/)
《Android安全攻防权威指南》