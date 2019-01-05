---
title: 为什么你应该使用ViewModel
date: 2019-01-05 17:24:01
tags: Android, 构架
photos:
- https://images.unsplash.com/photo-1524803504179-6d7ae4d283f7?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=1001&q=80
---
### MVP vs MVVM
#### MVP
MVP代表Model–view–presenter，用图来表示MVP的思想如下：
![](https://lh3.googleusercontent.com/-qCIouASXirE/XDB3kMH2Z7I/AAAAAAAAX0o/_xt_KXxew7gkfKMt2xw29H1YNJo5WMEQQCHMYCw/I/15466751424530.jpg)
*图片来之wikipedia*

Presenter作为中间人，负责在View和数据Model之间处理业务逻辑。其实这里隐藏了Presenter带来的问题，Presenter的作用不单纯他需要一边与View打交道，一边与数据打交道。只能剥离一些Activity/Fragment的代码。

下面已Google architecture sample里MVP的代码作为例子：
这是一个加载一个todo list的界面例子
MVP会先定义接口作为协议

```
interface TasksContract {

    interface View : BaseView<Presenter> {
        var isActive: Boolean

        fun setLoadingIndicator(active: Boolean)

        fun showTasks(tasks: List<Task>)

        fun showNoTasks()

        // 省略
    }

    interface Presenter : BasePresenter {

        fun loadTasks(forceUpdate: Boolean)

        fun result(requestCode: Int, resultCode: Int)

        // 省略
    }
}
```
TasksPresenter代码：

```
class TasksPresenter(val tasksRepository: TasksRepository, val tasksView: TasksContract.View)
    : TasksContract.Presenter {

       private fun loadTasks(forceUpdate: Boolean, showLoadingUI: Boolean) {
        // 这里需要知道view的状态来判断是否需要显示loading，showLoading由view传递过来。这里已经开始让逻辑不单一了
        if (showLoadingUI) {
            tasksView.setLoadingIndicator(true)
        }
        // 省略部分代码...

        // 加载数据
        tasksRepository.getTasks(object : TasksDataSource.LoadTasksCallback {
            override fun onTasksLoaded(tasks: List<Task>) {
                val tasksToShow = ArrayList<Task>()

                // 当View不是active的时候直接返回，这里有要考虑View被销毁，回调还在的问题，代码持续ugly
                if (!tasksView.isActive) {
                    return
                }
                if (showLoadingUI) {
                    tasksView.setLoadingIndicator(false)
                }
                processTasks(tasksToShow)
            }

            override fun onDataNotAvailable() {
                // The view may not be able to handle UI updates anymore
                if (!tasksView.isActive) {
                    return
                }
                tasksView.showLoadingTasksError()
            }
        })
    }

}
```
承担View职责的是一个Fragment

```
class TasksFragment : Fragment(), TasksContract.View {
    internal var itemListener: TaskItemListener = object : TaskItemListener {
        override fun onTaskClick(clickedTask: Task) {
            presenter.openTaskDetails(clickedTask)
        }

        override fun onCompleteTaskClick(completedTask: Task) {
            presenter.completeTask(completedTask)
        }

        override fun onActivateTaskClick(activatedTask: Task) {
            presenter.activateTask(activatedTask)
        }
    }

     // 省略代码...
    setOnRefreshListener { presenter.loadTasks(false) }
}
```
可以从上述代码了解到，MVP模式面临的问题是

* View和Presenter之间需要相互持有
* Presenter职责不单一，作为中间人同时要管理view和数据，让代码看起来丑陋。而我们期望Presenter的作用应该是只对数据进行逻辑操作而不需要关系View的状态。

#### MVVM
![](https://lh3.googleusercontent.com/-qiP0oo7E56Q/XDB3gFg8kQI/AAAAAAAAX0c/4VnI4hbRNxQZe2Tr9z2u9qv6iCC7xONEgCHMYCw/I/15466771241137.png)
*图片来之wikipedia MVVM*

MVVM是通过数据绑定，让原来的View和Presenter的职责更单一的改进。Model提供数据，ViewModel来做业务逻辑处理和管理状态，View只更具ViewModel的数据自己变化显示。
这样的想法很好，如何在Android里实现呢，答案是ViewModel和LiveData

### ViewModel + LiveData实现MVVM
Android里去年引入的ViewModel类，被设计用来保存和View显示逻辑相关的数据，它是忽略configuration change的一个类，就算因为configuration change数据依然会被保留。
LiveData可以简单理解为一个放数据的容器，并且知晓UI的生命周期，它是一个数据的观察者它可以知晓数据的变化，所以叫它LiveData活数据嘛。

同样是前面MVP展示的逻辑，一个todo list应用需要加todo列表

```
public class TasksViewModel extends AndroidViewModel {
    // 数据列表
    public final ObservableList<Task> items = new ObservableArrayList<>();
    // loading状态
    public final ObservableBoolean dataLoading = new ObservableBoolean(false);

     // 省略代码...

    private void loadTasks(boolean forceUpdate, final boolean showLoadingUI) {
        // 只用设置loading的状态，不再需要持有View
        if (showLoadingUI) {
            dataLoading.set(true);
        }

        mTasksRepository.getTasks(new TasksDataSource.LoadTasksCallback() {
            @Override
            public void onTasksLoaded(List<Task> tasks) {
                // 单纯的更新数据，而不再用操心View的状态
                List<Task> tasksToShow = new ArrayList<>();

                if (showLoadingUI) {
                    dataLoading.set(false);
                }
                mIsDataLoadingError.set(false);

                items.clear();
                items.addAll(tasksToShow);
                empty.set(items.isEmpty());
            }

            @Override
            public void onDataNotAvailable() {
                mIsDataLoadingError.set(true);
            }
        });
    }
}
```

View的逻辑，这里用了Android的DataBinding，loading的显示更具ViewModel里的值来判断

```
<SwipeRefreshLayout
    android:id="@+id/refresh_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:onRefresh="@{viewmodel}"
    app:refreshing="@{viewmodel.dataLoading}"/>

```

总结，MVVM可以让View的职责更加单一只是根据数据来显示。ViewModel可以单纯的处理数据而不用关心View的状态
### ViewModel的其他好处
* Fragment间传递数据
  Fragment之前直接传递数据是如此的痛苦，小数据放bundle put也就忍了，数据太大还只能通过直接持有Fragment的引用传入数据引用。现在有了ViewModel可以很方便的传递数据，因为在同一个Activity里可以获得到同一个ViewModel，数据可以直接拿来用。
  这样的好处显而易见，
  * Activity不需要了解Fragment直接通信的过程
  * Fragment直接也不需要互相了解，只需要从ViewModel里拿数据就好了
  * 数据会因为是通过引用传递过去的，其中一个Fragment因为生命周期而影响数据

* 测试方便
ViewModel不再持有具体的Android类，View啊Activity，Fragment啥的，给测试单独ViewModel提供了可能。
* 代替onSaveInstanceState保存状态
Fragment之前保存数据和恢复状态一直很疼，需要自己onSaveInstanceState，并且因为onSaveInstanceState用的bundle并不能放入大量数据，只能简单的保存一些简单的状态。但是现在有ViewModel它不会随着configuration change而变化，数据就静静的呆在那所以Fragment被回收后从新创建也不用担心数据丢失了。

### ViewModel的秘密
ViewModel的神奇的一点是它到底如何逃离configuration change导致的生命周期变化的呢？
其实非常简单，是利用了Fragment的
`Fragment.setRetainInstance(true)`。


看源码：
ViewModelStores是保存ViewModel的地方
![](https://lh3.googleusercontent.com/-TD3oTnqKz_w/XDB3gyhU-kI/AAAAAAAAX0g/QxAMBiU3VBkUDaZRxR8GbBJunB1Ct6HFgCHMYCw/I/15466795141530.jpg)
ViewModel是重HolderFragment里取出。

HolderFragment
![](https://lh3.googleusercontent.com/-dZ2I1pVEHKk/XDB3hSp65ZI/AAAAAAAAX0k/SkXAcL0skdADW9PLdF6qAnxJz0p6Ni7VwCHMYCw/I/15466795743944.jpg)

### 参考
* [ViewModels: Persistence, onSaveInstanceState(), Restoring UI State and Loaders](https://medium.com/androiddevelopers/viewmodels-persistence-onsaveinstancestate-restoring-ui-state-and-loaders-fc7cc4a6c090)

* [Google samples - Android Architecture Blueprints](https://github.com/googlesamples/android-architecture)
* [ViewModels and LiveData: Patterns + AntiPatterns](https://medium.com/androiddevelopers/viewmodels-and-livedata-patterns-antipatterns-21efaef74a54)
* [Android MVP vs MVVM and the winner is...](https://www.youtube.com/watch?v=ugpC98LcNqA&t=989s)


