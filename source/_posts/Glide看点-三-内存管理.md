Glide里的缓存是一个精妙的多级缓存，从文档里我们可以知道总的缓存策略如下：
> 默认情况下，Glide 会在开始一个新的图片请求之前检查以下多级的缓存：

> 活动资源 (Active Resources) - 现在是否有另一个 View 正在展示这张图片？
> 内存缓存 (Memory cache) - 该图片是否最近被加载过并仍存在于内存中？
> 资源类型（Resource） - 该图片是否之前曾被解码、转换并写入过磁盘缓存？
> 数据来源 (Data) - 构建这个图片的资源是否之前曾被写入过文件缓存？
> 前两步检查图片是否在内存中，如果是则直接返回图片。后两步则检查图片是否在磁盘上，以便快速但异步地返回图片。

> 如果四个步骤都未能找到图片，则Glide会返回到原始资源以取回数据（原始文件，Uri, Url等）。

本文讲分析Glide对于Bitmap的内存缓存逻辑：
## ActiveCache的使用
Glide里有内存里分别有两层缓存，一个是Active Cache一个是普通的Cache。他们本质上都是LinkedHashMap实现的LRUCache，只是调度策略让他们拥有了不同的功能。

从load方法来看策略：

```
 EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
      cb.onResourceReady(active, DataSource.MEMORY_CACHE);
      return null;
    }

    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
      cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
      return null;
    }
```
* ***加载时先从Active Cache里检查，如果没有再从普通的Cache里拿去***
* ***当从Normal cache里取出来后会把对象加入到Activie里，并且从普通的cache里移除***

```
private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
    if (!isMemoryCacheable) {
      return null;
    }

    EngineResource<?> cached = getEngineResourceFromCache(key);
    if (cached != null) {
      cached.acquire();
      activeResources.activate(key, cached);
    }
    return cached;
  }  
```

```
  private EngineResource<?> getEngineResourceFromCache(Key key) {
    Resource<?> cached = cache.remove(key);

    final EngineResource<?> result;
    if (cached == null) {
      result = null;
    } else if (cached instanceof EngineResource) {
      // Save an object allocation if we've cached an EngineResource (the typical case).
      result = (EngineResource<?>) cached;
    } else {
      result = new EngineResource<>(cached, true /*isMemoryCacheable*/, true /*isRecyclable*/);
    }
    return result;
  }

```
* ***当resource release的时候会在active resouce移除对象，在普通cache里加入***

```
 @Override
  public void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
    Util.assertMainThread();
    activeResources.deactivate(cacheKey);
    if (resource.isCacheable()) {
      cache.put(cacheKey, resource);
    } else {
      resourceRecycler.recycle(resource);
    }
  }

```
画个数据流图看得更直观一些
 ![-w766](https://lh3.googleusercontent.com/-MCF0Iy8U4zU/XB85_HGpyvI/AAAAAAAAXxE/Tm6vWU2dGUo9-cei08I3_CStVVaBxJrUwCHMYCw/I/15455419708995.jpg)

 
注： resource release时机解释：每个资源有一个自己的使用计数，每当使用的时候会调用acquire方法给计数加1，每次job完成后会计数减1，当计数为零时进行release

## Object Pool的使用
我们知道如果较短时间内频繁的新建对象是会造成内存的抖动，可能造成界面的卡顿。
  ![-w713](https://lh3.googleusercontent.com/-adg_-2ZY2s0/XB85_RujvbI/AAAAAAAAXxI/9YJPvcJ_fFApFxDpGBaDUvxKd5ho2zjdwCHMYCw/I/15455434429422.jpg)
Object Pool就是为了避免这种情况发生的一种常见设计模式。它会初始化一些列的对象，当需要对象的时候不是去创建一个新的对象，二手去复用闲置的对象资源。当没有闲置资源的时候进行扩容。
Glide里大量使用了Object Pool的实现。

## Bitmap Pool
Bitmap Pool是保证Bitmap回收使用管理的类，避免了Bitmap被大量创建造成频繁GC的问题。
下面已LRUBitmapPool为安利分析
LRUBitmapPool持有LruPoolStrategy(具体实现是SizeStrategy，SizeConfigStrategy），它是正常持有Bitmap缓存的地方

com.bumptech.glide.load.engine.bitmap_recycle.LruBitmapPool
```
  private final KeyPool keyPool = new KeyPool();
  private final GroupedLinkedMap<Key, Bitmap> groupedMap = new GroupedLinkedMap<>();
  private final NavigableMap<Integer, Integer> sortedSizes = new PrettyPrintTreeMap<>();
```
Key 根据width和height算出实际的bitmap大小的一个类，作为HashMap的键 (后面会解释为什么Key是bitmap大小）
KeyPool是用宽高生成一个Key的ObjectPool
groupedMap： 缓存Bitmap的对象，是一个修改过的LinkedHashMap作为LRU cache
sortedSizes： 记录每个Key下对应的Bitmap的数量，且已大小排序

### 为什么Bitmap Pool的Key是Bitmap的尺寸
Android里默认的API每次decode bitmap或则一些对bitmap的操作时默认是新创建一张Bitmap。API 11以后提供了BitmapFactory.Options.inBitmap这个变量来告诉系统去复用原来的已经存在的bitmap而不是重新创建bitmap从而达到节省内存的目的。
但是inBitmap的使用有一些限制，复用的bitmap的大小必须大于等于原来的尺寸。所以需要用bitmap的尺寸作为Key来缓存bitmap。
![-w482](https://lh3.googleusercontent.com/-BzN1rbQoWRY/XB85_2BlPnI/AAAAAAAAXxM/5HwR9_Cl_IkAmSk5ccAr5iQT_7gw67uzQCHMYCw/I/15455438718436.jpg)
### 其他Tips
* 不同的BitMapConfig不能复用，所以需要用不同的Pool来缓存bitmap。这里也就是Glide里的
com/bumptech/glide/load/engine/bitmap_recycle/SizeConfigStrategy.java 存在的意义
![-w465](https://lh3.googleusercontent.com/-rwxU4r4ksws/XB86AXKLzwI/AAAAAAAAXxQ/5mBtmKvaG_MN2ok2MOxtzt97dj3cweDnwCHMYCw/I/15455439182865.jpg)

## 总结
* 从Glide里我们可以学习到如何用LruCache建立多级缓存的逻辑。
* 如何使用inBitmap对bitmap进行缓存
* LinkedHashMap是如何实现LRU管理的（双向循环链表）
## 参考
[Glide v4 Caching](https://muyangmin.github.io/glide-docs-cn/doc/caching.html)
[Re-using Bitmaps](https://www.youtube.com/watch?v=_ioFW3cyRV0)

