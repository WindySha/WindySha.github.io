---
title: 免Root 实现App加载Xposed插件的工具Xpatch源码解析（二）
date: 2019-07-27 13:07:55
categories:
- Xposed
- Android逆向
- Xpatch
tags:
- Xposed
---
# 前言
Xpatch是笔者开发的一款破解Android App工具，源码地址：

[https://github.com/WindySha/Xpatch](https://github.com/WindySha/Xpatch)

本文接着上一篇Xpatch源码解析文章，继续分析Xpatch的实现原理。
# Xpatch加载Xposed插件流程
## 查找插件Apk
加载Xposed插件之前，首先需要遍历所有安装的应用，根据Xposed插件的特征，找到其中的Xposed插件。  

那什么样的应用才是Xposed插件呢？  

<!-- more -->

根据Xposed插件的书写规范中要求，插件Apk的Manifest文件中需要包含`android:name="xposedmodule"`这样的`meta-data`信息：
```
<application
        <meta-data
                android:name="xposedmodule"
                android:value="true"/>
</application>
```
根据此特征，我们获取App PackageInfo中的meta data，从而过滤出插件Apk，具体实现源码如下：
```
private static List<String> loadAllInstalledModule(Context context) {
        PackageManager pm = context.getPackageManager();
        List<String> modulePathList = new ArrayList<>();
//        modulePathList.add("mnt/sdcard/app-debug.apk");

        List<String> packageNameList = loadPackageNameListFromFile(true);
        List<Pair<String, String>> installedModuleList = new ArrayList<>();

        boolean configFileExist = configFileExist();

        for (PackageInfo pkg : pm.getInstalledPackages(PackageManager.GET_META_DATA)) {
            ApplicationInfo app = pkg.applicationInfo;
            if (!app.enabled)
                continue;
            if (app.metaData != null && app.metaData.containsKey("xposedmodule")) {
                String apkPath = pkg.applicationInfo.publicSourceDir;
                String apkName = context.getPackageManager().getApplicationLabel(pkg.applicationInfo).toString();
                if (TextUtils.isEmpty(apkPath)) {
                    apkPath = pkg.applicationInfo.sourceDir;
                }
                if (!TextUtils.isEmpty(apkPath) && (!configFileExist || packageNameList == null || packageNameList
                        .contains(app.packageName))) {
                    XLog.d(TAG, " query installed module path -> " + apkPath);
                    modulePathList.add(apkPath);
                }
                installedModuleList.add(Pair.create(pkg.applicationInfo.packageName, apkName));
            }
        }

        final List<Pair<String, String>> installedModuleListFinal = installedModuleList;

        // ...
        // ...
        return modulePathList;
    }
```
## 加载插件Apk
找到了插件Apk之后，就可以得到此Apk的路径（data/app/包名 目录下面），然后就是根据此路径加载插件。  
加载插件的方法是：`com.wind.xposed.entry.XposedModuleLoader.loadModule()`  
其主要流程参考了原版Xposed框架中的实现，过程如下：
>1. 根据插件Apk文件路径构造DexClassLoader；
>2. 读取Apk asset目录下''assets/xposed_init'文件中所有的类名；
>3.  根据类名和Classloader构造入口类，并执行类的入口方法`handleLoadPackage`。  

流程源码和注释：
```
public static int loadModule(final String moduleApkPath, String moduleOdexDir, String moduleLibPath,
                                 final ApplicationInfo currentApplicationInfo, ClassLoader appClassLoader) {
        // ...

        // 创建DexClassLoader
        ClassLoader mcl = new DexClassLoader(moduleApkPath, moduleOdexDir, moduleLibPath, appClassLoader);
        // 读取asset目录中文件里写入的所有类名
        InputStream is = mcl.getResourceAsStream("assets/xposed_init");
        // ...

        BufferedReader moduleClassesReader = new BufferedReader(new InputStreamReader(is));
        try {
            String moduleClassName;
            while ((moduleClassName = moduleClassesReader.readLine()) != null) {
                moduleClassName = moduleClassName.trim();
                if (moduleClassName.isEmpty() || moduleClassName.startsWith("#"))
                    continue;

                try {
                    XLog.i(TAG, "  Loading class " + moduleClassName);
                    // 构造对象
                    Class<?> moduleClass = mcl.loadClass(moduleClassName);

                    if (!XposedHelper.isIXposedMod(moduleClass)) {
                        Log.i(TAG, "    This class doesn't implement any sub-interface of IXposedMod, skipping it");
                        continue;
                    } else if (IXposedHookInitPackageResources.class.isAssignableFrom(moduleClass)) {
                        Log.i(TAG, "    This class requires resource-related hooks (which are disabled), skipping it.");
                        continue;
                    }

                    final Object moduleInstance = moduleClass.newInstance();
                    if (moduleInstance instanceof IXposedHookZygoteInit) {
                        XposedHelper.callInitZygote(moduleApkPath, moduleInstance);
                    }

                  // 执行对象中的`handleLoadPackage`入口方法，实现hook流程
                    if (moduleInstance instanceof IXposedHookLoadPackage) {
                        // hookLoadPackage(new IXposedHookLoadPackage.Wrapper((IXposedHookLoadPackage) moduleInstance));
                        IXposedHookLoadPackage.Wrapper wrapper = new IXposedHookLoadPackage.Wrapper((IXposedHookLoadPackage) moduleInstance);
                        XposedBridge.CopyOnWriteSortedSet<XC_LoadPackage> xc_loadPackageCopyOnWriteSortedSet = new XposedBridge.CopyOnWriteSortedSet<>();
                        xc_loadPackageCopyOnWriteSortedSet.add(wrapper);
                        XC_LoadPackage.LoadPackageParam lpparam = new XC_LoadPackage.LoadPackageParam(xc_loadPackageCopyOnWriteSortedSet);
                        lpparam.packageName = currentApplicationInfo.packageName;
                        lpparam.processName = currentApplicationInfo.processName;
                        lpparam.classLoader = appClassLoader;
                        lpparam.appInfo = currentApplicationInfo;
                        lpparam.isFirstApplication = true;
                        XC_LoadPackage.callAll(lpparam);
                    }
                } catch (Throwable t) {
                }
            }
        } catch (IOException e) {
        } finally {
    }
```

# Apk中注入代码的实现
往Apk中注入代码，一般来说，有两种主流方法：
>1. 最常用的方法，使用ApkTool将Apk反编译为smali代码，修改smali文件，然后再将修改后的文件使用ApkTool打包，从而实现代码的修改；
>2. 修改[dex2jar](https://github.com/pxb1988/dex2jar)工程源码，使得在dex转换为jar过程中能够插入java代码，然后再使用jar2dex工具将修改后的jar转换为dex文件，从而实现代码修改和回编。

这里，我们选取了第二种方法。第二种方法的难点是如何修改dex2jar工程源码实现代码的插入。

为此，需要先分析其实现原理。
Claud大神开源的[dex2jar](https://github.com/pxb1988/dex2jar)工具大致原理是，先根据dex文件格式规则解析dex文件中的所有类信息，然后再利用ASM工具根据这些信息生成Class文件。  

对Java开发比较熟悉的人，应该很熟悉ASM。ASM是一个Java字节码操作框架。它可以直接对class文件进行增删改的操作，能被用来动态生成类或者增强既有类的功能。Java中许多的框架的实现是基于ASM，比如Java AOP的实现，JavaWeb开发中的Spring框架的实现等等。可以说ASM就是一把利剑，是深入Java必须学习的一个点。  

这里，我们就不讲解ASM的原理和用法，只讲解如何利用ASM修改dex2jar工程源码，从而实现代码的注入。
## ASM代码生成
在上一篇源码解析文章中，我们说过，破解Apk，只需要在其Application类中注入这样一段静态代码块:
```
package com.test;
import android.app.Application;
import com.wind.xposed.entry.XposedModuleEntry;

public class MyApplication extends Application {
    static {
        XposedModuleEntry.init();
    }
}
```
那这样的一段代码，如何用ASM工具生成呢。 
假如对ASM的API熟悉的话，其实很容易就能实现这样一小段代码的生成。  
假如不熟悉的话，也没关系，我们可以利用Android Studio中的一个插件，查看这段代码的ASM的实现。这个插件的名字是：ASM Bytecode Viewer

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xNjM5MjM4LWRmZjBmN2YyZDlmYWIyNTkucG5n)  
通过这个插件，我们可以清晰的看到生成这段代码的ASM代码的实现：
```
public class MyApplicationDump implements Opcodes {

    public static byte[] dump() throws Exception {

        ClassWriter cw = new ClassWriter(0);
        FieldVisitor fv;
        MethodVisitor mv;
        AnnotationVisitor av0;

        cw.visit(V1_7, ACC_PUBLIC + ACC_SUPER, "com/test/MyApplication", null, "android/app/Application", null);

        cw.visitSource("MyApplication.java", null);

        {
            mv = cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
            mv.visitCode();
            Label l0 = new Label();
            mv.visitLabel(l0);
            mv.visitLineNumber(7, l0);
            mv.visitVarInsn(ALOAD, 0);
            mv.visitMethodInsn(INVOKESPECIAL, "android/app/Application", "<init>", "()V", false);
            mv.visitInsn(RETURN);
            Label l1 = new Label();
            mv.visitLabel(l1);
            mv.visitLocalVariable("this", "Lcom/test/MyApplication;", null, l0, l1, 0);
            mv.visitMaxs(1, 1);
            mv.visitEnd();
        }
        {
            mv = cw.visitMethod(ACC_STATIC, "<clinit>", "()V", null, null);
            mv.visitCode();
            Label l0 = new Label();
            mv.visitLabel(l0);
            mv.visitLineNumber(11, l0);
            mv.visitMethodInsn(INVOKESTATIC, "com/wind/xposed/entry/XposedModuleEntry", "init", "()V", false);
            Label l1 = new Label();
            mv.visitLabel(l1);
            mv.visitLineNumber(12, l1);
            mv.visitInsn(RETURN);
            mv.visitMaxs(0, 0);
            mv.visitEnd();
        }
        cw.visitEnd();

        return cw.toByteArray();
    }
}

```
这段代码中，第一个花括号中代码用来生成这个类的默认构造方法，第二个花括号中是用来生成静态代码块方法，去掉生成标签行数等无关代码后，最终需要的代码仅仅是：
```
            mv = cw.visitMethod(ACC_STATIC, "<clinit>", "()V", null, null);
            mv.visitCode();
            mv.visitMethodInsn(INVOKESTATIC, "com/wind/xposed/entry/XposedModuleEntry", "init", "()V", false);
            mv.visitInsn(RETURN);
            mv.visitMaxs(0, 0);
            mv.visitEnd();
```
下面再分析如何将这段ASM代码加到dex2jar工程中，从而实现代码植入。
## 修改dex2jar源码
通过不断调试dex2jar源码，我们可以找到使用ASM生成字节码的代码位置，在`Dex2jar.java`文件的`doTranslate ()`方法中：
```
// dex2jar项目源码
// com.googlecode.d2j.dex.Dex2jar.java
private void doTranslate(final Path dist) throws IOException {
    // ...
    new ExDex2Asm(exceptionHandler) {
            public void convertCode(DexMethodNode methodNode, MethodVisitor mv) {
                if ((readerConfig & DexFileReader.SKIP_CODE) != 0 && methodNode.method.getName().equals("<clinit>")) {
                    // also skip clinit
                    return;
                }
                super.convertCode(methodNode, mv);
            }
            @Override
            public void optimize(IrMethod irMethod) {
                // ...
                // ...
            }
            @Override
            public void ir2j(IrMethod irMethod, MethodVisitor mv) {
                new IR2JConverter(0 != (V3.OPTIMIZE_SYNCHRONIZED & v3Config)).convert(irMethod, mv);
            }
        }.convertDex(fileNode, cvf);
    // ...
}
```
`ExDex2Asm`方法`convertCode`是其父类中对外暴露的方法，用于处理每个方法生成。  
在这里，我们可以判断当前类是不是应用的Application类，以及方法是不是静态代码块方法`<clinit>`， 是的话，通过`visitMethodInsn`加上`XposedModuleEntry.init();`方法，代码如下：
```
    if (methodNode.method.getOwner().equals(applicationName) && methodNode.method.getName().equals("<clinit>")) {
            isApplicationClassFounded = true;
            mv.visitMethodInsn(Opcodes.INVOKESTATIC, XPOSED_ENTRY_CLASS_NAME, "init", "()V", false);
    }
```
还有另外一种情形，也需要处理，就是当前应用自定义的Application类没有方法静态方法块的情形。对于这种情形的处理，仅修改`ExDex2Asm`类中的代码，显然无法实现。我们需要在其父类`Dex2Asm`中增加一个非私有的空方法，暴露给子类`ExDex2Asm`。这个方法需要包含类的节点信息`DexClassNode `和ASM代码生成对象`ClassVisitor `。  
通过分析`Dex2Asm`类中代码，最终选择了在其`convertClass`方法后面的位置调用此方法，代码如下：
```java
   // com.googlecode.d2j.dex.Dex2jar.java
    public void convertClass(int dexVersion, DexClassNode classNode, 
        ClassVisitorFactory cvf, Map<String, Clz> classes) {
accept(classNode.anns, cv);
        // ...
        if (classNode.fields != null) {
            for (DexFieldNode fieldNode : classNode.fields) {
                convertField(classNode, fieldNode, cv);
            }
        }
       // 在这里调用新增加的方法
        addMethod(classNode, cv);

        if (classNode.methods != null) {
            for (DexMethodNode methodNode : classNode.methods) {
                convertMethod(classNode, methodNode, cv);
            }
        }
        cv.visitEnd();
    }

    // 这是新增加的方法，具体实现在子类中
    public void addMethod(DexClassNode classNode, ClassVisitor cv) {
    }
```
在`addMethod`具体实现中，先判断当前类是Application类，然后再遍历类的所有方法，如果没有静态代码块方法，通过ASM加上静态代码块方法，这段增加方法的ASM代码，就是上面用Android Studio中的ASM插件生成的。  
最终完整代码如下：
```
// 修改后的dex2jar项目代码
// com.googlecode.d2j.dex.Dex2jar.java
new ExDex2Asm(exceptionHandler) {
            public void convertCode(DexMethodNode methodNode, MethodVisitor mv) {
                // 增加的代码，用于在Application静态代码块中增加XposedModuleEntry.init();
                if (methodNode.method.getOwner().equals(applicationName) && methodNode.method.getName().equals("<clinit>")) {
                    isApplicationClassFounded = true;
                    mv.visitMethodInsn(Opcodes.INVOKESTATIC, XPOSED_ENTRY_CLASS_NAME, "init", "()V", false);
                }

                if ((readerConfig & DexFileReader.SKIP_CODE) != 0 && methodNode.method.getName().equals("<clinit>")) {
                    // also skip clinit
                    return;
                }
                super.convertCode(methodNode, mv);
            }

            // 增加的代码
            @Override
            public void addMethod(com.googlecode.d2j.node.DexClassNode classNode, ClassVisitor cv) {
                // 找到应用的Application类
                if (classNode.className.equals(applicationName)) {
                    isApplicationClassFounded = true;

                    boolean hasFoundClinitMethod = false;
                    if (classNode.methods != null) {
                         // 判断是否存在静态代码块
                        for (DexMethodNode methodNode : classNode.methods) {
                            if (methodNode.method.getName().equals("<clinit>")) {
                                hasFoundClinitMethod = true;
                                break;
                            }
                        }
                    }

                    // 通过ASM增加静态代码块方法，并注入初始化方法XposedModuleEntry.init();
                    if (!hasFoundClinitMethod) {
                        MethodVisitor mv = cv.visitMethod(Opcodes.ACC_STATIC, "<clinit>", "()V", null, null);
                        mv.visitCode();
                        mv.visitMethodInsn(Opcodes.INVOKESTATIC, XPOSED_ENTRY_CLASS_NAME, "init", "()V", false);
                        mv.visitInsn(Opcodes.RETURN);
                        mv.visitMaxs(0, 0);
                        mv.visitEnd();
                    }
                }
            }

            @Override
            public void optimize(IrMethod irMethod) {
                // ...
            }

            @Override
            public void ir2j(IrMethod irMethod, MethodVisitor mv) {
                new IR2JConverter(0 != (V3.OPTIMIZE_SYNCHRONIZED & v3Config)).convert(irMethod, mv);
            }
        }.convertDex(fileNode, cvf);
}
```
此外，`Dex2Jar`类对象`applicationName`是从外面传入的应用定义的Application类全名，在`Dex2jarCmd`类中传入，`Dex2jarCmd`类的修改点如下：
```
//  com.googlecode.dex2jar.tools.Dex2jarCmd.java
public class Dex2jarCmd extends BaseCmd {
    // ...
    // ...
    // 新增的命令行参数，用于传应用的Application全类名
    @Opt(opt = "app", longOpt = "applicationName", description = "application full name that method should be insert into", 
        argName = "application-name")
    private String applicationName;

    protected void doCommandLine() throws Exception {
        // ...
        // ...
        dex2jar = Dex2jar.from(reader);
        dex2jar.withExceptionHandler(handler).reUseReg(reuseReg).topoLogicalSort()
               .skipDebug(!debugInfo).optimizeSynchronized(this.optmizeSynchronized).printIR(printIR)
                .noCode(noCode).skipExceptions(skipExceptions)
                .setApplicationName(applicationName).to(file);  // 新增的代码
        // ...
        // ...
    }
    ...

    // 新增的方法，用于暴露给外面，判断当前Dex中是否存在应用的Application类
    public boolean isApplicationClassFounded() {
        if (dex2jar == null) {
            return false;
        }
        return dex2jar.isApplicationClassFounded();
    }
}
```
`Dex2jar`类增加的两个成员变量和相关方法如下：
```
// 修改后的dex2jar项目代码
// com.googlecode.d2j.dex.Dex2jar.java
public class Dex2jar {
    // ...
    // ...
    // 新增的两个成员变量
    private String applicationName;
    private boolean isApplicationClassFounded = false;

   // 增加应用application的名称
    public Dex2jar setApplicationName(String appName) {
        this.applicationName = appName;
        applicationName = applicationName.replace('.', '/');
        if (!applicationName.endsWith(";")) {
            applicationName += ";";
        }
        if (!applicationName.startsWith("L")) {
            applicationName = "L" + applicationName;
        }
        return this;
    }

    public boolean isApplicationClassFounded() {
        return isApplicationClassFounded;
    }
    // ...
    // ...
}
```
至此，我们完成了dex2jar工程的改造，顺利实现了给一个Apk注入代码。


# 打包及签名流程
有了上面的准备工作后，我们来分析Xpatch源码中，调用dex2jar工具修改apk流程，以及对修改后的apk打包签名的流程。

Xpatch源码的入口类`MainCommand`，其核心方法是`doCommandLine()`。  
在`doCommandLine()`方法的主流程执行之前，先做了以下准备工作：`
>  1. 解析命令行参数，主要是包括原Apk路径和生成的Apk路径；
>  2. 解析Apk压缩包，读取dex文件的个数；
>  3. 通过AxmlPrinter2工具解析Manifest文件中的Application全类名；

以上准备工具完成后，通过三个task处理Apk文件，源码如下：
```
        // 1. modify the apk dex file to make xposed can run in it
        mXpatchTasks.add(new ApkModifyTask(showAllLogs, keepBuildFiles, unzipApkFilePath, applicationName,
                dexFileCount));

        // 2. copy xposed so and dex files into the unzipped apk
        mXpatchTasks.add(new SoAndDexCopyTask(dexFileCount, unzipApkFilePath, getXposedModules(xposedModules)));

        // 3. compress all files into an apk and then sign it.
        mXpatchTasks.add(new BuildAndSignApkTask(keepBuildFiles, unzipApkFilePath, output));

        // 4. excute these tasks
        for (Runnable executor : mXpatchTasks) {
            executor.run();
        }
```
这三个task的作用分别是：
>1. 利用修改后的dex2jar工具和jar2dex工具修改Apk中应用Application类的代码；
>2. 将用于加载Xposed插件的dex文件和so文件复制到Apk解压后的文件目录下；
>3. 将Apk解压后的文件目录重新压缩为zip压缩包，并重新签名。

第二个task和第三个task比较简单，这里就不一一分析。  
主要分析一下第一个task，修改Apk源码的task:  `ApkModifyTask `。
`ApkModifyTask `的核心流程是遍历Apk解压出来的所有dex文件，对每个dex文件执行`Dex2jarCmd `，这个cmd的作用就是找到dex中应用的Application类，并插入代码，如果找到，就不继续处理下一个dex文件，因为每个App只有一个Application类，代码细节如下：
```
   private String dumpJarFile(int dexFileCount, String dexFilePath, String jarOutputPath, String applicationName) {
        ArrayList<String> dexFileList = createClassesDotDexFileList(dexFileCount);
        for (String dexFileName : dexFileList) {
            String filePath = dexFilePath + dexFileName;
            // 执行dex2jar命令，修改源代码
            boolean isApplicationClassFound = dex2JarCmd(filePath, jarOutputPath, applicationName);
            // 找到了目标应用主application的包名，说明代码注入成功，则返回当前dex文件
            if (isApplicationClassFound) {
                return dexFileName;
            }
        }
        return "";
    }

    private boolean dex2JarCmd(String dexPath, String jarOutputPath, String applicationName) {
        Dex2jarCmd cmd = new Dex2jarCmd();
        String[] args = new String[]{
                dexPath,
                "-o",
                jarOutputPath,
                "-app",
                applicationName,
                "--force"
        };
        cmd.doMain(args);

        // 执行完命令后，会返回查找Application Class的结果
        boolean isApplicationClassFounded = cmd.isApplicationClassFounded();
        if (showAllLogs) {
            System.out.println("isApplicationClassFounded ->  " + isApplicationClassFounded + "the dexPath is  " +
                    dexPath);
        }
        return isApplicationClassFounded;
    }
```
使用dex2jar修改完Apk的Application类之后，得到的是一个jar文件，再通过jar2dex工具转为dex文件：
```
private void jar2DexCmd(String jarFilePath, String dexOutPath) {
        Jar2Dex cmd = new Jar2Dex();
        String[] args = new String[]{
                jarFilePath,
                "-o",
                dexOutPath
        };
        cmd.doMain(args);
    }
```
最后删除生成的jar文件，新的dex文件就是完成代码注入后的dex。
最后，将这些dex文件和so文件压缩为Apk文件，并签名。

至此，完成Apk的篡改，并实现App启动时，加载设备上已安装的所有Xposed插件模块。
# 总结
最后，归纳一下Xpatch破解App的整体流程：
> 1. 利用Android Art Hook框架（whale或者SandHook），开发能够加载Xposed模块的Apk，并导出其中的dex和so文件；
> 2. 修改dex2jar工具，以实现在dex转换为jar的过程中，查找App的主Application类，并在此类中插入一段静态代码块，实现加载Xposed模块；
> 3. 将修改后的dex和加载Xposed模块的dex和so文件一起打包签名，从而完成代码注入，实现Xposed模块的加载。


欢迎扫二维码，关注我的技术公众号**Android葵花宝典**  ，获取高质量的Android干货分享：
 
![image](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xNjM5MjM4LTRkMzM1NzA4ZTFmY2MyNTI)