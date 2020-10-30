---
layout: post
title: "windows操作系统使用技巧"
author-id: "zqmalyssa"
tags: [windows]
---

windows最常用的操作系统了，有些技巧需要整理下，涵盖各个版本的

#### 一、 win10保存聚焦图片

win10在个性化配置中如果使用了`聚焦`图片，会随机生产屏保，都很赞，如何保存呢？可以进入目录

```html
C:\Users\Fan\AppData\Local\Packages\Microsoft.Windows.ContentDeliveryManager_cw5n1h2txyewy\LocalState\Assets
```

拷贝出所有文件，如何写个changeToJPG.bat脚本，内容简单如下

```html
ren * *.jpg
```
放到新目录下跑下，就能打开图片了，找到横屏的那张，保存吧


#### 二、 netstat的使用

不像linux中有grep，可以这样使用

```html
netstat -ano | findstr [port]
```
就能找到对应端口启动的进程Pid


#### 三、 如何查看机器信息

CMD进入命令行，在命令行中输入

```html
dxdiag
```
这就打开了Directx诊断工具，可以看到CPU核数

![cpuinfo]({{ "/assets/img/windows/cpuinfo.png" | relative_url}})

还有种方法，在CMD中，先输入

```html
wmic
```
电脑安装wmic并进入新的CMD，输入

```html
cpu get *
```
可以看到CPU信息

![wmic]({{ "/assets/img/windows/wmic.png" | relative_url}})

看到Name属性后面的NumberOfCores，NumberOfEnabledCore，NumberOfLogicalProcessors，还有ThreadCount这些跟核心有关的数据

#### 四、error launching installer

恢复系统后有些安装包报这样的错，系统设置的问题，主要是界面语言显示和非Unicode设置引起的，在控制面板，区域 界面的[管理]标签里，在非Unicode区域，更改系统区域设置里 选择[中文(简体，中国)]，而不是美国，同时本身界面的设置就是中文(简体)

#### 五、Win10自动修复无限重启

估计是某个程序下的驱动或者嘛的引起的，差点要摔电脑，还好恢复能抢救回来，但是要装一堆东西，先尝试关闭自动修复

```html
win标徽+X，打开cmd管理员模式--Windows Powershell(管理员)
//关闭自动修复
bcdedit /set recoveryenabled NO
//开启自动修复
cdedit /set recoveryenabled YES

去服务里面找Windows Update，将其禁用再
```

#### 六、电脑视频没有声音，其他有

电脑放电影什么的都ok，一视频就没有声音，用QQ调。。。尼玛，打开qq上的设置，视频那里，呼两口，测试出有声音就ok了，特么好像是音响选的不对，qq调完后又多出来一个设备，不懂
