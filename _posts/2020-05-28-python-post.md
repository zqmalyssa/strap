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

//pip如果配置到环境变量，可以直接使用（在python的scripts目录下面）
pip install D:\wheel\MySQL_python-1.2.5-cp27-none-win_amd64.whl

pip uninstall D:\wheel\MySQL_python-1.2.5-cp27-none-win_amd64.whl

// 这边补充一下，mysqlclient-1.4.6-cp27-cp27m-win_amd64.whl is not a supported wheel on this platform. 这种报错，是pip不能兼容版本的问题
// 进入python，import pip; print(pip.pep425tags.get_supported())，可以得到pip支持的版本，像上面这个，如果是cp27-none的话就能支持了

>>> import pip;
>>> print(pip.pep425tags.get_supported())
[('cp27', 'none', 'win_amd64'), ('py2', 'none', 'win_amd64'), ('cp27', 'none', 'any'), ('cp2', 'none', 'any'), ('cp26', 'none', 'any'), ('cp25', 'none', 'any'), ('cp24', 'none', 'any'), ('cp23', 'none', 'any'), ('cp22', 'none', 'any'), ('cp21', 'none', 'any'), ('cp20', 'none', 'any'), ('py27', 'none', 'any'), ('py2', 'none', 'any'), ('py26', 'none', 'any'), ('py25', 'none', 'any'), ('py24', 'none', 'any'), ('py23', 'none', 'any'), ('py22', 'none', 'any'), ('py21', 'none', 'any'), ('py20', 'none', 'any')]

// 把pip升级，是不是支持的版本更多了
pip install --upgrade pip  // 一直报错，升级到指定版本，成功了，python环境太老了
pip install --upgrade pip==20.1.1

// 再import后输出已经没有内容了

pip list查看安装的包

pip install D:\wheel\mysqlclient-1.4.6-cp27-cp27m-win_amd64.whl  // 升级完后可以通过wheel方式安装了

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

在linux系统中安装python3

```HTML

https://blog.csdn.net/KevinsCSDN/article/details/116565471

https://blog.csdn.net/HD243608836/article/details/121417965

```


#### Python中的if和循环

if x:
    print('True')

只要x是非零数值、非空字符串、非空list等，就判断为True，否则为False


在Python中，迭代是通过for ... in来完成的，而很多语言比如C语言，迭代list是通过下标完成的，比如C代码：

可以看出，Python的for循环抽象程度要高于C的for循环，因为Python的for循环不仅可以用在list或tuple上，还可以作用在其他可迭代对象上。

list这种数据类型虽然有下标，但很多其他数据类型是没有下标的，但是，只要是可迭代对象，无论有无下标，都可以迭代，比如dict就可以迭代：

```html

>>> d = {'a': 1, 'b': 2, 'c': 3}
>>> for key in d:
...     print(key)
...
a
c
b

因为dict的存储不是按照list的方式顺序排列，所以，迭代出的结果顺序很可能不一样。

默认情况下，dict迭代的是key。如果要迭代value，可以用for value in d.values()，如果要同时迭代key和value，可以用


for k, v in d.items()。

由于字符串也是可迭代对象，因此，也可以作用于for循环：


>>> for ch in 'ABC':
...     print(ch)
...
A
B
C

所以，当我们使用for循环时，只要作用于一个可迭代对象，for循环就可以正常运行，而我们不太关心该对象究竟是list还是其他数据类型。

那么，如何判断一个对象是可迭代对象呢？方法是通过collections.abc模块的Iterable类型判断：


>>> from collections.abc import Iterable
>>> isinstance('abc', Iterable) # str是否可迭代
True
>>> isinstance([1,2,3], Iterable) # list是否可迭代
True
>>> isinstance(123, Iterable) # 整数是否可迭代
False


最后一个小问题，如果要对list实现类似Java那样的下标循环怎么办？Python内置的enumerate函数可以把一个list变成索引-元素对，这样就可以在for循环中同时迭代索引和元素本身：

>>> for i, value in enumerate(['A', 'B', 'C']):
...     print(i, value)
...
0 A
1 B
2 C


```


#### Python基本使用

python的语法要看看，然后python中的数据结构是

列表
LIST: 可变数组
[]

