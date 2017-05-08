
title:  Stetho的通信原理
tag: android

---

## Stetho简介

[stetho](http://facebook.github.io/stetho/)是Facebook推出的安卓APP网络诊断和数据监控的工具，接入方便，功能强大，是Android开发者必备的友好工具。
主要功能包括：

- 实时查看App的布局
- 网络请求抓包
- 数据库、SharedPreferences文件内容监控
- 自定义dumpapp插件
- 对于JavaScript的支持

具体的使用方法可以看这篇[文章](https://www.figotan.org/2016/04/18/using-stetho-to-diagnose-data-on-android/)。
本文主要想讲一下自定义dumpapp插件的通信原理。

## dumpapp插件示例
在主机上给设备发送一个`files tree`命令，得到如下结果：
```bash
$ ./dumpapp files tree
+---lib
+---cache
|   +---com.android.opengl.shaders_cache
+---files
```
在app中对应这样一段java代码，来处理`files tree`命令。
```java
  private void doTree(PrintStream writer) throws DumpUsageException {
    File baseDir = getBaseDir(mContext);
    printDirectoryVisual(baseDir, 0, writer);
  }
```
问题是，为什么在主机上执行一段脚本([`dumpapp.py`](https://github.com/facebook/stetho/blob/master/scripts/dumpapp))后会让设备上的app执行相应的处理程序呢？

一般PushService可以完成类似的功能，后台下发一条指令，客户端完成指定的动作。对于Stetho这样的Android调试工具来说，显然不需要使用后台，用ADB就可以实现。

---

## ADB通信的原理

ADB的结构是一个client-server的结构，包含3个部分：

- Client ： 发送命令。客户端在PC主机上运行，在shell里使用Adb命令的时候就会开启一个client。
- Daemon : 在设备上执行命令。守护进程在设备上后台运行。(aabd运行在Andriod设备的底层)
- Server ： 管理客户端（client）和守护进程（daemon）的连接。server在PC主机上后台运行。
![](http://7viip0.com1.z0.glb.clouddn.com/17-3-12/10223878-file_1489286370152_c45a.jpg)


### smartsocket
android提供了smartsocket,详见[这里](https://android.googlesource.com/platform/system/core/+/master/adb/protocol.txt)
>--- smartsockets -------------------------------------------------------
Port 5037 is used for smart sockets which allow a client on the host
side to request access to a service in the host adb daemon or in the
remote (device) daemon.  The service is requested by ascii name,
preceeded by a 4 digit hex length.  Upon successful connection an
"OKAY" response is sent, otherwise a "FAIL" message is returned.  Once
connected the client is talking to that (remote or local) service.
client: <hex4> <service-name>
server: "OKAY"
client: <hex4> <service-name>
server: "FAIL" <hex4> <reason>

总结来说，就是可以给adb-server发送一条指令`<service-name>`，然后adb-server会转发给adbd，让adbd来执行`<service-name>`.
举例来说，当我们执行`adb shell cat /proc/net/unix`,最终就是通过adbd在设备上执行的。

Stetho的通信模型如下：
![](http://7viip0.com1.z0.glb.clouddn.com/17-3-12/75166063-file_1489297004530_117bc.png)
其中stetho-server就是app启的一个Thread用来accept客户端的connect。

---

## 程序流程
可以通过在关键位置打上断点的方式来看程序的流程。
Python可以用[Pycharm](https://www.jetbrains.com/pycharm/)来打断点。
如图配置一个debug版本,这样就可以以`./dumapp -l`的方式debug了。
![](http://7viip0.com1.z0.glb.clouddn.com/17-3-12/84731891-file_1489298066962_2a09.jpg)
Android app当然是用Android Studio打断点了。

### dumpapp.py流程分析
详见代码([`dumpapp.py`](https://github.com/facebook/stetho/blob/master/scripts/dumpapp))
![](http://7viip0.com1.z0.glb.clouddn.com/17-3-12/32838177-file_1489290757916_15931.png)

例子1：
`adb.select_service('shell:cat /proc/net/unix')`
![](http://7viip0.com1.z0.glb.clouddn.com/17-3-12/77046872-file_1489287023930_14c4e.png)
通过这个命令其实是在找到指定的Unix域套接字。
![](http://7viip0.com1.z0.glb.clouddn.com/17-3-12/74443269-file_1489287115563_17cdf.png)
在`/proc/net/unix`文件下可以看到所有的unix域套接字，Path字段前面有@符号的表示它是一个ABSTRACT类型的socket，如果是绝对路径则表示是FILESYSTEM类型的。
![](http://7viip0.com1.z0.glb.clouddn.com/17-3-12/30789563-file_1489287608725_1bbf.jpg)
例子2：
发起一个connect到Unix域套接字的请求
`adb.select_service('localabstract:%s' % (socket_name))`
![](http://7viip0.com1.z0.glb.clouddn.com/17-3-12/3363418-file_1489287410599_b640.png)

这里的python用到的几个service协议应该是android提供的smartsocket本身就支持的，在与adb的端口号连接后就能使用socket来发送service的名字给android设备了。
详见[这里](https://developer.chrome.com/devtools/docs/remote-debugging-legacy).
如下的命令就可以直接跟stetho-server连接。
![](http://7viip0.com1.z0.glb.clouddn.com/17-3-12/10703667-file_1489287758769_63f.png)

### stetho-server流程分析
详见代码[LocalSocketServer.java](https://github.com/facebook/stetho/blob/36aa5bd356d9cf5893b9424b06a83dda9ec5e44f/stetho/src/main/java/com/facebook/stetho/server/LocalSocketServer.java)
![](http://7viip0.com1.z0.glb.clouddn.com/17-3-12/85652381-file_1489291638319_11cf3.png)
这里创建ServerSocket时的address格式是`stetho_+进程名+_ devtools_remote `

---
## Unix域套接字

socket API原本是为网络通讯设计的，但后来在socket的框架上发展出一种IPC机制，就是UNIX Domain Socket（Unix域协议）。虽然网络socket也可用于同一台主机的进程间通讯（通过loopback地址127.0.0.1），但是UNIX Domain Socket用于IPC更有效率：不需要经过网络协议栈，不需要打包拆包、计算校验和、维护序号和应答等，只是将应用层数据从一个进程拷贝到另一个进程。这是因为，IPC机制本质上是可靠的通讯，而网络协议是为不可靠的通讯设计的。UNIX Domain Socket也提供面向流和面向数据包两种API接口，类似于TCP和UDP，但是面向消息的UNIX Domain Socket也是可靠的，消息既不会丢失也不会顺序错乱。

Unix域协议所用的API就是在不同主机上执行客户/服务通信所用的套接字API。


### Android中的Unix域套接字

在Android API中，有几个类对Unix域套接字（也叫localsocket）进行了封装，不仅可以用来应用程序之间进行IPC通信，还可以跨应用程序层和Linux层运行的程序进行通信。
`LocalSocket`在Unix域名空间创建一个套接字（非服务端）。
`LocalSocketImpl`是Framework层Socket的实现，通过JNI调用系统socket API。
`LocalServerSocket`创建服务器端Unix域套接字，与LocalSocket对应。
![](http://7viip0.com1.z0.glb.clouddn.com/17-3-12/89412003-file_1489294912573_1445f.png)

创建socket时指定的domain类型是AF_UNIX。
```java
/**
 * Creates a socket in the underlying OS.
 */
public void create (int sockType) throws IOException {
    //...
    fd = Os.socket(OsConstants.AF_UNIX, osType, 0);    
}
```

通过搜索发现LocalSocketImpl的native实现是在libandroid_runtime.so中。
![](http://7viip0.com1.z0.glb.clouddn.com/17-3-12/62841029-file_1489288277901_16a7e.png)
比如listen的native实现就是调用了socket的listen函数。
```cpp
/* private native void listen_native(int fd, int backlog) throws IOException; */
static void
socket_listen (JNIEnv *env, jobject object, jobject fileDescriptor, jint backlog)
{
    int ret,fd;
    fd = jniGetFDFromFileDescriptor(env, fileDescriptor);
    ret = listen(fd, backlog);
    if (ret < 0) {
        jniThrowIOException(env, errno);
        return;
    }
}
```

---

## 参考
[ADB原理，Wi-Fi连接，常用命令及拓展](http://www.jianshu.com/p/f82b733bd6ac)
《UNIX网络编程卷1》
[Android LocalSocket与Socket 区别](http://blog.csdn.net/shuzui1985/article/details/50178929)
[如何给安卓APP安装听诊器,检查数据问题](https://www.figotan.org/2016/04/18/using-stetho-to-diagnose-data-on-android/)