---
title: python循环之操作数据源
date: 2018-12-02 17:56:13
categories:

- python

tags:

- python

---

## 循环中的数据源操作

话不多少先来看段代码

``` python
for m_list in manual_list:
module_dict = m_list.get('module')
if module_dict['templateId'] == 1:
d_list = m_list.get('data')
for d in d_list:
if d.get('title') == '奥特曼之最强属性':
d_list.remove(d)
```

数据结构如下

![image-20181202163341608](/var/folders/56/dm9s_xsx5n5d885qn5cpfs000000gn/T/abnerworks.Typora/image-20181202163341608.png)

由于业务需求需要过滤掉data中title为“奥特曼之最强属性”的数据项，以上代码是否可以完成这一需求？

经由本地测试完全没问题，于是自信满满的上线了。

然而过段时间又有了新需求要求将title为“神奇宝贝之无限金币”的数据项也过滤掉，如法炮制,继续hardcode

```python
for m_list in manual_list:
                    module_dict = m_list.get('module')
                    if module_dict['templateId'] == 1:
                        d_list = m_list.get('data')
                        for d in d_list:
                            if d.get('title') == '奥特曼之最强属性' or d.get('title') == '神奇宝贝之无限金币':
                                d_list.remove(d)
                                
```

肯定没问题啊就是加个筛选条件吗！然而结果啪啪打脸，“神奇宝贝之无限金币”数据项并没有过滤掉，经过一番折腾打死也不相信自己这块儿代码写错了，然而只改动了这里，于是找python遍历过程操作数据源的详细过程。

python循环中删除数据源：

![image-20181202170502018](/var/folders/56/dm9s_xsx5n5d885qn5cpfs000000gn/T/abnerworks.Typora/image-20181202170502018.png)

删除“奥特曼之最强属性”后数据格式如下：

![image-20181202170934128](/var/folders/56/dm9s_xsx5n5d885qn5cpfs000000gn/T/abnerworks.Typora/image-20181202170934128.png)

所以，为了避免这种遗漏现象我们采用遍历数据要与要操作的数据源分离的操作方式，代码如下：

```python
for m_list in manual_list:
                    module_dict = m_list.get('module')
                    if module_dict['templateId'] == 1:
                        d_list = m_list.get('data')
                        for d in d_list[:]:
                            if d.get('title') == '奥特曼之最强属性' or d.get('title') == '神奇宝贝之无限金币':
                                d_list.remove(d)
```



d_list[:]拷贝一份作为遍历数据源这样删除操作则不会影响我们要操作的数据源。

## 说到拷贝不得不提下深浅拷贝

无论什么语言都涉及到深浅拷贝，浅拷贝引用同一块内存空间；深拷贝赋值操作后的两个变量的指向的是不同的内存地址。

python中切片赋值方式则为深拷贝代码：

![image-20181202173053054](/var/folders/56/dm9s_xsx5n5d885qn5cpfs000000gn/T/abnerworks.Typora/image-20181202173053054.png)

## 最后

推荐使用python自带的filter优化过滤，如果你有更好的方法或者要吐槽的内容欢迎留言。