```html

list是一个可变的有序表，所以，可以往list中追加元素到末尾：

>>> classmates.append('Adam')
>>> classmates
['Michael', 'Bob', 'Tracy', 'Adam']

也可以把元素插入到指定的位置，比如索引号为1的位置：

>>> classmates.insert(1, 'Jack')
>>> classmates
['Michael', 'Jack', 'Bob', 'Tracy', 'Adam']

要删除list末尾的元素，用pop()方法：

>>> classmates.pop()
'Adam'
>>> classmates
['Michael', 'Jack', 'Bob', 'Tracy']

要删除指定位置的元素，用pop(i)方法，其中i是索引位置：

>>> classmates.pop(1)
'Jack'
>>> classmates
['Michael', 'Bob', 'Tracy']

要把某个元素替换成别的元素，可以直接赋值给对应的索引位置：

>>> classmates[1] = 'Sarah'
>>> classmates
['Michael', 'Sarah', 'Tracy']

list里面的元素的数据类型也可以不同，比如：

>>> L = ['Apple', 123, True]

list元素也可以是另一个list，比如：

>>> s = ['python', 'java', ['asp', 'php'], 'scheme']
>>> len(s)
4


```

切片，可以取列表中的一部分
SLICE
[:]

```html

对应上面的问题，取前3个元素，用一行代码就可以完成切片：

>>> L[0:3]
['Michael', 'Sarah', 'Tracy']

L[0:3]表示，从索引0开始取，直到索引3为止，但不包括索引3。即索引0，1，2，正好是3个元素。

如果第一个索引是0，还可以省略：

>>> L[:3]
['Michael', 'Sarah', 'Tracy']


tuple也是一种list，唯一区别是tuple不可变。因此，tuple也可以用切片操作，只是操作的结果仍是tuple：

>>> (0, 1, 2, 3, 4, 5)[:3]
(0, 1, 2)

字符串'xxx'也可以看成是一种list，每个元素就是一个字符。因此，字符串也可以用切片操作，只是操作结果仍是字符串：

>>> 'ABCDEFG'[:3]
'ABC'
>>> 'ABCDEFG'[::2]
'ACEG'



```



元组，元组中的值是不能修改的，test[0] = '123', 这样是不行的，但可以这样赋值，test = (100, 123)
TUPLE
()

```html

另一种有序列表叫元组：tuple。tuple和list非常类似，但是tuple一旦初始化就不能修改，比如同样是列出同学的名字：

>>> classmates = ('Michael', 'Bob', 'Tracy')

现在，classmates这个tuple不能变了，它也没有append()，insert()这样的方法。其他获取元素的方法和list是一样的，你可以正常地使用classmates[0]，classmates[-1]，但不能赋值成另外的元素。

不可变的tuple有什么意义？因为tuple不可变，所以代码更安全。如果可能，能用tuple代替list就尽量用tuple。

tuple的陷阱：当你定义一个tuple时，在定义的时候，tuple的元素就必须被确定下来，比如：

>>> t = (1, 2)
>>> t
(1, 2)

如果要定义一个空的tuple，可以写成()：

>>> t = ()
>>> t
()

但是，要定义一个只有1个元素的tuple，如果你这么定义

>>> t = (1)
>>> t
1

定义的不是tuple，是1这个数！这是因为括号()既可以表示tuple，又可以表示数学公式中的小括号，这就产生了歧义，因此，Python规定，这种情况下，按小括号进行计算，计算结果自然是1。

所以，只有1个元素的tuple定义时必须加一个逗号,，来消除歧义：

>>> t = (1,)
>>> t
(1,)


Python在显示只有1个元素的tuple时，也会加一个逗号,，以免你误解成数学计算意义上的括号。

最后来看一个“可变的”tuple：

>>> t = ('a', 'b', ['A', 'B'])
>>> t[2][0] = 'X'
>>> t[2][1] = 'Y'
>>> t
('a', 'b', ['X', 'Y'])


表面上看，tuple的元素确实变了，但其实变的不是tuple的元素，而是list的元素。tuple一开始指向的list并没有改成别的list，所以，tuple所谓的“不变”是说，tuple的每个元素，指向永远不变。即指向'a'，就不能改成指向'b'，指向一个list，就不能改成指向其他对象，但指向的这个list本身是可变的！

```


字典
DICT: Key和Value的形式
{}

```html

要避免key不存在的错误，有两种办法，一是通过in判断key是否存在：

>>> 'Thomas' in d
False

通过dict提供的get()方法，如果key不存在，可以返回None，或者自己指定的value：

>>> d.get('Thomas')
>>> d.get('Thomas', -1)
-1

注意：返回None的时候Python的交互环境不显示结果。

要删除一个key，用pop(key)方法，对应的value也会从dict中删除

>>> d.pop('Bob')
75
>>> d
{'Michael': 95, 'Tracy': 85}

dict可以用在需要高速查找的很多地方，在Python代码中几乎无处不在，正确使用dict非常重要，需要牢记的第一条就是dict的key必须是不可变对象。

```

