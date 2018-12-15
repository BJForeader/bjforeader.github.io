---
title: Glide看点（二）巧用Fragment管理生命周期
date: 2018-12-15 16:21:27
tags: Android, Glide
photos:
- https://images.unsplash.com/photo-1527167598984-8802d8028eea?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=1500&q=80
---
# Motivation/动机
图片的一般是异步的，异步经常面临的问题是内存泄露和异步加载回来view已经销毁导致的空指针问题。而Glide在使用的时候只要求传入当前加载的view或者context，且没有用setLifeCyclelistener什么的方法就实现了生命周期的管理，它是如何做到的呢？
# Fragment Trick
我们知道Fragment和View最大区别其实就是多了生命周期，Glide采用的是用创建一个空白Fragment来实现生命周期的管理和监听的。这是如何实现的我们一步步看代码：
## Step 1：根据传入的Context得到FragmentManager
Glide.java

```
     @NonNull
      public static RequestManager with(@NonNull Context context) {
        return getRetriever(context).get(context);
      }
```

获取到正确的Context
com/bumptech/glide/manager/RequestManagerRetriever.java

```
      @NonNull
      public RequestManager get(@NonNull Context context) {
        if (context == null) {
          throw new IllegalArgumentException("You cannot start a load on a null Context");
        } else if (Util.isOnMainThread() && !(context instanceof Application)) {
          if (context instanceof FragmentActivity) {
            return get((FragmentActivity) context);
          } else if (context instanceof Activity) {
            return get((Activity) context);
          } else if (context instanceof ContextWrapper) {
            return get(((ContextWrapper) context).getBaseContext());
          }
        }
        return getApplicationManager(context);
      }
```

根据Context获取到FragmentManager

```
@NonNull
  public RequestManager get(@NonNull FragmentActivity activity) {
    if (Util.isOnBackgroundThread()) {
      return get(activity.getApplicationContext());
    } else {
      assertNotDestroyed(activity);
      FragmentManager fm = activity.getSupportFragmentManager();
      return supportFragmentGet(activity, fm, null /*parentHint*/);
    }
  }
```
### Step 2：RequestManagerFragment里注册Callback
获取SupportRequestManagerFragment，这个就是监听生命周期的主角Fragment了。下面去看看SupportRequestManagerFragment里有什么吧

```
  @NonNull
  private RequestManager supportFragmentGet(@NonNull Context context, @NonNull FragmentManager fm,
      @Nullable Fragment parentHint) {
    SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm, parentHint);
    RequestManager requestManager = current.getRequestManager();
   // 省略..部分
    return requestManager;
  }

```

其实里面就是一个持有ActivityFragmentLifecycle和RequestManager的空白Fragment。

```
public class SupportRequestManagerFragment extends Fragment {
  private static final String TAG = "SupportRMFragment";
  private final ActivityFragmentLifecycle lifecycle;
  @Nullable private RequestManager requestManager;
  
  // ..省略..
  @Override
  public void onStart() {
    super.onStart();
    lifecycle.onStart();
  }

  @Override
  public void onStop() {
    super.onStop();
    lifecycle.onStop();
  }

  @Override
  public void onDestroy() {
    super.onDestroy();
    lifecycle.onDestroy();
    unregisterFragmentWithRoot();
  }
}
```
RequestManager里又实现了ActivityFragmentLifecycle从而可以根据什么周期对请求做一些处理
如：在RequestManager里的onDestroy回调里取消所有图片加载请求，销毁监听器
com.bumptech.glide.RequestManager

```
  @Override
  public void onDestroy() {
    targetTracker.onDestroy();
    for (Target<?> target : targetTracker.getAll()) {
      clear(target);
    }
    targetTracker.clear();
    requestTracker.clearRequests();
    lifecycle.removeListener(this);
    lifecycle.removeListener(connectivityMonitor);
    mainHandler.removeCallbacks(addSelfToLifecycle);
    glide.unregisterRequestManager(this);
  }
```

