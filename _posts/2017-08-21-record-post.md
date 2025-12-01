---
layout: post
title: 方便追踪
tags: [code, java]
author-id: zqmalyssa
---

记录一些方便以后追踪

### git

1、git的常用命令是什么

2、git和svn的最大区别是什么

### maven

1、怎么看一个包是怎么样被引入的

2、依赖冲突的时候如何做选择？

3、dependencyManagement（传递依赖）和dependency的区别

4、maven依赖冲突的调节，同一个pom，相同的路径

### spring相关

1、springboot和spring关系

2、spirngboot的自动装配原理

3、springboot的启动流程

4、spring的aop及其实践（静态代理，动态代理）advice有哪几种

5、spring的IOC（实现思路，工厂+反射

6、spring的核心组件有哪些（web，jdbc，aop，beans，context，core）

7、spring中都用了哪些设计模式（工厂模式，单例模式，代理模式）

8、什么是bean装配

9、什么是@Autowired注解自动装配的过程

10、@Autowired和@Resource之间的区别，@Qualifier 注解有什么作用

### mysql

1、mysql的4大特性

2、mysql的事务隔离级别是什么 4大事务隔离级别

3、innodb和myisam

4、mysql的优化方式，最左匹配原则，加索引有什么原则，都有哪些索引，复合索引的特点是什么，为什么主键用自增比较好（B+树）

5、慢日志相关

6、优化思路

7、explain执行计划能看见什么

8、都有什么字段

### java基础

1、java

2、jvm相关，8,11,17默认垃圾收集器（发行版本） 8可以设置 11zgc 17zgc优化，jvm参数都有哪些，分别代表什么

3、juc中的内容，collections，tools，locks，atomic，executor（哪些线程池的创建，任务队列有哪些，丢弃策略有哪些）

线程数如何设置的

cas，乐观锁，悲观锁

4、Java agent相关

### linux


### chaos

1、关于chaosblade-exec-jvm中的结构，httpclient延迟的原理，db延迟的原理，异步client延迟的原理，pointcut是怎么切的（class和method）

2、卸载后metaspace空间释放问题

3、chaosblade的结构分享

4、如何不去重启实例，

5、tc是怎么做源目的故障注入的

6、chaosblade是怎么使用

7、agent的动态挂载？

### sre

```java



```

### AI

去做线的预测话，其实分类为时序方式，可以用Markov方式 和 潜变量。用Markov 可以结合 MLP（神经网络进行训练和预测）。在d2l的RNN开头的部分


### punchline

1、AI相关的预测数据，RNN

2、猴子的4层代理，搭建socks（1080），通过conn的socket去检测（代码里要设置proxyIp，proxyPort），因为ping检测是过不去的，其实当时没理解，真的vpn是能代理网络层的（可以ping）

3、dns的53端口的问题也是，DNS占用53号端口，同时使用TCP和UDP协议。那么DNS在什么情况下使用这两种协议？DNS在区域传输的时候使用TCP协议，其他时候使用UDP协议。UDP场景：客户端向DNS服务器进行普通查询（例如，查询A记录、CNAME记录等）。TCP场景：DNS服务器之间的区域传输（Zone Transfer），例如主DNS服务器向辅助DNS服务器同步数据时。（这个地方查下猴子做域名的那部分，应该是黑名单，通过写hosts，而白名单，是禁53，同时放行域名！！大致需要回忆一下）

4、连接池开的链接数，对于 域名 和 ip是不同的，ip是ip的配置，域名开的是域名，（想想dns分流中的rule规则 no-resolve）

5、blackhole，扔到黑洞里面，黑洞是在哪里使用来着的

6、http1/http2/http3 的区别 http3是用基于udp的quic协议的，如果有问题，回退到http2协议（禁用浏览器的quic功能）

7、域名申请完成，同时了解下ipv6

8、部署nginx，随便弄点东西，然后支持https，顺带解锁了nginx配置多站点，同一个80口，监听着不同的服务器域名。。。  server_name 可以填域名也可以填ip

https://help.aliyun.com/zh/ecs/user-guide/build-multiple-websites-on-a-linux-instance

那么nginx也能玩起来了

server {
    # 将原有 listen 80 修改为 listen 80 改为 listen 443 ssl
    listen 443 ssl;
    # 原有 server_name，可继续新增更多当前证书支持的域名
    server_name mingye888.dpdns.org;

    # ======================= 证书配置开始 =======================
    # 指定证书文件（中间证书可以拼接至该pem文件中），请将 /etc/nginx/cert/ssl.pem 替换为您实际使用的证书文件的绝对路径
    ssl_certificate /etc/nginx/ssl/fullchain.cer;
    # 指定私钥文档，请将 /etc/nginx/cert/ssl.key 替换为您实际使用的私钥文件的绝对路径
    ssl_certificate_key /etc/nginx/ssl/ssl.key;
    # 配置 SSL 会话缓存，提高性能
    ssl_session_cache shared:SSL:1m;
    # 设置 SSL 会话超时时间
    ssl_session_timeout 5m;
    # 自定义设置使用的TLS协议的类型以及加密套件（以下为配置示例，请您自行评估是否需要配置）
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    # 指定允许的 TLS 协议版本，TLS协议版本越高，HTTPS通信的安全性越高，但是相较于低版本TLS协议，高版本TLS协议对浏览器的兼容性较差
    ssl_protocols TLSv1.2 TLSv1.3;
    # 优先使用服务端指定的加密套件
    ssl_prefer_server_ciphers on;
    # ======================= 证书配置结束 =======================

    # 其它配置
    #charset koi8-r;
    access_log  /var/log/nginx/b.access.log  main;

    location / {
        root   /usr/share/nginx/html/testpage_1;    #测试站点路径。即您的项目代码路径。
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}


===简洁的===

server {
    # 将原有 listen 80 修改为 listen 80 改为 listen 443 ssl
    listen 443 ssl;
    # 原有 server_name，可继续新增更多当前证书支持的域名
    server_name mingye888.dpdns.org;

    ssl_certificate /etc/nginx/ssl/fullchain.cer;
    ssl_certificate_key /etc/nginx/ssl/ssl.key;

    #charset koi8-r;
    access_log  /var/log/nginx/b.access.log  main;

    location / {
        root   /usr/share/nginx/html/testpage_1;    #测试站点路径。即您的项目代码路径。
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}

应该确实不是配置的问题，是443被占用的问题

9、
