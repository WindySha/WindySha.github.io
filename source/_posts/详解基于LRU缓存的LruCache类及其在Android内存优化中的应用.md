---
title: 详解基于LRU缓存的LruCache类及其在Android内存优化中的应用
date: 2018-01-12 15:15:20
tags: 
- LRU 
- LruCache
- 缓存技术
---



今天与大家分享一下图片的缓存技术，利用它可以提高UI的流畅性、响应速度，给用户好的体验。

如何在内存中做缓存？

通过内存缓存可以快速加载缓存图片，但会消耗应用的内存空间。LruCache类（通过兼容包可以支持到sdk4）很适合做图片缓存，它通过LinkedHashMap保持图片的强引用方式存储图片，当缓存空间超过设置定的限值时会释放掉早期的缓存。

  注：在过去，常用的内存缓存实现是通过SoftReference或WeakReference，但不建议这样做。从Android2.3（API等级9）垃圾收集器开始更积极收集软/弱引用，这使得它们相当无效。此外，在Android 3.0（API等级11）之前，存储在native内存中的可见的bitmap不会被释放，可能会导致应用程序暂时地超过其内存限制并崩溃。

什么是LruCache？ 
LruCache实现原理是什么？   

要回答这个两个问题，先要知道什么是LRU。
LRU是Least Recently Used 的缩写，翻译过来就是“最近最少使用”，LRU缓存就是使用这种原理实现，简单的说就是缓存一定量的数据，当超过设定的阈值时就把一些过期的数据删除掉，比如我们缓存100M的数据，当总数据小于100M时可以随意添加，当超过100M时就需要把新的数据添加进来，同时要把过期数据删除，以确保我们最大缓存100M，那怎么确定删除哪条过期数据呢，采用LRU算法实现的话就是将最老的数据删掉。利用LRU缓存，我们能够提高系统的performance.
<!-- more -->
# LruCache源码分析
LruCache.java是 android.support.v4包引入的一个类，其实现原理就是基于LRU缓存算法，
要想实现LRU缓存，我们首先要用到一个类 LinkedHashMap。 用这个类有两大好处：一是它本身已经实现了按照访问顺序的存储，也就是说，最近读取的会放在最前面，最不常读取的会放在最后（当然，它也可以实现按照插入顺序存储）。第二，LinkedHashMap本身有一个方法用于判断是否需要移除最不常读取的数，但是，原始方法默认不需要移除（这是，LinkedHashMap相当于一个linkedlist），所以，我们需要override这样一个方法，使得当缓存里存放的数据个数超过规定个数后，就把最不常用的移除掉。LinkedHashMap的API写得很清楚，推荐大家可以先读一下。
要基于LinkedHashMap来实现LRU缓存，可以选择inheritance, 也可以选择 delegation， android源码选择的是delegation，而且写得很漂亮。下面，就来剖析一下源码的实现方法：
```
public class LruCache<K, V> {
    //缓存 map 集合，要用LinkedHashMap    
    private final LinkedHashMap<K, V> map;

    private int size; //已经存储的大小
    private int maxSize; //规定的最大存储空间
  
    private int putCount;  //put的次数
    private int createCount;  //create的次数
    private int evictionCount;  //回收的次数
    private int hitCount;  //命中的次数
    private int missCount;  //丢失的次数

    //实例化 Lru，需要传入缓存的最大值,这个最大值可以是个数，比如对象的个数，也可以是内存的大小
    //比如，最大内存只能缓存5兆
    public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }

   //重置最大存储空间
    public void resize(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }

        synchronized (this) {
            this.maxSize = maxSize;
        }
        trimToSize(maxSize);
    }

    //通过key返回相应的item，或者创建返回相应的item。相应的item会移动到队列的头部，
    // 如果item的value没有被cache或者不能被创建，则返回null。
    public final V get(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V mapValue;
        synchronized (this) {
            mapValue = map.get(key);
            if (mapValue != null) {
                hitCount++;
                return mapValue;
            }
            missCount++;
        }

        //如果丢失了就试图创建一个item
        V createdValue = create(key);
        if (createdValue == null) {
            return null;
        }

        synchronized (this) {
            createCount++;
            mapValue = map.put(key, createdValue);

            if (mapValue != null) {
                // There was a conflict so undo that last put
                map.put(key, mapValue);
            } else {
                size += safeSizeOf(key, createdValue);
            }
        }

        if (mapValue != null) {
            entryRemoved(false, key, createdValue, mapValue);
            return mapValue;
        } else {
            trimToSize(maxSize);
            return createdValue;
        }
    }

    //创建cache项，并将创建的项放到队列的头部
    public final V put(K key, V value) {
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }

        V previous;
        synchronized (this) {
            putCount++;
            size += safeSizeOf(key, value);
            previous = map.put(key, value);
            if (previous != null) {  //返回的先前的value值
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
            entryRemoved(false, key, previous, value);
        }

        trimToSize(maxSize);
        return previous;
    }

    //清空cache空间
    public void trimToSize(int maxSize) {
        while (true) {
            K key;
            V value;
            synchronized (this) {
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }

                if (size <= maxSize || map.isEmpty()) {
                    break;
                }

                Map.Entry<K, V> toEvict = map.entrySet().iterator().next();
                key = toEvict.getKey();
                value = toEvict.getValue();
                map.remove(key);
                size -= safeSizeOf(key, value);
                evictionCount++;
            }

            entryRemoved(true, key, value, null);
        }
    }

    //删除key相应的cache项，返回相应的value
    public final V remove(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V previous;
        synchronized (this) {
            previous = map.remove(key);
            if (previous != null) {
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
            entryRemoved(false, key, previous, null);
        }

        return previous;
    }

    /**
     * Called for entries that have been evicted or removed. This method is
     * invoked when a value is evicted to make space, removed by a call to
     * {@link #remove}, or replaced by a call to {@link #put}. The default
     * implementation does nothing.
     * 当item被回收或者删掉时调用。改方法当value被回收释放存储空间时被remove调用，
     * 或者替换item值时put调用，默认实现什么都没做
     * <p>The method is called without synchronization: other threads may
     * access the cache while this method is executing.
     *
     * @param evicted true if the entry is being removed to make space, false
     *     if the removal was caused by a {@link #put} or {@link #remove}.
     *  true---为释放空间被删除；false---put或remove导致
     * @param newValue the new value for {@code key}, if it exists. If non-null,
     *     this removal was caused by a {@link #put}. Otherwise it was caused by
     *     an eviction or a {@link #remove}.
     */
    protected void entryRemoved(boolean evicted, K key, V oldValue, V newValue) {}

    //当某Item丢失时会调用到，返回计算的相应的value或者null
    protected V create(K key) {
        return null;
    }

    private int safeSizeOf(K key, V value) {
        int result = sizeOf(key, value);
        if (result < 0) {
            throw new IllegalStateException("Negative size: " + key + "=" + value);
        }
        return result;
    }

    //这个方法要特别注意，跟我们实例化LruCache的maxSize要呼应，怎么做到呼应呢，比如maxSize的大小为缓存 
    //的个数，这里就是return 1就ok，如果是内存的大小，如果5M，这个就不能是个数了，就需要覆盖这个方法，返回每个缓存 
    //value的size大小，如果是Bitmap，这应该是bitmap.getByteCount();
    protected int sizeOf(K key, V value) {
        return 1;
    }

    //清空cacke
    public final void evictAll() {
        trimToSize(-1); // -1 will evict 0-sized elements
    }

    /**
     * For caches that do not override {@link #sizeOf}, this returns the number
     * of entries in the cache. For all other caches, this returns the sum of
     * the sizes of the entries in this cache.
     */
    public synchronized final int size() {
        return size;
    }

    public synchronized final int maxSize() {
        return maxSize;
    }

    /**
     * Returns the number of times {@link #get} returned a value that was
     * already present in the cache.
     */
    public synchronized final int hitCount() {
        return hitCount;
    }

    /**
     * Returns the number of times {@link #get} returned null or required a new
     * value to be created.
     */
    public synchronized final int missCount() {
        return missCount;
    }

    public synchronized final int createCount() {
        return createCount;
    }

    public synchronized final int putCount() {
        return putCount;
    }

    //返回被回收的数量
    public synchronized final int evictionCount() {
        return evictionCount;
    }

    //返回当前cache的副本，从最近最少访问到最多访问
    public synchronized final Map<K, V> snapshot() {
        return new LinkedHashMap<K, V>(map);
    }

    @Override public synchronized final String toString() {
        int accesses = hitCount + missCount;
        int hitPercent = accesses != 0 ? (100 * hitCount / accesses) : 0;
        return String.format("LruCache[maxSize=%d,hits=%d,misses=%d,hitRate=%d%%]",
                maxSize, hitCount, missCount, hitPercent);
    }
}
```
从源代码中，我们可以清晰的看出LruCache的缓存机制。
此外，还有一个开源的使用磁盘缓存的方法DiskLruCache，源代码：https://github.com/JakeWharton/DiskLruCache
其详细的使用方法可以参考链接：http://www.tuicool.com/articles/JB7RNj
# LruCache应用实例
根据上面的代码，当我们用LruCache来缓存图片时，一定要重写` protected int sizeOf(K key, V value) {} `方法，否则，最大缓存的是数量而不是占用内存大小。重写` protected void entryRemoved(boolean evicted, K key, V oldValue, V newValue) {} `方法，在里面处理内存回收。
下面例子继承LruCache，实现相关方法：
```
public class BitmapCache<K, V extends Bitmap> extends LruCache<K, V>{
	private BitmapRemovedCallBack mEnterRemovedCallBack;

	public BitmapCache(int maxSize, BitmapRemovedCallBack callBack) {
		super(maxSize);
		mEnterRemovedCallBack = callBack;
	}

    //当缓存大于我们设定的最大值时，会调用这个方法，我们在这里面做内存释放操作
	@Override
	protected void entryRemoved(boolean evicted, K key, V oldValue, V newValue) {
		super.entryRemoved(evicted, key, oldValue, newValue);
		if (evicted && oldValue != null){
		//在回收bitmap之前，务必要先在下面的回调方法中将bitmap设给的View的bitmapDrawable设为null
		//否则，bitmap被回收后，很容易出现cannot draw recycled bitmap的报错。切记！
			mEnterRemovedCallBack.onBitmapRemoved(key);
			oldValue.recycle();
		}
	}

    //获取每个 value 的大小
	@Override
	protected int sizeOf(K key, V value) {
		int size = 0;
		if (value != null) {
			size = value.getByteCount();
		}
		return size;
	}

	public interface BitmapRemovedCallBack<K>{
		void onBitmapRemoved(K key);
	}
}
```
使用BitmapCache时，在构造方法中传入最大缓存量和一个回掉方法就行：
```
private BitmapCache<String, Bitmap> mMemoryCache;  

private BitmapCache.BitmapRemovedCallBack<String> mEnteryRemovedCallBack =
			new BitmapCache.BitmapRemovedCallBack<String>() {
		@Override
		public void onBitmapRemoved(String key) {
		//处理回收bitmap前，清空相关view的bitmap操作			
		}
	};
Override
protected void onCreate(Bundle savedInstanceState) { 
    // 获取到可用内存的最大值，使用内存超出这个值会引起OutOfMemory异常。 
    // BitmapCache通过构造函数传入缓存值，以bit为单位。 
    int memClass = ((ActivityManager) activity.getSystemService(Context.ACTIVITY_SERVICE)).getMemoryClass();
    // 使用单个应用最大可用内存值的1/8作为缓存的大小。 
    int cacheSize = 1024 * 1024 * memClass / 8;
    mMemoryCache = new BitmapCache<String, Bitmap>(cacheSize， mEnteryRemovedCallBack);
} 
    
public void addBitmapToMemoryCache(String key, Bitmap bitmap) { 
    if (getBitmapFromMemCache(key) == null) { 
        mMemoryCache.put(key, bitmap); 
    } 
} 
    
public Bitmap getBitmapFromMemCache(String key) { 
    return mMemoryCache.get(key); 
}
```
当向 ImageView 中加载一张图片时,首先会在 BitmapCache 的缓存中进行检查。如果找到了相应的键值，则会立刻更新ImageView ，否则开启一个后台线程来加载这张图片：
```
public void loadBitmap(int resId, ImageView imageView) { 
    final String imageKey = String.valueOf(resId); 
    final Bitmap bitmap = getBitmapFromMemCache(imageKey); 
    if (bitmap != null) { 
        imageView.setImageBitmap(bitmap); 
    } else { 
        imageView.setImageResource(R.drawable.image_placeholder); 
        BitmapLoadingTask task = new BitmapLoadingTask(imageView); 
        task.execute(resId); 
    } 
}
```
BitmapLoadingTask后台线程，把新加载出来的图片以键值对形式放到缓存中：
```
class BitmapLoadingTask extends AsyncTask<Integer, Void, Bitmap> { 
    // 在后台加载图片。 
    @Override
    protected Bitmap doInBackground(Integer... params) { 
        final Bitmap bitmap = decodeSampledBitmapFromResource( 
                getResources(), params[0], 100, 100); 
        addBitmapToMemoryCache(String.valueOf(params[0]), bitmap); 
        return bitmap; 
    } 
}
```



# 总结
1、LruCache 是基于 Lru算法实现的一种缓存机制； 
2、Lru算法的原理是把近期最少使用的数据给移除掉，当然前提是当前数据的量大于设定的最大值。 
3、LruCache 没有真正的释放内存，只是从 Map中移除掉数据，真正释放内存还是要用户手动释放。
4、手动释放bitmap的内存时，需要先清除相关view中的bitmap。

