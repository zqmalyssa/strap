---
layout: post
title: nginx使用及运维
tags: [nginx]
author-id: zqmalyssa
---

#### Nginx的基本介绍

Nginx目前常用作应用服务器，实现反向代理等操作

之前先将正代和反代都说下，看一下它们的区别

（1）服务对象不同

正向代理服务器的服务对象是客户端，可以将客户端和代理服务器看作一个整体。

反向代理服务器的服务对象是服务端，可以将服务端和代理服务器看作一个整体。

（2）配置方法不同

正向代理需要在客户端配置代理服务器的地址，比如在浏览器上设置代理服务器地址

反向代理服务器上要配置上服务端的地址。比如：nginx 配置反向代理地址

（3）作用

正向代理当客户端没有办法和服务器直接进行通信的时候，这个时候使用代理服务器是让客户端和服务端通信的好方法。客户端指定请求给代理服务器，代理服务器将请求发送给服务端，将服务端的内容取来，返给客户端。这样完成客户端和服务端的访问。

反向代理可以保证内网的安全，客户端没有办法直接和服务端进行通信，当客户端的请求直接发送给代理服务器，代理服务器去取过服务端的内容，返回给客户端。还有就是做负载均衡，这个是用的最多的，可以保证网站的高并发，高可用性

nginx即可以正向代理也可以反向代理，反向代理用的最多：

正向代理的3个常用设置

1、resolver指令，该指令用于指定DNS服务器的IP地址。DNS服务器的主要工作是进行域名的解析，将域名映射为对应的IP地址。该指令的语法结构为：resolver address ... [valid=time];

2、resolver_timeount指令，resolver_timeount time;该指令用于设置DNS服务器域名解析超时时间，语法结果为：

3、proxy_pass指令，该指令用于设置代理服务器的协议和地址，它不仅仅用于Nginx服务器的代理服务器，更主要的是应用于反向代理服务，我们马上就会谈及。该指令的语法结构为：proxy_pass URL;其中，URL即为设置的代理服务器协议和地址。

示例：

..
server
{
    resolver 8.8.8.8;
    listen 82;
    location /
    {
        proxy_pass http://$http_host$request_uri;
    }
}

实现的片段很简单，设置DNS服务器地址为8.8.8.8，使用默认的53号端口作为DNS服务器的服务端口，代理服务的监听端口设置为82端口，Nginx服务器接收到的所有请求都由第5行的location块进行过滤处理

Nginx服务器代理服务使用的场合不多，从上一节的配置指令来看，功能也相对简单。在使用过程中，有一些需要注意的事项在这里说明一下。

首先，我们在上面提到过，设置Nginx服务器的代理服务器，一般是配置到一个server块中，注意，在该server块中，不要出现 server_name指令，即不要设置虚拟主机的名称或IP。而resolver指令是必需的，如果没有该指令，Nginx服务器无法处理接收到的域名。

其次，Nginx服务器的代理服务器不支持正向代理HTTPS站点（也已经有版本支持了把）

反向代理的设置

Nginx服务器的反向代理服务是其最常用的重要功能之一，在实际工作中应用广泛，涉及的配置指令也比较多，各类指令完成的功能也不尽相同

Nginx服务器提供的反向代理服务也是比较高效的。它能够同时接收的客户端连接由worker_processes指令和worker_connections指令决定，计算方法为：worker_processes * worker_connections / 4.

配置Nginx服务器反向代理用到的指令如果没有特别说明，原则上可以出现在Nginx配置文件的http块、server块或者location块中，但同正向代理服务的设置一样，一般是在搭建的Nginx服务器中单独配置一个server块来设置反射代理服务。这些指令主要由ngx_http_proxy_module模块进行解析和处理。该模块是Nginx服务器的标准HTTP模块。

一般常用的是27个指令，尤其是proxy_pass指令，在实际应用过程中需要注意一些配置细节，需要小心使用。

1、proxy_pass指令

该指令用来设置被代理服务器的地址，可以是主机名称、IP地址加端口号等形式。其语法结构为：proxy_pass URL;

