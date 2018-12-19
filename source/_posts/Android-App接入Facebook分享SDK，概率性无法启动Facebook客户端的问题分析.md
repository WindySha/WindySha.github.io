---
title: Android App接入Facebook分享SDK时，无法启动Facebook客户端的问题分析
date: 2018-12-20 02:07:45
categories:
- Android
- 问题分析
tags:
- Share
- Facebook
- 分享
---

## 问题来源
   **由**于我司的android app产品主要是面向海外，因此，app中的分享功能[接入facebook分享][1]是必不可少的。最近在接入[facebook android sdk][2]进行分享时，发现一个非常奇怪的现象，明明手机上已经安装了facebook客户端，但却经常出现无法调起客户端分享，而是调起了facebook sdk内置的网页分享。
   
   在网页端分享时，用户又需要重新输入账号密码才能分享（客户端不用，因为一般都已经登录过）。这样用户体验就非常差，进而会导致很多用户会因为要输入账户密码而放弃分享。因此，这是一个严重影响体验的问题，需要紧急修复。
   
本文主要就是介绍该问题的分析思路，然后给出一个比较完美的解决方案，给同样遇到此坑的程序员们提供一个思路。

<!-- more -->

## 问题分析
为了分析此问题，我们需要进入sdk代码内部，借助android studio debug工具，debug每一个流程，观察是在哪个流程出了问题。我这里接入的facebook-android-sdk版本是当前最新的版本, 4.38.1，以下引用的sdk代码都是基于该版本。
首先看看分享功能的简单实现：

### 1. 分享功能的实现
这里我们已分享一个链接为例，展示sdk的调用方式。

分享链接时，首先new一个ShareLinkContent([参考文档][3])，然后new一个ShareDialog，并调用其show方法，这样就能启动facebook分享了，具体代码如下：
```
 ShareDialog shareDialog = new ShareDialog(activity)
 ShareLinkContent content = new ShareLinkContent.Builder()
        .setContentUrl(Uri.parse("https://developers.facebook.com"))
        .build();
 shareDialog.show(content, ShareDialog.Mode.AUTOMATIC)
```
按照正常流程，用户手机安装了facebook app就应该启动facebook app分享，未安装app就启动网页，让用户在网页里登录后再分享。但实际情况是，facebook app被杀死时，无法启动facebook app分享，这是为何呢？难道facebook sdk里有严重的bug吗？下面，来追溯源码，分析问题本因。

### 2.源码分析
上面的shareDialog的show方法最终调用到了showImpl方法中，其具体的实现为：
```
// Pass in BASE_AUTOMATIC_MODE when Automatic mode choice is desired
    protected void showImpl(final CONTENT content, final Object mode) {
        AppCall appCall = createAppCallForMode(content, mode);
        if (appCall != null) {
            if (fragmentWrapper != null) {
                DialogPresenter.present(appCall, fragmentWrapper);
            } else {
                DialogPresenter.present(appCall, activity);
            }
        } else {
            // If we got a null appCall, then the derived dialog code is doing something wrong
            String errorMessage = "No code path should ever result in a null appCall";
            Log.e(TAG, errorMessage);
            if (FacebookSdk.isDebugEnabled()) {
                throw new IllegalStateException(errorMessage);
            }
        }
    }
```
在showImpl方法中构造了一个AppCall，AppCall是最终页面跳转的地方，也是intent的简单包装，所以，需要弄清楚AppCall是怎么构造出来的。
先看上面的`createAppCallForMode`方法是如何实现的：
```
private AppCall createAppCallForMode(final CONTENT content, final Object mode) {
        boolean anyModeAllowed = (mode == BASE_AUTOMATIC_MODE);

        AppCall appCall = null;
        for (ModeHandler handler : cachedModeHandlers()) {
            if (!anyModeAllowed && !Utility.areObjectsEqual(handler.getMode(), mode)) {
                continue;
            }
            if (!handler.canShow(content, true /*isBestEffort*/)) {
                continue;
            }

            try {
                appCall = handler.createAppCall(content);
            } catch (FacebookException e) {
                appCall = createBaseAppCall();
                DialogPresenter.setupAppCallForValidationError(appCall, e);
            }
            break;
        }

        if (appCall == null) {
            appCall = createBaseAppCall();
            DialogPresenter.setupAppCallForCannotShowError(appCall);
        }

        return appCall;
    }
```
通过上面代码可知，AppCall是通过ModeHandler的createAppCall方法来创建的，前提是这个handler需要满足上面的两个if条件才行。那么这里的cachedModeHandlers()里面缓存了哪些handler呢？
再看代码：
```
private List<ModeHandler> cachedModeHandlers() {
        if (modeHandlers == null) {
            modeHandlers = getOrderedModeHandlers();
        }

        return modeHandlers;
    }
```
getOrderedModeHandlers()是抽象方法，在ShareDialog中的实现是：
```
    @Override
    protected List<ModeHandler> getOrderedModeHandlers() {
        ArrayList<ModeHandler> handlers = new ArrayList<>();
        handlers.add(new NativeHandler());
        handlers.add(new FeedHandler()); // Feed takes precedence for link-shares for Mode.AUTOMATIC
        handlers.add(new WebShareHandler());
        handlers.add(new CameraEffectHandler());
        handlers.add(new ShareStoryHandler());//Share into story

        return handlers;
    }
```
原来handler有这么多个，不过，通过名字可知我们需要的是第一个handler，即NativeHandler()，本地调用，也就是调用本地客户端。问题应该是出在了这里，本应使用NativeHandler构造一个AppCall，实际上使用了WebShareHander。

