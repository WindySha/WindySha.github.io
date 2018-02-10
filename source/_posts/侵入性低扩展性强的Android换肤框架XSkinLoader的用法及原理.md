---
title: 侵入性低扩展性强的Android换肤框架XSkinLoader的用法及原理
date: 2018-02-10 17:12:33
categories:
- Android
- 换肤框架
tags:
- 换肤框架
- XSkinLoader
- 插件化开发
- LayoutInflater
---


## **前言**
Android发展到现在，很多成熟的应用上已经集成了插件式换肤的功能，比如网易云音乐，手机QQ，QQ音乐等等。但是，成熟稳定易用的开源换肤框架并没有出现。

国内最早的插件式换肤框架是[Android-Skin-Loader][2]。后面也出现了一些在此基础上的改进版，比如：hongyang的[ChangeSkin][3]，[andSkin][4]，[Android-skin-support][5]，[injor][6]，[QSkinLoader][7]等等。大家都对**Android-Skin-Loader**做了一些改进，以使换肤过程侵入性更低，扩展性更强，使用更简单。但是还是会有一些不足之处，因此，XSkinLoader就诞生了。

XSkinLoader是在Android-Skin-Loader和QSkinLoader的基础上又进行了一次重大改进，主要的改进点有如下：
>1. 侵入性更低，换肤Activity并不用实现某个接口或者继承某个BaseActivity
2. 支持布局里style中定义的属性换肤，默认支持了TextView的textColor和ProgressBar的indeterminateDrawable，并支持扩展；
3. 更好地支持了AppCompatActivity中的控件换肤，由于AppCompatActivity中的TextView，ImageView等控件会被转为AppCompatTextView,AppCompatImageView，XSkinLoader换肤时并不会覆盖此转换，其他换肤框架会覆盖；
4. 支持状态栏颜色换肤，并可以通过相似方法扩展支持标题栏和虚拟导航栏的换肤；
5. 支持xml中指定的属性换肤

