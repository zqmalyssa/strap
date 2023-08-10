---
layout: post
title: Java开发规范
tags: [java, code]
author-id: zqmalyssa
---

Java开发的一些守则以及一些问题的解决方法

#### 阿里规范

阿里巴巴Java开发手册中的DO、DTO、BO、AO、VO、POJO定义

分层领域模型规约：

DO（ Data Object）：与数据库表结构一一对应，通过DAO层向上传输数据源对象。
DTO（ Data Transfer Object）：数据传输对象，Service或Manager向外传输的对象。
BO（ Business Object）：业务对象。 由Service层输出的封装业务逻辑的对象。
AO（ Application Object）：应用对象。 在Web层与Service层之间抽象的复用对象模型，极为贴近展示层，复用度不高。
VO（ View Object）：显示层对象，通常是Web向模板渲染引擎层传输的对象。
POJO（ Plain Ordinary Java Object）：在本手册中， POJO专指只有setter/getter/toString的简单类，包括DO/DTO/BO/VO等。
Query：数据查询对象，各层接收上层的查询请求。 注意超过2个参数的查询封装，禁止使用Map类来传输。
领域模型命名规约：

数据对象：xxxDO，xxx即为数据表名。
数据传输对象：xxxDTO，xxx为业务领域相关的名称。
展示对象：xxxVO，xxx一般为网页名称。
POJO是DO/DTO/BO/VO的统称，禁止命名成xxxPOJO。

#### 疑难杂症

1、Java 9-11 中的PostConstruct和PreDestroy注释错误怎么解决

注释位于模块java.xml.ws.annotation中。因为它是一个Java EE模块，所以不推荐使用它在Java 9中删除，并且默认情况下不会解析，因此需要手动添加它--add-modules。

在Java 11中，模块将完全消失，并且--add-modules会失败，因为java.xml.ws.annotation不再存在。最好的解决方案是立即用第三方依赖替换它。使用Java Commons Annotations可以在Maven Central上找到它：

```java
<dependency>
    <groupId>javax.annotation</groupId>
    <artifactId>javax.annotation-api</artifactId>
    <version>1.3.1</version>
</dependency>
```

这边也可以直接将项目版本降到1.8版本

2、import com.sun.tools.attach.VirtualMachine 或者 import com.sun.tools.attach.VirtualMachineDescriptor 异常

也是包依赖的问题，直接在idea里的library里面增加tools.jar包，这个jar包在jdk安装目录的lib目录下面

3、java中的科学计数法

小数点后连续超过3个0就变成科学计数法了

sout(0.0001)   1.0E-4

sout(-0.0001)  -1.0E-4

解决的办法是 NumberFormat、DecimalFormat、BigDecimal这三种API实现方式。

### 正则的使用

1、正则中想要匹配一些特殊的字符，需要"\"进行修饰

```html
result\.detail\[[0-9]*\]\.caseList\[[0-9]*\]\.status
```
如上面这种，`.`和`[]`都要记得加上特殊标识

### BigDecimal

前后端四舍五入的问题


如果前端

(100 - x * 100 / 7862399).toFixed(4) // 对减后的值进行四舍五入

和后端

(100 - x * 100 / 7862399)  // 对除法进行四舍五入，然后再减，效果一样

上面这两个效果是一样的，这特么是数学之美么。。。。。所以也无需修改