下面，再看NativeHandler的具体实现：
我们具体只看它的canShow()方法，因为肯定是因为canShow方法返回了false，导致无法调用其createAppCall方法。
```
private class NativeHandler extends ModeHandler {
        @Override
        public Object getMode() {
            return Mode.NATIVE;
        }

        @Override
        public boolean canShow(final ShareContent content, boolean isBestEffort) {
            if (content == null || (content instanceof ShareCameraEffectContent)
                || (content instanceof ShareStoryContent)) {
                return false;
            }

            boolean canShowResult = true;
            if (!isBestEffort) {
                if (content.getShareHashtag() != null) {
                    canShowResult = DialogPresenter.canPresentNativeDialogWithFeature(
                        ShareDialogFeature.HASHTAG);
                }
                if ((content instanceof ShareLinkContent) &&
                    (!Utility.isNullOrEmpty(((ShareLinkContent)content).getQuote()))) {
                    canShowResult &= DialogPresenter.canPresentNativeDialogWithFeature(
                        ShareDialogFeature.LINK_SHARE_QUOTES);
                }
            }
            return canShowResult && ShareDialog.canShowNative(content.getClass());
        }

        @Override
        public AppCall createAppCall(final ShareContent content) {
            // 代码省略，这个就是构造跳转到facebook客户端的intent
        }
    }
```
上面的createAppCallForMode中调用canShow方法时，传入的isBestEffort是true：
```
if (!handler.canShow(content, true /*isBestEffort*/)) {
                continue;
            }
```
因此` if (!isBestEffort) `中的代码不会执行，只需看`ShareDialog.canShowNative(content.getClass())`这个方法就返回结果就行。
```
// ShareDialog.java
private static boolean canShowNative(Class<? extends ShareContent> contentType) {
        DialogFeature feature = getFeature(contentType);

        return feature != null && DialogPresenter.canPresentNativeDialogWithFeature(feature);
}

    
// DialogPresenter.java
public static boolean canPresentNativeDialogWithFeature(
            DialogFeature feature) {
        return getProtocolVersionForNativeDialog(feature).getProtocolVersion()
                != NativeProtocol.NO_PROTOCOL_AVAILABLE;
}


// DialogPresenter.java
public static NativeProtocol.ProtocolVersionQueryResult getProtocolVersionForNativeDialog(
            DialogFeature feature) {
        String applicationId = FacebookSdk.getApplicationId();
        String action = feature.getAction();
        int[] featureVersionSpec = getVersionSpecForFeature(applicationId, action, feature);

        return NativeProtocol.getLatestAvailableProtocolVersionForAction(
                action,
                featureVersionSpec);
    }
    
   
// NativeProtocol.java
public static ProtocolVersionQueryResult getLatestAvailableProtocolVersionForAction(
        String action,
        int[] versionSpec) {
        List<NativeAppInfo> appInfoList = actionToAppInfoMap.get(action);
        return getLatestAvailableProtocolVersionForAppInfoList(appInfoList, versionSpec);
    }
    
    
// NativeProtocol.java
private static ProtocolVersionQueryResult getLatestAvailableProtocolVersionForAppInfoList(
        List<NativeAppInfo> appInfoList,
        int[] versionSpec) {
        // Kick off an update
        updateAllAvailableProtocolVersionsAsync();

        if (appInfoList == null) {
            return ProtocolVersionQueryResult.createEmpty();
        }

        // Could potentially cache the NativeAppInfo to latestProtocolVersion
        for (NativeAppInfo appInfo : appInfoList) {
            int protocolVersion =
                computeLatestAvailableVersionFromVersionSpec(
                    appInfo.getAvailableVersions(),
                    getLatestKnownVersion(),
                    versionSpec);

            if (protocolVersion != NO_PROTOCOL_AVAILABLE) {
                return ProtocolVersionQueryResult.create(appInfo, protocolVersion);
            }
        }

        return ProtocolVersionQueryResult.createEmpty();
    }
```
通过上面一系列调用可知，最终是通过appInfo.getAvailableVersions()得到的版本号计算的到一个协议版本(protocolVersion)，如果取到了protocolVersion值（不是NO_PROTOCOL_AVAILABLE），就是返回一个结果，canShow方法就返回true。显然，问题就出在这里，这里获取不到protocolVersion的值。