其中，URL为要设置的被代理服务器的地址，包含传输协议、主机名称或IP地址加端口号、URI等要素。传输协议通常是“http”或者“https”。指令同时还接受以“unix”开始的UNIX-domain套接字路径。例如：
proxy_pass http://www.myweb.name/uri;
proxy_pass http://localhost/uri;
proxy_pass http://unix:/tmp/backend.socket:/uri/;

这边是域名

如果被代理服务器是一组服务器的话，可以使用upstream指令配置后端服务器组。例如：

```html
#多个服务器
...
upstream proxy_svrs                  #配置后端服务器
{
    server http://192.168.1.1:8001/uri/;
    server http://192.168.1.2:8001/uri/;
    server http://192.168.1.3:8001/uri/;
}

server
{
    ...
    listen 80;
    server_name www.myweb.name;
    location /
    {
        proxy_pass proxy_svrs;      #使用服务器组名称
    }
}

```

这里首先需要提醒大家proxy_pass指令在使用服务器组名称时应该注意一个细节。在上例中，在组内的各个服务器中都指明了传输协议“http://”，而在proxy_pass指令中就不需要指明了。如果 现在将upstream指令的配置改为：

```html
#不指明http
...
upstream proxy_svrs                  #配置后端服务器
{
    server 192.168.1.1:8001/uri/;
    server 192.168.1.2:8001/uri/;
    server 192.168.1.3:8001/uri/;
}

我们就需要在proxy_pass指令中指明传输协议“http://”；

proxy_pass http://proxy_svrs;

```

在使用该指令的过程中还需要注意，URL中是否包含有URI，Nginx服务器的处理方式是不同的。如果URL中不包含URI，Nginx服务器不会改变原地址的URI；但是如果包含了URI，Nginx服务器将会使用新的URI替代原来的URI。我们举例来说明。

请看下面的Nginx配置片段：

```html
..
server
{
    ...
    server_name www.myweb.name;
    resolver 8.8.8.8;
    listen 82;
    location /server/
    {
        ...
        proxy_pass http://192.168.1.1;
    }
}

```

如果客户端使用“http://www.myweb.name/server ”发起请求，该请求被配置中显示的location块进行处理，由于proxy_pass指令变量不含有URI，所以转向的地址为“http:///192.168.1.1/server ”；我们再来看下面的Nginx片段：

```html

..
server
{
    ...
    server_name www.myweb.name;
    resolver 8.8.8.8;
    listen 82;
    location /server/
    {
        ...
        proxy_pass http://192.168.1.1/loc;
    }
}

```

在该配置实例中，proxy_pass指令的URI包含了URI“/loc”；如果客户端仍然使用“http://www.myweb.name/server ”发起请求，Nginx服务器将会把地址转向“http://192.168.1.1/loc/ ”

通过上面的实例，我们可以总结出，在使用proxy_pass指令时，如果不想改变原地址中的URI，就不要在URL变量中配置URI。 明白了上面这两个例子的用法，我们来解释大家经常讨论的一个问题，就是proxy_pass指令的URL变量末尾是否加斜杠“/”的问题。请看这两个配置示例：

#配置1 proxy_pass http://192.168.1.1;
#配置2 proxy_pass http://192.168.1.1/;

实例1：

```html
..
server
{
    ...
    listen 80;
    server_name www.myweb.name;         #注意location的uri变量
    location /
    {
        ...
        #配置1 proxy_pass http://192.168.1.1;
        #配置2 proxy_pass http://192.168.1.1/;
    }
}

```

在该配置中，location块使用“/”作为uri变量的值来匹配不包含URI的请求URL。由于请求URL中不包含URL，因此配置1和配置2的效果是一样的。比如客户端的请求URL为“http://www.myweb.name/index.htm”，其将会被实例1中的location块匹配成功并进行处理。不管使用配置1不是配置2，转向的URL都为：“http://192.168.1.1/index.htm”。

实例2：

```html
..
server
{
    ...
    listen 80;
    server_name www.myweb.name;         #注意location的uri变量
    location /server/
    {
        ...
        #配置1 proxy_pass http://192.168.1.1;
        #配置2 proxy_pass http://192.168.1.1/;
    }
}

```

