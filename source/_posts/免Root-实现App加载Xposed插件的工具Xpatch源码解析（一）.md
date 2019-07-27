---
title: 免Root 实现App加载Xposed插件的工具Xpatch源码解析（一）
date: 2019-07-27 13:07:43
categories:
- Xposed
- Android逆向
- Xpatch
tags:
- Xposed
---


# 前言
Xpatch是一款免Root实现App加载Xposed插件的工具，可以非常方便地实现App的逆向破解（再也不用改smali代码了），源码也已经上传到[Github](https://github.com/WindySha/Xpatch)上，欢迎各位Fork and Star。  

本文主要介绍Xpatch的实现原理。由于其原理比较复杂，所以分二篇文章来详细讲解。  

由于Xpatch处理Xposed module的方法参考了Xposed框架部分源码，所以本文先介绍Xposed框架加载Xposed模块原理，再详细讲解Xpatch如何兼容Xposed模块。

<!-- more -->

# Xposed框架加载Xposed Module的原理
Xposed是github上rovo89大神设计的一个针对Android平台的动态劫持项目，其主要原理是通过替换/system/bin/app_process程序控制zygote进程，使得app_process在启动过程中会加载XposedBridge.jar这个jar包，从而完成对Zygote进程及其创建的app进程的劫持。

XposedBridge.jar的入口方法是main()，其主要逻辑如下：
```
//de.robv.android.xposed.XposedBridge.java
	protected static void main(String[] args) {
		// Initialize the Xposed framework and modules
		try {
			if (!hadInitErrors()) {
				initXResources();

				SELinuxHelper.initOnce();
				SELinuxHelper.initForProcess(null);

				runtime = getRuntime();
				XPOSED_BRIDGE_VERSION = getXposedVersion();

				if (isZygote) {
					XposedInit.hookResources();
					XposedInit.initForZygote();
				}

				XposedInit.loadModules();
			} else {
				Log.e(TAG, "Not initializing Xposed because of previous errors");
			}
		} catch (Throwable t) {
			Log.e(TAG, "Errors during Xposed initialization", t);
			disableHooks = true;
		}

		// Call the original startup code
		if (isZygote) {
			ZygoteInit.main(args);
		} else {
			RuntimeInit.main(args);
		}
	}
```
这里最核心的一行代码是:
```
XposedInit.loadModules();
```
在这个方法里，通过读取`/data/data/de.robv.android.xposed.installer/conf/modules.list`这个文件，找到需要加载的Xposed插件(APK)路径。而这些路径都是通过Xposed Installer这个App里的开关控制的。在Xposed Installer App里，有一个已安装的Xposed插件列表，用户选定某个插件后，就会将该插件APK路径写到modules.list文件里，从而实现插件开关的控制。

在modules.list文件里查找到所有插件APK路径后，根据Apk的绝对路径构造一个PathClassLoader()，然后用此Classloader加载全类名写在资源文件`assets/xposed_init`里的入口类，其核心逻辑代码如下：
```
//de.robv.android.xposed.XposedInit.java
...
...
ClassLoader mcl = new PathClassLoader(apk, XposedBridge.BOOTCLASSLOADER);
		BufferedReader moduleClassesReader = new BufferedReader(new InputStreamReader(is));
		try {
			String moduleClassName;
			while ((moduleClassName = moduleClassesReader.readLine()) != null) {
				moduleClassName = moduleClassName.trim();
				if (moduleClassName.isEmpty() || moduleClassName.startsWith("#"))
					continue;

				try {
					Log.i(TAG, "  Loading class " + moduleClassName);
					Class<?> moduleClass = mcl.loadClass(moduleClassName);

					if (!IXposedMod.class.isAssignableFrom(moduleClass)) {
						Log.e(TAG, "    This class doesn't implement any sub-interface of IXposedMod, skipping it");
						continue;
					} else if (disableResources && IXposedHookInitPackageResources.class.isAssignableFrom(moduleClass)) {
						Log.e(TAG, "    This class requires resource-related hooks (which are disabled), skipping it.");
						continue;
					}
					...
					...
```
加载到这些类之后，将这些类使用全局变量保存起来：
```
if (moduleInstance instanceof IXposedHookLoadPackage)
	XposedBridge.hookLoadPackage(new IXposedHookLoadPackage.Wrapper((IXposedHookLoadPackage) moduleInstance));

	...
	...
	
	//保存在全局变量sLoadedPackageCallbacks里
public static void hookLoadPackage(XC_LoadPackage callback) {
		synchronized (sLoadedPackageCallbacks) {
			sLoadedPackageCallbacks.add(callback);
		}
	}

```
保存起来后，何时执行这些类里的入口方法呢？
下面一段代码，给出了答案：
```
// normal process initialization (for new Activity, Service, BroadcastReceiver etc.)
		findAndHookMethod(ActivityThread.class, "handleBindApplication", "android.app.ActivityThread.AppBindData", new XC_MethodHook() {
			@Override
			protected void beforeHookedMethod(MethodHookParam param) {
			    ...
			
				XC_LoadPackage.LoadPackageParam lpparam = new XC_LoadPackage.LoadPackageParam(XposedBridge.sLoadedPackageCallbacks);
				lpparam.packageName = reportedPackageName;
				lpparam.processName = (String) getObjectField(param.args[0], "processName");
				lpparam.classLoader = loadedApk.getClassLoader();
				lpparam.appInfo = appInfo;
				lpparam.isFirstApplication = true;
				XC_LoadPackage.callAll(lpparam);
				...
		});
```
通过上面代码可知，是在main入口处拦截了`ActivityThread`的`handleBindApplication`方法，在这个方法执行之前，加载了Xposed插件里的Hook代码(入口代码)。而`ActivityThread`的`handleBindApplication`方法的主要功能就是创建`Application`，并调用其`attachBaseContext`，`onCreate`等方法。因此，在App的Application创建之前就实现了Xposed Hook。

至此，Xposed框架加载Xposed module的流程就非常清晰了。

# 免Root实现Xposed的探索
由于Xposed框架是修改了/system/bin/app_process程序，控制的zygote进程的启动，从而在app进程启动之前执行了加载xposed模块，实现了App代码的Hook。因此，只有Root的手机才能使用Xposed。

那有没有办法实现免Root下也能让App加载Xposed模块呢？

其中一种已经被探索过的方法是使用App双开工具，让其他App运行在自己的App里面。比如，利用大名鼎鼎的开源双开工具**VirtualApp**，让其他App运行其中，这样VirtualApp就可以控制其他App的进程启动了，当然也就可以实现免Root加载Xposed模块了。

不过，这样做也有一些问题，其兼容性和稳定性比较差。而且，由于VirtualApp的开源版本已经很少维护，bug会比较多，有些App在里面启动非常卡顿，甚至无法启动

那有没有更好的方案呢？

有，那就是基于Apk二次打包的Xpatch方案。

为了实现免Root Xposed，我们可以修改App入口代码，在App的Application初始化时，插入我们加载Xposed模块的代码，并对App进行重新打包签名即可。


# Xpatch加载Xposed模块的方法

通过以上分析，Xposed框架执行Xposed模块的入口位置是通过Hook `ActivityThread`的`handleBindApplication`方法，从而使在创建应用的Application之前就执行Xposed模块里的入口方法（执行Hook流程）。

由于我们是修改应用代码，因此入口只能在应用的`Application`里，可以是`Application`的静态方法块，也可以是`attachBaseContext`方法或`onCreate`方法。

那到底应该选择哪个作为加载Xposed模块的入口呢？

首先，肯定是越早Hook越好，否则可能会出现有些方法调用之后才执行Hook方法，导致方法没有被Hook到。而且加载Xposed插件需要用到Applicatin的context参数，所以笔者选了在`attachBaseContext`方法的第一行代码执行加载Xposed模块：
```
    // MyApplication.java
    @Override
    protected void attachBaseContext(Context base) {
        XposedModuleEntry.init(base);
        ...
        //App其他业务代码
        ...
        super.attachBaseContext(base);
    }
```
通过对一些app进行测试，发现大多数应用走这个流程都没问题，唯一有问题的是微信，修改后的微信一启动就奔溃，具体原因暂时不清楚。

因此，我尝试将加载Xposed模块的入口代码放在Application的静态代码块里，静态代码块在类创建的时候就会执行，比`attachBaseContext`方法执行的时机更早。

通过测试，修改后的微信的确能够成功启动！！

```
    // MyApplication.java中的静态代码块
    static {
        XposedModuleEntry.init();
    }
```

但是，在Application的静态代码块中，并没有Application的context参数，而加载Xposed模块时，需要传递参数，参数中的applicationInfo和应用的classLoader都是需要从Context中取得，没有Context怎么办？总不能传个空过去吧。

既然没有Context，我们就自己构造一个Context。

### 创建App Context流程
Android sdk并没有提供应用自己创建context的方法，为了找到构建一个Application Context的方法，我们先来了解android Framework里是如何创建Application的Context。

Application里最早出现Context的地方是`attachBaseContext`方法:
```
// Application的父类android.content.ContextWrapper.java
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
```
这个方法唯一调用的地方是：
```
// android.app.Application.java
    /**
     * @hide
     */
    /* package */ final void attach(Context context) {
        attachBaseContext(context);
        mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
    }
```
Application里的`attach`方法是在new Application之后立即调用的，具体是在`android.app.Instrumentation.java`里：
```
// android.app.Instrumentation.java
public Application newApplication(ClassLoader cl, String className, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
        return newApplication(cl.loadClass(className), context);
    }
    
static public Application newApplication(Class<?> clazz, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
    Application app = (Application)clazz.newInstance();
    app.attach(context);
    return app;
}
```
Instrumentation类的`newApplication`方法最终又是在`android.app.LoadedApk.java`类里的`makeApplication`方法调用的：
```
// android.app.LoadedApk.java
public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
            ...
            //代码省略
            ...
            String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }
           ...
           //代码省略
           ...
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
          
            ...
            //代码省略
            ...
}
```
终于看到构造context的方法了
```
ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
```
这个方法的实现是：
```
// android.app.ContextImpl.java
static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) {
        if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
        ContextImpl context = new ContextImpl(null, mainThread, packageInfo, null, null, null, 0,
                null);
        context.setResources(packageInfo.getResources());
        return context;
    }
```
`createAppContext`方法需要传两个参数，一个是mActivityThread，另一个是this，也就是LoadedApk对象。mActivityThread这个对象比较容易找到，因为一个进程只有一个ActivityThread对象，只用通过反射调用ActivityThread的静态方法`currentActivityThread`即可：
```
// android.app.ActivityThread.java
public static ActivityThread currentActivityThread() {
        return sCurrentActivityThread;
}
```
反射：
```
//反射调用ActivityThread.java的静态方法currentActivityThread()
Class activityThreadClass = Class.forName("android.app.ActivityThread");
Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread");
currentActivityThreadMethod.setAccessible(true);
Object activityThreadObj = currentActivityThreadMethod.invoke(null);
```
另外一个对象LoadedApk该如何获取呢？

### LoadedApk对象的获取
上面代码分析过，App启动时最先调用 `ActivityThead`的`handleBindApplication(AppBindData data)`方法，并在其这个方法里创建Application，而创建Application的唯一方法是
```
// android.app.LoadedApk.java
public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation){}
```
查看`handleBindApplication`方法具体实现过程，发现`makeApplication`确实被调用到：
```
// android.app.ActivityThread.java
private void handleBindApplication(AppBindData data) {
        ...
        //其他代码省略
        ...
    mBoundApplication = data;
    mConfiguration = new Configuration(data.config);
    mCompatConfiguration = new Configuration(data.config);
         ...
        //其他代码省略
        ...
    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);
         ...
        //其他代码省略
        ...
        // 创建Application
    Application app = data.info.makeApplication(data.restrictedBackupMode, null);
    mInitialApplication = app;
        ...
        //其他代码省略
        ...
}
```
根据以上代码可知，`LoadedApk`的实例就是`data.info`，而`data.info`是通过方法`getPackageInfoNoCheck`来获取的，而且`data.info`对象存到了全局变量`mBoundApplication`里，因此，`mBoundApplication`对象里的`info`变量就是我们要找的LoadedApk实例。

我们可以通过反射来获取它：
```
// 获取ActivityThread的mBoundApplication变量
Field boundApplicationField = activityThreadClass.getDeclaredField("mBoundApplication");
boundApplicationField.setAccessible(true);
Object mBoundApplicationObj = boundApplicationField.get(activityThreadObj);   // mBoundApplication: AppBindData

// 获取mBoundApplication的info变量（LoadedApk）
Field infoField = mBoundApplicationObj.getClass().getDeclaredField("info");   // info: LoadedApk
infoField.setAccessible(true);
Object loadedApkObj = infoField.get(mBoundApplicationObj);  // LoadedApk
```

获取到`ActivityThread`和`LoadedApk`后，通过反射调用`ContextImpl`的静态方法`createAppContext`就可以构造一个context对象：
```
Class contextImplClass = Class.forName("android.app.ContextImpl");
//Get createAppContext method
Method createAppContextMethod = contextImplClass.getDeclaredMethod("createAppContext", activityThreadClass, loadedApkObj.getClass());
createAppContextMethod.setAccessible(true);

// call method: ContextImpl.createAppContext()
Object context = createAppContextMethod.invoke(null, activityThreadObj, loadedApkObj);
```
至此，我们在Application的静态代码块中成功得到Application context。


#  Comming soon...
下一篇Xpatch源码解析中，我们将接着这部分内容介绍`XposedModuleEntry.init();`这个方法的具体实现逻辑，然后再介绍如何利用dex2jar工具修改Apk中dex文件。

欢迎扫二维码，关注我的技术公众号**Android葵花宝典**  ，获取高质量的Android干货分享：
 
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xNjM5MjM4LWFiNmUwZmNlYWJmZmZkZGEuanBn)
