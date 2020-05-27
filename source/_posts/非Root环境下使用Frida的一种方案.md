---
title: 非Root环境下使用Frida的一种方案
date: 2020-05-28 13:07:43
categories:
- Firda
- Xposed
- Android逆向
- Xpatch
tags:
- Frida
- Xposed
---


## 前言
在Android逆向过程中，除了Xposed，还有一个必不可少的Hook神器，那就是[Frida](https://frida.re/)。而且，Firida比Xposed功能更加强大，不仅可以实现Java层Hook，还可以实现native层Hook。但是在使用过程中，只能在Root设备才能实现代码的hook。那么是否有免Root使用Frida的方案呢？

关于免Root使用Frida，业内也有一些方案，大致有以下几种：
1. 使用Apktool反编译Apk，修改smali文件和manifest文件，实现Firda的加载。
这种方案原理比较简单，实现起来也不算复杂。这篇英文文档中，对此有非常详细的介绍：
[Using Frida on Android without root](https://koz.io/using-frida-on-android-without-root/)  
这种方案的本质就是将frida-gadget.so放到反编译后的apk so目录下，并修改反编译后的smali文件，插入`System.loadLibrary("frida-gadget") `对应的smali代码，从而实现frida的加载。

<!-- more -->

2. 使用开源工具Objection
源码地址：[https://github.com/sensepost/objection](https://github.com/sensepost/objection)
其原理跟方法1类似，也是使用Apktool反编译apk然后植入代码，只是它将这个流程封装成一个工具，使用起来更方便。
3. 使用LIEF工具修改原so文件，实现对frida-gadget.so的加载
LIEF工具的官方文档对此有详细的介绍：[https://lief.quarkslab.com/doc/latest/tutorials/09_frida_lief.html](https://lief.quarkslab.com/doc/latest/tutorials/09_frida_lief.html)，
也可参考本文对应的中文翻译文档：[https://bbs.pediy.com/thread-229970.htm](https://bbs.pediy.com/thread-229970.htm)  
这种方法的基本原理是，利用LIEF工具将frida-gadget.so与原Apk中的某个so文件链接起来，使得加载原so时，同时也加载了frida-gadget.so文件，从而实现Frida工具。  
这种方法有以下几个缺点：
- 需要向APK里添加文件
- 需要程序有至少一个native库
- 注入进去的库的加载顺序不能控制

## 使用Xpatch实现免Root的Frida功能
Xpatch是笔者开发的一个免Root加载Xposed插件工具
源码地址：[https://github.com/WindySha/Xpatch](https://github.com/WindySha/Xpatch)
Xpatch Apk版本Xposed Tool下载地址：https://xposed-tool-app.oss-cn-beijing.aliyuncs.com/data/xposed_tool_v2.0.2.apk

既然Xpatch可以实现免Root加载Xposed插件，那么，Xpatch应该也可以实现免Root使用Frida。

方法其实也比较简单，只需编写一个专门用于加载frida-gadget.so文件的Xposed插件，然后使用Xpatch处理原Apk文件并安装，最后让经Xpatch处理后的Apk加载该该Xposed插件即可。

下面，详细介绍该Xposed插件的实现方法。
## 用于加载Frida.so的Xposed模块
为了实现Xposed模块中的Frida.so能被其他进程加载，可以通过以下几个步骤：
1. 将frida-gadget.so文件内置到Xposed插件Apk的lib目录下；
2. 获取当前插件Apk文件路径，这里有两种方式可以取到：
- 通过PackageManager和插件Apk的包名获取Apk的安装路径，代码如下：
```
apkPath = context.getPackageManager().getApplicationInfo(packageName, 0).sourceDir;
```
这种方法的前提是，插件已经在设备上安装了。
- 根据加载插件的classLoader，反射获取classLoader里保存的apk路径
加载Xposed插件模块的classLoader都是一个BaseDexClassLoader，里面保存了此插件的文件路径，通过反射可以获取到：
```
Field fieldPathList = getClassLoader().getClass().getSuperclass().getDeclaredField("pathList");
fieldPathList.setAccessible(true);
Object dexPathList = fieldPathList.get(AppUtils.class.getClassLoader());

 Field fieldDexElements = dexPathList.getClass().getDeclaredField("dexElements");
fieldDexElements.setAccessible(true);
Object[] dexElements = (Object[]) fieldDexElements.get(dexPathList);
Object dexElement = dexElements[0];

Field fieldDexFile = dexElement.getClass().getDeclaredField("dexFile");
fieldDexFile.setAccessible(true);
Object dexFile = fieldDexFile.get(dexElement);
String apkPath = ((DexFile) dexFile).getName();
```
这种方法相对比较麻烦，但是其优点是，插件Apk即使没有安装，也可以取到其被加载时的路径，这样更加灵活。当插件是被内置到Apk中加载时，也可以成功获取到其原路径。
3. 将插件Apk中的frida-gadget.so文件拷贝到主应用的data/data目录下。
复制so文件也有两种方法，第一种最简单的方案是使用ZipFile读取Apk压缩包中的so文件流，然后写入到指定的文件目录下。
另外一种方案是利用Android sdk中的隐藏类`NativeLibraryHelper`，该类在Android源码中的文件路径为：`framework/base/core/java/com/android/internal/content/NativeLibraryHelper.java`。其中的`copyNativeBinaries`就是将Apk中的lib文件复制到指定的文件目录下。其实现原理是在native层遍历ZipFile，并将符合条件的so文件复制出来。由于是native层实现的，其运行效率比java层复制效率高，因此推荐使用。
下面是Android9.0中`copyNativeBinaries`方法的源码：
```                                                                                      
* Copies native binaries to a shared library directory.                                   
 *                                                                                         
 * @param handle APK file to scan for native libraries                                     
 * @param sharedLibraryDir directory for libraries to be copied to                         
 * @return {@link PackageManager#INSTALL_SUCCEEDED} if successful or another               
 *         error code from that class if not                                               
 */                                                                                        
public static int copyNativeBinaries(Handle handle, File sharedLibraryDir, String abi) {   
    for (long apkHandle : handle.apkHandles) {                                             
        int res = nativeCopyNativeBinaries(apkHandle, sharedLibraryDir.getPath(), abi,     
                handle.extractNativeLibs, HAS_NATIVE_BRIDGE, handle.debuggable);           
        if (res != INSTALL_SUCCEEDED) {                                                    
            return res;                                                                    
        }                                                                                  
    }                                                                                      
    return INSTALL_SUCCEEDED;                                                              
}                                                                                          

```
4. 调用System.loadLibrary()方法，加载data/data目录下的frida-gadget.so文件。
so加载成功，便实现了免Root使用Frida。
## 验证
为了验证该Xposed插件可以实现免Root使用Frida，我们将此插件安装到基于Android6.0.1的Nexus5手机上，并安装Xposed Tool(Xpatch App版本)工具，使用Xposed Tool破解任意一个未加固未防二次打包的应用。然后，在Xposed模块管理中启用此Xposed插件。



![](https://upload-images.jianshu.io/upload_images/1639238-c51128c288b18a54.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/320)

我们启动被破解的应用后，发现应用卡在启动界面，这说明，Xposed插件中的frida-gadget.so文件加载成功。
然后将手机连接PC，启动USB调试模式。并在命令行输入：
`frida -U gadget -l ../test.js`
其中，在test.js中，我们简单地拦截了Activity的onCreate方法和onResume方法，其实现如下：
```
// test.js
Java.perform(function() {
   var Activity = Java.use("android.app.Activity");
    Activity.onCreate.overload('android.os.Bundle').implementation = function(arg1) {
       console.log("Activity onCreate() got called!  ");
       this.onCreate(arg1);
   }

   Activity.onResume.implementation = function() {
       console.log(" Activity onResume() got called!  ");
       this.onResume();
   }
})
```
命令行回车后，App成功进入主界面，并在命令行中打印出如下日志：
```
[Nexus 5::gadget]-> Activity onCreate() got called!  
 Activity onResume() got called! 
```
成功hook了Activity的onCreate和onResume方法。
这说明test.js中hook代码已生效，免Root环境成功跑通了Frida！！
## 不足之处
经过测试发现，在部分机型上，部分App上使用`frida -U gadget -l ../test.js`命令，提示“ Server terminated”或者“Failed to load script: timeout was reached ”，暂时未找到问题原因。
## 源码
此Xposed插件源码已上传到Github：[FridaXposedModule](https://github.com/WindySha/FridaXposedModule)
欢迎Star and Fork。
