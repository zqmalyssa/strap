---
layout: post
title: Saltstack相关
tags: [code, operations]
author-id: zqmalyssa
---

工作需要Saltstack走一波，需要和ansible放到一起说下

#### Saltstack安装

简单说下安装

```html
首先是配置阿里云的源

服务端安装
yum -y install salt-master salt-minion

客户端安装
yum -y install salt-minion

照理说测试环境master都不要配置什么

客户端把master改成服务端的hostname或者ip，文件位置

/etc/salt/minion

然后分别启动即可，服务端也就是客户端，所以有两个客户端节点

服务端操作去配置认证：

salt-key -L //查看
salt-key -A //全部接受

进行测试
salt '*' test.ping

基本没什么问题

有点问题的地方是如果有定制的地方，那么salt的配置方法就可能有所不一样，比如2015的salt，其实配置文件在了/etc/salt/minion.d/master.conf去进行的配置
然后删除 rm /etc/salt/pki/minion/minion_master.pub才行，这样测试才能通过

```


#### Saltstack使用

salt有一个module的使用，与python的函数结合起来的

这里说明下，一个module就是一个python文件，系统模块在site-packages/salt/modules下面，自定义的模块在/srv/salt/_modules下面

一般没有上面的路径，需要创建一下，简单写个python

```python
def world():
  return 'Hello World'
```

然后将模块同步刷新给所有client

```html
salt '*' saltutil.sync_modules

成功话会有module的名字的哦

执行一下

salt '*' helloworld.world

python的文件名.函数名
```
