---
title: 'Xpatch: 免Root实现App加载Xposed插件的一种方案'
date: 2019-04-18 01:00:55
categories: Android
tags:
- Xpatch
- Xposed
- Hook
- Android Safety
---


# 源码
Github: [**https://github.com/WindySha/Xpatch**](https://github.com/WindySha/Xpatch)

# 基本原理
Xpatch的原理是对Apk文件进行二次打包，重新签名，并生成一个新的可加载Xposed插件的Apk。  

在Apk二次打包过程中，插入加载Xposed插件的逻辑，这样，新的Apk文件就可以加载任意Xposed插件，从而实现App免Root使用Xposed插件进行Hook。

Xposed Hook底层暂时使用的lody的[whale](https://bbs.pediy.com/thread-249212.htm)方案。
支持Android 5.0 ~ 9.0。

<!-- more -->

# 使用方法
在github release界面下载jar包，命令行中执行如下命令即可：
```
$ java -jar ../xpatch.jar ../src.apk
```

这条命令之后，在原Apk文件(src.apk)相同的文件夹中，会生成一个名称为src-xposed-signed.apk的新apk，这就是重新签名之后的支持xposed插件的apk。卸载了Android系统里的原apk，安装新apk即可。  

当然，也可以指定新Apk生成的路径，加上`-o`参数即可:
```
$ java -jar ../xpatch.jar ../src.apk -o ../dst.apk
```
详细用法请参考Github中的[Readme文档]([https://github.com/WindySha/Xpatch/blob/2538d86ecccc9fa62afec290a754cd23e56ccdb6/README.md](https://github.com/WindySha/Xpatch/blob/2538d86ecccc9fa62afec290a754cd23e56ccdb6/README.md)

# How to discuss with me
**Any questions or suggestions can be left under this article.**  
**I will reply when I have time.**
