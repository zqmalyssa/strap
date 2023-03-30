---
layout: post
title: 域名相关
tags: [domain]
author-id: zqmalyssa
---

域名相关内容放在这边

#### 域名基本介绍

域名、顶级域名、一级域名、二级域名、子域名 // no


主机名.次级域名.顶级域名.根域名   // 顶级域名也是一级域名，每个域名还有个根域名，就是.root（13个根域名服务器），储存了每个域（如.com .net .cn）的解析和域名服务器的地址信息。可以理解为根域名服务器就是存放顶级域名服务器地址的

# 即

host.sld.tld.root

域名（英语：Domain Name），又称网域，是由一串用点分隔的名字组成的Internet上某一台计算机或计算机组的名称，用于在数据传输时对计算机的定位标识（有时也指地理位置）。由于IP地址具有不方便记忆并且不能显示地址组织的名称和性质等缺点，人们设计出了域名，并通过网域名称系统（DNS，Domain Name System）来将域名和IP地址相互映射，使人更方便地访问互联网，而不用去记住能够被机器直接读取的IP地址数串。

Top-level domains，first-level domains（TLDs），也翻译为国际顶级域名，也成一级域名。
　　.com 供商业机构使用，但无限制最常用
　　.net 原供网络服务供应商使用，现无限制
　　.org 原供不属于其他通用顶级域类别的组织使用，现无限制
　　.edu / .gov / .mil 供美国教育机构/美国政府机关/美国军事机构。因历史遗留问题一般只在美国专用
　　.aero 供航空运输业使用
　　.biz 供商业使用
　　.coop 供联合会（cooperatives）使用
　　.info 供信息性网站使用，但无限制
　　.museum 供博物馆使用
　　.name 供家庭及个人使用
　　.pro 供部分专业使用
　　.asia 供亚洲社区使用
　　.tel 供连接电话网络与因特网的服务使用
　　.post 供邮政服务使用
　　.mail 供邮件网站使用
　　国家顶级域名：cn（中国大陆）、de（德国）、eu（欧盟）、jp（日本）、hk（中国香港）、tw（中国台湾）、uk（英国）、us（美国）


二级域（或称二级域名；英语：Second-level domain；英文缩写：SLD）是互联网DNS等级之中，处于顶级域名之下的域。二级域名是域名的倒数第二个部分，例如在域名example.baidu.com中，二级域名是Baidu。
　　.com 顶级域名/一级域名，更准确的说叫顶级域
　　baidu.com 二级域名，更准确的说叫二级域
　　tieba.baidu.com 三级域名，更准确的说叫三级域
　　detail.tieba.baidu.com 四级域名，更准确的说叫四级域

子域名（或子域；英语：Subdomain）是在域名系统等级中，属于更高一层域的域。比如，mail.example.com和calendar.example.com是example.com的两个子域，而example.com则是顶级域.com的子域。凡顶级域名前加前缀的都是该顶级域名的子域名，而子域名根据技术的多少分为二级子域名，三级子域名以及多级子域名。

通常我们把.com成为一级域名，但严格意义上这样讲不太准确，真正的一级域名是由一个合法的字符串+域名后缀组成，所以，guanghe.com这种形式的域名才是一级域名，guanghe是域名主体，.com是域名后缀，我们也可以把.com也称为顶级域。

　　一级域名又称为顶级域名，比如单独的gunaghe.com如果指向一个ip，这个域名就是一级域名。但需要注意的是，www.guanghe.com这种形式的域名并不是一级域名，它只是一个二级域名，也就是说www只是一个主机名。


一级域名（qmzhang.com）需要备案，而二级域名不需要单独备案，只要它所处的一级域名已经备案，就能直接解析。比如guanghe.com已经备案，若还需使用www.guanghe.com、music.guanghe.com等二级域名，不需要单独备案，但需要在域名申请的机构网站设置一下开启二级域名及绑定相应IP即可。

　　域名有顶级域名和二级，三级之分，一般网站只用到顶级域名即可，有时候一个网站系统比较庞大，那么就可能使用多个域名，如果去申请多个域名肯定不划算，这个时候，使用已申请的一个域名的二级域名的处理方式就应运而生。比如百度买下了baidu.com这一顶级域名，将baidu.com绑定了一个地址，map.baicu.com、music.baicu.com也绑定到了各个地址，不用单独花钱，只是购买了baidu.com这一个顶级域名而已。

#### 域名解析过程

