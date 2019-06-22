---
title: 免Root环境，破解爱奇艺和腾讯视频广告的一个简易方法
date: 2019-06-23 00:19:20
categories: Android
tags:
- Xpatch
- Hook
---

# 前言
笔者推出免Root破解App工具[Xpatch]([https://github.com/WindySha/Xpatch](https://github.com/WindySha/Xpatch)也有一段时间了，但一直没有介绍详细的应用案例。本文介绍如何利用Xpatch实现一个非常实用的功能，腾讯视频App和爱奇艺App去广告功能。 

# 如何去广告
为了破解腾讯视频，首先我们需要反编译Apk，获取Java源码。庆幸的是腾讯视频没有做加固处理，使用jadx工具可以反编译成功。  
获取到源码后，通过分析源码，我们发现一个叫做`VideoInfo`的类:

<!-- more -->

```
// com.tencent.qqlive.ona.player.VideoInfo.java
public class VideoInfo {
    ....
    private boolean isAd;
    private boolean isAdSkip;
    private boolean isAutoPlay;
    private boolean isAutoPlayNext;
    private boolean isBlockAutoPlay;
    private boolean isCharged;
    ....
    private boolean isHotChannelPlayer;
    private boolean isHotspot;
    private boolean isMiniVideo;
    private boolean isNotStroeWatchedHistory;
    private boolean isTryWatch;
    private boolean isUserCheckedMobileNetWork;
    private boolean isVip7CanPlay;
    ....

    public boolean isAdSkip() {
        return this.isAdSkip;
    }
    ....
}
```
这个类没有做混淆处理，很可能是从服务端返回的每个video的信息，其中有一个成员变量叫做`isAdSkip`，很可能就是用来控制广告播放与否的字段。
为了验证这种猜想，写个Xposed插件，将获取这个参数值的方法`isAdSkip`Hook掉，使其返回true。Hook代码如下：
```
// kotlin
findAndHookMethod("com.tencent.qqlive.ona.player.VideoInfo",
            classLoader,
            "isAdSkip",
            object : XC_MethodHook() {
                override fun beforeHookedMethod(param: MethodHookParam) {
                    param.result = true
                }
            })
```
让Xpatch重打包后的腾讯视频app加载这个Xposed插件，启动后发现，播放任何一个视频都没有广告了，因此，成功实现腾讯视频广告破解.  

爱奇艺的破解过程类似，只是破解的点不太一样，爱奇艺app中视频信息的实体类中并没有包含类似`isAdSkip`的变量。爱奇艺反编译后的代码比腾讯视频复杂很多，而且很多核心的代码都是在Native中实现的，因此暂时没能完美实现广告破解。  
通过不断尝试，发现hook `StateManager `类的`updateVideoType `方法，使其返回空，可以实现二次打开视频无广告。也就是说，需要点击点击一个视频进入播放页，再返回，再进入此视频播放页，此时便无广告。具体hook代码如下:
```
// kotlin
XposedHelpers.findAndHookMethod("com.iqiyi.video.qyplayersdk.player.state.StateManager",
            classLoader,
            "updateVideoType",
            Int::class.java,
            object : XC_MethodReplacement() {
                override fun replaceHookedMethod(param: MethodHookParam?): Any? {
                    return null
                }
            })
```
# Xposed插件
有了以上的hook代码，写成一个Xposed插件就很容易了。
完整代码已经上传到了笔者的Github上，地址为：
https://github.com/WindySha/RemoveVideoAdsPlugin
编译出来的插件apk可以在Release页面下载：
https://github.com/WindySha/RemoveVideoAdsPlugin/releases

# 利用Xpatch制作无广告App
[Xpatch](https://github.com/WindySha/Xpatch)是笔者开发的一个免Root加载Xposed插件的工具。利用Xpatch和上面的插件apk可以制作一个无广告的腾讯视频App和爱奇艺App，具体使用流程如下：
1. 下载Xpatch Jar包，下载链接为：https://github.com/WindySha/Xpatch/releases/download/v1.4/xpatch-1.4.jar.zip；
2. 下载腾讯视频apk（爱奇艺apk）以及上文的无广告Xposed插件apk；
3. 在PC的终端执行如下命令：`$ java -jar ../xpatch.jar ../tencent_video.apk -xm ../xposed_module.apk`
4. 命令执行完成之后，在tencent_video.apk相同的目录下，会生成一个名称为tencent_video-xposed-signed.apk，卸载掉设备上原来的apk，安装这个无广告的apk即可。


# 下载
为了方便大家使用，笔者通过上面方法，制作出了这个两个无广告apk，百度网盘下载链接为：  
1. 腾讯视频无广告版本：https://pan.baidu.com/s/18nKg7AM-Rwb0y0SWi5Vkow
2. 爱奇艺无广告版本: https://pan.baidu.com/s/1X_sTJPGobChO1tNoTENFTg 
 
关注公众号Android葵花宝典可获取到文件提取码。
再次温馨提醒：爱奇艺需要进入播放页后退出播放页，再次进入才会无广告。

# 文件提取码
先关注公众号：Android葵花宝典  
![](https://upload-images.jianshu.io/upload_images/1639238-61a070908946025d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/160)  
点击公众号中的提取码按钮，获取文件提取码：

![](https://upload-images.jianshu.io/upload_images/1639238-1ee44a6a2ba242db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/150)
# 拓展
在Root手机中，不需要使用Xpatch处理App，只需要下载XposedInstaller App安装Xposed框架，然后再安装破解广告的Xposed插件即可实现视频无广告。  

在非Root环境下，还有一种方法可实现此功能。
由于对App的修改点并不多，使用Xpatch来破解似乎有点大才小用。我们也可以使用[Apktool]([https://github.com/iBotPeaches/Apktool](https://github.com/iBotPeaches/Apktool)
)工具，反编译Apk，生成smali代码，然后修改smali代码。因为只是修改一个方法的返回值，所以改起来并不会太难。修改完smali后，然后再利用Apktool将修改后的smali代码和其他文件一起打包成apk文件，最后给apk签名，这样，也能制作出一个无广告的腾讯视频App。
# 声明
本文中提到的技术仅能用于个人学习研究，请勿用于任何商业目的。下载的文件仅能用于个人试用，并请在24小时内删除。