在该配置中，location块使用“/server/”作为uri变量的值来匹配包含的URI“/server/”的请求URL。这时，使用配置1和配置2的转向结果就不相同了。使用配置1和配置2的转向效果就不相同了。使用配置1时候，proxy_pass指令中的URL变量不包含URI，Nginx服务器将不改变原地址的URI，使用配置2的时候，proxy_pass指令中的URL变量包含URI“/”，Nginx服务器会将原地址的URI替换为"/"。

比如客户端的请求URI为“http://www.myweb.name/server/index.htm”将会被实例2的location块匹配成功并进行处理。使用配置1的时候，转向的URL为“http://192.168.1.1/server/index.htm”，原地址的URI“、server/”示被改变；使用配置2时，转向的URL为“http://192.168.1.1/index.htm”,可以看到原地址的URI“/server/”被替换为“/”。

2、proxy_hide_header指令

该指令用于设置Nginx服务器在发送HTTP响应时，隐藏一些头域信息。其语法结构为：

proxy_hide_header field;

其中，field为需要隐藏的头域。该指令可以在http块、server块或者location块中进行配置。

3、proxy_pass_header指令

默认情况下，Nginx服务器在发送响应报文时，报文头中不包含“Date”、“Server”、“X-Accel”等来自被代理服务器的头域信息。该指令可以设置这些头域信息以被发送，其语法结构为：

proxy_pass_header field;

4、proxy_pass_request_body指令

该指令用于配置是否将客户端请求的请求体发送给代理服务器，其语法结构为：

proxy_pass_request_body on | off;

默认开启（on），开头可以在http块、server块或者location块中进行配置。

5、proxy_pass_request_headers指令

该指令用于配置是否将客户端请求的请求头发送给代理服务器，其语法结构为：

proxy_pass_request_headers  on | off;

默认开启（on），开头可以在http块、server块或者location块中进行配置。

6、proxy_set_header指令

该指令可以理发Nginx服务器接收到的客户端请求的请求头信息，然后将新的请求头发送给被代理的服务器，其语法结构为：

proxy_set_header field value;

field，要更新的信息所在的区域；value，更改的值，支持使用文本、变量或者变量的组合。

默认情况下，该指令的设置为：

proxy_set_header Host $proxy_host;
proxy_set_header Connection close;

请看一些设置实例：

proxy_set_header Host $http_host;           #将目前Host头域的值填充成客户端的地址
proxy_set_header Host $$host;               #将当前location块的server_name指令填充到Host头域
proxy_set_header Host $$host:$proxy_port;   #listener指令值一起填充到Host头域.

7、proxy_set_body指令

指该指令可以更改Nginx服务器接收到的客户端请求的请求信息，然后将新的请求体发送给被代理的服务器。其语法结构为：

proxy_set_body_value;

其中，value为更改的信息，支持使用文本、变量或者变量的组合。

8、proxy_bind指令

官方文档中对该指令的解释是，强制将与代理主机的连接绑定到指定的IP地址。通俗来讲就是，在配置多个基于名称或者基于IP地址。通俗来讲就是，在配置了多个基于名称或者基于IP主机的情况下，如果我们希望代理连接由指定的主机处理，就可以使用该指令进行配置，其语法结构为：

proxy_bind adress;

其中，adress为指定主机的IP地址。

9、proxy_connect_timeout指令

该指令配置Nginx服务器与后端被代理服务器尝试建立连接的超时时间，其语法结构为:

proxy_connect_timeout time;

其中，time为设置的超时时间，默认60s。

10、proxy_read_timeout指令

该指令配置Nginx服务器向后端被代理服务器（组）发出的read请求后，等待响应的超时时间，其语法结构为：

proxy_read_timeout time;

其中，time为设置的超时时间，默认60s。

11、proxy_send_timeout指令

该指令配置Nginx服务器向后端被代理服务器（组）发出的write请求后，等待响应的超时时间，其语法结构为：

proxy_write_timeount time

其中，time为设置的超时时间，默认60s。

12、proxy_http_version指令

该指令设置用于Nginx服务器提供代理服务的HTTP协议版本，其语法结构为：

proxy_http_version 1.0 |  1.1;

默认版本为1.0版本，1.1版本支持upstream服务器组设置的keepalive指令。

13、proxy_method指令