那么，appInfo.getAvailableVersions()这里是如何得到AvailableVersion的呢？
是通过`updateAllAvailableProtocolVersionsAsync()`方法吗？
显然不是，因为这个方法是在异步线程中执行的，因此本段代码执行结束后才会执行线程里面的异步方法，这里，显然是为了更新之前已经取到的AvailableVersion的值。

再看看`updateAllAvailableProtocolVersionsAsync()`方法被哪里调用过，发现在sdkInitialize方法里有调用：
```
public static synchronized void sdkInitialize(
            final Context applicationContext,
            final InitializeCallback callback) {
            ...省略代码
            // Fetch available protocol versions from the apps on the device
            NativeProtocol.updateAllAvailableProtocolVersionsAsync();
            ...省略代码
        }
```
原来，是在应用启动初始化SDk时异步获取的。
这样做应该是为了提高分享时界面响应速度，避免在主线程进行contentProvider跨进程调用耗时，导致主线程阻塞。

下面在看看异步线程里到底做了什么耗时的操作：
```
public static void updateAllAvailableProtocolVersionsAsync() {
        if (!protocolVersionsAsyncUpdating.compareAndSet(false, true)) {
            return;
        }

        FacebookSdk.getExecutor().execute(new Runnable() {
            @Override
            public void run() {
                try {
                    for (NativeAppInfo appInfo : facebookAppInfoList) {
                        appInfo.fetchAvailableVersions(true);
                    }
                } finally {
                    protocolVersionsAsyncUpdating.set(false);
                }
            }
        });
    }
```
最终调用到NativeAppInfo的fetchAvailableVersions方法：
```
private synchronized void fetchAvailableVersions(boolean force) {
            if (force || availableVersions == null) {
                availableVersions = fetchAllAvailableProtocolVersionsForAppInfo(this);
            }
        }
```
fetchAllAvailableProtocolVersionsForAppInfo方法的实现为：
```
private static TreeSet<Integer> fetchAllAvailableProtocolVersionsForAppInfo(
        NativeAppInfo appInfo) {
        TreeSet<Integer> allAvailableVersions = new TreeSet<>();

        Context appContext = FacebookSdk.getApplicationContext();
        ContentResolver contentResolver = appContext.getContentResolver();

        String [] projection = new String[]{ PLATFORM_PROVIDER_VERSION_COLUMN };
        Uri uri = buildPlatformProviderVersionURI(appInfo);
        Cursor c = null;
        try {
            // First see if the base provider exists as a check for whether the native app is
            // installed. We do this prior to querying, to prevent errors from being output to
            // logcat saying that the provider was not found.
            PackageManager pm = FacebookSdk.getApplicationContext().getPackageManager();
            String contentProviderName = appInfo.getPackage() + PLATFORM_PROVIDER;
            ProviderInfo pInfo = null;
            try {
                pInfo = pm.resolveContentProvider(contentProviderName, 0);
            } catch (RuntimeException e) {
                Log.e(TAG, "Failed to query content resolver.", e);
            }
            if (pInfo != null) {
                try {
                    c = contentResolver.query(uri, projection, null, null, null);
                } catch (NullPointerException|SecurityException|IllegalArgumentException ex) {
                    Log.e(TAG, "Failed to query content resolver.");
                    c = null;
                }

                if (c != null) {
                    while (c.moveToNext()) {
                        int version = c.getInt(c.getColumnIndex(PLATFORM_PROVIDER_VERSION_COLUMN));
                        allAvailableVersions.add(version);
                    }
                }
            }
        } finally {
            if (c != null) {
                c.close();
            }
        }

        return allAvailableVersions;
    }
```
这段代码的意图很明显，就是通过contentProvider跨进程调用，获取当前安装的facebook app支持的sdk分享的版本号。
通过debug这段代码，可知ConentProvider调用的主要参数为：
```
URI = "content://com.facebook.katana.provider.PlatformProvider/versions"
projection = {"version"}
packageName = "com.facebook.katana"
contentProviderName = "com.facebook.katana.provider.PlatformProvider"
```
ContentProvider返回的结果为：
![屏幕快照 2018-11-25 23.31.06.png-173.3kB][4]
这些日期信息，猜测应该是当前的app版本支持的分享sdk版本的日期。

