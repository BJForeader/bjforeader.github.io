---
title: Kotlin coroutine（一）异步回调的故事
date: 2018-12-15 16:41:13
tags: Android, kotlin
photos:
- https://images.unsplash.com/photo-1459792323369-14e51bb68de0?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=1500&q=80
---

## Part 1, Motivation
在前端的世界里，为了保证UI的流畅性，耗时的操作都应该丢到异步任务里去。但是传统的异步回调却在实际工程里带来巨大的麻烦。

    * 当多个异步任务需要组合执行的时候，嵌套起来可读性，维护性很差 （ *Callback Hell 回调地狱* ）
    * 界面销毁时，异步任务结束后的回调要进行判断，不然会造成异常

下面一个例子展示一个常见的开发场景下的问题：
同步用户书架，并获取最新章节标志，流程如下：

```
* 从数据库查询用户的本地数据
* 从服务器拉取用户云端备份数据
* 两份数据去重合并
* 更具合并后的数据请求服务端拉取最新章节信息
```

传统异步回调式写法：
![](https://lh3.googleusercontent.com/-B5W1pTTBp6A/XAJhjDkI3QI/AAAAAAAAXY8/yRT7LxcpR-0-Ofj1QJtMdGoKLLWYy1uYgCHMYCw/I/15436514890844.jpg)

这样的代码异常难读且非常容易出bug
但是如果代码可以改为类似如下的代码呢? 瞬间可读性，强了数倍。

    val localData = fetchUserBookFromDB()
    val remoteData = fetchUserBookFromRemote()
    val mergeData = mergeData(localData)
    val result = fetchBookUpdate(mergeData)

## Part 2, 异步问题的改进史
### Future/Promise/RxJava
如果每个异步函数都要传递一个callback，来处理问题很容易造成嵌套的结构，那有没有办法回调扁平化处理呢？更具这样的思路，于是就产生了promise和Future。
其实他们都是相当于封装了一个统一的类作为回调把回调从参数列表里移除并做为返回值包装了起来，就可以用来进行链式调用。
Before:

    fun requestSomethingAsync(callback: Callabck<Data>)
    
    requestSomethingAsync(object: Callback {
        override onSuccess() {
            doAnotherThingAsync(...)
        }
        override onFailed() {
        }
    })

After:

     fun requestSomethingAsync():Promise<Data> {}
     
     requestSomethingAsync()
        .then(res -> {})
        .catch(err -> {})
### async/await
链式调用虽然解决了层级嵌套的问题，但是仍然不是很优雅（想想RxJava那么多种操作符）。与是有了async和await关键字的概念，它可以把异步任务改为同步的方法进行编写。
 
    suspend fun fetchBookUpdateInfo() {
        val localData = fetchUserBookFromDB().await()
        val remoteData = fetchUserBookFromRemote().await()
        val mergeData = mergeData(localData).await()
        val result = fetchBookUpdate(mergeData).await()
    }
## Part 3, Kotlin coroutine基本概念


### coroutine Builder
如字面上的意思，这写类提供创建coroutine的方式
launch, async, runningBlock 等等

* launch： 按coroutine作者Roman Elizarov说的那样，就像调用new Thread开启一个新的线程做任务，属于Fire and Forget（用完就走，即不用操心任务结果成功失败的那种）
![](https://lh3.googleusercontent.com/-rLe_-FttLlk/XAJhje5L4EI/AAAAAAAAXZA/VD2qdSEF_1kOQgEB6N9wf4hlzjf8DpT6ACHMYCw/I/15436567921769.jpg)
它会立即返回一个Job，Job里拥有如下这样类似Thread的方法。
![](https://lh3.googleusercontent.com/-EAQGb2nCql0/XAJhjobs0EI/AAAAAAAAXZE/YzmKFHbuiTsfge-4g__fY8OF95TgnJ_TwCHMYCw/I/15436568390391.jpg)
* async： 返回的是Deferred,Deferred继承与Job，比一般Job多的是

下面是一个栗子：
![](https://lh3.googleusercontent.com/-ZqSO_O5AhEQ/XAJhj6gdwNI/AAAAAAAAXZI/oxonltVPUVIyF45EjLIubKtItmgGAQLXgCHMYCw/I/15436584851861.jpg)

### coroutine context
coroutine里运行的上下文。规定运行使用的环境是什么。这里和RxJava里决定使用什么类型的线程池一样的概念。
分为Main，IO，Default，Unconfine。
Main就是android里的UI线程，IO是一个线程池使用与做IO操作。
给coroutine制定不同的context可以大幅度的简化UI和取数据的逻辑。
下面我改写上面的栗子：
![](https://lh3.googleusercontent.com/-1U5rCf-ZnOk/XAJhkJ9hsFI/AAAAAAAAXZM/FqFcKAYVirwsLWZRKCmflZI1QgIS0S5JwCHMYCw/I/15436590604220.jpg)
我们在UI事件上进行对View的操作，在load数据的时候使用的IO的context来进行。

### suspend关键字
suspend可以标志这个方法会时当前运行时挂起。suspend函数只能在coroutine里运行。

### 预告
改用async/await的编程形式大大简化了异步复杂的交互逻辑。并且对错误处理很友好（可以直接对catch）。下一篇将会简单介绍Coroutine背后实现的一些原理。