## Bouns拓展
空白Fragment还有什么其他妙用呢？
* 动态申请权限。对于不是必要的权限不同意就退出App是非常糟糕的用户体验，最佳实践应该是当要用到该权限的时候再向用户申请。这样写面临的问题就会是授权代码会散落在各个地方，这样显然是很糟糕的。所以用一个空白的Fragment，把权限检查和申请逻辑全部丢到里面，在需要用到地方只要持有这个Fragment做一下检查就好。
如：

```
public class CameraMicPermissionHelper extends Fragment {
  private static final int REQUEST_CAMERA_MIC_PERMISSIONS = 10;
  public static final String TAG = "CamMicPerm";

  private CameraMicPermissionCallback mCallback;
  private static boolean sCameraMicPermissionDenied;

  public static CameraMicPermissionHelper newInstance() {
    return new CameraMicPermissionHelper();
  }

  public CameraMicPermissionHelper() {
    // Required empty public constructor
  }

  @Override
  public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setRetainInstance(true);
  }

  @Override
  public void onAttach(Activity activity) {
    super.onAttach(activity);
    if (activity instanceof CameraMicPermissionCallback) {
      mCallback = (CameraMicPermissionCallback) activity;
    } else {
      throw new IllegalArgumentException("activity must extend BaseActivity and implement LocationHelper.LocationCallback");
    }
  }

  @Override
  public void onDetach() {
    super.onDetach();
    mCallback = null;
  }

  public void checkCameraMicPermissions() {
    if (PermissionUtil.hasSelfPermission(getActivity(), new String[]{Manifest.permission.CAMERA, Manifest.permission.RECORD_AUDIO})) {
      mCallback.onCameraMicPermissionGranted();
    } else {
      // UNCOMMENT TO SUPPORT ANDROID M RUNTIME PERMISSIONS
      if (!sCameraMicPermissionDenied) {
        requestPermissions(new String[]{Manifest.permission.CAMERA, Manifest.permission.RECORD_AUDIO}, REQUEST_CAMERA_MIC_PERMISSIONS);
      }
    }
  }

  public void setCameraMicPermissionDenied(boolean cameraMicPermissionDenied) {
    this.sCameraMicPermissionDenied = cameraMicPermissionDenied;
  }

  public static boolean isCameraMicPermissionDenied() {
    return sCameraMicPermissionDenied;
  }

  /**
   * Callback received when a permissions request has been completed.
   */
  @Override
  public void onRequestPermissionsResult(int requestCode, String[] permissions,
                                         int[] grantResults) {

    if (requestCode == REQUEST_CAMERA_MIC_PERMISSIONS) {
      if (PermissionUtil.verifyPermissions(grantResults)) {
        mCallback.onCameraMicPermissionGranted();
      } else {
        Log.i("BaseActivity", "LOCATION permission was NOT granted.");
        mCallback.onCameraMicPermissionDenied();
      }

    } else {
      super.onRequestPermissionsResult(requestCode, permissions, grantResults);
    }
  }


  public interface CameraMicPermissionCallback {
    void onCameraMicPermissionGranted();
    void onCameraMicPermissionDenied();
  }

}
```
* 不建议的hack手段：
   在ViewModel出现之前，有一种很hack的方式用Fragment保存数据。在设备的configurationChange的时候保存大块数据。
关键方法, 把Fragment设置为true后可以在configurationChange的时候不被销毁，利用这一点保存数据。现在用ViewModel显然是更好的方法

```
    /* Control whether a fragment instance is retained across Activity re-creation (such as from a configuration change).*/
    
    setRetainInstance(boolean retain)

```

* 声明周期的监听，现在用Android Jetpack里的 Lifecycle-Aware Components来做是一个更简单的做法。


## 参考
* [Headless fragments](https://luboganev.github.io/blog/headless-fragments/)
* [Use headless Fragment for Android M run-time permissions and to check network connectivity](https://medium.com/@ali.muzaffar/use-headless-fragment-for-android-m-run-time-permissions-and-to-check-network-connectivity-b48615f6272d)


