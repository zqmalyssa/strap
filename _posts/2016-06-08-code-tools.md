---
layout: post
title: "常用工具总结"
author-id: "zqmalyssa"
tags: [code, tools]
---

干IT，好的工具就像武器，用起来顺手的，效率也会提高

#### 1.Intellij IDEA

Intellij IDEA感觉是革新产品，之前使用过eclipse，spring tools suite，感觉这个更加聪明，灵活，使用起来是有点区别的，但是兼容很好，比如快捷键就设置成之前的eclipse的熟悉的快捷键

| 编号 | 说明 | eclipse | idea |
| :----: | :----:  | :----: | :----: |
| 1 | 向下复制一行 | ctrl + alt + down |  |
| 2 | 查找类相关，区别于查找 | ctrl + shift + t |  |
| 3 | 万能解错 | alt + enter |  |
| 4 | 查看继承关系(type hierarchy) | F4 |  |
| 5 | 查看类的构造 | ctrl + o |  |
| 6 | 大小写转换 | ctrl + shift + y |  |
| 7 | 生成try-catch等 | alt + shift + z |  |
| 8 | 查找(全局) | ctrl + h |  |
| 9 | 查找文件 | double shift |  |
| 10 | 查看方法的多层重写结构 | ctrl + alt + h |  |
| 11 | 抽取方法 | alt + shift + m |  |
| 12 | 关闭打开的所有代码栏 | ctrl + shift + w |  |
| 13 | 查找方法在哪里被调用(文件角度) | ctrl + shift + h |  |
| 14 | 查找方法在哪里被使用(列出详细地方) | ctrl + g |  |
| 15 | 查找具体的实现类 | ctrl + alt + b |  |
| 16 | 显示全部的继承关系（与F4类似） | ctrl + t |  |
| 17 | 按单词删除 | ctrl + backspace |  |
| 18 | 移动代码的结构 | alt + up/down |  |
| 19 |  |  |  |
| 20 |  |  |  |
| 21 |  |  |  |
| 22 |  |  |  |
| 23 |  |  |  |

#### 2.Atom

Atom是一个比较流行的编辑器，而且它有些插件很酷炫

