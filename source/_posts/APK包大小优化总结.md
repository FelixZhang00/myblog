# APK包大小优化总结
title: APK包大小优化总结
tag: android
---

## 导言

随着业务快速增长，我们面临apk包体积不断增长的问题，在4.2版本包体积达到了历史最高点37.16MB，远远大于各竞品。包体积过大会影响到下载转化率、升级成功率，从线上数据看，apk升级失败的错误码主要集中在下载空间不足、socket异常和md5校验失败， 主要是包体积过大直接导致的。
为此我们成立了包大小优化项目，经过几个迭代的优化走后，在5.0版本包体积减少到15.16MB，领先竞品，同时apk升级失败率也大幅降低。


---
## 优化方案

### native优化

1、使用共享C++ 运行时库
如果使用静态STL将会在每个 so 库中出现重复代码，增加应用大小。通过下面的配置可以改为使用共享C++运行时库。
```
# for cmake
externalNativeBuild {
    cmake {
        arguments "-DANDROID_STL=c++_shared"
  }
}

# 解决ibc++_shared.so 冲突问题
packagingOptions {
    pickFirst 'lib/*/libc++_shared.so'
}
```
```
#for ndk-build
APP_STL := c++_shared
```
通过[Matrix-ApkChecker](https://github.com/Tencent/matrix/tree/master/matrix/matrix-android/matrix-apk-canary)可以检测出使用静态STL的so。
![2401578127039_.pic.jpg](http://ww1.sinaimg.cn/large/8b331ee1gy1gakm9lfy8gj214e0fgdia.jpg)

2、精简cocos代码
项目早期是基于cocos2d-x框架开发的，现已全面完成java化，C层的业务代码大部分可以删去。精简之后，apk包大小减少了约5MB。

3、开启编译优化

如果使用 GCC 可以 -Os 打开优化，如果使用 Clang 可以 -Oz 打开优化。

---

### 资源优化

#### 删除无用资源
随着业务快速迭代，产生了大量资源，由于初期对包大小不够重视，堆积了很多没有用到的资源，亟需检测出这部分无用资源并批量删除。

Android Lint可以检测无用资源，关于lint更详细的介绍可以看这篇[文章](https://cloud.tencent.com/developer/article/1014614)。
![WeChat4bb3d7df2a8adac6f2bb2c3e98dd676f.png](http://ww1.sinaimg.cn/large/8b331ee1gy1g97p3x72d7j21c80rwnpd.jpg)

Lint 作为一个静态扫描工具，它最大的问题在于没有考虑到 ProGuard 的代码裁剪。在 ProGuard 过程我们会 shrink 掉大量的无用代码，但是 Lint 工具并不能检查出这些无用代码所引用的无用资源。lint也不支持配置资源白名单，防止误删。
正好[Matrix-ApkChecker](https://github.com/Tencent/matrix/tree/master/matrix/matrix-android/matrix-apk-canary)就具备这个功能。Matrix-ApkChecker还支持检测asset目录下的无用资源。

Matrix-ApkChecker搜索无用资源的原理:

>  1. 找出apk中声明的资源：通过R.txt 文件（资源名与资源id的映射关系）中读取apk中包含的所有资源；
>  2. 找出资源的引用：针对代码中对资源的引用，反编译dex成smali文件，搜索资源id常量；针对资源文件中对其他资源的引用，编译 xml 资源文件 和 AndroidManifest.xml，然后搜索 xml 文件中的每个节点，查看 attribute 和 text
> 中是否引用到其他资源

通过`getIdentifier`去访问的资源需要配置到白名单，防止误删。
扫描的结果会输出到`apk-checker-result.json`文件中，执行下面的脚本可以批量删除项目中的资源文件。
```bash
#!/usr/bin/env bash
# 用于删除apk-checker扫描出来的unused resources
# 输入参数1:unused-resources filename
# 输入参数2:待处理资源目录路径
for line in $(cat "$1")
do
    type=''
    value=''
    eval $(echo "${line}" | awk -F '.' '{print "type="$2";value="$3}')
    name="<""${type}"" name=\"""${value}""\">"
    echo $name
    find "$2" -type f -name '*.xml' | xargs  grep -l "${name}" | xargs sed -i '' "/${name}/d"

done
```

删除无用资源后，apk包大小减少了280K，仅无用的dimens资源就占用了137K。
![企业微信截图_608b1862-eb8c-4916-8ecb-277a565ed2d7.png](http://ww1.sinaimg.cn/large/8b331ee1gy1g97om9fjjqj21dc01pdg5.jpg)
在接入[AndroidAutoSize](https://github.com/JessYanCoding/AndroidAutoSize)屏幕适配方案后，原先手工定义的定义的dimen都可以这种删除。

#### 删除重复的资源
随着工程的增大，开发人员的变动，有些资源文件名字不同，但是内容却完全相同。
可以通过扫描文件的MD5值，找出内容相同的文件。
在构建流水线中接入ipt腾讯云安装包检查工具后，可以检测出重复文件资源
![企业微信截图_11e4c25c-1262-41d1-84b8-9ac5f4c9f336.png](http://ww1.sinaimg.cn/large/8b331ee1gy1g97qygqgvfj20gw04wjrv.jpg)
重复资源涉及到代码修改，需要手工删除。

Android在适配图片资源的时候，如果只有一套资源，低密度设备会缩放图片，高密度设备会拉伸图片。我们利用这个特性，存放一套资源图就可以供所有密度的设备使用。
综合考虑图片清晰度，包大小和内存占用情况，一般采用xhdpi下的资源图片。


#### 图片格式
在Google I/O 2016中提到了针对图片格式的选择,
其建议是:如果能用VectorDrawable来表示的话优先使用VectorDrawable，如果支持WebP则优先用WebP，如果有透明度要求则使用PNG，其它场景可以使用JPG格式。
![图片格式选择](http://ww1.sinaimg.cn/large/8b331ee1gy1g96x375bmhj20zk0jyabg.jpg)
目前4.2及以上的Android版本已经支持WebP，4.0,4.1的Android版本只支持不含透明度的有损压缩，
因为我们项目支持的最低版本是4.0，项目中的不含透明度的图片较少，考虑到兼容性问题，因此项目apk中的图片还是采用了PNG和JPG格式。

#### 图片压缩
Android项目构建过程中默认会使用[pngcrush压缩res/drawable/下的图片资源](https://developer.android.com/guide/topics/graphics/drawables#drawables-from-images),
但是PNG cruncher的压缩率并不高，有必要选择更合适的压缩工具。

[png图片压缩工具对比](https://cloud.tencent.com/developer/article/1034208)一文中，对比了主流的图片压缩工具。
> tinypng、pngquant、ImageAlpha、pngnq都是有损压缩，基本采用的都是quantization算法，将24位的PNG图片转换为8位的PNG图片，减少图片的颜色数；
> pngcrush、optipng、pngout、advpng都是无损压缩，采用的都是基于LZ/Huffman的DEFLATE算法，减少图片IDAT chunk区域的数据。一般有损压缩的压缩率会大大高于无损压缩。

我们项目选择了公司内开源的组件[pnghelper](https://git.oa.com/chunyujin/auto_compress_pic)，
它集成了pngquant、pngcrush、optipng，支持自动扫描，省去的人为压缩工作量的问题。
在保证图片质量的前提下，极大缩减了图片的大小。
同时需要关闭cruncherEnabled来禁止默认的优化算法，避免图片增大。
```gradle
android {
    aaptOptions {
        cruncherEnabled false
    }
}
```

#### 图片网络化
为了进一步减少应用内图片资源，我们采用图片网络化的方案，根据产品功能有选择得将应用内比较大的图片配置到CDN上，客户端使用纯色或者小尺寸图片的兜底图，并会对图片做预下载，保证在使用到图片的场景中，可以尽快的展示出来。图片网络化的另一个好处是可以很方便支持动态更换图片，可以在线上配置，不用跟客户端版本。
对于图片预下载，Glide提供了downloadOnly的模式，支持仅下载图片文件，防止使用bytes解码出现内存的瞬间增长。

![screenshot.png](http://ww1.sinaimg.cn/large/8b331ee1gy1gd519moe13j20n00f9jto.jpg)

对于新增超过10k的图片我们要求尽量做到网络化。
经过大图网络化方案的改造后，做到了产品体验与包体积兼顾，apk包大小减少了2.3M,并且也可持续化。

#### 资源混淆压缩
我们项目中使用[AndResGuard](https://github.com/shwenzhang/AndResGuard)实现资源混淆和极限压缩处理。
通过将资源文件名混淆替换成短路径:
```sh
res/drawable/icon -> res/s/a
```
达到减少 resources.arsc、签名文件以及 ZIP 文件大小的目的。
AndResGuard利用了 7-Zip 的大字典优化，提升APK整体压缩率，可以强制压缩PNG、JPG 以及 GIF 等Android默认不会打包压缩的文件，进一步减少包大小。

因为资源名字会做混淆处理，项目中通过`getIdentifier`去访问的资源，需要配置到白名单。
经过AndResGuard处理后，apk包大小减少了约800K。
![屏幕快照 2019-10-08 上午11.14.15.png](http://ww1.sinaimg.cn/large/8b331ee1gy1g97rvke7bwj22dq034q3n.jpg)

---

### 代码优化

#### ProGuard
由于项目初期对于包大小不够重视，对于项目中的ProGuard配置文件关注不够，导致存在了很多过度keep的问题。

通过在`proguard.cfg`配置下面的规则，可以输出 ProGuard 的最终配置。
```sh
-printconfiguration build/final_proguard_config.txt
```
查看最终配置文件是否有过度keep的问题，比如：
```sh
-keep class org.apache.** { *; }
-keep class android.support.** { *; }
```
很多情况下，我们只需要keep其中的某个包、某个方法，或者是类名就可以了。

通过Android Studio的APK Analyzer可以很方便得查看apk中不同库代码占比情况
![APK Analyzer](http://ww1.sinaimg.cn/large/8b331ee1gy1g97trsd67hj20lg07ugml.jpg)
support包里的类名都没有混淆，是被过度keep了。

ProGuard 在每次运行时都会在工程 `build/outputs/mapping/release`目录输出结果文件：

 1. seeds.txt : 列出未混淆的类和成员
 2. usage.txt : 列出从 .apk 删除的代码
 3. mapping.txt ：列出原始与混淆后的类、方法和字段名称之间的对应关系
 4. dump.txt ：描述 .apk 文件中所有类文件的内部结构

我们可以根据 seeds.txt 文件检查未被混淆的类和成员中是否已包含所有期望保留的，再根据 usage.txt 文件查看是否有被误移除的代码。

需要注意的是，宿主提供给插件使用(compileonly)的代码需要keep住，否则会报找不到类方法的错误
```java
type:java.lang.NoSuchMethodError
message:No virtual method toJsonTree(Ljava/lang/Object;)Lcom/google/gson/JsonElement; in class Lcom/google/gson/Gson; or its super classes (declaration of 'com.google.gson.Gson' appears in /data/app/com.ktcp.video==/base.apk)
stack:
com.ktcp.tvagent.stat.StatProperties.fromKVObject(StatProperties.java:xx)
```

修改项目中不合理的配置规则后，apk包大小减少了约600k。

#### Dex方法数优化

jce协议自动生成的java代码可以选择精简模式，去掉set、get、display等方法。
![jce精简模式](http://ww1.sinaimg.cn/large/8b331ee1gy1g97ukq91thj21ea0cqwi4.jpg)
去除jce大部分多余方法后，apk包大小减少了约40K，dex方法数减少了6千多个。

优化前：
![jce优化前](http://ww1.sinaimg.cn/large/8b331ee1gy1g97urkys3pj20k70ig793.jpg)

优化后：
![jce优化后](http://ww1.sinaimg.cn/large/8b331ee1gy1g97us0yx6pj20l70lm0xo.jpg)

#### D8 & R8
D8 作为下一代 dex 编译器，与之前的DX编译器相比，D8 运行速度更快，生成的 .dex 文件更小且具有同等或更好的运行时性能。
R8是一种用于执行代码压缩和混淆的新工具，可以兼容Proguard配置。

![之前的编译流程](http://ww1.sinaimg.cn/large/8b331ee1gy1g97vkangu7j216l0g5wfm.jpg)
![新一代的编译流程](http://ww1.sinaimg.cn/large/8b331ee1gy1g97vl213aqj215s0j5dgr.jpg)

有关D8 & R8更多内容可以看这几篇文章：
[Android CPU, Compilers, D8 & R8](https://proandroiddev.com/android-cpu-compilers-d8-r8-a3aa2bfbc109)
[D8 Optimizations](https://jakewharton.com/d8-optimizations/)
[R8 Optimization: Staticization](https://jakewharton.com/r8-optimization-staticization/)

为了使用R8，需要将项目AGP版本升级到3.4.1以上。
升级过程中遇到以下：
1、gradle api变动的问题，需要修改gradle plugin兼容性的api。
[GRADLE Support AGP 3.4.0](https://github.com/getsentry/sentry-java/pull/724/files)
```groovy
try { // Android Gradle Plugin >= 3.3.0
    return variantOutput.processManifestProvider.get().manifestOutputDirectory.get().asFile
} catch (Exception ignored) {}
try { // Android Gradle Plugin < 3.0.0
    return new File(variantOutput.processManifest.manifestOutputFile).parentFile
} catch (Exception ignored) {}
```
2、3.4.2版本的data binding需要把包名改成小写，大写的包名会被databinding-compiler误当做类名，编译报错。

配置打开D8 & R8
```java
android.enableD8=true
android.enableR8=true
```

开启D8 & R8后，apk包大小减少了约1.3M，减少了1w多个方法。
但是在测试tinker升级的过程中发现，开启R8会造成mapping冲突问题，暂时只能先关闭R8。

#### 删除R类

经过删除R类优化后，apk包大小减少245K。
![R文件剔除效果](http://ww1.sinaimg.cn/large/8b331ee1gy1g97vql8v2gj21ty01ndfy.jpg)

可以通过下面命令检测R类是否删除成功。
```sh
find . -name '*.dex' | xargs dexdump | grep 'Class descriptor'|grep -e 'R\$'
```

Android组件化工程会导致生成很多冗余的R类字段
Android多项目构建时，会将子项目生成R文件中的字段都声明为非 final类型，将 ID 的最终分配延迟到编译的打包期间，这样虽然可以提升构建性能和防止资源索引冲突，但是却带来了很多冗余的字段被打包到了最终的 APK，体现在两点：
- 每个子项目都会产生一个R文件，造成R文件数量增多
- 项目依赖层级越深，产生冗余的R类字段就越多

首先想到的是通过Proguard对R类做内联优化，但是项目中有反射R类的情况，proguard处理有bug，不会内联，第2种方案是基于 ASM 在字节码层面内联优化，优点是可增加白名单支持反射R类的场景。

基本原理是注册Gradle Transform，在由class到dex转换阶段，通过ASM技术操作修改class文件，读取所有的R类，将字段名和值记录起来，然后替换有对R类字段引用的地方。
由于存在反射R类的情况，需要额外支持一下白名单配置。
![screenshot.png](http://ww1.sinaimg.cn/large/8b331ee1gy1gd5122v3kmj20s20jzjus.jpg)

### 其它优化

#### 渠道包
有些代码资源只会在某些渠道apk中用到,没有必要打包到基线中，可以通过`flavors`配置。

#### 插件化
通过将功能较独立的业务做成插件，独立下发的方式，从而达到减少包体积的效果。

项目中由于语音模块相对较独立，风险点较小，我们首先对它进行了插件化改造试点。语音模块插件化之后，apk包大小减少了约2.7MB。

经过一个版本的线上数据观察（crash率无明显异常、插件安装成功率在99%以上），验证了插件化方案的可行性，于是我们开始着手Hippy模块的插件化
Hippy模块插件化之后，apk包大小减少了约4.2MB。


#### 自动化持续监控
包大小的持续监控方面，我们通过在蓝盾上构建一个用于监控apk大小的流水线，配置Git事件触发，每次代码push触发apk构建，通过配置IPT可检查每个版本跟上一个版本包大小的差值，超过阈值会导致构建失败，并通知开发者。这时可以通过查看具体的代码变更记录找出造成包体积增大的具体原因。
![screenshot.png](http://ww1.sinaimg.cn/large/8b331ee1gy1gd51cigcfmj20v80kv0vv.jpg)
![apk_monitor pipeline](http://ww1.sinaimg.cn/large/8b331ee1gy1gajjze1e2ij20ia0jcjth.jpg)

结合IPT和[Matrix-ApkChecker](https://github.com/Tencent/matrix/wiki/Matrix-Android-ApkChecker)可以帮助我们快速分析包体积增大的原因，如无用资源、大文件、重复文件、R 文件等。

---
# 总结
经过了几个迭代版本，通过native优化、删除无用重复资源、图片压缩、图片网络化、资源压缩、proguard配置、Dex方法数优化、删除R类、插件化等优化手段，
将apk从37.16MB减少到15.16MB。
通过自动化持续监控apk包大小，缩短问题发现路径，提升优化效率。



