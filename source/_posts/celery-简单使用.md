---
title: celery 简单使用
date: 2018-12-23 22:20:21
tags:
---

# celery

工作中我们经常会遇到一种业务场景，用户触发某种操作，这个操作很耗时，但是又不能影响用户体验，让用户等待降低产品体验。那些针对耗时操作处理比较好的应用是怎么做到的？举一个最常见的应用场景，大家应该都有过在亚马逊购物的经历，应该也参加过一些秒杀活动。除了系统繁忙，应该还有过下单成功，结果过了段时间却收到库存不足的致歉信。这种快速给用户反馈，实际却还没做的处理方式被称为异步任务队列。

今天，介绍一个基于python语言实现的一个分布式队列服务celery

### 适用场景

- 即时任务 **task**
- 定时任务 **beat**

### 5大核心角色

- Task 服务端接到请求快速给出响应，并把耗时任务放到任务队列中交由celery处理，相当于生产者的角色

- Broker 中间商，简单理解即存放任务的队列

- Worker 顾名思义执行任务的人，即消费者

- Beat 定时任务调度器，定时任务中的生产者

- Backend 任务执行的返回值都存放在Backend中

  

### celery具体使用

安装 pip install -U celery

创建celery实例

```
from celery import Celery
celery=Celery('tasks',broker='redis://127.0.0.1:6379/2',backend='redis://127.0.0.1:6379/2')#名称，任务队列，和返回结果存放位置
```

创建任务

```python
@celery.task
def send_email(email):
    print "send email"
```

启动 worker

```
celery -A tasks worker --loglevel=info
```

### Flask项目中应用

![image-20181223220212852](/var/folders/56/dm9s_xsx5n5d885qn5cpfs000000gn/T/abnerworks.Typora/image-20181223220212852.png)

Service.py调用

```python
@classmethod
def add_read_time(cls,  book_id, user_id, duration):
    from services.read_time.read_time_upload import add_read_time
    add_read_time.delay( book_id, user_id, duration)#通过delay方式调用被celery.task
    
```

Config.py

```
CELERY_IMPORTS = ('services.read_time.read_time_upload',)#task和celery实例不在一个目录引入使用
```

Celery_worker.py

```
from app import create_app, celery
app = create_app()
app.app_context().push()
```



启动work：celery -A celery_worker.celery  worker --loglevel=info

![image-20181223221602082](/var/folders/56/dm9s_xsx5n5d885qn5cpfs000000gn/T/abnerworks.Typora/image-20181223221602082.png)

Redis 

![image-20181221160659535](/var/folders/56/dm9s_xsx5n5d885qn5cpfs000000gn/T/abnerworks.Typora/image-20181221160659535.png)

方法触发之后



![image-20181221161530681](/var/folders/56/dm9s_xsx5n5d885qn5cpfs000000gn/T/abnerworks.Typora/image-20181221161530681.png)

![image-20181223221423786](/var/folders/56/dm9s_xsx5n5d885qn5cpfs000000gn/T/abnerworks.Typora/image-20181223221423786.png)

![image-20181221181509694](/var/folders/56/dm9s_xsx5n5d885qn5cpfs000000gn/T/abnerworks.Typora/image-20181221181509694.png)