[下载](http://xiazai.zol.com.cn/detail/44/433456.shtml)个Atom先，压缩版的，解压就能用。然后装下activate-power-mode这个酷炫的插件，首先从github上下载下来

```html
git clone https://github.com/JoelBesada/activate-power-mode.git
```

然后将整个文件夹copy到`C:\Users\Fan\.atom\packages`目录，进入文件夹，运行

```html
npm install  // 这步需要执行，不然打开报错
```

如果没有`npm`命令，就要安装一个node，[下载](http://nodejs.cn/download/)之，然后安装并测试

```html
node -v
npm -v
```

之后再去文件夹下安装，成功后会有`add packages`的显示，重新打开Atom，就默认有了，用`ctrl + alt + o`进行开启和关闭

还有的插件如markdown-preview，能够很好的预览markdown格式，atom高版本自带了这个package，可以在packages里面搜索`markdown`，然后根据setting获得快捷键的键值对，这样在keybinding就可以自定义快捷键，比如markdown的预览功能

还有列功能的插件atom-sublime

```html

cd ~/.atom/packages/
git clone https://github.com/bigfive/atom-sublime-select.git
cd 文件夹
npm install // 不知道要不要做，反正成功了


```

#### 3.Intellij IDEA写java的一些技巧

| 编号 | 说明 | 操作 |
| :----: | :----:  | :----: |
| 1 | 快速打出public static void main | psvm + tab |
| 2 | 快速打出System.out.println | sout + tab |
| 3 | 快速打出for循环 | fori + tab |
| 4 | 快速打出foreach循环 | iter + tab |
| 5 | 快速打出list集合循环 | itli + tab |
| 6 | 快速打出iterator迭代器 | itit + tab |
| 7 | 快速打出System.out.println("") | soutp + tab |
| 8 | 快速打出System.out.println("变量名= " + 变量) | soutv + tab |
| 9 | 快速打出System.out.println("当前类名.当前方法") | soutm + tab |
| 10 | 快速打出switch中enum内容 | 打完switch体后用旁边的黄色叹号进行补齐 |

#### 4.Intellij IDEA中隐藏.idea文件夹和.iml文件

idea中的.idea文件夹和.iml是平常几乎不使用的文件，在创建父子工程或者聚合工程时反而会对我们操作产生干扰，所以，一般情况下，我们都将其隐藏掉，步骤如下

File——>settings——>Editor——>File Types——>Ignore files and foloders中输入*.iml和*.idea，以;结尾

打开的所有工程都会刷新一波的

#### 5.Intellij IDEA中一个工程启动多次

将工程中的Single Instance Only去掉，如下

![idea_configuration]({{ "/assets/img/idea/idea_configuration.png" | relative_url}})

然后配置多个不同端口，`server.port`属性不一样就可以了，如果设置为0，就是随机端口

#### 6.GoLang的激活

Golang的永久激活和IJ是类似的，首先要下载`JetbrainsCrack-release-enc.jar`放到Go安装的bin目录，然后修改其中的两个启动文件

1、goland.exe.vmoptions
2、goland64.exe.vmoptions

在文件里添加

```html
-javaagent:E:\intelliJ Goland\GoLand 2018.3.3\bin\JetbrainsCrack-release-enc.jar
```
然后启动Golang，在`Activation code`里输入如下code，如果invalid的话去其他地方再找找

```html
ThisCrackLicenseId-{
　　"licenseId":"ThisCrackLicenseId",
　　"licenseeName":"WeChat",
　　"assigneeName":"IT--Pig",
　　"assigneeEmail":"1113449881@qq.com",
　　"licenseRestriction":"",
　　"checkConcurrentUse":false,
　　"products":[
　　{"code":"II","paidUpTo":"2099-12-31"},
　　{"code":"DM","paidUpTo":"2099-12-31"},
　　{"code":"AC","paidUpTo":"2099-12-31"},
　　{"code":"RS0","paidUpTo":"2099-12-31"},
　　{"code":"WS","paidUpTo":"2099-12-31"},
　　{"code":"DPN","paidUpTo":"2099-12-31"},
　　{"code":"RC","paidUpTo":"2099-12-31"},
　　{"code":"PS","paidUpTo":"2099-12-31"},
　　{"code":"DC","paidUpTo":"2099-12-31"},
　　{"code":"RM","paidUpTo":"2099-12-31"},
　　{"code":"CL","paidUpTo":"2099-12-31"},
　　{"code":"PC","paidUpTo":"2099-12-31"}
　　],
　　"hash":"2911276/0",
　　"gracePeriodDays":7,
　　"autoProlongated":false
}

```

然后在help中的about就有了截止激活日期


这边要提一嘴了，MAC的激活跟win思路是一样，但是jar和activation code都不太一样，还没有找到稳定的，有个到2020年的下载地址，可以生成activation code

```html
http://idea.medeming.com/jihuoma
```

jar包放到google drive里了，比win方便的是只要修改一个文件就行了`goland.vmoptions`

```html
-javaagent:JetbrainsCrack-2.9-release-enc.jar
```

应用程序文件夹中右键应用`显示包内容`


#### 7.IJ的插件

IJ经常要用到一些插件，现在举个代码格式的插件例子，因为要严格遵守代码风格，又要和eclipse保持一致，所以需要安装一个插件，EclipseFormatter，安装完后本地install在IJ的plugin配置中

然后用`ctrl+alt+l`的时候发现还是不行，报错是缺少下载文件中的jar包

```html
Plugin jar file not found: D:\Users\qmzhang\.IntelliJIdea2018.2\config\plugins\EclipseFormatter\lib\adapter.jar
```

所以把下载下来的jar扔到这个路径下ok了吧，每开启一个新的项目后，需要enable这个插件，然后才能生效

#### 8.Google插件

google浏览器有许多附加的插件很实用，比如postman，有个画图的工具，`gliffy diagrams`可以尝试下载用下

#### 9.PyCharm的激活

其实永久激活的步骤和IDEA，GOLANG是一致的，需要注意的是其实用的jar包和安装的版本是有依赖关系的

比如我下的PY的3.7版本，那么我就无法用3.1或者3.4版本的jar包就行激活，会提示`key is invalid`，所以下载低版本的PY，比如20180205，那么key就不报错了

还有如果是key不报错的话，看下help->Edit Custom VM Options是不是有用户自己的配置文件导致启动没有调到

#### 10.idea的变量的使用

```html

# VM Arguments 是设置的虚拟机的属性
# VM options
# 环境变量参数  非虚拟机参数需要指定-D参数
-server -XX:MetaspaceSize=512M -XX:MaxMetaspaceSize=1024m -Dfile.encoding=UTF-8


# Program arguments的值作为args[] 的参数传入的


# Environment variable 环境变量  这里不需要-D 参数
-D 系统属性
-X* jvm参数

# 两个横杠是springboot参数
--server.port=8088


# VM options 优先级 高于  Environment variable



优先级:
Program arguments (--priority=program-agrs) > VM options (-Dpriority=vm-options) > Environment variable (priority=environment-variables)


XXXX_SIDECAR_INJECT，比如直接带这样的参数就行了

```
