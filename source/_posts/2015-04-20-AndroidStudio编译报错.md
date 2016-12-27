title: 在AndroidStudio编译过程中遇到Error:duplicate files during packaging of APK问题的解决方法
tag: Debug
categotry: Android
---

##问题描述
	Error:duplicate files during packaging of APK /Users/sample/app/build/outputs/apk/app-debug-unaligned.apk

	Path in archive: META-INF/LICENSE.txt

	Origin 1: /Users/sample/app/libs/commons-codec-1.3.jar

	Origin 2: /Users/sample/app/libs/commons-httpclient-3.1.jar


##问题原因
libs文件夹下的多个jar包中有相同的`LICENSE.txt` 、`NOTICE.txt`文件，所以编译器会报重复文件的错误。

##解决方案
只需要在`build.gradle`文件中添加如下内容即可

	android {
    packagingOptions {
        exclude 'META-INF/LICENSE.txt'
        exclude 'META-INF/NOTICE.txt'
    }
	}