## 问题发现
再次通过断点调试无法掉起facebook客户端时，方法`fetchAllAvailableProtocolVersionsForAppInfo`的返回结果，我们发现，这里返回的是cursor是空，这样其返回值`allAvailableVersions`（TreeSet<Integer>），也是空的（内容是空，对象不为空）。
这样：
```
public static ProtocolVersionQueryResult getLatestAvailableProtocolVersionForAction(
        String action,
        int[] versionSpec) {
        List<NativeAppInfo> appInfoList = actionToAppInfoMap.get(action);
        return getLatestAvailableProtocolVersionForAppInfoList(appInfoList, versionSpec);
    }
```
这里返回的结果也就是`NO_PROTOCOL_AVAILABLE`
`canShowNative`返回的也就是false，从而导致无法使用`NativeHandler`来创建`AppCall`,进而无法掉起facebook客户端分享。
## 问题原因
明明客户端已经按安装，为什么通过contentProvider无法获取facebook客户端的存储的内容呢？并且，只是概率性问题。
再仔细测试问题，我们发现，在google的pixel 2手机上没有此问题，在主流国产手机（魅族，小米等）上都有此问题。
然而，当打开手机设置里（或者手机管家等系统app里）facebook自启动权限（或者应用间互相启动权限）之后，就没有问题了，这说明当app没有自启动权限时，该app进程被杀掉（没有做保活）后，其他app是无法通过service, broadcast receiver, contentProvider启动该app进程（通过activity可以）。为了验证这一点，本人亲自写了个demo验证了一番，结论的确是如此！！至此，我们发现了根本问题原因。

四大组件中三个跨进程调用时，很有可能会调用失败，这样也太不靠谱了。难怪国内的很多app要做保活的功能，这应该也是主要原因之一。
## 问题修复
既然知道问题原因，修复起来就比较Easy了。既然无法通过contentProvider启动facebook进程，那我们通过startActivty先启动facebook app再走分享的流程不就行了。
这样又有一个不太友好的体验，就是界面会跳转两次，先跳到facebook主界面，再跳到facebook分享界面。但是，这种问题确实也无法规避，只能跳转两次。