SET

```html

set和dict类似，也是一组key的集合，但不存储value。由于key不能重复，所以，在set中，没有重复的key。

要创建一个set，需要提供一个list作为输入集合：

>>> s = set([1, 2, 3])
>>> s
{1, 2, 3}

注意，传入的参数[1, 2, 3]是一个list，而显示的{1, 2, 3}只是告诉你这个set内部有1，2，3这3个元素，显示的顺序也不表示set是有序的

重复元素在set中自动被过滤：

>>> s = set([1, 1, 2, 2, 3, 3])
>>> s
{1, 2, 3}

通过add(key)方法可以添加元素到set中，可以重复添加，但不会有效果：

>>> s.add(4)
>>> s
{1, 2, 3, 4}
>>> s.add(4)
>>> s
{1, 2, 3, 4}

通过remove(key)方法可以删除元素：

>>> s.remove(4)
>>> s
{1, 2, 3}

set可以看成数学意义上的无序和无重复元素的集合，因此，两个set可以做数学意义上的交集、并集等操作：

>>> s1 = set([1, 2, 3])
>>> s2 = set([2, 3, 4])
>>> s1 & s2
{2, 3}
>>> s1 | s2
{1, 2, 3, 4}

set和dict的唯一区别仅在于没有存储对应的value，但是，set的原理和dict一样，所以，同样不可以放入可变对象，因为无法判断两个可变对象是否相等，也就无法保证set内部“不会有重复元素”。试试把list放入set，看看是否会报错。

```

#### ptyhon函数

定义默认参数要牢记一点：默认参数必须指向不变对象！ 不能是List这种东西

```html

def add_end(L=[]):
    L.append('END')
    return L

>>> add_end()
['END']

>>> add_end()
['END', 'END']
>>> add_end()
['END', 'END', 'END']

所以上面的情况不行，Python函数在定义的时候，默认参数L的值就被计算出来了，即[]，因为默认参数L也是一个变量，它指向对象[]，每次调用该函数，如果改变了L的内容，则下次调用时，默认参数的内容就变了，不再是函数定义时的[]了。

要修改上面的例子，我们可以用None这个不变对象来实现：

def add_end(L=None):
    if L is None:
        L = []
    L.append('END')
    return L

现在，无论调用多少次，都不会有问题：


>>> add_end()
['END']
>>> add_end()
['END']



```

可变参数

```html

如果利用可变参数，调用函数的方式可以简化成这样：

>>> calc(1, 2, 3)
14
>>> calc(1, 3, 5, 7)
84

所以，我们把函数的参数改为可变参数：


def calc(*numbers):
    sum = 0
    for n in numbers:
        sum = sum + n * n
    return sum

如果已经有一个list或者tuple，要调用一个可变参数怎么办？可以这样做：


>>> nums = [1, 2, 3]
>>> calc(nums[0], nums[1], nums[2])
14

这种写法当然是可行的，问题是太繁琐，所以Python允许你在list或tuple前面加一个*号，把list或tuple的元素变成可变参数传进去：


>>> nums = [1, 2, 3]
>>> calc(*nums)
14

*nums表示把nums这个list的所有元素作为可变参数传进去。这种写法相当有用，而且很常见。

```

关键字参数

```html


可变参数允许你传入0个或任意个参数，这些可变参数在函数调用时自动组装为一个tuple。而关键字参数允许你传入0个或任意个含参数名的参数，这些关键字参数在函数内部自动组装为一个dict。请看示例：

def person(name, age, **kw):
    print('name:', name, 'age:', age, 'other:', kw)

函数person除了必选参数name和age外，还接受关键字参数kw。在调用该函数时，可以只传入必选参数：

>>> person('Michael', 30)
name: Michael age: 30 other: {}

也可以传入任意个数的关键字参数：

>>> person('Bob', 35, city='Beijing')
name: Bob age: 35 other: {'city': 'Beijing'}
>>> person('Adam', 45, gender='M', job='Engineer')
name: Adam age: 45 other: {'gender': 'M', 'job': 'Engineer'}

和可变参数类似，也可以先组装出一个dict，然后，把该dict转换为关键字参数传进去：

>>> extra = {'city': 'Beijing', 'job': 'Engineer'}
>>> person('Jack', 24, city=extra['city'], job=extra['job'])
name: Jack age: 24 other: {'city': 'Beijing', 'job': 'Engineer'}

当然，上面复杂的调用可以用简化的写法：

>>> extra = {'city': 'Beijing', 'job': 'Engineer'}
>>> person('Jack', 24, **extra)
name: Jack age: 24 other: {'city': 'Beijing', 'job': 'Engineer'}

**extra表示把extra这个dict的所有key-value用关键字参数传入到函数的**kw参数，kw将获得一个dict，注意kw获得的dict是extra的一份拷贝，对kw的改动不会影响到函数外的extra。


```

