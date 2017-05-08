
---
title: 用Android Studio调试Framework层代码
tag: Android DEBUG

---

Android程序员不得不知的调试技巧。
本文以webview loadUrl和域名解析为例,介绍配合使用LLDB和Android Studio调试Framework代码的技巧。

## java 层调试

首先需要把AOSP源码导入到Android Studio中，如果是macOS系统可以参考[这篇文章](http://blog.csdn.net/u012455213/article/details/54647010)。
导入后如下图所示：
![](http://7viip0.com1.z0.glb.clouddn.com/17-1-23/37996598-file_1485133690721_e86b.png)

### 调试原理
Java平台的调试是有一个规范化的标准的，那就是JPDA（Java Platform Debugger Architecture）；通过 JPDA 提供的 API，开发人员可以方便灵活的搭建 Java 调试应用程序。 JPDA 主要由三个部分组成：Java 虚拟机工具接口（JVMTI），Java 调试线协议（JDWP），以及 Java 调试接口（JDI）。
调试需要堆栈、符号等信息都保存在JVM中，调试器（debugger）需要通过一种渠道获取这些信息，并通过这个渠道发送调试指令给JVM，JDWP就是调试器与JVM通信的渠道。在JVM内部有一个专门的jdwp线程，Android系统的adbd守护进程通过socket与各个虚拟机的jdwp线程进行通信，外部调试器通过主机的adb与adbd通信进而完成与jdwp的通信。具体过程如下图：
![调试架构图](http://7viip0.com1.z0.glb.clouddn.com/17-1-23/20297449-file_1485133715439_160b3.png)

### 配置Debug选项
在菜单栏上依次点击Run -> Edit Configurations -> Remote，打开并配置成如下的页面
![aosp_java_debug](http://7viip0.com1.z0.glb.clouddn.com/17-1-23/32613201-file_1485133728789_eace.png)

### Exclued 不必要的文件夹
在断点调试时，JVM会告诉AS自己在xx.java的第xx行被断住了，AS就会定位到这个位置，但是如果有重复的文件的名的，往往会出现定位不准的情况，所以需要把不必要的文件夹排除在整个源码结构之外。打开Project Structure,做如下修改
![Exclued](http://7viip0.com1.z0.glb.clouddn.com/17-1-23/98071738-file_1485133747899_256c.png)
如果遇上断点文件对不上的情况时，就手动在这里Exclued好了。
也可以直接修改`aosp-root/development/tools/idegen/excluded-paths`文件中的内容，添加exclude，再运行`idegen.sh` 重新生成IDE代码树。


### 在源码处打断点
我们在WebView.java的loadUrl处打断点
![](http://7viip0.com1.z0.glb.clouddn.com/17-1-23/38678927-file_1485133770804_2652.png)
点击调试按钮，你会看到Console中的提示
```bash
Connected to the target VM, address: 'localhost:8700' , transport: 'socket' 
```

### 打开DDMS
在菜单栏上依次点击Tools ->Android -> Android Device Monitor，打开DDMS后,点击
![ddms](http://7viip0.com1.z0.glb.clouddn.com/17-1-23/59482474-file_1485133844742_d58b.png)

在monitor中我们可以看到有3列，分别是

* 进程名(以包名显示)
* PID(Process ID)
* 端口号(映射端口号/实际端口号)
点击我们要调试的browser程序的那一行，会出现一只绿色的bug，表示我们的Debugger已经跟设备上的程序联系上，可以调试了。

### 开始调试
当在浏览器中加载一个网页时，就能触发之前设置的loadUrl的断点了，如此就可以使用各种调试手段了。
![loadUrl堆栈](http://7viip0.com1.z0.glb.clouddn.com/17-1-23/60962148-file_1485133869972_a8b8.png)

---
## C++层调试
Android Framework中native代码的调试方法采用的是 2.2以上版本的Android Studio配合LLDB调试器。
这里以调试webview的dns查找过程为例，说明native调试的方法。

### 调试原理
LLDB作为Android Native层的调试工具，其原理跟gdb一样，也是采用C/S架构，通过push一个lldb-server到设备上，pc机的debugger作为lldb-client与其通信，以达到调试的效果。
C++在编译时有一个选项`-g`表示编译出来的可执行文件是带有调试信息的，比如源文件、行号信息，都会存放在ELF文件中的
`.debug_* `段之中， 知道了这些调试信息后，调试器配合IDE就可以定位代码了。
这里还需要保证你的符号文件和设备上真正运行的动态链接库或者可执行文件是对应的，就是同一份，不然调试信息就对不上了。
最简单的办法就是使用模拟器。我们编译完源码之后，一个主要的编译产物就是 system.img，这个 system.img会在启动之后挂载到设备的 /system 分区，而system分区包含了Android系统运行时的绝大部分可执行文件和动态链接库，而这些文件就是我们的编译输出，正好可以与编译得到的调试符号进行配合调试。模拟器有一个 -system选项用来指定模拟器使用的 system.img文件。
```bash
$ emulator -avd Nexus5-API22 -verbose -no-boot-anim -system (the path of system.img)
```
我这里的做法是使用烧录了自己编译源码的Nexus手机。

### 配置Debugger
这里需要新建一个Android Demo工程了，直接用AOSP源码那个工程，没有是Native Debug那个选项的。
按如下方式配置符号表，需要与设备上用的so是同一份。并且改Debug type 为Native。
![配置Debugger](http://7viip0.com1.z0.glb.clouddn.com/17-1-23/98365902-file_1485133891159_d5e1.png)
符号表的添加也可以通过lldb命令行的方式添加
![lldb-pause](http://7viip0.com1.z0.glb.clouddn.com/17-1-23/11135935-file_1485133911409_87f8.png)
![lldb-add-dsym](http://7viip0.com1.z0.glb.clouddn.com/17-1-23/81500034-file_1485133908937_c39f.png)
LLDB需要这些符号信息才能帮你定位到调试断点的代码。

### 配置源码到AS
当LLDB告诉AS源文件行号信息时，AS需要定位到对应的代码处，所以必须先把源文件导入到AS中，最简单的做法是建立软链接。在Android Demo工程下建立一个source文件夹，然后执行如下命令。
```bash
$ ln -s xx/external/chromium_org  xx/source/chromium_org
$ ln -s xx/bionic/libc  xx/source/libc
```
这里只是把需要用到的源文件导入进来，当然也可以把整个AOSP源码导入AS中，但是这样会比较耗时。

### 打断点
我在getaddrinfo.c的getaddrinfo方法处打一个断点，看看webview在加载网页时的域名解析会不会走到这里。
![getaddrinfo](http://7viip0.com1.z0.glb.clouddn.com/17-1-23/25829366-file_1485133943240_17e29.png)
点击Debug按钮，当Demo程序开始LoadUrl之后，就会被Debug断住，如下是chromium域名解析线程的堆栈（这里的方法名真够长的。。。），这样我们就可以进一步了解webview加载网页时域名解析的过程了。
![getaddrinfo-stack](http://7viip0.com1.z0.glb.clouddn.com/17-1-23/58831938-file_1485134083110_4efc.png)
让我们看看其他线程在干啥，整个世界都停止了。
![chromium-threads](http://7viip0.com1.z0.glb.clouddn.com/17-1-23/98647443-file_1485134070646_62ea.png)
![renderthread](http://7viip0.com1.z0.glb.clouddn.com/17-1-23/78278941-file_1485133978101_8702.png)
![jdwp](http://7viip0.com1.z0.glb.clouddn.com/17-1-23/9092603-file_1485134218822_d5f1.png)

--- 
## 不足之处
目前的调试framework方案只能把java层和c++ native层的分开来做，还不能做到java层和c++ native层互相跳转的效果。虽然目前我们开发Android App用AS调试时能做大这一点，要是framework的调试也能做到这一点就好了。获取真有这样的方法，如果有知道的大神，还请赐教。

---
## 相关链接

[Debugging AOSP Platform code with Android Studio - Part I - Java Debugger](http://ronubo.blogspot.com/2016/01/debugging-aosp-platform-code-with.html)
[Android Debugging: Old School bringup routines - Command line Java debugging with JDWP](http://ronubo.blogspot.com/2016/01/android-debugging-old-school-bringup.html)
[如何调试Android Framework](http://weishu.me/2016/05/30/how-to-debug-android-framework/index.html)
[如何调试Android Native Framework](http://weishu.me/2017/01/14/how-to-debug-android-native-framework-source/index.html)
[在macOS 10.12 上编译 Android 5.1](http://blog.csdn.net/u012455213/article/details/54647010)