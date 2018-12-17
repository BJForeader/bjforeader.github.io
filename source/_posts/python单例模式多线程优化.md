---
title: 单例模式多线程优化
date: 2018-12-16 17:05:36
tags:
---

# 单例模式（singleton Pattern）

#### 定义：

顾名思义，在程序的生命周期中只有一个该类的对象

### 首先回顾下之前语言中的单例模式优化

简单的声明方式:

```
public sealed class Singleton
{
static Singleton instance=null;
Singleton()
{

}

public static Singleton Instance
{
get
{
if(instance ==null)
{
  instance =new Singleton();
}
return instance;
}
}
}
```

#### singleton Pattern  的优点：

- 直到类想要产生一个实例时才会初始化，被叫做“惰性实例化”，避免了程序初始化时生成没必要的实例；
- 类的实例化放在属性内部，这样可以有一些附加功能（例如对子类的实例化）；

#### 单例模式简单声明的弊端：

（多线程访问Instance属性要求实例化情况下会产生多个实例，违背了设计模式的用意）

#### 解决方案：

```
public sealed class Singleton
{
static readonly object auxiliaryLock=new object();//进程辅助对象
static Singleton instance=null;
Singleton()
{

}

public static Singleton Instance
{
get
{
lock(auxiliaryLock )//加了锁的这部分代码只有一个线程可以访问
{
if(instance ==null)
{
  instance =new Singleton();
}
return instance;
           }

}
}
}
```

这样做避免了多线程触发问题，但存在一个问题每次访问Instance时不管instance是否为null都会加锁独占影响性能 

进一步改进 

```python
public sealed class Singleton
{
static readonly object auxiliaryLock=new object();//进程辅助对象
static Singleton instance=null;
Singleton()
{

}

public static Singleton Instance
{
get
{
if(instance ==null )//加了这个判断就可以排除已经有该实例的没必要的加锁操作
{
lock(auxiliaryLock )//加了锁的这部分代码只有一个线程可以访问
{
if(instance ==null)
{
  instance =new Singleton();
}

           }
}
return instance;

}
}
}
```

这种方式仍然有很多缺点：无法实现延迟初始化。

进一步改进：静态初始化

```
public sealed class Singleton
{
static readonly Singleton instance=new Singleton();
static Singleton()
{

}
Singleton()
{

}
public static Singleton Instance
{
get
{
return instance;
}
}
}
```

### python中优化

#### 首先还是通过最熟悉的类的形式创建

```
class Singleton(object):

    def __init__(self):
        import time
        time.sleep(2)#模拟执行IO操作

    @classmethod
    def instance(cls, *args, **kwargs):
        if not hasattr(Singleton, "_instance"):
            Singleton._instance = Singleton(*args, **kwargs)
        return Singleton._instance
```

#### 和上面一样还是来测试下多线程情况下会发生什么

```
import threading

def task(arg):
    obj = Singleton.instance()
    print(obj)

for i in range(10):
    t = threading.Thread(target=task,args=[i,])
    t.start()
```



改进

```
import time
import threading
class Singleton(object):
    _instance_lock = threading.Lock()

    def __init__(self):
        time.sleep(2)

   
    @classmethod
    def instance(cls, *args, **kwargs):
        if not hasattr(Singleton, "_instance"):#选择性加锁
            with Singleton._instance_lock:
                if not hasattr(Singleton, "_instance"):
                    Singleton._instance = Singleton(*args, **kwargs)
        return Singleton._instance


def task(arg):
    obj = Singleton.instance()
    print(obj)
for i in range(10):
    t = threading.Thread(target=task,args=[i,])
    t.start()
time.sleep(20)
obj = Singleton.instance()
print(obj)
```

现在则是可以多线程可以调用的单例，实例化方式Singleton.instance()

#### 能不能像正常类一样初始化呢，不想区别对待

python中一个对象的创建过程是先执行“__new__“方法然后执行”__init__“方法，我们创建单例的过程可以放到new中

```
import threading
class Singleton(object):
    _instance_lock = threading.Lock()

    def __init__(self):
        time.sleep(2)


    def __new__(cls, *args, **kwargs):
        if not hasattr(Singleton, "_instance"):
            with Singleton._instance_lock:
                if not hasattr(Singleton, "_instance"):
                    Singleton._instance = object.__new__(cls)  
        return Singleton._instance
```

这样则可以像实例化正常类一样来实例化单例Singleton()

python提供metaclass（元类）方便对类的动态操作，很多静态语言中也有通过反射获取类的类型例如C#中的

```
typeof(Int32)
i.GetType()
```

那么使用元类实现的单例模式岂不更加合适

```

import threading

class SingletonType(type):
    _instance_lock = threading.Lock()
    def __call__(cls, *args, **kwargs):
        if not hasattr(cls, "_instance"):
            with SingletonType._instance_lock:
                if not hasattr(cls, "_instance"):
                    cls._instance = super(SingletonType,cls).__call__(*args, **kwargs)
        return cls._instance

class Singleton(metaclass=SingletonType):
    def __init__(self,name):
        self.name = name
```

调用则是Singleton('only')