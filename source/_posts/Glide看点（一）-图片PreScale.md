---
title: Glide看点（一） 图片PreScale
date: 2018-12-15 16:37:53
tags: Android, Glide
photos:
- https://images.unsplash.com/photo-1529651795107-e5a141e34843?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=1500&q=80
---

#### Glide是如何根据View大小对图片进行缩放优化呢 （pre scale)
我们都知道图片实际大小和实际显示的区域一般不能是一样的，所以如果图片尺寸本身很大但是展示区域很小的话就会，如果已完整尺寸对图片进行加载就会效率很低浪费内存，加载速度慢。所以一般我们会根据图片实际需要显示的实际大小对图片进行缩放。
Android内置了inSampleSize方法来对图片进行缩放, Glide抽象出了一个类叫Downsampler.class来根据期望的图片大小和缩放策略进行向下采样。所以关键点变为了Glide如何知道我们的View实际需要的显示区域。下面将通过一个性能问题的debug故事分析glide源码
####  Wrap Content引发的性能问题“血案”
一个类似聊天信息的界面，如果列表里有4，5张图片后，滑动就会变得异常卡顿。![screen_shoot-w540](https://lh3.googleusercontent.com/-nIhLIng7BgQ/XAzlDXhtuQI/AAAAAAAAXi4/gLy4BR_ctw06PVZhvl2zvTcWt78_5SsMACHMYCw/I/screen_shoot.jpeg)

### 诊断
打开Android Monitor抓取性能Trace，发现咦咋性能消耗最大的是Glide decode Bitmap。
![performance](https://lh3.googleusercontent.com/-w0li48k0DMs/XAzlD3LjLnI/AAAAAAAAXi8/pHdkLKcqx7YezzLYe6XcAtgdKNy5cfgigCHMYCw/I/performance.png)
感觉不应该啊，我图片虽然可能很大但是实际显示区域不大，Glide在显示前应该已经会根据ImageView的实际显示效果进行了缩放采样（DownSample）的优化不应该产生卡顿啊。
*****等等*******
难道缩放采样出现了问题？

我们根据Glide的调用链找分析。
`
    GlideApp.with(context).load(imgUrl).into(target)
`
RequestBuilder的into方法其实是调用管理图片加载请求的RequestTrack里的track方法最后调用到了具体Request的begin方法。
Step1:
SingleRequest.run()方法 -> ViewTarget.getSize() 
![](https://lh3.googleusercontent.com/-PA1liE7Xv4Q/XAzlEgt23mI/AAAAAAAAXjA/y6i3o5QSFGQcoiLiSWMCj6oy09FZOEEPQCHMYCw/I/15443470777195.jpg)
如果开发者有制定大小OverrideSize就会直接采用该尺寸，如果没有就去ViewTarget里去拿size
ViewTarget.getSize -> sizeDeterminer.getSize(cb); 
![](https://lh3.googleusercontent.com/-P8wZJt_gZow/XAzlFuAXigI/AAAAAAAAXjE/McvQrudFhAopNkYxflkrFZ2JmZnYjrOQwCHMYCw/I/15443472154597.jpg)
ViewTarget里会根据SizeDeterminer去获取TargetView的具体尺寸。如果能直接从View里读取到具体的尺寸就直接用。
![](https://lh3.googleusercontent.com/-c21fwwJI7xY/XAzlGrjViaI/AAAAAAAAXjI/KjDCqb9dyr0kKMOHjhaaOWQxKkG3m3E0QCHMYCw/I/15443473205402.jpg)
如果读取不到就用PreDrawListener在layout完成后去监听view的实际尺寸：
observer.addOnPreDrawListener(layoutListener); 
关键点来了，去看添加的preDrawListener,里面有个特殊的if语句，如果view定义了WRAP_CONTENT, Glide就无法判断到底使用什么大小去preScale图片，所以它取了屏幕的最大宽度作为图片的加载大小。（不取图片的Target.SIZE_ORIGINAL的原因是可能图片会很大，远远大于屏幕尺寸）
SizeDeterminerLayoutListener:
`
 
      if (!view.isLayoutRequested() && paramSize == LayoutParams.WRAP_CONTENT) {
        if (Log.isLoggable(TAG, Log.INFO)) {
          Log.i(TAG, "Glide treats LayoutParams.WRAP_CONTENT as a request for an image the size of"
              + " this device's screen dimensions. If you want to load the original image and are"
              + " ok with the corresponding memory cost and OOMs (depending on the input size), use"
              + " .override(Target.SIZE_ORIGINAL). Otherwise, use LayoutParams.MATCH_PARENT, set"
              + " layout_width and layout_height to fixed dimension, or use .override() with fixed"
              + " dimensions.");
        }
    return getMaxDisplayLength(view.getContext());
      }
`
### 元凶
看到这里刚才的性能问题迎刃而解，虽然我们界面里的ImageView实际显示区域很小（maxWitdh和maxHeight限制的），但是Glide不知道。一看我们list item的布局果然如此

![](https://lh3.googleusercontent.com/-d6Q239URgRs/XAzlH3SplII/AAAAAAAAXjM/fNRZf4EgtRM2jl9XDngeqqhz3ixkH7DvACHMYCw/I/15161493230346.jpg)
所以在列表里的每一张图片都是以手机的最大宽度去加载的，所以decode bitmap才花了巨大的时间。

所以Fix这个问题只需要，在加载图片前设置好需要的加载宽度就可以解决啦。

### 总结
图片加载的一个重要手段是，“按需显示”更具实际需要显示的图片大小进行预缩放（Pre Scale)， PreScale要根据图片的显示大小和图片本身实际大小来计算inSampleSize来进行。Glide在处理Wrap Content的ImageView的时候，会不知道如何进行PreScale需要指定好具体显示的尺寸，不然可能会引发性能问题。