#### Python函数式编程


```html

>>> def f(x):
...     return x * x
...
>>> r = map(f, [1, 2, 3, 4, 5, 6, 7, 8, 9])
>>> list(r)
[1, 4, 9, 16, 25, 36, 49, 64, 81]

map()传入的第一个参数是f，即函数对象本身。由于结果r是一个Iterator，Iterator是惰性序列，因此通过list()函数让它把整个序列都计算出来并返回一个list。

比如，把这个list所有数字转为字符串：

>>> list(map(str, [1, 2, 3, 4, 5, 6, 7, 8, 9]))
['1', '2', '3', '4', '5', '6', '7', '8', '9']

再看reduce的用法。reduce把一个函数作用在一个序列[x1, x2, x3, ...]上，这个函数必须接收两个参数，reduce把结果继续和序列的下一个元素做累积计算，其效果就是：

reduce(f, [x1, x2, x3, x4]) = f(f(f(x1, x2), x3), x4)

比方说对一个序列求和，就可以用reduce实现：

>>> from functools import reduce
>>> def add(x, y):
...     return x + y
...
>>> reduce(add, [1, 3, 5, 7, 9])
25

如果要把序列[1, 3, 5, 7, 9]变换成整数13579，reduce就可以派上用场：

>>> from functools import reduce
>>> def fn(x, y):
...     return x * 10 + y
...
>>> reduce(fn, [1, 3, 5, 7, 9])
13579

这个例子本身没多大用处，但是，如果考虑到字符串str也是一个序列，对上面的例子稍加改动，配合map()，我们就可以写出把str转换为int的函数：


>>> from functools import reduce
>>> def fn(x, y):
...     return x * 10 + y
...
>>> def char2num(s):
...     digits = {'0': 0, '1': 1, '2': 2, '3': 3, '4': 4, '5': 5, '6': 6, '7': 7, '8': 8, '9': 9}
...     return digits[s]
...
>>> reduce(fn, map(char2num, '13579'))
13579

可见用filter()这个高阶函数，关键在于正确实现一个“筛选”函数。

注意到filter()函数返回的是一个Iterator，也就是一个惰性序列，所以要强迫filter()完成计算结果，需要用list()函数获得所有结果并返回list。

要进行反向排序，不必改动key函数，可以传入第三个参数reverse=True：

>>> sorted(['bob', 'about', 'Zoo', 'Credit'], key=str.lower, reverse=True)
['Zoo', 'Credit', 'bob', 'about']

```

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

#### Python的uv

```html

// uv一直起不来

uv run --with fastmcp fastmcp dev /Users/zqmalyssa/Code/image-gen-server/server.py

// 通过cursor进行排障

I see you're using uv to run the FastMCP server. The issue is that the brotli dependency needs to be installed in the uv environment. Let me help you install the dependencies using uv:

➜  image-gen-server git:(main) ✗ uv add brotli
error: No `pyproject.toml` found in current directory or any parent directory
➜  image-gen-server git:(main) ✗

Since there's no pyproject.toml file, let me try installing the dependencies from the requirements.txt file using uv pip:

➜  image-gen-server git:(main) ✗ uv pip install -r requirements.txt
Resolved 44 packages in 3.23s
      Built brotlipy==0.7.0
Prepared 15 packages in 10.36s

// 重点是上面这句话（uv pip install）就好了，所以uv这个环境有点搞，之前没有用python自带的uv，另外装了一个，用这个方法https://www.cnblogs.com/akapril/p/18671748删除

// uv用python自带的

```

#### Python的conda

```html

// 如果按照d2l的book中的安装方法的话

zh-v2.d2l

pip install d2l==0.17.6 // pip version是不是又会引起异常

// 创建激活完指定环境后，到指定的pip环境中去下载包

conda create --name d2l python=3.9 -y
conda activate d2l

// 用下面的方式下载依赖包
/Users/zqmalyssa/miniconda3/envs/d2l/bin/pip --version
/Users/zqmalyssa/miniconda3/envs/d2l/bin/pip install jupyter d2l torch torchvision

// 启动book
jupyter notebook

```