该指令用于设置Nginx服务器请求被代理服务器时使用的请求方法，一般为POST或者GET。设置了该指令，客户端的请求方法将被忽略。其语法结构为：

proxy_method method;

其中，method的值可以设置为POST或者GET，注意不加引号。

14、proxy_ignore_client_abort指令

该指令用过设置在客户端中断网络请求时，Nginx服务器是否中断对被代理服务器的请求，其语法结构为：

proxy_ignore_client_abort on | off

默认设置为off，当客户端中断网络请求时，Nginx服务器中断对被代理服务器的请求。

15、proxy_ignore_header指令

该指令用于设置一些HTTP响应头的头域，Nginx服务器接收到被代理服务器的响应数据后，不会处理被设置的头域。其语法结构为：

proxy_ignore_header field ...;

其中,field为要设置的HTTP响应头的头域，例如“X-Accel-Redirect”、“X-Accel-Expires”、“Cache-Control”、“Expires”或“Set-Cookie”等。

16、proxy_redirect指令

该指令用于修改被代理服务器返回的响应头中的Location头域和“Refresh”头域，与proxy_pass指令配合使用。比如，Nginx服务器通过proxy_pass指令将客户端的请求地址重写为被代理服务器的地址，那么Nginx服务器返回客户端的响应头中“Location”头域显示的地址就应该和客户端发起请求的地址相对应，而不是代理服务器直接返回的地址信息，否则就会出问题。该指令解决了这个问题，可以把代理服务器返回的地址信息更改为需要的地址信息。其语法结构为:

proxy_redirect redirect replacement
proxy_redirect default;
proxy_redirect off;

redirect，匹配“Location”头域值的字符串，支持变量的使用和正则表达式。

replacement，用于替换redirect变量内容的字符串，支持变量的使用。

该指令的用法我们通过几个配置实例来解释。

对于第1个结构，假设被代理服务器返回的响应头中的“Location”头域为：

Location: http://localhost:8081/proxy/some/uri

该指令设置为：

proxy_redirect http://localhost:8081/proxy/ http://myweb/fronted/;

Nginx服务器会将“Location”头域信息更改为：

Location：http://myweb/frontend//some/uri;

这样，客户端收到的响应信息头部中的“Location”头域就被更改了。

结构 2使用default，代表使用location块的uri变量作为replacement，并使用proxy_pass变量作为redirect。请看下面两段配置，它们的配置效果是等同的。

#配置1
location /server/
{
    proxy_pass http://proxyserver/source/;
    proxy_redirect default;

}

#配置2
location /server/
{
    proxy_pass http//proxyserver/source/;
    proxy_redirect http://proxyserver/source/ /server/;

}

使用结构3可以将当前作用域下所有的proxy_redirect指令全部设置为无效。

17、proxy_intercept_errors指令

该指令用于配置一个状态是否开启还是关闭。在开启状态时，如果被代理的服务器返回的HTTP状态码为400或者大于400，则Nginx服务器使用自己定义的错误页（使用error_page指令）；如果是关闭状态，Nginx服务器直接将被代理服务器返回的HTTP状态返回给客户端。其请求结构为

proxy_intercept_errors on | off

18、proxy_headers_hash_max_size指令

该指令用于配置HTTP报文头哈希表的容量，其语法结构为：

proxy_headers_hash_max_size size;

其中，size为HTTP报文头哈希表的容量上限，默认为512个字符，即：

proxy_headers_hash_max_size 512;

Nginx服务器为了能够快速检索HTTP报文头中的各项信息，比如服务器名称、MIME类型、请求头名等，使用哈希表存储这些信息。Nginx服务器在申请存放HTTP报文头的空间时，通常以固定大小为单位申请，该大小由proxy_headers_hash_bucket_size指令配置。

在Nignx配置中，不仅能够配置整个哈希表的大小上限，对大部分内容项，也可以配置其大小上限，比如server_names_hash_max_size指令和server_names_hash_bucket_size指令用来设置服务器名称的字符数长度。

19、proxy_headers_hash_bucket_size指令

该指令用于设置Nginx服务器申请存放HTTP报文头的哈希表容量的单位大小。该指令的具体作用在上面proxy_headers_bucket_size指令的使用中已经说明。其语法结构为：