既然无法规避，我们可以通过判断facebook进程是否启动，来决定是否需要主动启动facebook app后再分享。这样可以降低跳转两次的频率，用户体验较好。

那问题又来了，如何判断一个app进程是否启动了呢？有没有现成的api？
google一搜，发现还真有：

```
//ActivityManager.java
@Deprecated
    public List<RunningTaskInfo> getRunningTasks(int maxNum)
            throws SecurityException {
        try {
            return getService().getTasks(maxNum, 0);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
但是很抱歉，此方法的注释里有一段很重要的话：
```
 @deprecated As of {@link android.os.Build.VERSION_CODES#LOLLIPOP}, this method
     * is no longer available to third party
     * applications: the introduction of document-centric recents means
     * it can leak person information to the caller.  For backwards compatibility,
     * it will still retu rn a small subset of its data: at least the caller's
     * own tasks, and possibly some other tasks
     * such as home that are known to not be sensitive.
```
此方法已经不对第三方应用开放，它会泄露用户信息给调用者，现在只会返回调用者自己的任务栈信息了。OMG！此路不同！那有没有其他办法呢？
## 另辟蹊径
我们是如何知道facebook进程无法被启动的呢？
再回过头看看上面的分析过程，是因为通过contentProvider无法获取到facebook app里的数据，我们得知facebook进程没有被调起。

既然如此，那我们何不直接copy sdk中通过contentProvider获取facebook数据的代码，自己先获取一遍，取到了`allAvailableVersions`，说明facebook进程已经启动了可以走正常分享流程，没有取到，先启动facebook的主activity，再走正常的分享流程。

多说无益，直接看代码吧:（Talk is cheap,show me the code！）
kolin代码：

```
object FacebookShareWrapper {

    private val FACEBOOK_PACKAGE_NAME = "com.facebook.katana"

    private val TAG = "FacebookShareWrapper"

    fun startFacebookShareChecking(activity: Activity, shareAction: (() -> Unit)?): Disposable? {

        //这一行是从NativeProtocol中fetchAllAvailableProtocolVersionsForAppInfo方法复制过来的代码
        val versionSet = FacebookProtocolVersionHelper.fetchAllAvailableProtocolVersionsForAppInfo(activity, FACEBOOK_PACKAGE_NAME)

        val am = activity.getSystemService(Context.ACTIVITY_SERVICE) as ActivityManager

        var isFacebookStarted = false

        //facebook未安装，或者facebook进程无法启动
        if (versionSet.size <= 0) {
            val intent = activity.packageManager.getLaunchIntentForPackage(FACEBOOK_PACKAGE_NAME)
            if (intent != null) {
                //Facebook进程无法启动，启动它吧！！
                try {
                    isFacebookStarted = true
                    activity.startActivity(intent)
                } catch (e: Exception) {
                    isFacebookStarted = false
                    Log.e(TAG, " start activity failed intent = $intent, error msg = ${e.message}", e)
                }
            }
        }

        //facebook没有被启动，走正常分享流程就行
        if (!isFacebookStarted) {
            shareAction?.invoke()
            return null
        }

        //检查facebook启动情况的标志
        var intervalCheckFacebookFlag = false

        //一直轮询，直到可以取到facebook的ProtocolVersion数据
        return Flowable.intervalRange(0, 20, 0, 20, TimeUnit.MILLISECONDS, AndroidSchedulers.mainThread())
                .subscribe {
                    if (!intervalCheckFacebookFlag) {
                        val versionSet = FacebookProtocolVersionHelper.fetchAllAvailableProtocolVersionsForAppInfo(activity, FACEBOOK_PACKAGE_NAME)
                        if (versionSet.size > 0) {
                            intervalCheckFacebookFlag = true

                            //将应用界面移到前台，并开始facebook分享
                            am.moveTaskToFront(activity.taskId, 0)
                            shareAction?.invoke()
                        }
                    }
                }
    }
}    
```
使用的时候，只用在原来的分享流程上面包一层就行了，非常easy:
```
FacebookShareWrapper.startFacebookShareChecking(activity!!) {
                val shareDialog = ShareDialog(activity)
                val content = ShareLinkContent.Builder()
                        .setContentUrl(Uri.parse("https://developers.facebook.com"))
                        .build()
                shareDialog.show(content, ShareDialog.Mode.AUTOMATIC)
            }
```


## 新的问题
经过多轮测试，上面的修改方式的确在能够大多数情况下能够调起客户端分享。但是，还是有非常小的概率出现先启动了facebook app，再打开facebook网页分享，而不是客户端分享！！这又是哪里出了问题呢？

我们再回头zai看看`getLatestAvailableProtocolVersionForAppInfoList`方法：
```
private static ProtocolVersionQueryResult getLatestAvailableProtocolVersionForAppInfoList(
        List<NativeAppInfo> appInfoList,
        int[] versionSpec) {
        // 这里使用线程池异步获取ProtocolVersions信息
        // Kick off an update
        updateAllAvailableProtocolVersionsAsync();

        if (appInfoList == null) {
            return ProtocolVersionQueryResult.createEmpty();
        }

        // Could potentially cache the NativeAppInfo to latestProtocolVersion
        for (NativeAppInfo appInfo : appInfoList) {
            int protocolVersion =
                computeLatestAvailableVersionFromVersionSpec(
                    appInfo.getAvailableVersions(), 
                    getLatestKnownVersion(),
                    versionSpec);

            if (protocolVersion != NO_PROTOCOL_AVAILABLE) {
                return ProtocolVersionQueryResult.create(appInfo, protocolVersion);
            }
        }

        return ProtocolVersionQueryResult.createEmpty();
    }
```

`updateAllAvailableProtocolVersionsAsync`方法的具体实现为：
```
public static void updateAllAvailableProtocolVersionsAsync() {
        if (!protocolVersionsAsyncUpdating.compareAndSet(false, true)) {
            return;
        }

        FacebookSdk.getExecutor().execute(new Runnable() {
            @Override
            public void run() {
                try {
                    for (NativeAppInfo appInfo : facebookAppInfoList) {
                        appInfo.fetchAvailableVersions(true);
                    }
                } finally {
                    protocolVersionsAsyncUpdating.set(false);
                }
            }
        });
    }
```
这里也是调用了
```
    appInfo.fetchAvailableVersions(true);
```

`getAvailableVersions`方法的具体实现为：
```
private static abstract class NativeAppInfo {
        abstract protected String getPackage();
        abstract protected String getLoginActivity();

        private TreeSet<Integer> availableVersions;

        public TreeSet<Integer> getAvailableVersions() {
            if (availableVersions == null) {
                fetchAvailableVersions(false);
            }
            return availableVersions;
        }

        private synchronized void fetchAvailableVersions(boolean force) {
            if (force || availableVersions == null) {
                availableVersions = fetchAllAvailableProtocolVersionsForAppInfo(this);
            }
        }
    }
```
再看看`updateAllAvailableProtocolVersionsAsync`这个方法有哪些地方调用过了：

![屏幕快照 2018-11-26 10.58.49.png-188.4kB][5]

`sdkInitialize`中的关键代码为：
```
public static synchronized void sdkInitialize(
            final Context applicationContext,
            final InitializeCallback callback){
            // 省略其他代码
             // Fetch available protocol versions from the apps on the device
            NativeProtocol.updateAllAvailableProtocolVersionsAsync();
           // 省略其他代码
            }
```

我们发现，在sdkInitialize方法中它也被调用过，而sdkInitialize是在Application启动时被初始化的，因此这个方法在应用刚启动时被调用了，但此时facebook是未启动的，并且无法被contentProvider拉起，因此返回的ProtocolVersions是空的，这里请注意，contentProvider查到的cursor为空时，**返回一个size为0的TreeSet<Integer>给availableVersions**，此时，NativeAppInfo中的availableVersions是非空的**（not null, but size = 0)**，这样，调用其`getAvailableVersions`返回的是大小为空的非空对象，因此不会执行``fetchAvailableVersions`方法，无获取到真实的`availableVersions`，进而无法facebook客户端分享。

这个bug显然是facebook share sdk自己的bug，显然没有考虑到app启动时无法调起facebook，分享时才调起facebook的流程，在sdk代码里直接修改的话，是非常方便的, 只需要在：NativeAppInfo的两个方法里加上`size=0`的判断就行了：
```
private static abstract class NativeAppInfo {
        abstract protected String getPackage();
        abstract protected String getLoginActivity();

        private TreeSet<Integer> availableVersions;

        public TreeSet<Integer> getAvailableVersions() {
            // 这里加上size() = 0的条件
            if (availableVersions == null || availableVersions.size() = 0) {
                fetchAvailableVersions(false);
            }
            return availableVersions;
        }

        private synchronized void fetchAvailableVersions(boolean force) {
            // 这里加上size() = 0的条件
            if (force || availableVersions == null ||  || availableVersions.size() = 0) {
                availableVersions = fetchAllAvailableProtocolVersionsForAppInfo(this);
            }
        }
    }
```
这里，我给facebook share sdk的GitHub代码库提交了一个Pull Request：
[https://github.com/facebook/facebook-android-sdk/pull/538][6]
提交几天之后，就被facebook的开发人员合并到主分支中。

## 完整修复
虽然提交了PR，但他们不会立即为了你出一个版本。
所以，还是要想办法在我们自己app里来修复。其实，我们只需要通过反射将我们上面通过contentProvider获取到的availableVersion个设到NativeAppInfo中就行了，非常容易实现。
完整的修改代码如下(Kotlin)：
```
object FacebookShareWrapper {

    private val FACEBOOK_PACKAGE_NAME = "com.facebook.katana"

    private val TAG = "FacebookShareWrapper"

    fun startFacebookShareChecking(activity: Activity, shareAction: (() -> Unit)?): Disposable? {

        //这一行是从NativeProtocol中fetchAllAvailableProtocolVersionsForAppInfo方法复制过来的代码
        val versionSet = FacebookProtocolVersionHelper.fetchAllAvailableProtocolVersionsForAppInfo(activity, FACEBOOK_PACKAGE_NAME)

        val am = activity.getSystemService(Context.ACTIVITY_SERVICE) as ActivityManager

        var isFacebookStarted = false

        //facebook未安装，或者facebook进程无法启动
        if (versionSet.size <= 0) {
            val intent = activity.packageManager.getLaunchIntentForPackage(FACEBOOK_PACKAGE_NAME)
            if (intent != null) {
                //Facebook进程无法启动，启动它吧！！
                try {
                    isFacebookStarted = true
                    activity.startActivity(intent)
                } catch (e: Exception) {
                    isFacebookStarted = false
                    Log.e(TAG, " start activity failed intent = $intent, error msg = ${e.message}", e)
                }
            }
        }

        //facebook没有被启动，走正常分享流程就行
        if (!isFacebookStarted) {
            hookNativeProtocalFacebookAppInfoList(versionSet, FACEBOOK_PACKAGE_NAME)
            shareAction?.invoke()
            return null
        }

        //检查facebook启动情况的标志
        var intervalCheckFacebookFlag = false

        //一直轮询，直到可以取到facebook的ProtocolVersion数据
        return Flowable.intervalRange(0, 20, 0, 20, TimeUnit.MILLISECONDS, AndroidSchedulers.mainThread())
                .subscribe {
                    if (!intervalCheckFacebookFlag) {
                        val versionSet = FacebookProtocolVersionHelper.fetchAllAvailableProtocolVersionsForAppInfo(activity, FACEBOOK_PACKAGE_NAME)
                        if (versionSet.size > 0) {
                            hookNativeProtocalFacebookAppInfoList(versionSet, FACEBOOK_PACKAGE_NAME)
                            intervalCheckFacebookFlag = true

                            //将应用界面移到前台，并开始facebook分享
                            am.moveTaskToFront(activity.taskId, 0)
                            shareAction?.invoke()
                        }
                    }
                }
    }
    
    private fun hookNativeProtocalFacebookAppInfoList(protocolVersionSet: TreeSet<Int>, packageName: String) {
        try {
            if (protocolVersionSet.isEmpty()) {
                return
            }
            val facebookAppInfoList = ReflectUtils.getField("com.facebook.internal.NativeProtocol", "facebookAppInfoList") as? List<*>

            if (facebookAppInfoList?.isEmpty() != false) {
                return
            }
            facebookAppInfoList.forEach {
                val thisPackageName = ReflectUtils.callMethod(it, "getPackage")
                Log.d(TAG, " hookNativeProtocalFacebookAppInfoList thisPackageName = $thisPackageName")
                if (thisPackageName == packageName) {
                    ReflectUtils.setField(it, "availableVersions", protocolVersionSet)
                }
            }
        } catch (e: Exception) {
            Log.d(TAG, " hookNativeProtocalFacebookAppInfoList failed ", e)
        }
    }
}    
```
其中，两个反射方法的封装为：
```
//获取类的静态变量的值
    public static Object getField(String className, String fieldName) {
        return getField(className,null, fieldName);
    }
    
    private static Object getField(String className, Object receiver, String fieldName) {
        Class<?> clazz = null;
        Field field;
        if (!TextUtils.isEmpty(className)) {
            try {
                clazz = Class.forName(className);
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
        } else {
            if (receiver != null) {
                clazz = receiver.getClass();
            }
        }
        if (clazz == null) return null;

        try {
            field = findField(clazz, fieldName);
            if (field == null)  return null;
            field.setAccessible(true);
            return field.get(receiver);
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (IllegalArgumentException e) {
            e.printStackTrace();
        } catch (NullPointerException e) {
            e.printStackTrace();
        }
        return null;
    }
    
    public static Object callMethod(Object receiver, String methodName, Object... params) {
        return callMethod(null, receiver, methodName, params);
    }
    
    private static Object callMethod(String className, Object receiver, String methodName, Object... params) {
        Class<?> clazz = null;
        if (!TextUtils.isEmpty(className)) {
            try {
                clazz = Class.forName(className);
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
        } else {
            if (receiver != null) {
                clazz = receiver.getClass();
            }
        }
        if (clazz == null) return null;
        try {
            Method method = findMethod(clazz, methodName, params);
            if (method == null) {
                return null;
            }
            method.setAccessible(true);
            return method.invoke(receiver, params);
        } catch (IllegalArgumentException e) {
            e.printStackTrace();
        }  catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
        return null;
    }

```
至此，我们完美地解决了facebook分享无法调起facebook客户端的问题。

## 总结
最后，对上面的问题原因以及修复方法作一个简单的总结。

主要有两个原因导致facebook分享无法调起客户端：
1. facebook分享sdk在分享时，会先过contentProvider跨进程调用获取facebook App保存的protocolVersion数据，并根据该数据决定客户端是否支持此次分享。由于在一些国产Rom上增加了禁止第三方自启动（或者禁止应用间互相启动）功能，导致此次跨进程启动facebook app失败，facebook sdk获取不到客户端数据，因此，就无法调起客户端分享，而是调起了网页分享。
2. facebook sdk本身存在一个TreeSet非空判断的bug，导致先起自己的app，再起facebook app时，获取facebook app数据失败，从而导致分享失败。

这两个问题对应的修复方法为：
1. 先判断跨进程调用facebook app是否会失败，失败的话，先启动facebook app，再跳回自己的应用走正常分享流程。
2. 通过反射修复TreeSet非空判断的bug。

以上就本人对该问题的一些思考与总结，希望能给大家带来一些帮助。


  [1]: https://developers.facebook.com/docs/sharing/android
  [2]: https://github.com/facebook/facebook-android-sdk
  [3]: https://developers.facebook.com/docs/reference/android/current/class/ShareLinkContent
  [4]: http://static.zybuluo.com/Wind729/4utue56qk9se3bpc1e00ar19/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-11-25%2023.31.06.png?imageView/2/w/350/q/80
  [5]: http://static.zybuluo.com/Wind729/t41th8mcp5phz23yhvfh42k8/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-11-26%2010.58.49.png?imageView/2/w/850/q/80
  [6]: https://github.com/facebook/facebook-android-sdk/pull/538
