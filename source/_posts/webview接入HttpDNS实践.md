# webview接入HttpDNS实践
title: webview接入HttpDNS实践
tag: android
---

本文是对去年做的webview接入HttpDNS工作的一个总结，拖的时间有点久了。主要分享了GOT Hook webview中域名解析函数的方法。

## HttpDNS简介
首先简单介绍下移动App接入HttpDNS后有什么好处，这里直接引用腾讯云文档中的[说明](https://www.qcloud.com/document/product/379/3520):
>HttpDNS是通过将移动APP及桌面应用的默认域名解析方式，替换为通过Http协议进行域名解析，以规避由运营商Local DNS服务异常所导致的用户网络接入异常。减少用户跨网访问，降低用户解析域名时延。

更详细的内容可以参考这篇文章：[【鹅厂网事】全局精确流量调度新思路-HttpDNS服务详解](https://mp.weixin.qq.com/s?__biz=MzA3ODgyNzcwMw==&mid=201837080&idx=1&sn=b2a152b84df1c7dbd294ea66037cf262&scene=2&from=timeline&isappinstalled=0&utm_source=tuicool)

## 移动端的实现原理
域名的解析工作将在HttpDNS服务器上完成，客户端只要把待解析的域名作为参数发起一个HTTP请求，HttpDNS服务器就会把解析结果下发给客户端了。
在客户端，默认的域名解析是系统的`getaddrinfo()`库函数实现的，默认的域名解析请求会走到LocalDNS。
所以域名解析的工作必须要交给app应用层来实现。下面介绍几种实现方案。

### 1、okhttp
okhttp的实现是建立在socket之上的，并且实现了HTTP协议。对于客户端发起的http请求，okhttp首先会跟远端服务器建立socket连接，在此之前okhttp会根据http请求中url的domain做域名解析，默认的实现是java网络库提供的`InetAddress.getAllByName`。

如果项目中用的网络库是okhttp，所有的网络请求都是通过它完成的话就可以使用okhttp提供的DNS解析接口，实现自己的DNS resolver，代码如下：
```java
public class HttpDns implements Dns {
    @Override
    public List<InetAddress> lookup(String hostname) throws UnknownHostException {
	    //DNSHelper完成DNS解析的具体工作，向HttpDNS服务器请求服务。
        String ip = DNSHelper.getIpByHost(hostname);
        List<InetAddress> inetAddresses = Arrays.asList(InetAddress.getAllByName(ip));
        return inetAddresses;
    }
}
```

### 2、native hook的方法
通过Hook libc的`getaddrinfo`库函数，将函数指针指向app应用层实现的DNS解析函数地址。
要深入了解linux native hook的技术的话，需要了解ELF文件格式和动态链接的相关知识，可参考[ELF文件及android hook原理](http://blog.csdn.net/u012455213/article/details/53862637)。

`getaddrinfo`是在libc.so中的定义的，其它库如`libandroid_runtime.so`、`libjavacore.so`要使用这个函数的话，只能通过动态导入符号的形式，好在java网络库底层是就是通过这个方式实现的。

### android nativehook原理

下面代码是arm架构的一种实现方案，具体实现参考Andrey Petrov的[blog](http://shadowwhowalks.blogspot.com/2013/01/android-hacking-hooking-system.html). 

```cpp
#include "linker.h"  // get it from bionic
 
unsigned elfhash(const char *_name); //hash函数
//查找散列表。经典的链接法解决散列冲突

static Elf32_Sym *soinfo_elf_lookup(soinfo *si, unsigned hash, const char *name)  
 {  
   Elf32_Sym *s;  
   Elf32_Sym *symtab = si->symtab;  
   const char *strtab = si->strtab;  
   unsigned n;  
   n = hash % si->nbucket;  
   for(n = si->bucket[hash % si->nbucket]; n != 0; n = si->chain[n]){  
     s = symtab + n;  
     if(strcmp(strtab + s->st_name, name)) continue;  
       return s;  
     }  
   return NULL;  
 }  


 //soname:动态库名称;
 //symbol:待hook的函数名;
 //newval:新函数地址
 int hook_call(char *soname, char *symbol, unsigned newval) {  
  soinfo *si = NULL;  
  Elf32_Rel *rel = NULL;  
  Elf32_Sym *s = NULL;   
  unsigned int sym_offset = 0;  
  //打开动态库，得到soinfo对象
  si = (soinfo*) dlopen(soname, 0);  
 //通过查找散列表找到symbol对应符号表的索引
  s = soinfo_elf_lookup(si, elfhash(symbol), symbol);  
  sym_offset = s - si->symtab;  
  
  rel = si->plt_rel;//指向plt表的起始位置  
  //遍历plt表
  for (int i = 0; i < si->plt_rel_count; i++, rel++) {  
   unsigned type = ELF32_R_TYPE(rel->r_info);  
   unsigned sym = ELF32_R_SYM(rel->r_info);  
   //加上动态库的基址，定位到该符号重定向元素的内存
   unsigned reloc = (unsigned)(rel->r_offset + si->base);  
   uint32_t page_size = 0;
   uint32_t entry_page_start = 0;
   unsigned oldval = 0;  
   if (sym_offset == sym) {  //是否是待hook的符号位置
    switch(type) {  
      case R_ARM_JUMP_SLOT:
	     //修改内存页的属性为可读写  
         page_size = getpagesize();
         entry_page_start = reloc& (~(page_size - 1));
         int ret = mprotect((uint32_t *)entry_page_start, page_size, PROT_READ | PROT_WRITE); 
         
         oldval = *(unsigned*) reloc;  
         *((unsigned*)reloc) = newval;  //成功替换这块内存的值为新函数的地址值 
         return 1;  
      default:  
         return 0;  
    }  
   }  
  }  
  return 0;  
 } 

```
程序中调用mprotect的作用是： 修改一段指定内存区域的保护属性。以防万一，将这它改为可读写，因为后面就要对这块内存做写操作了。
[函数原型](https://linux.die.net/man/2/mprotect)为：`int mprotect(const void *start, size_t len, int prot); `
mprotect()函数把自start开始的、长度为len的内存区的保护属性修改为prot指定的值。
需要指出的是，指定的内存区间必须包含整个内存页（4K）。区间开始的地址start必须是一个内存页的起始地址，并且区间长度len必须是页大小的整数倍。

用法：
```
hook_call("libjavacore.so", "getaddrinfo", &my_getaddrinfo);  
```

- 1.调用dlopen拿到so的句柄,得到soinfo,它包含了符号表、重定位表、plt表等信息。
- 2.查找需要hook的函数的符号，得到它在符号表中的索引。
- 3.遍历plt表，直到匹配第2步中找到的符号索引。
 如果是JUMP_SLOT类型（函数调用），替换为新的符号地址（函数指针）。
如下图所示，`my_code_func`的函数地址替换了GOT表项中原来指向libc中的`getaddrinfo`函数地址,达到了hook的效果。
![](http://ww1.sinaimg.cn/large/8b331ee1gy1fknbjaa5c4j215m0q0jur.jpg)

跟进一步地，可以把设备上的`libjavacore.so`导出，用IDA Pro打开，观察`getaddrinfo`的引用关系，将有助于理解上面的代码。

找到`libjavacore.so`中`getaddrinfo`导入符号的位置：
![](http://ww1.sinaimg.cn/large/8b331ee1gy1fknbmf8o6oj20lc09o75g.jpg)
![](http://ww1.sinaimg.cn/large/8b331ee1gy1fknbmu8d3cj218i0203z9.jpg)

定位到`getaddrinfo`在plt表中引用的位置：
![](http://ww1.sinaimg.cn/large/8b331ee1gy1fknbn5ojkjj21ga03ygn7.jpg)

定位到`getaddrinfo`在GOT表中引用的位置：
![](http://ww1.sinaimg.cn/large/8b331ee1gy1fknbnfg0q4j215q02ota6.jpg)

定位到在代码段中调用`getaddrinfo`的位置：
![](http://ww1.sinaimg.cn/large/8b331ee1gy1fknbnwwei1j20yy044q44.jpg)

通过分析得知，虽然`getaddrinfo`是`libc.so`的导出函数，但是这种方法无法hook导出函数，没有一劳永逸的方法，只能hook导入函数，因为这种方案是通过修改GOT表项实现的，这是它的缺陷。

### 3、webview
webview作为H5的容器，在做网络请求的时候也需要做DNS域名解析，要对其接入HttpDNS的一般做法是通过拦截WebView的各类网络请求，截取URL请求的host，然后调用HttpDns解析该host，通过得到的ip组成新的URL来请求网络地址。
不过这种方式只能处理资源，处理正常的http/https请求会存在问题。
```java
public WebResourceResponse shouldInterceptRequest(WebView view, WebResourceRequest request) { 
	if (request.getMethod().equalsIgnoreCase("get")) { 
		String url = request.getUrl().toString(); 
		// HttpDns解析css文件的网络请求及图片请求 
		if (url.contains(".css") || url.endsWith(".png")) { 
		try { 
			URL oldUrl = new URL(url); 
			URLConnection connection = oldUrl.openConnection(); 
			// 获取HttpDns域名解析结果 
			String ips = MSDKDnsResolver.getInstance().getAddrByName(oldUrl.getHost()); 
			String newUrl = url.replaceFirst(oldUrl.getHost(), ip); 
			// 设置HTTP请求头Host域 
			connection = (HttpURLConnection) new URL(newUrl).openConnection(); 
			connection.setRequestProperty("Host", oldUrl.getHost()); 
			}
			return new WebResourceResponse("text/css", "UTF-8", connection.getInputStream()); 
		}
	}}
```
必须要对webview的DNS域名解析函数进行拦截替换。
webview的DNS域名解析函数具体实现是在`chromiumn.so`，不同版本的实现也不同，5.0版本的代码见[host_resolver.h](http://androidxref.com/5.0.0_r2/xref/external/chromium_org/ppapi/cpp/host_resolver.h)
webview的DNS域名解析函数是否也跟java的网络库一样最终调用的`libc.so`动态库中`getaddrinfo`呢？
通过源码注释得知确实如此。
![](http://ww1.sinaimg.cn/large/8b331ee1gy1fknbo84ymdj21aa08sq77.jpg)
[用Android Studio调试Framework层代码](http://blog.csdn.net/u012455213/article/details/54691340)中也对其进行过断点调试。
所以解决方法很简单，只需要hook `libchromium_net.so`中`getaddrinfo`导入符号即可。
```
hook_call("libchromium_net.so", "getaddrinfo", &my_getaddrinfo);  
```

## 机型问题
在实践中我们发现，不同机型不同版本的android在实现DNS解析函数的导出符号是不同的，更糟糕的是调用DNS解析函数的动态库也不一定就是`libjavacore.so`。
我之前定位过Android5.0设备的DNS解析函数，发现它的名字改为`android_getaddrinfofornet`。
![](http://ww1.sinaimg.cn/large/8b331ee1gy1fknbok6ktgj213y0f20tk.jpg)
webview的so库位置也曾遇到过找不到的问题。

解决方法是通过一个脚本，pull下测试设备上的所有so到主机上，然后用readelf工具查找so的导入符号，观察是否有`getaddrinfo`字样的导入符号。
为此我写了一个[脚本](https://github.com/FelixZhang00/so-finder)，方便自动化进行。运行如下命令即可
```sh
$ python sofinder.py -e getaddrinfo
```
![](http://ww1.sinaimg.cn/large/8b331ee1gy1fknbotls24j20wj0bw74u.jpg)
在上面输出的第一行可以看到，android 5.0以上版本webview的so已经被放在system/app目录中了。
需要写全so的路径：
```
hook_call("/system/app/WebViewGoogle/lib/arm/libwebviewchromium.so", "getaddrinfo", &my_getaddrinfo);  
```

---
## 参考

[【鹅厂网事】全局精确流量调度新思路-HttpDNS服务详解](https://mp.weixin.qq.com/s?__biz=MzA3ODgyNzcwMw==&mid=201837080&idx=1&sn=b2a152b84df1c7dbd294ea66037cf262&scene=2&from=timeline&isappinstalled=0&utm_source=tuicool)
[ELF文件及android hook原理](http://blog.csdn.net/u012455213/article/details/53862637)
[Andrey Petrov's blog](http://shadowwhowalks.blogspot.com/2013/01/android-hacking-hooking-system.html)