proxy_headers_hash_bucket_size size;

20、proxy_next_upstream指令

在配置Nginx服务器反向代理功能时，如果使用upstream指令配置了一组服务器作为代理 服务器，服务器组中各服务器的访问规则遵循upstream指令配置的轮询规则 ，同时可以使用该指令配置在发生哪些异常情况时，将请求顺次交由下一个组内服务器处理。该指令的语法结构为：

proxy_next_upstream status ...;

其中，status为设置的服务器返回状态，可以是一个或者多个。这些状态包括：

error，建立连接、向被代理服务器发送请求或者读取响应头时服务器发生连接错误。

timeout，建立连接、向被代理服务器发送请求或者读取响应头时服务器发生连接超时。

invalid_header，被代理的服务器返回的响应头为空或者无效。

http_500|http_502|http_503|http_504|http_404，被代理的服务器返回500、502、503、504或者404状态代码。

off，无法将请求发送给被代理的服务器。

注意

与被代理的服务器进行数据传输的过程中发送错误的请求，不包含在该指令支持的状态之内。

21、proxy_ssl_session_reuse指令

该指令用于配置是否使用基于SSL安全协议的会话连接（“https://”）被代理的服务器，其语法结构为：

proxy_ssl_session_reuse on | off


#### Nginx的机制

当你启动nginx以后，使用ps命令查看nginx进程， 会发现nginx进程不只有一个，默认情况下， 你会看到至少两个nginx进程，如下

```html

ps -ef | grep nginx

```

当你启动nginx以后，使用ps命令查看nginx进程， 会发现nginx进程不只有一个，默认情况下， 你会看到至少两个nginx进程，如下

```html

useradd -u 900 nginx

id nginx

nginx -s reload

```

当我启动nginx以后，有两个nginx进程，一个master进程，一个worker进程，这两个nginx进程都有各自的作用，见名知意, "worker"进程天生就是来"干活"的，真正负责处理请求的进程就是你看到的"worker"进程，那么"master"进程有什么用呢? “master"进程其实是负责管理"worker"进程的，除了管理” worker"进程，master"进程还负责读取配置文件、判断配置文件语法的工作，“master进程"也叫"主进程”，在nginx中，"master"进程只能有一个,而"worker"进程可以有多个，worker"进程的数量可以由管理员自己进行定义，那么怎么定义"worker"进程的数量呢?

默认的nginx.conf配置文件中有这样一条配置

worker_ processes 1;

上述配置的意思就是启动nginx后只有1个worker进程，你想要多少个worker进程，将worker_ processe指令的值设置成多少就好了,非常简单, worker_ processes指令只能在main区域中使用,通常情况下,

worker_ processes的值通常不会大于服务器中cpu的核心数量,

换句话说就是，worker进程的数通常与服务器有多少
cpu核心有关，比如，nginx所在主机拥有4核cpu,那么worker_ processes的值通常不会大于4，这样做的原因是为了尽力让每个worker进程都有一个cpu可以使用，尽量避免了多个worker进程抢占同一个cpu的情况,我们也可以将worker_ processes的值设置为"auto"
当worker_ processes的值为auto时，nginx会自动检测当前主机的cpu核心数,并启动对应数量的worker进程，比如，nginx检测到当前主机一共有4个cpu核心，那么nginx就会启动4个worker进程

同时，为了避免cpu在切换进程时产生性能损耗，我们也可以将worker进程与cpu核心进行"绑定"，当worker进程与cpu核心绑定以后，worker进程可以更好的专注的使用某个cpu核心上的缓存，从而减少因为cpu切换不同worker进程而带来的缓存失效，如果想要让worker进程与某个cpu核心绑定,则需要借助另外一个配置指令,它就是"worker_ cpu_ affinity"指令

想要搞明白怎样使用"worker_ cpu_ affinity" 指令,最好先来了解一个概念，这个概念就是"cpu掩码",我们可以通过"cpu掩码"表示某个cpu核心，比如，当前机器上一共有4个cpu核心，那么我们就用4个0表示这4个核，也就是说，我们可以使用如下字符表示这4个核: 0000
那么第一个核就用如下字符表示
0001
第二个核就用如下字符表示
0010
第三个核就用如下字符表示
0100

