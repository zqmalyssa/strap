---
layout: post
title: Python相关
tags: [code, python]
author-id: zqmalyssa
---

python好几年不碰了，工作需要总结一下

#### Python使用前置

简答的一个项目下来，基本在这边或多或少有点问题

```python
import

from import
```

首先是本机环境的安装，最好装两个版本，一个是2.7还有一个是3的版本，然后将3版本的.exe换成python3这样，就可以在机器上同时用cmd跑p2和p3了

好了，安装破解版pycharm，intellij的全家桶，破解方式看工具文章，可以选python版本，每个版本下的依赖包也会展示

然后可以用pip进行包管理，有时候需要wheel辅助

```python

//一般这样
python -m pip install salt

//有的时候需要升级pip到新版本
python -m pip install --upgrade pip

//安装wheel的方式，先下载，再指定绝对路径
python -m pip install "D:/XXX/XXX"

//写requirements.txt的方式安装

requests>=2.9.1
suds>=0.4
redis>=2.10.0

python -m pip install -r requirements.txt

//通过gitlab等远程安装
python -m pip install git+http://git.XXX.XX.com/XXX/serviceclient.git

//下载别人的依赖包，放到site-packages里面
可以去down源码，然后解压到这个目录就可以了，也可以从别人的机器那边copy过来

```

#### Python基本使用

python的语法要看看，然后python中的数据结构是

列表
LIST: 可变数组
[]

切片，可以取列表中的一部分
[:]

元组，元组中的值是不能修改的，test[0] = '123', 这样是不行的，但可以这样赋值，test = (100, 123)
()

字典
DICT: Key和Value的形式
{}

#### Python容器

python用Django写的web项目也可以打包成镜像，发到kubernetes中，这边Dockerfile可以参考下面

```html
FROM python:3.6.8
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
MAINTAINER zqmalyssa@hotmail.com
ADD ./djangoPro2p3 /code
WORKDIR /code
RUN pip install -r requirements.txt
EXPOSE 8087
CMD ["python", "/code/manage.py", "runserver", "0.0.0.0:8087"]
```
网不好的时候，pip install的时候可能会卡，还有CMD最后的0.0.0.0:8087要加，不然docker run后外面的访问会被301掉的，因为它默认是127.0.0.1，是一个回环地址，从容器外进不来的

然后部署到kubernetes的时候也需要注意，需要将Django中的`settings.py`文件中的allowed_hosts

```html
ALLOWED_HOSTS = []
ALLOWED_HOSTS = ['*'] #修改成这样
```
kubernetes才能访问，不想重新打镜像也可以直接在容器中修改，然后，docker commit成新的镜像就可以了