XSkinLoader项目源码地址为：**[https://github.com/WindySha/XSkinLoader][8]**

<!-- more -->

下面，先简单介绍XSkinLoader的基本用法，再通过分析源码来解析这些改进点的实现原理。

## **XSkinLoader的使用方法**
XSkinLoader的使用方式特别简单，对代码的侵入性很低，需要换肤的Activity中只用在调用一行代码即可：
```
    SkinInflaterFactory.setFactory(this);
```
用法跟其他换肤框架基本相同，先在Application中初始化，然后在相关xml中加上`skin:enable="true"`即可， 详细用法如下：
### **初始化**
首先在`Application`的`onCreate`中进行初始化：
```
        SkinManager.get().init(this);
```
如果代码中需要经常使用Application Context的LayoutInflater加载View，最好同时加上这样一行代码：
```
        SkinInflaterFactory.setFactory(LayoutInflater.from(this));  // for skin change
        SkinManager.get().init(this);
```
如此，使用LayoutInflater.from(context.getApplicationContext()).inflate()加载的view也是可以换肤的

### **XML换肤**
xml布局中的View需要换肤的，只需要在布局文件中相关View标签下添加`skin:enable="true"`即可,例如：
```
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:skin="http://schemas.android.com/android/skin"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <TextView
        android:id="@+id/status_bar"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        skin:enable="true"
        android:background="@color/title_color">
    </TextView>
<RelativeLayout/>
```
能换肤的前提是解析这个xml的LayoutInflater设置Factory接口：SkinInflaterFactory
因此，在相关activity的`onCreate()`中`setContentView()`方法之前添加：
```
    //干涉xml中view的创建，实现xml中资源换肤
    SkinInflaterFactory.setFactory(this);  //for skin change in XML
```
>**PS:** 对于AppCompatActivity，务必要在`onCreate()`的`super.onCreate()`之前添加，否则不会使用AppComt包装的控件，比如：AppCompatTextView等。

某些view的资源是在代码中动态设置的，使用以下方式来设置资源，才能实现换肤效果：

```
    //设置imageView的src资源
    SkinManager.get().setImageDrawable(imageView, R.drawable.ic_action);
    //设置imageView的backgroud资源
    SkinManager.get().setViewBackground(imageView, R.drawable.ic_action);
    //设置textVie的color资源
    SkinManager.get().setTextViewColor(textView, R.color.title_color);
    //设置Activity的statusBarColor
    SkinManager.get().setWindowStatusBarColor(MainActivity.this.getWindow(), R.color.title_color);
    ...
```

### **xml中指定换肤属性**
xml中假如出现了多个可换肤属性，但只需要换其中部分属性，而不是全部属性，比如：
```
<Button
        android:id="@+id/use_sdcard_skin"
        android:layout_width="180dp"
        android:layout_height="40dp"
        skin:enable="true"
        android:background="@drawable/confirm_skin_btn_border"
        android:textColor="@color/music_skin_change_button_color" />
```
这个布局中，包含两个换肤属性：`background`，`textColor`，假如只想换`textColor`,那该怎么办？
此处，借鉴了[andSkin][9]中的一个办法，增加一个属性`attrs`，在此属性中声明需要换肤的属性。
具体到上面的例子，只需要增加这样一行代码`skin:attrs="textColor"`就行：
```
<Button
        android:id="@+id/use_sdcard_skin"
        android:layout_width="180dp"
        android:layout_height="40dp"
        skin:enable="true"
        skin:attrs="textColor"
        android:background="@drawable/confirm_skin_btn_border"
        android:textColor="@color/music_skin_change_button_color" />
```
如果支持多个属性，使用`|`分割就行：
```
        skin:attrs="textColor|background"
```
其实，大多数情况下并不用在Xml中加此属性来控制，如若不想此属性换肤，也可以在相应的皮肤apk中去掉此属性指定的资源。

### **新增换肤属性**
对已经成型的大型项目来说，XSkinLoader中提供的换肤属性是不够用的，需要额外增加的换肤属性该怎么办？
在sample中写好了相应的模板，具体参考ExtraAttrRegister.java
```
public static final String CUSTIOM_VIEW_TEXT_COLOR = "titleTextColor";

    static {
        //增加自定义控件的自定义属性的换肤支持
        SkinResDeployerFactory.registerDeployer(CUSTIOM_VIEW_TEXT_COLOR, new CustomViewTextColorResDeployer());

    }
```

### **新增style中的换肤属性**
假如style中的换肤属性不够用，需要新增，该怎么办？
sample中也写了一个模板，在ExtraAttrRegister.java中:
```
static {
        //增加xml里的style中指定的View background属性换肤
        StyleParserFactory.addStyleParser(new ViewBackgroundStyleParser());
    }
```
## **XSKinLoader的实现原理分析**
换肤框架核心的技术原理和Android-skim-loader以及由此衍生出来的那些框架都差不多。主要就是实现`LayoutInflater.Factory`接口干涉xml中view解析的过程，并将解析出来的熟悉和view保存到list(map)中，换肤的时候，遍历此list(map)，重新设置此view的换肤属性对应的资源（用皮肤包对应的Resources来设置）。

具体细节，如若不清楚可以参考QSkinLoader的源码解析：
[Android换肤功能实现与换肤框架QSkinLoader使用方式介绍][10]
或者 andSkin的源码解析：
[Android 换肤原理分析和总结][11]
核心原理都差不多，都来自于Android-skin-loader，此处就不再啰嗦。

这里，主要讲XSkinLoader的改进点。

### **使用WeakHashMap**
将View和对应的换肤属性保存在全局的WeakHashMap中，这样activity退出后，WeakHashMap中的view会被GC回收掉，因此不会出现内存泄漏的问题。
```
    //使用这个map保存所有需要换肤的view和其对应的换肤属性及资源
    //使用WeakHashMap两个作用，1.避免内存泄漏，2.避免重复的view被添加
    //使用HashMap存SkinAttr，为了避免同一个属性值存了两次
    private WeakHashMap<View, HashMap<String, SkinAttr>> mSkinAttrMap = new WeakHashMap<>();
```
WeakHashMap中键值对的值使用HashMap<String, SkinAttr>，是为了避免view的属性重复添加，比如，在xml中设置了TextView的textColor换肤资源，在代码中又设置了textColor换肤资源
`SkinManager.get().setTextViewColor(textView, R.color.title_color);`
这样代码中设置的换肤资源会覆盖掉xml中设置的。（xml中设置的属性资源也会覆盖style中设置的属性资源）

### **支持AppCompatActivity换肤**
由于AppCompatActivity会设置LayoutInflater.Factory，干涉view的创建过程，并将TextView,ImageView等替换为AppCompatTextView,AppCompatImageView。假如不做特殊处理，会覆盖掉AppCompatActivity中设置的Factory，因此，没有兼容到AppCompatActivity的一些属性。

查阅AppCompatActivity，可知，为了兼容不同的android版本，它是通过AppCompatDelegate来设置LayoutInflater的Factory,代码细节如下：
```
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        final AppCompatDelegate delegate = getDelegate();
        delegate.installViewFactory();
        delegate.onCreate(savedInstanceState);
        ...
        ...
        super.onCreate(savedInstanceState);
    }
```
`delegate.installViewFactory();`最终调到了`AppCompatDelegateImplV9.java`中的`installViewFactory()`。
```
    @Override
    public void installViewFactory() {
        LayoutInflater layoutInflater = LayoutInflater.from(mContext);
        if (layoutInflater.getFactory() == null) {
            LayoutInflaterCompat.setFactory2(layoutInflater, this);
        } else {
            if (!(layoutInflater.getFactory2() instanceof AppCompatDelegateImplV9)) {
                Log.i(TAG, "The Activity's LayoutInflater already has a Factory installed"
                        + " so we can not install AppCompat's");
            }
        }
    }
```
`AppCompatDelegateImplV9.java`中的Factory2的接口实现为：
```
    /**
     * From {@link LayoutInflater.Factory2}.
     */
    @Override
    public final View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
        // First let the Activity's Factory try and inflate the view
        final View view = callActivityOnCreateView(parent, name, context, attrs);
        if (view != null) {
            return view;
        }

        // If the Factory didn't handle it, let our createView() method try
        return createView(parent, name, context, attrs);
    }
```
在`createView`中使用`AppCompatViewInflater`来创建View，并将TextView,ImageView等替换为AppCompatTextView,AppCompatImageView：
```
    public final View createView(View parent, final String name, @NonNull Context context,
            @NonNull AttributeSet attrs, boolean inheritContext,
            boolean readAndroidTheme, boolean readAppTheme, boolean wrapContext) {
        ...
        ...
        switch (name) {
            case "TextView":
                view = new AppCompatTextView(context, attrs);
                break;
            case "ImageView":
                view = new AppCompatImageView(context, attrs);
                break;
            case "Button":
                view = new AppCompatButton(context, attrs);
                break;
            case "EditText":
                view = new AppCompatEditText(context, attrs);
                break;
            case "Spinner":
                view = new AppCompatSpinner(context, attrs);
                break;
            case "ImageButton":
                view = new AppCompatImageButton(context, attrs);
                break;
            ...
            ...

        return view;
    }
```
为了使我们的SkinInflaterFactory不干涉AppCompatActivity的view创建过程，我们可以这样做：
```
    public static void setFactory(Activity activity) {
        LayoutInflater inflater = activity.getLayoutInflater();
        SkinInflaterFactory factory = new SkinInflaterFactory();
        if (activity instanceof AppCompatActivity) {
            //AppCompatActivity本身包含一个factory,将TextView等转换为AppCompatTextView.java, 参考：AppCompatDelegateImplV9.java
            final AppCompatDelegate delegate = ((AppCompatActivity) activity).getDelegate();
            factory.setInterceptFactory(new Factory() {
                @Override
                public View onCreateView(String name, Context context, AttributeSet attrs) {
                    //创建view的过程还是交给AppCompatDelegate来做
                    return delegate.createView(null, name, context, attrs);
                }
            });
        }
        inflater.setFactory(factory);
    }
    
    //因为LayoutInflater的setFactory方法只能调用一次，当框架外需要处理view的创建时，可以调用此方法
    public void setInterceptFactory(Factory factory) {
        mViewCreateFactory = factory;
    }

    @Override
    public View onCreateView(String name, Context context, AttributeSet attrs) {
        View view = null;
        if (mViewCreateFactory != null) {
            //给框架外提供创建View的机会
            view = mViewCreateFactory.onCreateView(name, context, attrs);
        }
        if (isSupportSkin(attrs)) {
            if (view == null) {
                view = createView(context, name, attrs);
            }
            if (view != null) {
                parseAndSaveSkinAttr(attrs, view);
            }
        }

        return view;
    }
```
### **Activity的statusBar颜色换肤**
首先将Activity对应的Window传过来，然后获取Window对应的DecorView，对DecorView实施换肤：
```
    public void setWindowStatusBarColor(Window window, @ColorRes int resId) {
        View decorView = window.getDecorView();
        setSkinViewResource(decorView, SkinResDeployerFactory.ACTIVITY_STATUS_BAR_COLOR, resId);
    }
```
正真换肤的时候，又通过DecorView反射获取其对应的Window，然后设置window的StatusBarColor：
```
public class ActivityStatusBarColorResDeployer implements ISkinResDeployer {
    @Override
    public void deploy(View view, SkinAttr skinAttr, ISkinResourceManager resource) {
        //the view is the window's DecorView
        Window window = (Window) ReflectUtils.getField(view, "mWindow");
        if (window == null) {
            throw new IllegalArgumentException("view is not a DecorView, cannot get the window");
        }
        if (SkinConfig.RES_TYPE_NAME_COLOR.equals(skinAttr.attrValueTypeName)) {
            window.setStatusBarColor(resource.getColor(skinAttr.attrValueRefId));
        }
    }
}
```
### **支持style中的换肤属性**
style中的换肤属性支持方法主要是根据传入的`AttributeSet`和控件的styleable列表获取控件中属性对应的资源id，并将view，属性，资源id保存起来。以TextView的textColor为例，具体实现细节如下：
```
public class TextViewTextColorStyleParser implements ISkinStyleParser{

    private static int[] sTextViewStyleList;
    private static int sTextViewTextColorStyleIndex;

    @Override
    public void parseXmlStyle(View view, AttributeSet attrs, Map<String, SkinAttr> viewAttrs, String[] specifiedAttrList) {
        if (!TextView.class.isAssignableFrom(view.getClass())) {
            return;
        }
        Context context = view.getContext();
        int[] textViewStyleable = getTextViewStyleableList();
        int textViewStyleableTextColor = getTextViewTextColorStyleableIndex();

        TypedArray a = context.obtainStyledAttributes(attrs, textViewStyleable, 0, 0);
        if (a != null) {
            int n = a.getIndexCount();
            for (int j = 0; j < n; j++) {
                int attr = a.getIndex(j);
                if (attr == textViewStyleableTextColor &&
                        SkinConfig.isCurrentAttrSpecified(SkinResDeployerFactory.TEXT_COLOR, specifiedAttrList)) {
                    int colorResId = a.getResourceId(attr, -1);
                    SkinAttr skinAttr = SkinAttributeParser.parseSkinAttr(context, SkinResDeployerFactory.TEXT_COLOR, colorResId);
                    if (skinAttr != null) {
                        viewAttrs.put(skinAttr.attrName, skinAttr);
                    }
                }
            }
            a.recycle();
        }
    }

    private static int[] getTextViewStyleableList() {
        if (sTextViewStyleList == null || sTextViewStyleList.length == 0) {
            sTextViewStyleList = (int[]) ReflectUtils.getField("com.android.internal.R$styleable", "TextView");
        }
        return sTextViewStyleList;
    }

    private static int getTextViewTextColorStyleableIndex() {
        if (sTextViewTextColorStyleIndex == 0) {
            Object o = ReflectUtils.getField("com.android.internal.R$styleable", "TextView_textColor");
            if (o != null) {
                sTextViewTextColorStyleIndex = (int) o;
            }
        }
        return sTextViewTextColorStyleIndex;
    }
}
```
具体细节如若不明白，可以参考`TextView.java`第四个构造方法中对`AttributeSet`和`style`的处理。


## **总结**
XSkinLoader虽然说已经很完美了，但是还有一些不足之处：

1. 无法支持Theme中定义的属性换肤，无论是Activity中的Theme还是Application还是控件中指定的Theme，都是无法支持换肤。暂时没能找到解决方法，而且其他的换肤框架也没有解决这个问题，比较坑。
2. 暂时没能支持Glide控件设置默认图片的换肤，一般使用Glide设置默认图是这样：
`Glide.with(context).load(url).placeholder(R.drawable.default).into(imageView);`
暂时不能支持`R.drawable.default`的换肤，不过，此问题应该可解，毕竟Glide的扩展性非常强。
3. RecyclerView的缓存问题，可能会导致换肤RecyclerView的item换肤失败，不过暂时未碰到。假如遇到此问题，可以参考**[QSkinLoader][12]**的清除缓存的方法。
3. 可能存在的性能问题.


  [1]: https://windysha.github.io/
  [2]: https://github.com/fengjundev/Android-Skin-Loader
  [3]: https://github.com/hongyangAndroid/ChangeSkin
  [4]: https://github.com/RrtoyewxXu/andSkin
  [5]: http://blog.csdn.net/ximsfei/article/details/54586827
  [6]: https://github.com/hackware1993/injor
  [7]: http://blog.csdn.net/u013478336/article/details/53083054
  [8]: https://github.com/WindySha/XSkinLoader
  [9]: https://github.com/RrtoyewxXu/andSkin
  [10]: http://blog.csdn.net/u013478336/article/details/53083054
  [11]: http://blog.csdn.net/zhi184816/article/details/53436761
  [12]: http://blog.csdn.net/u013478336/article/details/53083054