规律就是，有几个核,就用几个0表示，如果想要使用某个核，就将对应位的0改成1,位从右边开始
比如，如果有8个核,我就可以使用如下字符表示这8个核中的第二个核:
00000010

```html

cat /proc/cpuinfo | grep "physical id" | sort | uniq | wc -l // 核心的数量

worker_cpu_affinity 1000 0100 0010 0001;

```

#### SLB中

配置文件还是nginx.conf，但是include了upstreams 和 vhosts的conf，这里面的id名为vs的id名


backed具体的机器在upstreams里面

proxy_paas在vhosts里面

```html

proxy_pass $upstream_scheme://$upstream;

前后端的vs是一个，不同 IDC 才会有两个（一个VS意思就是 协议+域名+端口是一个）

前后端一个VS后面的话是两个group，后端的成员服务器信息是写在一个vs.conf中的



```


#### Nginx的一些高可用

Keepalived介绍
Keepalived是一个基于VRRP协议来实现的服务高可用方案，可以利用其来避免IP单点故障，类似的工具还有heartbeat、corosync、pacemaker。但是它一般不会单独出现，而是与其它负载均衡技术（如lvs、haproxy、nginx）一起工作来达到集群的高可用。

keepalived与heartbeat/corosync等比较

Heartbeat、Corosync、Keepalived这三个集群组件我们到底选哪个好呢？

首先要说明的是，Heartbeat、Corosync是属于同一类型，Keepalived与Heartbeat、Corosync，根本不是同一类型的。
Keepalived使用的vrrp协议方式，虚拟路由冗余协议 (Virtual Router Redundancy Protocol，简称VRRP)；
Heartbeat或Corosync是基于主机或网络服务的高可用方式；

简单的说就是，Keepalived的目的是模拟路由器的高可用，Heartbeat或Corosync的目的是实现Service的高可用。
所以一般Keepalived是实现前端高可用，常用的前端高可用的组合有，就是我们常见的LVS+Keepalived、Nginx+Keepalived、HAproxy+Keepalived。而Heartbeat或Corosync是实现服务的高可用，常见的组合有Heartbeat v3(Corosync)+Pacemaker+NFS+Httpd 实现Web服务器的高可用、Heartbeat v3(Corosync)+Pacemaker+NFS+MySQL 实现MySQL服务器的高可用。

总结一下，Keepalived中实现轻量级的高可用，一般用于前端高可用，且不需要共享存储，一般常用于两个节点的高可用。而Heartbeat(或Corosync)一般用于服务的高可用，且需要共享存储，一般用于多节点的高可用。这个问题我们说明白了。

那heartbaet与corosync又应该选择哪个好？

一般用corosync，因为corosync的运行机制更优于heartbeat，就连从heartbeat分离出来的pacemaker都说在以后的开发当中更倾向于corosync，所以现在corosync+pacemaker是最佳组合。

双机高可用一般是通过虚拟IP（飘移IP）方法来实现的，基于Linux/Unix的IP别名技术。

双机高可用方法目前分为两种：

1）双机主从模式：即前端使用两台服务器，一台主服务器和一台热备服务器，正常情况下，主服务器绑定一个公网虚拟IP，提供负载均衡服务，热备服务器处于空闲状态；当主服务器发生故障时，热备服务器接管主服务器的公网虚拟IP，提供负载均衡服务；但是热备服务器在主机器不出现故障的时候，永远处于浪费状态，对于服务器不多的网站，该方案不经济实惠。

2）双机主主模式：即前端使用两台负载均衡服务器，互为主备，且都处于活动状态，同时各自绑定一个公网虚拟IP，提供负载均衡服务；当其中一台发生故障时，另一台接管发生故障服务器的公网虚拟IP（这时由非故障机器一台负担所有的请求）。这种方案，经济实惠，非常适合于当前架构环境。

操作系统：centos6.8，64位
master机器（master-node）：103.110.98.14/192.168.1.14
slave机器（slave-node）：103.110.98.24/192.168.1.24
公用的虚拟IP（VIP）：103.110.98.20 //负载均衡器上配置的域名都解析到这个VIP上

