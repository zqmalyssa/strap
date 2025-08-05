---
layout: post
title: maven细节
tags: [maven]
author-id: zqmalyssa
---

说下maven的细节相关

#### 看maven依赖关系

一般是想某个依赖包是怎样被引入的

mvn --settings D:\\code_config\\settings.xml dependency:tree -Dverbose -Dincludes=com.fasterxml.jackson.core:jackson-annotations

注意这里要指定--settings的文件（如果是公司有相关配置），还是报下面这个错

Could not resolve dependencies for project com.xxx.xxx.app:xxx-data:jar:1.2.5: Failure to find com.xxx.xxx.app:xxx-common:jar:1.2.5 in http://maven.release.ctripcorp.com/nexus/content/groups/public was cached in the local repository, resolution will not be reattempted until the update interval of nexus has elapsed or updates are forced

然后setting文件里面添加了，报错变成下面
<releases>
        <enabled>true</enabled>
        <updatePolicy>always</updatePolicy>
</releases>

Could not resolve dependencies for project com.xxx.xxx.app:xxx-data:jar:1.2.5: Could not find artifact com.xxx.xxx.app:xxx-common:jar:1.2.5 in nexus (http://maven.release.ctripcorp.com/nexus/content/groups/public)

会一直从远端拉，突然发现其实nexus远端也没有这些基础包啊（不会deploy过去的），于是mvn install（加上--settings D:\\code_config\\settings.xml）一下到本地，去掉<updatePolicy>always</updatePolicy>，没有指定maven_repo的地址，就可以了

#### maven的依赖检测与升级等等

1、移除maven

<exclusions>
    <exclusion>
        <artifactId>fastjson</artifactId>
        <groupId>com.alibaba</groupId>
    </exclusion>
</exclusions>

java.lang.NoClassDefFoundError: com/alibaba/fastjson/TypeReference

如果移除某个依赖，项目不一定能启动起来的，要解决版本还是要升级

2、通过查看maven依赖关系，得到的结果总结是

最后写着compile的就是编译成功的。
最后写着omitted for duplicate的就是有jar包被重复依赖了，但是jar包的版本是一样的。
最后写着omitted for conflict with xxxx的，说明和别的jar包版本冲突了，而该行的jar包不会被引入。比如上面有一行最后写着omitted for conflict with 3.4.6，那么该行的zookeeper:jar:3.4.8不会被引入，会引入3.4.6版本
最后写着version managed from 2.3 ;omitted for duplicate ,表示最终使用commons-pool2最终会使用2.4.2，拒绝使用<dependencyManagement></dependencyManagement>中声明的2.3版本‘’
最后写着version managed from 1.16.8 ;表示最终使用lombok:jar:1.16.22版本

最后两个都是用前面的版本，区别是第一个是明确指定使用2.4.2，可以看log4j升级的例子

[INFO] com.xxx.xxx.app:alerts-xxx-cache:jar:1.0.0
[INFO] \- com.xxx.xxx.app:common:jar:1.5.170:compile
[INFO]    \- org.springframework.boot:spring-boot-starter:jar:2.2.5.RELEASE:compile
[INFO]       \- org.springframework.boot:spring-boot-starter-logging:jar:2.2.5.RELEASE:compile
[INFO]          \- org.apache.logging.log4j:log4j-to-slf4j:jar:2.12.1:compile
[INFO]             \- org.apache.logging.log4j:log4j-api:jar:2.15.0:compile (version managed from 2.12.1)

这个log4j-to-slf4j:jar:2.12.1:compile用到了log4j-api，但是最终使用了2.15.0版本，而不是依赖的2.12.1版本，同时

这边的原则，有高版本尽量保留高版本，因为原则还是向下兼容的

3、再说下scope这个参数

compile : (编译依赖范围) , 在编译 , 测试 , 运行/打包时都会使用这个依赖

test : (测试依赖范围) , 测试时会使用 , 编译 和 运行/打包 不使用 , 如 Junit

runtime : (运行时依赖范围) , 测试 和 运行/打包 时需要 , 编译不需要 , 如 JDBC 驱动包

provided : (已提供依赖范围) , 编译 和 测试时需要 , 运行/打包 时不需要 , 如 servlet-api （环境切换，运行的时候没添加tomcat依赖，起补鸟）

system : (系统依赖范围) , 本地依赖 , 不在 maven 中央仓库 , 从参与度来说也 provided 相同 , 不过被依赖项不会从 maven 仓库抓 , 而是从本地文件系统拿 , 一定需要配合 systemPath 属性使用

还有一个import的scope：

import这个scope仅仅在<dependencyManagement>内部定义的pom<dependency>支持，作用就是这个定义会被替换成一系列的<dependency>，由于它们是被替换的，所以这些被import的scope所导入的dependency不会受限依赖继承策略

就是说这些dependency可以等价于直接在当前pom文件中定义的，而不是从父项目继承来的。就是复制黏贴

另外，一个小tip，如果在B中要对A中定义的dependency覆盖，应该注意顺序，将对A的依赖卸载下面。（？？？？不是应该在后买你？？？？）



4、在升级时候的一些覆盖问题

首先父pom中的dependencyManagement，官方定义：pom文件中没有指定版本的依赖或是传递的依赖，如果在dependencyManagement中有指定此依赖版本，那就使用dependencyManagement中定义的版本号

一个是jackson的例子（升级版本），还有比如你子项目中用

<artifactId>spring-context</artifactId>
<groupId>org.springframework</groupId>
<version>5.0.1.RELEASE</version>

此时传递依赖的包有 context下有 aop和expression两个子包，版本也都是5.0.1.RELEASE

这时候parent的dependencyManagement加入个

<dependencyManagement>
<dependencise>
<dependency>
<artifactId>spring-boot-dependencies</artifactId>
<groupId>org.springframework.boot</groupId>
<version>1.5.13.RELEASE</version>
<type>pom</type>
<scope>import</scope>
</dependency>
</dependencise>
</dependencyManagement>

这个版本的springboot，那么context还是5.0.1.RELEASE，但是传递依赖进来的aop和expression版本就变成了4.3.17.RELEASE

解决这种例子，直接在子pom中再覆盖（因为父的dependencyManagement不让动）

-- 冲突的调节

依赖冲突的调节

A -> B -> C -> X (1.0)
A -> D -> X (2.0)
由于只能引入一个版本的包 , 此时 Maven 按照最短路径选择导入 X (2.0)

A -> B -> X (1.0)
A -> D -> X (2.0)
路径长度一致 , 且不在同一个 POM 文件，则优先选择第一个先声明的依赖 , 此时导入 X (1.0)

A -> X (1.0)
A -> X (2.0)
路径长度一致 , 且在同一个 POM 文件，则后面声明的依赖会覆盖前面的依赖 , 此时导入 X (2.0)

注意路径长度的两种不同，一个是在同一级pom中，一个是传递依赖

补充：同一个pom中，后面的会覆盖前面的，本级的dependencyManagement大于父类的dependencyManagement


5、有的时候父pom去package成功，但是子的package不成功

先说多个子项目的构建顺序

```html

<modules>
    <module>A</module>
    <module>B</module>
    <module>C</module>
    <module>D</module>
</modules>

假设各个子模块间，配置的相互依赖关系如下：


A 依赖 B
B 依赖 C
D 依赖 A


[INFO] ------------------------------------------------------------------------
[INFO] Reactor Build Order:
[INFO]
[INFO] C
[INFO] B
[INFO] A
[INFO] D
[INFO]
[INFO] ------------------------------------------------------------------------


```

这是因为子模块的构建顺序受两个因素影响

1、父模块中各子模块的声明次序
2、子模块间的依赖关系

maven按照次序读取pom，如果该pom没有依赖其他子模块，就构建该模块，否则就构建其依赖的模块，如果该依赖模块还依赖于其他的模块，那么就进一步构建依赖的依赖。

在示例中，A模块依赖B，而B模块又依赖C，因此要先构建C，再构建B，然后才能构建A。而D依赖的模块A已经构建了，因此直接构建它。

模块间的依赖关系会将反应堆(Reactor)构成一个有向非循环图，各个模块是该图的节点，依赖关系构成了有向边。这个图不允许出现循环。如果A依赖B，B又依赖A，这样就产生了循环依赖，Maven会报错。

maven的操作

清理（clean）：删除以前的编译结果，为重新编译做好准备
编译（compile）：将Java 源程序编译为字节码文件
测试（test）：针对项目中的关键点进行测试，确保项目在迭代开发过程中关键点的正确性
报告（）：在每一次测试后以标准的格式记录和展示测试结果
打包（package）：将一个包含诸多文件的工程封装为一个压缩文件用于安装或部署。Java 工程对应 jar 包，Web工程对应 war 包。
安装（install）：在 Maven 环境下特指将打包的结果——jar 包或 war 包安装到本地仓库中。
部署（deploy）：将打包的结果部署到远程仓库或将 war 包部署到服务器上运行。


所以，注意当只是package父的时候，没有install，打包子的时候是没有获取正确的依赖关系的（子的会有包冲突问题等等），一般有子项目的需要本地install一下，这个时候再去package别的子项目就行了，处理这种问题的时候留个神，再举下面这个例子


比如简单的，有一个pp工程，pp下包含两个Maven Module：pp-Api， pp-Service。

```html

父工程

<groupId>com.xxx.center.pp</groupId>
<artifactId>xxx-center-pp</artifactId>
<version>0.0.1-SNAPSHOT</version>
<packaging>pom</packaging>

<name>xxx-center-pp</name>

<modules>
<module>xxx-center-pp-api</module>
<module>xxx-center-pp-service</module>
</modules>

pp-apigoon工程

<parent>
  <groupId>com.xxx.center.pp</groupId>
  <artifactId>xxx-center-pp</artifactId>
  <version>0.0.1-SNAPSHOT</version>
</parent>


<artifactId>xxx-center-pp-api</artifactId>
<name>xxx-center-pp-api</name>

pp-service工程

<parent>
  <groupId>com.xxx.center.pp</groupId>
  <artifactId>xxx-center-pp</artifactId>
  <version>0.0.1-SNAPSHOT</version>
</parent>


<artifactId>xxx-center-pp-service</artifactId>
<packaging>jar</packaging>
<name>xxx-center-pp-service</name>

<dependencies>

<dependency>
  <groupId>com.xxx.center.pp</groupId>
  <artifactId>xxx-center-pp-api</artifactId>
  <version>0.0.1-SNAPSHOT</version>
</dependency>

</dependencies>

```

由此可见，pp-service 依赖了pp-api工程，因此pp-service打包之前，必须先打包pp-api。但仅此也不行（没有install依赖关系），pp主工程也需要打包一下。所以打包的顺序是：

1. 对 pp 工程执行 mvn install

2. 对pp-api 工程执行 mvn install（需要吗，不一定）

3. 对pp-service工程执行 mvn install


#### maven的一些例子

1、springboot和spring，两者的版本号相辅相成，需要有一些匹配，在升级spring cve时候体现

```html

<spring-boot.version>2.5.12</spring-boot.version>
<spring.version>5.3.18</spring.version>

```

2、三个项目，引用差不多，为什么就其中一个springboot启动的时候报错

Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/commons/logging/LogFactory

一开始以为引个commons-log就行，但其他两个项目也没引啊，后来发现spring包装了commons-log，叫做spring-jcl

然后强制加下就可以启动项目了，但是另两个项目也强制引入啊，找下maven的依赖，发现另两个项目也没有比较明显的spring-jcl

结合package时候的报错

```html

Found in:
    org.springframework:spring-jcl:jar:5.3.18:compile
    org.slf4j:jcl-over-slf4j:jar:1.7.30:compile
  Duplicate classes:
    org/apache/commons/logging/Log.class
    org/apache/commons/logging/impl/NoOpLog.class
    org/apache/commons/logging/impl/SimpleLog.class
    org/apache/commons/logging/LogFactory.class

```

发现原来另两个项目不一定用的是spring-jcl，而是jcl-over-slf4j，然后maven去搜索下这个包的依赖链，发现是的，在有问题的项目中被exclude掉了

所以启动的时候报错

3、想提高速度，其实不用全部去package

```html

-pl  --projects   Build specified reactor projects instead of all projects   选项后可跟随{groupId}:{artifactId}或者所选模块的相对路径(多个模块以逗号分隔)

-am  --also-make  If project list is specified, also build projects required by the list    表示同时处理选定模块所依赖的模块

-amd  --also-make-dependents  If project list is specified, also build projects that depend on projects on the list   表示同时处理依赖选定模块的模块

-N   --Non-recursive  Build projects without recursive  表示不递归子模块

-rf   --resume-from  Resume reactor from specified project  表示从指定模块开始继续处理

clean package -pl xxx-restapi -am -Pxxx -DskipTests

parent
  - common
  - web

1. 在dailylog-parent目录运行`mvn clean install -pl org.lxp:dailylog-web -am`，结果

dailylog-common成功安装到本地库
dailylog-parent成功安装到本地库
dailylog-web成功安装到本地库
该命令等价于`mvn clean install -pl ../dailylog-web -am`

2. 在dailylog-parent目录运行`mvn clean install -pl ../dailylog-common -am`，结果

dailylog-common成功安装到本地库
dailylog-parent成功安装到本地库
3. 在dailylog-parent目录运行`mvn clean install -pl ../dailylog-common -amd`，结果

dailylog-common成功安装到本地库
dailylog-web成功安装到本地库
由于dailylog-parent并不依赖dailylog-common模块，故没有被安装

4. 在dailylog-parent目录运行`mvn clean install -pl ../dailylog-common,../dailylog-parent -amd`，结果

dailylog-common成功安装到本地库
dailylog-parent成功安装到本地库
dailylog-web成功安装到本地库
5. 在dailylog-parent目录运行`mvn clean install -N`，结果

dailylog-parent成功安装到本地库
-N表示不递归，那么dailylog-parent管理的子模块不会被同时安装

6. 在dailylog-parent目录运行`mvn clean install -pl ../dailylog-parent -N`，结果

dailylog-parent成功安装到本地库
7. 在dailylog-parent目录运行`mvn clean install -rf ../dailylog-common`，结果

dailylog-common成功安装到本地库
dailylog-web成功安装到本地库

```
