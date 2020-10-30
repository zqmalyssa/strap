---
layout: post
title: "Web相关工具"
author-id: "zqmalyssa"
tags: [code, web]
---

介绍一些web常用，实用的工具

### Jmeter

Jmeter比较知名，用jmeter去做压测啊什么的比较方便

直接官方下载后解压，运行.bat文件就可以打开配置，但是真正跑的时候不建议用GUI，用command去做

GUI提示的意思就是：不要使用GUI运行压力测试，GUI仅用于压力测试的创建和调试；执行压力测试请不要使用GUI。使用下面的命令来执行测试：

```html
jmeter -n -t [jmx file] -l [results file] -e -o [Path to web report folder]
```

并且修改JMeter批处理文件的环境变量：HEAP="-Xms1g -Xmx1g -XX:MaxMetaspaceSize=256m"

创建线程组后->配置元件进行一些header的配置->建立http请求->建立结果树->建立报告

```html
jmeter -n -t testplan/RedisLock.jmx -l testplan/result/result.txt -e -o testplan/webreport
```
说明：
testplan/RedisLock.jmx 为测试计划文件路径
testplan/result/result.txt 为测试结果文件路径
testplan/webreport 为web报告保存路径。