应用系统              域名/虚拟目录             对应的后端应用服务器及URL
svn                 http://dev.xxx.com/svn      http://192.168.1.108/svn
svn web管理         http://dev.xxx.com/submin   http://192.168.1.108/submin
网站                http://www.xxx.com          http://192.168.1.101
                                                http://192.168.1.102
                                                http://192.168.1.118
OA                  http://oa.xxx.com           http://192.168.1.101:8080
                                                http://192.168.1.102:8080


Nginx配置，其中:
多域名指向是通过虚拟主机（配置http下面的server）实现;
同一域名的不同虚拟目录通过每个server下面的不同location实现;
到后端的服务器在vhost/LB.conf下面配置upstream,然后在server或location中通过proxy_pass引用。

upstream LB-WWW {
      ip_hash;
      server 192.168.1.101:80 max_fails=3 fail_timeout=30s;     #max_fails = 3 为允许失败的次数，默认值为1
      server 192.168.1.102:80 max_fails=3 fail_timeout=30s;     #fail_timeout = 30s 当max_fails次失败后，暂停将请求分发到该后端服务器的时间
      server 192.168.1.118:80 max_fails=3 fail_timeout=30s;
    }

upstream LB-OA {
      ip_hash;
      server 192.168.1.101:8080 max_fails=3 fail_timeout=30s;
      server 192.168.1.102:8080 max_fails=3 fail_timeout=30s;
}

server {
  listen      80;
  server_name dev.wangshibo.com;

  access_log  /usr/local/nginx/logs/dev-access.log main;
  error_log  /usr/local/nginx/logs/dev-error.log;

  location /svn {
     proxy_pass http://192.168.1.108/svn/;
     proxy_redirect off ;
     proxy_set_header Host $host;
     proxy_set_header X-Real-IP $remote_addr;
     proxy_set_header REMOTE-HOST $remote_addr;
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
     proxy_connect_timeout 300;             #跟后端服务器连接超时时间，发起握手等候响应时间
     proxy_send_timeout 300;                #后端服务器回传时间，就是在规定时间内后端服务器必须传完所有数据
     proxy_read_timeout 600;                #连接成功后等待后端服务器的响应时间，已经进入后端的排队之中等候处理
     proxy_buffer_size 256k;                #代理请求缓冲区,会保存用户的头信息以供nginx进行处理
     proxy_buffers 4 256k;                  #同上，告诉nginx保存单个用几个buffer最大用多少空间
     proxy_busy_buffers_size 256k;          #如果系统很忙时候可以申请最大的proxy_buffers
     proxy_temp_file_write_size 256k;       #proxy缓存临时文件的大小
     proxy_next_upstream error timeout invalid_header http_500 http_503 http_404;
     proxy_max_temp_file_size 128m;
     proxy_cache mycache;
     proxy_cache_valid 200 302 60m;
     proxy_cache_valid 404 1m;
   }

  location /submin {
     proxy_pass http://192.168.1.108/submin/;
     proxy_redirect off ;
     proxy_set_header Host $host;
     proxy_set_header X-Real-IP $remote_addr;
     proxy_set_header REMOTE-HOST $remote_addr;
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
     proxy_connect_timeout 300;
     proxy_send_timeout 300;
     proxy_read_timeout 600;
     proxy_buffer_size 256k;
     proxy_buffers 4 256k;
     proxy_busy_buffers_size 256k;
     proxy_temp_file_write_size 256k;
     proxy_next_upstream error timeout invalid_header http_500 http_503 http_404;
     proxy_max_temp_file_size 128m;
     proxy_cache mycache;
     proxy_cache_valid 200 302 60m;
     proxy_cache_valid 404 1m;
    }
}

server {
   listen       80;
   server_name  www.wangshibo.com;

    access_log  /usr/local/nginx/logs/www-access.log main;
    error_log  /usr/local/nginx/logs/www-error.log;

   location / {
       proxy_pass http://LB-WWW;
       proxy_redirect off ;
       proxy_set_header Host $host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header REMOTE-HOST $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_connect_timeout 300;
       proxy_send_timeout 300;
       proxy_read_timeout 600;
       proxy_buffer_size 256k;
       proxy_buffers 4 256k;
       proxy_busy_buffers_size 256k;
       proxy_temp_file_write_size 256k;
       proxy_next_upstream error timeout invalid_header http_500 http_503 http_404;
       proxy_max_temp_file_size 128m;
       proxy_cache mycache;
       proxy_cache_valid 200 302 60m;
       proxy_cache_valid 404 1m;
      }
}