整个域名解析看[这篇](https://www.ruanyifeng.com/blog/2016/06/dns.html)，总结一下就是

1.本机一定要知道`DNS服务器`的IP地址，否则上不了网。通过DNS服务器，才能知道某个域名的IP地址到底是什么，DNS服务器的IP地址，有可能是动态的，每次上网时由网关分配，这叫做DHCP机制；也有可能是事先指定的固定地址。Linux系统里面，DNS服务器的IP地址保存在/etc/resolv.conf文件

2.本机只向自己的DNS服务器查询，dig命令有一个@参数，显示向其他DNS服务器查询的结果。

3.dig的域名解析math.stackexchange.com显示为math.stackexchange.com. ，这个.不是疏忽，而是所有域名的尾部，实际上都有一个根域名。www.example.com真正的域名是www.example.com.root，简写为www.example.com.。因为，根域名.root对于所有域名都是一样的，所以平时是省略的

4.DNS服务器根据域名的层级，进行分级查询，每一级域名都有自己的NS记录，NS记录指向该级域名的域名服务器。这些服务器知道下一级域名的各种记录。所谓"分级查询"，就是从根域名开始，依次查询每一级域名的NS记录，直到查到最终的IP地址。从"根域名服务器"查到"顶级域名服务器"的NS记录和A记录（IP地址），从"顶级域名服务器"查到"次级域名服务器"的NS记录和A记录（IP地址），从"次级域名服务器"查出"主机名"的IP地址

5.DNS服务器怎么知道"根域名服务器"的IP地址。回答是"根域名服务器"的NS记录和IP地址一般是不会变化的，所以内置在DNS服务器里面，目前，世界上一共有十三组根域名服务器，从A.ROOT-SERVERS.NET一直到M.ROOT-SERVERS.NET

6.由于CNAME记录就是一个替换，所以域名一旦设置CNAME记录以后，就不能再设置其他记录了（比如A记录和MX记录），这是为了防止产生冲突。举例来说，foo.com指向bar.com，而两个域名各有自己的MX记录，如果两者不一致，就会产生问题。由于顶级域名通常要设置MX记录，所以一般不允许用户对顶级域名设置CNAME记录。

补充下：这种说法应该ok

从"根域名服务器"查到"顶级域名服务器"的NS记录和A记录（IP地址）

从"顶级域名服务器"查到"次级域名服务器"的NS记录和A记录（IP地址）

从"次级域名服务器"查出"主机名"的IP地址

#### 域名的记录类型

域名与IP之间的对应关系，称为"记录"（record）。根据使用场景，"记录"可以分成不同的类型（type），前面已经看到了有A记录和NS记录。

（1） A：地址记录（Address），返回域名指向的IP地址。

（2） NS：域名服务器记录（Name Server），返回保存下一级域名信息的服务器地址。该记录只能设置为域名，不能设置为IP地址。

（3）MX：邮件记录（Mail eXchange），返回接收电子邮件的服务器地址。

（4）CNAME：规范名称记录（Canonical Name），返回另一个域名，即当前查询的域名是另一个域名的跳转，详见下文。

（5）PTR：逆向查询记录（Pointer Record），只用于从IP地址查询域名，详见下文。

一般来说，为了服务的安全可靠，至少应该有两条NS记录（Name Server的缩写，即哪些服务器负责管理stackexchange.com的DNS记录），而A记录和MX记录也可以有多条，这样就提供了服务的冗余性，防止出现单点失败

facebook.github.io的CNAME记录指向github.map.fastly.net。也就是说，用户查询facebook.github.io的时候，实际上返回的是github.map.fastly.net的IP地址。这样的好处是，变更服务器IP地址的时候，只要修改github.map.fastly.net这个域名就可以了，用户的facebook.github.io域名不用修改。

由于CNAME记录就是一个替换，所以域名一旦设置CNAME记录以后，就不能再设置其他记录了（比如A记录和MX记录），这是为了防止产生冲突。举例来说，foo.com指向bar.com，而两个域名各有自己的MX记录，如果两者不一致，就会产生问题。由于顶级域名通常要设置MX记录，所以一般不允许用户对顶级域名设置CNAME记录。


注意：

（1）一个域名可以指向多个A记录
（2）一个域名可以同时指向A记录和Cname，A记录优先生效
（3）多个域名可以解析到同一个Cname，这样修改起来方便
（4）一个域名可以指向多个Cname吗（就一个Cname）
（5）多个域名可以指向一个A记录吗（www.baidu.com，mail.baidu.com 指向同一个A）

这边就是多种指向的关系，除了一个域名只能指向一个cname外？？应该也能指向多个cname把

#### 域名解析缓存

Linux上装了nscd

nscd的DNS缓存TTL

positive-time-to-live  正向（解析成功后的时间）
negative-time-to-live  反向（解析不成功就不要缓存勒）

```html

#       logfile                 /var/log/nscd.log
#       threads                 4
#       max-threads             32
server-user             nscd
#       stat-user               somebody
debug-level             0
#       reload-count            5
paranoia                no
#       restart-interval        3600

#       enable-cache            passwd          yes
#       positive-time-to-live   passwd          600
#       negative-time-to-live   passwd          20
#       suggested-size          passwd          211
#       check-files             passwd          yes
#       persistent              passwd          yes
#       shared                  passwd          yes
#       max-db-size             passwd          33554432
#       auto-propagate          passwd          yes
#
#       enable-cache            group           yes
#       positive-time-to-live   group           3600
#       negative-time-to-live   group           60
#       suggested-size          group           211
#       check-files             group           yes
#       persistent              group           yes
#       shared                  group           yes
#       max-db-size             group           33554432
#       auto-propagate          group           yes

enable-cache            hosts           yes
positive-time-to-live   hosts           60
negative-time-to-live   hosts           0
suggested-size          hosts           211
check-files             hosts           yes
persistent              hosts           no
shared                  hosts           no
max-db-size             hosts           33554432

#       enable-cache            services        yes
#       positive-time-to-live   services        28800
#       negative-time-to-live   services        20
#       suggested-size          services        211
#       check-files             services        yes
#       persistent              services        yes
#       shared                  services        yes
#       max-db-size             services        33554432
#
#       enable-cache            netgroup        yes
#       positive-time-to-live   netgroup        28800
#       negative-time-to-live   netgroup        20
#       suggested-size          netgroup        211
#       check-files             netgroup        yes
#       persistent              netgroup        yes
#       shared                  netgroup        yes
#       max-db-size             netgroup        33554432

```

`小插曲`

从hosts中什么也没有，到加 8.8.8.8 阻断域名，那么服务器上的curl是立马生效，但是应用没生效，查看netstat -anp中有est的连接，（也就是应用仍然在用这个连接，curl的话不用这个连接）
等到连接消失了，应用也被阻断了

当把8.8.8.8 的阻断域名恢复，那么服务器上的curl是立马生效，应用也应该立马生效了，（没有连接了，重新建连）？？

争对上面的问题做了实验，本质还是有连接存在的问题，而不是linux上的啥DNS缓存

```html

服务器上的每一次curl，都是短连接，有就是每一次都要建连，在客户端随机开端口，curl完就释放连接，可以看到，time wait，这个写一个curl的循环脚本就能看出

而应用不一样是因为，一般应用的http客户端，如httpClient和restTemplate都使用了连接池，实现了Tcp的连接复用，也就是多次请求，都是复用一个连接，netstat可以看到都是同样一个端口

在windows下简单验证

RestTemplate restTemplate = new RestTemplate();

    for (int i = 0; i < 30; i++) {
      ResponseEntity<ListResult> result = restTemplate.exchange("http://XX.XX.com/XX/XX/XX",
          HttpMethod.GET, null, new ParameterizedTypeReference<ListResult>() {});
      result.getBody();

      System.out.println("ok " + i);

      try {
        Thread.sleep(2000);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
    }

可以看到只有一个tcp建连！！

而如果修改域名会发生什么呢？在for循环期间，修改了hosts（即使用了ipconfig /flushdns），并没有产生connect timeout，这跟长连接一直保持是有关的，而等循环结束，连接断开，再运行立马就报错了，是connect timeout

所以这种情况下 有长连接的存在，要重启应用杀光连接才生效

那么恢复的时候呢，就不用了，windows和linux更改了hosts后会生效，所以之前一直connect timeout的会自动ok！！

```

#### 输入http://www.baidu.com发生了什么

1、解析域名

首先，浏览器本身是可以缓存DNS域名解析的，浏览器会先在自己内部的DNS缓存中查找是否有现成的记录

否则，查询操作系统级别的DNS缓存，windows中，有一个叫做DNS Cache的服务，Linux中可能是NSCD，也可能是DNSmasq，如果有命中，直接返回结果

如果没有，查询本机Hosts文件记录，windows和linux中目录不同

如果没有，像电脑配置的DNS服务器发出递归查询请求，等待DNS返回最终响应

域名解析的结果，我们可以通过ping命令捕获

2、连接服务器

连接80或者443端口

3、发送请求

发送HTTP请求，HTTP请求包括请求方法，URL，协议版本三个最基本的要素，以及种类繁多的各式HTTP Header

4、等待响应

5、传输响应内容

6、浏览器渲染处理
