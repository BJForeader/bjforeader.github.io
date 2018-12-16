---
title: pycharm远程调试
date: 2018-12-16 21:47:24
categories:

- tools

tags:

- python

photos:
 -   https://ws4.sinaimg.cn/large/006tNbRwly1fy9104uo7tj31hc0u0tid.jpg
---

## 动机

1. 一些bug由于本地环境和线上环境的不一致可能导致本地无法复现
2. 本地依赖和线上依赖版本不一致也可以导致一些问题
3. 有时一些bug跟数据相关，本地数据无法和线上数据一致
4. 有些三方平台会验证服务器的合法性或者异步回调结果，如微信支付，这时候本地无法测试

如上所诉，要是有一个很方便调试远程服务器的方法，岂不美哉。通过PyCharm我们可以很方便地实现远程调试，下面详细介绍下PyCharm这个牛叉的功能。

## 添加远程部署

1. 打开pycharm，tools-->Deployment-->Configuration

![](https://ws1.sinaimg.cn/large/006tNbRwly1fy8xrz7r2oj310d0u07wh.jpg)

2. 点击左边“+”添加远程服务器，随便起个名字，选择SFTP

![](https://ws2.sinaimg.cn/large/006tNbRwly1fy8zol12boj315g0p2tab.jpg)

3. 配置connection

![](https://ws1.sinaimg.cn/large/006tNbRwly1fy8yhdwo9jj310t0u0ah8.jpg)

4. 配置Mappings

   ![](https://ws1.sinaimg.cn/large/006tNbRwly1fy8ynlzintj30zz0u0aep.jpg)

5. 配置Excluded Paths（可选）

   ![](https://ws3.sinaimg.cn/large/006tNbRwly1fy8yrh6qtwj31720sc40w.jpg)

最后点击OK即可

再次打开部署选项，可以发现现在可以直接部署代码到服务器了，也可以直接下载带代码到本地，同时自动上传（Automatic Upload）是默认勾选的，我一般会把它去掉，防止一些本地测试代码上传上去

![](https://ws4.sinaimg.cn/large/006tNbRwly1fy8yw6bdovj313r0u07wh.jpg)

新增一个文件，查看deployment 选项，这时候就可以上传到远程服务器了

![](https://ws2.sinaimg.cn/large/006tNbRwly1fy8z53cccrj31by0msjyp.jpg)



## 添加远程解释器

远程部署仅仅只是同步和拷贝文件，要真正实现远程调试还需要配置远程解释器

1. 设置页面找到“Project Interpreter” --> 设置-->Add

![](https://ws4.sinaimg.cn/large/006tNbRwly1fy8zb2ooqoj31kw0u0ws6.jpg)

2. 选择“SSH Interpreter” --> "Existing server configuration" --> 选择刚才创建的部署配置，使用‘’Create“或者”Move“ 都OK

![](https://ws2.sinaimg.cn/large/006tNbRwly1fy8zdpxfewj31hg0u0tgu.jpg)

3. 点击下一步，这时会进行连接远程服务器，稍等一会，会出现以下界面，稍微配置下，点击“Finish”即可

![](https://ws3.sinaimg.cn/large/006tNbRwly1fy8zl5cur0j31fs0u00y3.jpg)

最后使用远程解释器,点击“OK”，返回到项目界面，等待同步完成即可

![](https://ws3.sinaimg.cn/large/006tNbRwly1fy8zmqlj4mj31fi0cg0uo.jpg)



## 远程调试

> 这里根据自己的具体项目情况而定，我这里是**Flask+阿里云+nginx+运行脚本**的一个例子

1. 新增一个 run configuration

![](https://ws2.sinaimg.cn/large/006tNbRwly1fy8zxkrmqmj30bm06amyi.jpg)

![](https://ws4.sinaimg.cn/large/006tNbRwly1fy900lq1qsj30d00tgafe.jpg)

![](https://ws2.sinaimg.cn/large/006tNbRwly1fy91brol9sj315a0lktco.jpg)

2. 运行脚本代码如下，这里使用了8000的端口

> **Host要配置为0.0.0.0**

![](https://ws1.sinaimg.cn/large/006tNbRwly1fy904q9vcyj30pc03g74i.jpg)

3. nginx 配置8000 端口

```shell
    server {
        listen       8000 ;
        listen       [::]:8000 ;
        server_name  _;
        root         /usr/share/nginx/html;
        access_log  /var/log/nginx/access_8000.log  main;
        error_log  /var/log/nginx/error_8000.log;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;
        location / {
		proxy_pass http://127.0.0.1:8000;
		proxy_set_header Host $host;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
```

4. 要是使用阿里云，还需要安全组开放8000 端口
5. 断点运行

![](https://ws4.sinaimg.cn/large/006tNbRwly1fy90aeah0dj30e0048glz.jpg)

![](https://ws3.sinaimg.cn/large/006tNbRwly1fy9170wpemj30qg01w3yv.jpg)

6. 出现上图所示的时候，恭喜你，你已经可以断点调试远程服务器了

我们打一个断点，然后试着访问一个API服务：http://xx.xx.xx.xx:8000/api/pages/bookstore

完美断上

![](https://ws3.sinaimg.cn/large/006tNbRwly1fy90trr9yij318s0u0n2c.jpg)

### 补充说明

不建议在正式服务器使用这个功能，可以在测试服务器使用