server {
     listen       80;
     server_name  oa.wangshibo.com;

    access_log  /usr/local/nginx/logs/oa-access.log main;
    error_log  /usr/local/nginx/logs/oa-error.log;

     location / {
       proxy_pass http://LB-OA;
       proxy_redirect off ;
       proxy_set_header Host $host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header REMOTE-HOST $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_connect_timeout 300;
       proxy_send_timeout 300;
       proxy_read_timeout 600;
       proxy_buffer_size 256k;
       proxy_buffers 4 256k;
       proxy_busy_buffers_size 256k;
       proxy_temp_file_write_size 256k;
       proxy_next_upstream error timeout invalid_header http_500 http_503 http_404;
       proxy_max_temp_file_size 128m;
       proxy_cache mycache;
       proxy_cache_valid 200 302 60m;
       proxy_cache_valid 404 1m;
      }
}


LB -> SLB 4层  SLB -> Server 7层 基于Tengine

实现 硬件LB → SLB → 应用集群 的三层路由架构
a.请求通过DNS解析出来的IP, 使用4层协议到达硬件LB
b.硬件LB根据到达的VIP，通过4层协议将请求负载到SLB服务器
c.SLB服务器根据7层协议HTTP中的信息，域名和访问的URI，将请求路由负载到具体的应用服务器

SLB基于开源软件Nginx实现
a.SLB底层使用开源软件Nginx实现具体的负载路由功能
b.SLB自研发了管理系统，通过抽象业务建模，最终实现和XX生态系统、Nginx的双向交互功能

举例：

XXXmonitor.xxx.xxx.com  rb的slb是这套

http://slbxxx.xxxx.com/portal/slb/conf#?env=pro&slbId=200 （portal）

随便登到上面的slb的server上，slb server上的配置是一样的，就是nginx

配置在 /opt/app/nginx/conf，每台server上的配置是一样的

在upstreams中，有个31188.conf，里面的内容是

upstream backend_115484 {
 server 10.56.22.21:8080 weight=5 max_fails=3 fail_timeout=30;

// 有长连接哦

 keepalive 50;

 keepalive_timeout 55s;


}

upstream backend_115486 {
 server 10.61.107.249:80 weight=5 max_fails=3 fail_timeout=30;
 keepalive 50;

 keepalive_timeout 55s;


}

就是nginx的upstreams了，这其中有前端的80，有后端的8080

slb新增server，负载配置得去LB上配置


#### Nginx的Win简单搭建及使用，老文

在[官网](http://nginx.org/en/download.html)下载nginx，一般下个stable的zip版本就行，然后下载完解压

下载完成后启动nginx，有很多启动的方式
1)直接双击nginx.exe可以
2)在命令行中输入nginx.exe或者start nginx

检查nginx启动是否成功，浏览器输入`http://localhost:80`

看看nginx的配置

```html
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}

```

默认是80端口启动，修改后不需要重启nginx，直接执行命令

```html
nginx -s reload
```

关闭nginx的话，如果是命令行启动的话，关闭窗口是不能结束nginx进程的，两种方法
1) nginx -s stop(快速停止nginx)  或者 nginx -s quit(完整有序的停止nginx)
2) 使用taskkill taskkill /f /t /im nginx.exe

nginx配置请求转发地址

```html
upstream tomcat_server {
  server localhost:8080;
}

server {
  location / {
    proxy_pass http://tomcat_server;
  }
}

//也可以配置多个目标服务器，weight权重越高被访问的几率越高
upstream tomcat_server {
  server localhost:8080 weight=2;
  server 192.168.101.9:8080 weight=1;
}

```
nginx也可以配置静态资源，将静态资源配置在`nginx-1.16.1\static`这样的目录下，然后nginx中做如下配置

```html
location / {
  root E:\nginx16\nginx-1.16.1\static;
  index index.html index.htm;
}
```
