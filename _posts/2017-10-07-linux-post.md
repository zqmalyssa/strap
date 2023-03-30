---
layout: post
title: "Linux相关内容"
author-id: "zqmalyssa"
tags: [code, operations]
---

Linux运维在日常工作中真的是太常见了，所有安装在linux服务器上的基本组件都要比较熟，能够基本掌握使用方法，能够编写一些脚本，不管是本身的shell语言，python/go等，都会节约大量时间，提高工作效率

## 不定期补充

### 一、脚本跟定时器的使用

下面写一段简单的脚本，主要作用就是获取K8s运行的所有pod中的镜像，并从私有镜像库中拉取

```html
#!/bin/bash
for podName in $(kubectl get pod | awk 'NR == 1 {next} {print $1}')
do
 image=$(kubectl describe pod $podName | grep Image: | awk '{print $2}')
 echo $podName" image is "$image
 docker pull $image
done
```

需要用到`awk`命令，熟悉一下，然后进行拉取

之后，我们在CentOS中用自带的定时任务跑起来

```html
crontab -e
crontab -l
```

-e 进行任务编辑，-l 展示已有的任务，编辑采用时间间隔方式

```html
*/2 * * * * /bin/sh /home/pullImages.sh
```

```html
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
40 17 * * * bash /opt/install/test/pullImages.sh
```

保存后即可，可以发现指定间隔时间或者指定时间点后会运行脚本，crontab的日志会存在`/var/log/cron`中，如果是前几天的应该会在同级目录下存在对应日期的日志文件，crontab的一些常用的配置如下

```html
#每晚的21:30重启apache。
30 21 * * * /usr/local/etc/rc.d/lighttpd restart
#每月1、10、22日
45 4 1,10,22 * * /usr/local/etc/rc.d/lighttpd restart
#每天早上6点10分
10 6 * * * date
#每两个小时
0 */2 * * * date
#晚上11点到早上8点之间每两个小时，早上8点
0 23-7/2，8 * * * date
#每个月的4号和每个礼拜的礼拜一到礼拜三的早上11点
0 11 4 * mon-wed date
#1月份日早上4点
0 4 1 jan * date 
```


### 二、CentOS7 无法启动，enter emergency mode 报错 Failed to mount /sysroot 解决方法

CentOS7 无法启动，进入紧急模式，enter emergency mode

```html
# journalctl
```

根据提示查看日志，发现报错：Failed to mount /sysroot

根据，老外的网站提供的线索，执行这个命令

```html
# xfs_repair -v -L /dev/dm-0
```
然后，就修复了，再执行init 6 重新reboot，就OK

```html
# init 6
```

0 表示关机，1 表示单用户模式，2 表示多用户模式，3 表示切换到命令行模式  服务一般处于这种模式，4 表示未被使用的模式，5 表示切换到桌面模式，6 表示重启


### 三、mysql访问某个库中的表出错，ERROR 1018 (HY000): Can't read dir of './test/'

原因是用集群的root用户启动的mysql，创建了database及表，这时候`/var/lib/mysql`下面对应的数据库的所属用户及用户组是root，修改成mysql的用户及用户组

```html
# chown -R mysql /var/lib/mysql/test
# chgrp -R mysql /var/lib/mysql/test
```
可以正常去`show tables`了

### 四、Linux中awk使用技巧

awk能使linux的工作非常方便，可以联合bash进行使用，举例如下

```html
# kubectl get pod | awk 'NR!=2 {next} {print "kubectl describe pod "$1}'
# kubectl get pod | awk '{if(NR>=2 && NR<=4) print "kubectl describe pod "$1}'
```
上面指定某行或者某些行进行输出

```html
# kubectl get pod | awk 'NR==1 {next} {print "kubectl describe pod "$1}' | bash
```
上面将指定命令进行执行

### 五、Linux中xargs使用技巧

xargs一般可以跟find联合使用，将结果作为参数传入

```html
# find . -name "qwer-ze*" | xargs -I {} cp {} /root/test_z/
```
上面将搜索的文件结果拷贝到指定目录，其中-I参数要在后面跟{}，而-i参数则不需要在后面跟{}

```html
# find . -name "*.yaml" | xargs grep "name"
```

查找当前目录中所有扩展名是yaml的文件，并找出包含name的行

### 六、Linux中的快速操作

比较实用的一些：

`<ctrl+w>` 移除光标前的一个单词

`<ctrl+u>` 清除光标前至行首的间的所有内容

`<ctrl+k>` 清除光标后至行尾的所有内容

`<ctrl+y>` 恢复上面三种操作

`<ctrl+方向左键>` 一个单词一个单词的向左跳

`<ctrl+方向右键>` 一个单词一个单词的向右跳

### 七、Linux中Ip和ifconfig操作

linux的ip命令和ifconfig类似，但前者功能更强大，并旨在取代后者。使用ip命令，只需一个命令，你就能很轻松地执行一些网络管理任务。ifconfig是net-tools中已被废弃使用的一个命令，许多年前就已经没有维护了。iproute2套件里提供了许多增强功能的命令，ip命令即是其中之一，大多数Linux发行版已经预装了iproute2工具。

要给你的机器设置一个IP地址，可以使用下列ip命令

```html
# ip addr add 192.168.0.193/24 dev wlan0
```

按照上述方式设置好IP地址后，需要查看是否已经生效

```html
# ip addr show wlan0
```

有几个路由条目。这个结果显示有几个设备通过不同的网络接口连接起来。它们包括WIFI、以太网和一个点对点连接

```html
# ip route show
```

假设现在你有一个IP地址，你需要知道路由包从哪里来。可以使用下面的路由选项

```html
# ip route get 10.42.0.47
```

要更改默认路由，使用下面ip命令

```html
# ip route add default via 192.168.0.196
```

当你需要获取一个特定网络接口的信息时，在网络接口名字后面添加选项ls即可。使用多个选项-s会给你这个特定接口更详细的信息。特别是在排除网络连接故障时，这会非常有用

```html
# ip -s -s link ls p2p1
```

地址解析协议（ARP）用于将一个IP地址转换成它对应的物理地址，也就是通常所说的MAC地址。使用ip命令的neigh或者neighbour选项，你可以查看接入你所在的局域网的设备的MAC地址

```html
# ip neighbour
```

也可以使用ip命令查看netlink消息。monitor选项允许你查看网络设备的状态。比如，所在局域网的一台电脑根据它的状态可以被分类成REACHABLE或者STALE。使用下面的命令

```html
# ip monitor all
```

你可以使用ip命令的up和down选项来激某个特定的接口，就像ifconfig的用法一样

```html
# ip link set ppp0 down
# ip link set ppp0 up
```
这里要说的是ip link类型的操作只显示链路层的信息，不显示ip地址信息等

帮助

```html
# ip route help
```

ip neighbour的使用，也就是邻居和ARP表管理

邻居对象为共享相同链路的主机建立协议地址和链路层地址之间的绑定。邻接条目被组织成表。IPv 4邻居表的另一个名称是ARP表。相应的命令显示邻居绑定及其属性，添加新的邻居项并删除旧条目

1）ip neighbour add，增加邻居表
2）ip neighbour change，改变已经存在的邻居表
3）ip neighbour replace，增加一个表或者修改已经存在的表

这些命令创建新的邻居记录或更新现有记录。上面的三个命令使用方法如下:
toADDRESS(default)，邻居的协议地址。它要么是IPv4，要么是IPv6地址。
devNAME，连接到邻居的接口。
lladdrLLADDRESS，邻居的链路层地址，可以是null。
nudNUD_STATE，邻居的状态，可以是下面的值：
Ⅰ）permanent，邻居项永远有效，只能内管理员删除。
Ⅱ）noarp，邻居项有效。将不会尝试验证此条目，但可以在其生存期届满时删除该条目。
Ⅲ）reachable，邻居项在可达超时过期之前是有效的。
Ⅳ）stale，邻居的进入是有效的，但却是可疑的。如果邻居状态有效且此命令未更改地址，则此选项不会更改邻居状态。

举例：

```html
ip neigh add 10.0.0.3 lladdr 0:0:0:0:0:1 dev eth0 nud perm
ip neigh chg 10.0.0.3 dev eth0 nud reachable
ip neigh del 10.0.0.3 dev eth0
ip -s n ls 193.233.7.254
#清除邻接条目
ip neighbour flush
```

### 八、Linux中tcpdump的操作

Tcpdump这个工具在分析一些运维问题时非常有用

tcpdump [-nn] [-i 介面] [-w 儲存檔名] [-c 次數] [-Ae] [-qX] [-r 檔案] [所欲擷取的資料內容]


抓包选项：
-c：指定要抓取的包数量。注意，是最终要获取这么多个包。例如，指定"-c 10"将获取10个包，但可能已经处理了100个包，只不过只有10个包是满足条件的包。
-i interface：指定tcpdump需要监听的接口。若未指定该选项，将从系统接口列表中搜寻编号最小的已配置好的接口(不包括loopback接口，要抓取loopback接口使用tcpdump -i lo)，
           ：一旦找到第一个符合条件的接口，搜寻马上结束。可以使用'any'关键字表示所有网络接口。
-n：对地址以数字方式显式，否则显式为主机名，也就是说-n选项不做主机名解析。
-nn：除了-n的作用外，还把端口显示为数值，否则显示端口服务名。
-N：不打印出host的域名部分。例如tcpdump将会打印'nic'而不是'nic.ddn.mil'。
-P：指定要抓取的包是流入还是流出的包。可以给定的值为"in"、"out"和"inout"，默认为"inout"。
-s len：设置tcpdump的数据包抓取长度为len，如果不设置默认将会是65535字节。对于要抓取的数据包较大时，长度设置不够可能会产生包截断，若出现包截断，
     ：输出行中会出现"[|proto]"的标志(proto实际会显示为协议名)。但是抓取len越长，包的处理时间越长，并且会减少tcpdump可缓存的数据包的数量，
     ：从而会导致数据包的丢失，所以在能抓取我们想要的包的前提下，抓取长度越小越好。
-p : 不让网络接口进入混杂模式。默认情况下使用 tcpdump 抓包时，会让网络接口进入混杂模式。一般计算机网卡都工作在非混杂模式下，此时网卡只接受来自网络端口的目的地址指向自己的数据。当网卡工作在混杂模式下时，网卡将来自接口的所有数据都捕获并交给相应的驱动程序。如果设备接入的交换机开启了混杂模式，使用 -p 选项可以有效地过滤噪声。



输出选项：
-e：输出的每行中都将包括数据链路层头部信息，例如源MAC和目标MAC。
-q：快速打印输出。即打印很少的协议相关信息，从而输出行都比较简短。
-X：输出包的头部数据，会以16进制和ASCII两种方式同时输出。
-XX：输出包的头部数据，会以16进制和ASCII两种方式同时输出，更详细。
-v：当分析和打印的时候，产生详细的输出。
-vv：产生比-v更详细的输出。
-vvv：产生比-vv更详细的输出。
其他功能性选项：
-D：列出可用于抓包的接口。将会列出接口的数值编号和接口名，它们都可以用于"-i"后。
-F：从文件中读取抓包的表达式。若使用该选项，则命令行中给定的其他表达式都将失效。
-w：将抓包数据输出到文件中而不是标准输出。可以同时配合"-G time"选项使得输出文件每time秒就自动切换到另一个文件。可通过"-r"选项载入这些文件以进行分析和打印。
-r：从给定的数据包文件中读取数据。使用"-"表示从标准输入中读取。
-A：表示使用 ASCII 字符串打印报文的全部数据，这样可以使读取更加简单，方便使用 grep 等工具解析输出内容。-X 表示同时使用十六进制和 ASCII 字符串打印报文的全部数据。这两个参数不能一起使用。
-l：如果想实时将抓取到的数据通过管道传递给其他工具来处理，需要使用 -l 选项来开启行缓冲模式（或使用 -c 选项来开启数据包缓冲模式）。使用 -l 选项可以将输出通过立即发送给其他命令，其他命令会立即响应。


一些基本示例如下

直接启动tcpdump将监视第一个接口上所有流过的数据包

```html
# tcpdump，其中10.15.198.4是远端的服务

tcpdump -nn -i eth0  host 10.15.198.4

#指定host的抓包，-nn 代表显示的是Ip+port分别是 地址+端口号，不是域名+对应的协议名

tcpdump -nn -P in -i eth0  host 10.15.198.4
tcpdump -nn -i eth0 src host 10.15.198.4

#只抓入向的包，-P默认为inout，指定src字段的话也可以达到这个效果

tcpdump -i eth0 udp
tcpdump -i eth0 proto 17

tcpdump -i eth0 tcp
tcpdump -i eth0 proto 6

#抓取指定协议，上面两组意思相同




```

tcpdump的过滤器

过滤的真正强大之处在于你可以随意组合它们，而连接它们的逻辑就是常用的 与/AND/&& 、 或/OR/|| 和 非/not/!。

and or &&
or or ||
not or !

```html

1、Host 过滤器

用来过滤某个主机的数据报文。例如：

tcpdump host 1.2.3.4

2、Network 过滤器

Network 过滤器用来过滤某个网段的数据，使用的是 CIDR 模式。

可以使用四元组（x.x.x.x）、三元组（x.x.x）、二元组（x.x）和一元组（x）。四元组就是指定某个主机，三元组表示子网掩码为 255.255.255.0，二元组表示子网掩码为 255.255.0.0，一元组表示子网掩码为 255.0.0.0

抓取所有发往网段 192.168.1.x 或从网段 192.168.1.x 发出的流量：

tcpdump net 192.168.1

抓取所有发往网段 10.x.x.x 或从网段 10.x.x.x 发出的流量：

tcpdump net 10

和 Host 过滤器一样，这里也可以指定源和目的：

tcpdump src net 10

3、Proto 过滤器

Proto 过滤器用来过滤某个协议的数据，关键字为 proto，可省略。proto 后面可以跟上协议号或协议名称，支持 icmp, igmp, igrp, pim, ah, esp, carp, vrrp, udp和 tcp。因为通常的协议名称是保留字段，所以在于 proto 指令一起使用时，必须根据 shell 类型使用一个或两个反斜杠（/）来转义。Linux 中的 shell 需要使用两个反斜杠来转义，MacOS 只需要一个。

例如，抓取 icmp 协议的报文：

tcpdump -n proto \\icmp

或者

tcpdump -n icmp

4、Port 过滤器

Port 过滤器用来过滤通过某个端口的数据报文，关键字为 port。例如：

tcpdump port 389



```

理解 tcpdump 的输出

```html

此外，上面的三条数据还是 tcp 协议的三次握手过程，第一条就是 SYN 报文，这个可以通过 Flags [S] 看出。下面是常见的 TCP 报文的 Flags:

[S] : SYN（开始连接）
[.] : 没有 Flag
[P] : PSH（推送数据）
[F] : FIN （结束连接）
[R] : RST（重置连接）

而第二条数据的 [S.] 表示 SYN-ACK，就是 SYN 报文的应答报文。


```

再来一些实际的例子

```html

从 HTTP 请求头中提取 HTTP 用户代理：

tcpdump -nn -A -s1500 -l | grep "User-Agent:"

通过 egrep 可以同时提取用户代理和主机名（或其他头文件）：

tcpdump -nn -A -s1500 -l | egrep -i 'User-Agent:|Host:'

抓取 HTTP GET 流量：

tcpdump -s 0 -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'

也可以抓取 HTTP POST 请求流量：

tcpdump -s 0 -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504f5354'

注意：该方法不能保证抓取到 HTTP POST 有效数据流量，因为一个 POST 请求会被分割为多个 TCP 数据包。

提取 HTTP 请求的 URL

tcpdump -s 0 -v -n -l | egrep -i "POST /|GET /|Host:"   // 看是-l开启了立即缓冲

提取 Cookies

提取 Set-Cookie（服务端的 Cookie）和 Cookie（客户端的 Cookie）：

$ tcpdump -nn -A -s0 -l | egrep -i 'Set-Cookie|Host:|Cookie:'

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on wlp58s0, link-type EN10MB (Ethernet), capture size 262144 bytes
Host: dev.example.com
Cookie: wordpress_86be02xxxxxxxxxxxxxxxxxxxc43=admin%7C152xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxfb3e15c744fdd6; _ga=GA1.2.21343434343421934; _gid=GA1.2.927343434349426; wordpress_test_cookie=WP+Cookie+check; wordpress_logged_in_86be654654645645645654645653fc43=admin%7C15275102testtesttesttestab7a61e; wp-settings-time-1=1527337439

抓取非 ECHO/REPLY 类型的 ICMP 数据包

tcpdump 'icmp[icmptype] != icmp-echo and icmp[icmptype] != icmp-echoreply'

抓取 SMTP/POP3 协议的邮件

tcpdump -nn -l port 25 | grep -i 'MAIL FROM\|RCPT TO'

抓取 NTP 服务的查询和响应

tcpdump dst port 123

抓取 IPv6 流量

tcpdump -nn ip6 proto 6

抓取 DNS 请求和响应

向 Google 公共 DNS 发起的出站 DNS 请求和 A 记录响应可以通过 tcpdump 抓取到：

tcpdump -i wlp58s0 -s0 port 53

抓取 HTTP 有效数据包

$ tcpdump 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'

将输出内容重定向到 Wireshark

$ ssh root@remotesystem 'tcpdump -s0 -c 1000 -nn -w - not port 22' | /Applications/Wireshark.app/Contents/MacOS/Wireshark -k -i -

例如，如果想分析 DNS 协议，可以使用下面的命令：

$ ssh root@remotesystem 'tcpdump -s0 -c 1000 -nn -w - port 53' | /Applications/Wireshark.app/Contents/MacOS/Wireshark -k -i -

-c 选项用来限制抓取数据的大小。如果不限制大小，就只能通过 ctrl-c 来停止抓取，这样一来不仅关闭了 tcpdump，也关闭了 wireshark。

找出发包最多的 IP

$ tcpdump -nnn -t -c 200 | cut -f 1,2,3,4 -d '.' | sort | uniq -c | sort -nr | head -n 20

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp7s0, link-type EN10MB (Ethernet), capture size 262144 bytes
200 packets captured
261 packets received by filter
0 packets dropped by kernel
    108 IP 10.10.211.181
     91 IP 10.10.1.30
      1 IP 10.10.1.50

抓取用户名和密码

$ tcpdump port http or port ftp or port smtp or port imap or port pop3 or port telnet -l -A | egrep -i -B5 'pass=|pwd=|log=|login=|user=|username=|pw=|passw=|passwd=|password=|pass:|user:|username:|password:|login:|pass |user '

抓取 DHCP 报文

最后一个例子，抓取 DHCP 服务的请求和响应报文，67 为 DHCP 端口，68 为客户机端口。

$ tcpdump -v -n port 67 or 68


```


### 九、Linux脚本结合操作

1) 将某个文件夹下应用中的所有yaml文件Pod及镜像的关系信息报错后，与实际运行的pod进行比对，看实际运行的是否与启动的yaml一致
1. echo后面如果是单引号，就全是文本内容，引用变量也没有，如果是双引号，可以引用变量
2. if操作有格式限制[]加空格，if内置了不少方法，注意熟悉
3. awk用了个一般的方法转换成数组，应该有更简便的方法
4. []和[[]]确实是有区别的
5. echo有问题，长度的问题，会覆盖之前的字符？？在某个服务器上有这种情况，自己服务器没有，有设置？？
6. 记得=前后不要有空格，就是赋值，不然会有报错

```html
#!/bin/bash
compareFile=yamlsInfo.txt
if [ -f "$compareFile" ];then
 rm $compareFile
fi

for yamlFile in $(ls *.yaml)
do
 podName=$(grep -C 2 "kind: Deployment" $yamlFile | awk 'END {print $2}')
# echo $podName
 if [ -n "$podName" ];then
   imageName=$(grep 'image:' $yamlFile | grep -v '#' | awk '{print $3}')
   echo $yamlFile" file pod name is "$podName" image name is "$imageName >> $compareFile
#    echo "$yamlFile file pod name is $podName image name is $imageName"
#    echo $yamlFile" file pod name is "$podName
 fi
done

for podName in $(kubectl get pod | awk 'NR == 1 {next} {print $1}')
do
 image=$(kubectl describe pod $podName | grep Image: | awk '{print $2}')
 #echo $podName" image is "$image
 tmp=${podName%-*}
 name=${tmp%-*}
 i=0
 j=0
 for file_list in $(grep "$name" $compareFile | awk '{print $1}')
   do
     file_arr[i]=$file_list
     i=$i+1
   done
 for image_list in $(grep "$name" $compareFile | awk '{print $10}')
   do
     image_arr[j]=$image_list
     j=$j+1
   done
 count=$(grep -c "$name" $compareFile)
 notfound=1
 for ((i=0;i<$count;i++))
   do
     if [ $image == ${image_arr[i]} ];then
       echo "pod $podName found same image, file name is ${file_arr[i]}"
       notfound=0
     fi
   done
 if [ $notfound == 1 ];then
   echo "pod $podName not found same image, please check"
 fi
 #echo "true name is "$name
done
```

### 十、Iptables的使用

iptables在运维及K8S的使用中起到很关键的作用

1. 规则的顺序很重要，开始优先级高，后面就算匹配也不执行了，-I加的规则默认在前面，-A加的规则在后面
2. 多个匹配条件（例如源地址，目的地址），采用“与”关系
3. iptables做为防火墙时，转发的使用forward链，进行规则配置，转发如果是WEB类型，存在双向问题，发出去的请求，在forward中也能响应回来。
4. iptables作白名单时，将链的默认策略反过来，设置成ACCEPT，通过在链的最后设置REJECT实现白名单机制，而不是将默认策略设置成DROP（白名单意思对的，只是默认是ACCEPT，是黑名单）
  链默认策略是GROP，链中的规则对应应该是ACCEPT，只有匹配到的才会被放行，白名单，所有人都当成坏人，只放行好人
  链默认策略是ACCEPT，链中的规则对应应该是DROP和REJECT，只有匹配到的才会被拒绝，黑名单，所有人都当成好人，只拒绝坏人
5. iptables可以自定义链，用于存一类相同的规则，方便管理
6. iptables有各种各样的扩展，方便写语句，-m参数就是使用的扩展模块（这块自行学习）
7. 包到主机网卡，是先从内核空间的INPUT转到用户空间的web服务（发起/终点），反回则是从用户空间的web服务转到内核空间的OUTPUT，从主机网卡出去。如果不是发给本机的，就涉及到其他链，对应路由器前（PREROUTING），转发（FORWARD），路由后（POSTROUTING），都在内核

PREROUTING的规则可以存放在：raw表，mangle表，nat表 // 没有filter，所以链上也做不了阻断
INPUT的规则可以存放在：mangle表，filter表（Centos7还有nat表，6没有）
FORWARD的规则可以存放在：mangle表，filter表
OUTPUT的规则可以存放在：raw表，mangle表，nat表，filter表
POSTROUTING的规则可以存放在：mangle表，nat表

raw表的规则可以被哪些链使用：PREROUTING，OUTPUT
mangle表的规则可以被哪些链使用：PREROUTING，INPUT，FORWARD，OUTPUT，POSTROUTING
nat表的规则可以被哪些链使用：PREROUTING，OUTPUT，POSTROUTING（Centos7还有INPUT，6没有）
filter表的规则可以被哪些链使用：INPUT，FORWARD，OUTPUT // 反过来看会更清晰


但是我们知道，iptables为我们定义了4张”表”,当他们处于同一条”链”时，执行的优先级如下。
优先级次序（由高而低）：
raw –> mangle –> nat –> filter
但是我们前面说过，某些链天生就不能使用某些表中的规则，所以，4张表中的规则处于同一条链的目前只有output链

简单说明：

iptables其实不是真正的防火墙，我们可以把它理解成一个客户端代理，用户通过iptables这个代理，将用户的安全设定执行到对应的”安全框架”中，这个”安全框架”才是真正的防火墙，这个框架的名字叫netfilter

netfilter才是防火墙真正的安全框架（framework），netfilter位于内核空间。

iptables其实是一个命令行工具，位于用户空间，我们用这个工具操作真正的框架。netfilter/iptables（下文中简称为iptables）组成Linux平台下的包过滤防火墙，与大多数的Linux软件一样，这个包过滤防火墙是免费的，它可以代替昂贵的商业防火墙解决方案，完成封包过滤、封包重定向和网络地址转换（NAT）等功能。

Netfilter是Linux操作系统核心层内部的一个数据包处理模块，它具有如下功能：网络地址转换(Network Address Translate)，数据包内容修改以及数据包过滤的防火墙功能

所以说，虽然我们使用service iptables start启动iptables”服务”，但是其实准确的来说，iptables并没有一个守护进程，所以并不能算是真正意义上的服务，而应该算是内核提供的功能。

我们知道iptables是按照规则来办事的，我们就来说说规则（rules），规则其实就是网络管理员预定义的条件，规则一般的定义为”如果数据包头符合这样的条件，就这样处理这个数据包”。规则存储在内核空间的信息包过滤表中，这些规则分别指定了源地址、目的地址、传输协议（如TCP、UDP、ICMP）和服务类型（如HTTP、FTP和SMTP）等。当数据包与规则匹配时，iptables就根据规则所定义的方法来处理这些数据包，如放行（accept）、拒绝（reject）和丢弃（drop）等。配置防火墙的主要工作就是添加、修改和删除这些规则。

表的概念：

我们再想想另外一个问题，我们对每个”链”上都放置了一串规则，但是这些规则有些很相似，比如，A类规则都是对IP或者端口的过滤，B类规则是修改报文，那么这个时候，我们是不是能把实现相同功能的规则放在一起呢，必须能的。

我们把具有相同功能的规则的集合叫做”表”，所以说，不同功能的规则，我们可以放置在不同的表中进行管理，而iptables已经为我们定义了4种表，每种表对应了不同的功能，而我们定义的规则也都逃脱不了这4种功能的范围，所以，学习iptables之前，我们必须先搞明白每种表 的作用。

iptables为我们提供了如下规则的分类，或者说，iptables为我们提供了如下”表”

filter表：负责过滤功能，防火墙；内核模块：iptables_filter

nat表：network address translation，网络地址转换功能；内核模块：iptable_nat

mangle表：拆解报文，做出修改，并重新封装 的功能；iptable_mangle

raw表：关闭nat表上启用的连接追踪机制；iptable_raw

也就是说，我们自定义的所有规则，都是这四种分类中的规则，或者说，所有规则都存在于这4张”表”中。

但是我们需要注意的是，某些”链”中注定不会包含”某类规则”，也就是每个”链”上的规则决定每个”链”上有哪些表，看上面一个对应关系

所以，整个经过的链和链中表的顺序可以看下图

![iptables]({{ "/assets/img/iptables/iptables.png" | relative_url}})

匹配与动作的概念：

匹配条件分为基本匹配条件与扩展匹配条件
基本匹配条件：
源地址Source IP，目标地址 Destination IP
上述内容都可以作为基本匹配条件。
扩展匹配条件：
除了上述的条件可以用于匹配，还有很多其他的条件可以用于匹配，这些条件泛称为扩展条件，这些扩展条件其实也是netfilter中的一部分，只是以模块的形式存在，如果想要使用这些条件，则需要依赖对应的扩展模块。
源端口Source Port, 目标端口Destination Port
上述内容都可以作为扩展匹配条件

处理动作
处理动作在iptables中被称为target（这样说并不准确，我们暂且这样称呼），动作也可以分为基本动作和扩展动作。
此处列出一些常用的动作，之后的文章会对它们进行详细的示例与总结：

ACCEPT：允许数据包通过。

DROP：直接丢弃数据包，不给任何回应信息，这时候客户端会感觉自己的请求泥牛入海了，过了超时时间才会有反应。// 超时，没有返回

REJECT：拒绝数据包通过，必要时会给数据发送端一个响应的信息，客户端刚请求就会收到拒绝的信息。

SNAT：源地址转换，解决内网用户用同一个公网地址上网的问题。

MASQUERADE：是SNAT的一种特殊形式，适用于动态的、临时会变的ip上。

DNAT：目标地址转换。

REDIRECT：在本机做端口映射。

LOG：在/var/log/messages文件中记录日志信息，然后将数据包传递给下一条规则，也就是说除了记录以外不对数据包做任何其他操作，仍然让下一条规则去匹配。

参数的概念：

pkts:对应规则匹配到的报文的个数。

bytes:对应匹配到的报文包的大小总和。

target:规则对应的target，往往表示规则对应的”动作”，即规则匹配成功后需要采取的措施。

prot:表示规则对应的协议，是否只针对某些协议应用此规则。

opt:表示规则对应的选项。

in:表示数据包由哪个接口(网卡)流入，我们可以设置通过哪块网卡流入的报文需要匹配当前规则。

out:表示数据包由哪个接口(网卡)流出，我们可以设置通过哪块网卡流出的报文需要匹配当前规则。

source:表示规则对应的源头地址，可以是一个IP，也可以是一个网段。

destination:表示规则对应的目标地址。可以是一个IP，也可以是一个网段。


iptables的一些基本使用：

1. iptables -FXZ去清除之前所有规则

2. iptables -t filter -L // 指定filter表，默认是，会出3条链，因为INPUT，OUTPUT和FORWARD中都有filter表

3. iptables -t filter -L INPUT // 指定链

4. iptables -t filter -I INPUT ! -s 192.168.1.146 -j ACCEPT // !是取反的作用

5. iptables -I INPUT -s 182.168.1.146 -p tcp -m multiport --dports 22,80:88 -j DROP // 阻断22及范围的端口，用到了扩展模块 -m multiport，multiport扩展只能用于tcp协议与udp协议，即配合-p tcp或者-p udp使用

6. iptables -I INPUT -m iprange --src-range 192.168.1.127-192.168.1.146 -j DROP // iprange的扩展模块，能够使用”!”取反

7. 其他扩展模块，String模块（过滤字串），time模块（过滤生效的时间段），connlimit扩展模块（每个IP最多只能占用2个ssh链接），limit模块（报文到达速率）

8. iptables -I INPUT -p tcp -m tcp --dport 22 --tcp-flags SYN,ACK,FIN,RST,URG,PSH SYN -j REJECT // TCP扩展模块中的--tcp-flags，匹配tcp报文头部的标识位，需要匹配SYN,ACK,FIN,RST,URG,PSH这6个标志位，这些标志位中哪些必须是1呢，第二部分是SYN，第一部分需要匹配的标志位列表中，SYN标志位的值必须为1，其他标志位必须为0，也就匹配到了tcp握手的第一次握手，也可以把第一部分替换成ALL，tcp扩展模块还为我们专门提供了一个选项，可以匹配上文中提到的”第一次握手”，那就是–syn选项

9. iptables -I INPUT -p udp -m udp --dport 137 -j ACCEPT // udp扩展，这个扩展模块中能用的匹配条件比较少，只有两个，就是–sport与–dport，放行samba服务的137与138这两个UDP端口（对的，没错，就是samba，CIFS，投影仪用的那个协议。。）当使用扩展匹配条件时，如果未指定扩展模块，iptables会默认调用与”-p”对应的协议名称相同的模块，所以，当使用”-p udp”时，可以省略”-m udp”。冒号可以指定连续，如果既要连续又要离散，那么可以用multiport扩展块

10. iptables -I INPUT -p icmp -j REJECT  // icmp扩展，ping发和回都是icmp协议，但是还是有区别的，发的是类型8的icmp报文，回的是类型0的icmp报文，具体的type和code看下面的表，主要看的type类型是0，3，8，11，这条会造成的效果是我们无法ping通别人，别人也无法ping通我们（当然了，我们无法是因为回包也是icmp的，回不来），只想我们能ping别人不能ping我们呢，iptables -I INPUT -p icmp -m icmp --icmp-type 8/0 -j REJECT， -m 扩展模块选择可以省略 8/0是ping请求的type/code，回包的是0/0，因为type是8的只有0这一个code，所以也可以简写成--icmp-type 8，也可以写成--icmp-type "echo-request"

11. iptables -I INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT // state模块，可以判断到底是回的包，还是主动发送的包，可以防止攻击，当前主机IP为104，当放行ESTABLISHED与RELATED状态的包以后，并没有影响通过本机远程ssh到IP为77的主机上，但是无法从77上使用22端口主动连接到104上。禁止ssh连接可以这样玩？？

12. iptables -t filter -N IN_WEB // 假设，我们自定义一条链，链名叫IN_WEB，我们可以将所有针对80端口的入站规则都写入到这条自定义链中，当以后想要修改针对web服务的入站规则时，就直接修改IN_WEB链中的规则就好了，即使默认链中有再多的规则，我们也不会害怕了，因为我们知道，所有针对80端口的入站规则都存放在IN_WEB链中，-N创建完后还没有被引用，使用iptables -I INPUT -p tcp --dport 80 -j IN_WEB，使其被INPUT链使用，会显示Chain IN_WEB (1 references)，用-E参数修改链名，删除自定义的用-X，需要满足两个条件1、自定义链没有被任何默认链引用，即references为0，自定义链中没有任何规则

13. forward的功能，充当网络防火墙的过滤和转发，先白名单建个所有REJECT，iptables -I FORWARD -s 10.1.0.0/16 -p tcp --dport 80 -j ACCEPT，iptables -I FORWARD -d 10.1.0.0/16 -p tcp --sport 80 -j ACCEPT，出和入都要放行，响应的报文不用每次都写，删掉上面的第二个，改成iptables -I FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT 就可以使绝大部分响应报文放行了，不管是外部响应还是内部响应，一条规则就能搞定，作为网络防火墙，一定要考虑双向的问题！！！！，上述一条规则就能搞定，再加一条内部主机访问外部ssh的规则，iptables -I FORWARD -s 10.1.0.0/16 -p tcp --dport 22 -j ACCEPT

14. action是SNAT和DNAT时候的一些操作，iptables -t nat -A POSTROUTING -s 10.1.0.0/16 -j SNAT --to-source 192.168.1.146 (模拟公网IP)，iptables -t nat -nvL，将来自10.1.0.0/16网段的报文的源地址改为公司的公网地址，这样内网的机器就能访问外网了，只是用于配置SNAT的话，我们并不用 手动的进行DNAT设置，iptables会自动维护NAT表，并将响应报文的目标地址转换回来。在内网的机器和外网的机器上分别抓包，内网显示的可以是外网的Ip地址返回，但是外网的机器上显示的是边界机器的外网地址，不会显示内网地址，起到了保护作用。再来看下DNAT，我们对外宣称，公司的公网IP上既提供了web服务，也提供了windows远程桌面，不管是访问web服务还是远程桌面，只要访问这个公网IP就行了，我们利用DNAT，将公网客户端发送过来的报文的目标地址与端口号做了映射，将访问web服务的报文转发到了内网中的C主机中，将访问远程桌面的报文转发到了内网中的D主机中iptables -t nat -F，iptables -t nat -I PREROUTING -d 192.168.1.146 -p tcp --dport 3389 -j DNAT --to-destination 10.1.0.6:3389，理论上只要完成上述DNAT配置规则即可，但是在测试时，只配置DNAT规则后，并不能正常DNAT，经过测试发现，将相应的SNAT规则同时配置后，即可正常DNAT，于是我们又配置了SNAT，同理3389是windows的远程访问端口，那么web服务，再加，iptables -t nat -I PREROUTING -d 192.168.1.146 -p tcp --dport 801 -j DNAT --to-destination 10.1.0.1:80

SNAT规则只配置在POSTROUTING链与INPUT链中

为什么SNAT规则必须定义在POSTROUTING链中，我们可以这样认为，POSTROUTING链是iptables中报文发出的最后一个”关卡”，我们应该在报文马上发出之前，修改报文的源地址，否则就再也没有机会修改报文的源地址了，在centos7中，SNAT规则也可以定义在INPUT链中，我们可以这样理解，发往本机的报文经过INPUT链以后报文就到达了本机，如果再不修改报文的源地址，就没有机会修改了

DNAT规则只配置在PREROUTING链与OUTPUT链中

15. action是MASQUERADE，与SNAT相似的动作，当我们拨号上网时，每次分配的IP地址往往不同，不会长期分给我们一个固定的IP地址，如果这时，我们想要让内网主机共享公网IP上网，就会很麻烦，因为每次IP地址发生变化以后，我们都要重新配置SNAT规则，通过MASQUERADE即可解决这个问题，MASQUERADE会动态的将源地址转换为可用的IP地址，其实与SNAT实现的功能完全一致，都是修改源地址，只不过SNAT需要指明将报文的源地址改为哪个IP，而MASQUERADE则不用指定明确的IP，会动态的将报文的源地址修改为指定网卡上可用的IP地址iptables -t nat -I POSTROUTING -s 10.1.0.0/16 -o eno50332184 -j MASQUERADE，我们指定，通过外网网卡出去的报文在经过POSTROUTING链时，会自动将报文的源地址修改为外网网卡上可用的IP地址，这时，即使外网网卡中的公网IP地址发生了改变，也能够正常的、动态的将内部主机的报文的源IP映射为对应的公网IP，可以把MASQUERADE理解为动态的、自动化的SNAT，如果没有动态SNAT的需求，没有必要使用MASQUERADE，因为SNAT更加高效

16. action是REDIRECT，使用REDIRECT动作可以在本机上进行端口映射，比如，将本机的80端口映射到本机的8080端口上，iptables -t nat -A PREROUTING -p tcp –dport 80 -j REDIRECT –to-ports 8080，经过上述规则映射后，当别的机器访问本机的80端口时，报文会被重定向到本机的8080端口上。REDIRECT规则只能定义在PREROUTING链或者OUTPUT链中。跟D类似

别人通过公网Ip访问你的服务就要做DNAT，你的内网要去访问公网服务就要做SNAT！！！

![icmp]({{ "/assets/img/network/icmp_type.png" | relative_url}})

`一些实战`

原始
iptables -I OUTPUT -j REJECT && iptables -I FORWARD -j REJECT && iptables -I INPUT -j REJECT && iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -I INPUT -p tcp --dport 22 -j ACCEPT
iptables -D OUTPUT -j REJECT && iptables -D FORWARD -j REJECT && iptables -D INPUT -j REJECT && iptables -D OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -D INPUT -p tcp --dport 22 -j ACCEPT

iptables -A INPUT -p tcp -m tcp --dport 6379 -j DROP
iptables -A INPUT -p tcp -m tcp --dport 6379 -j REJECT --reject-with icmp-port-unreachable
iptables -A INPUT -p tcp -m tcp --dport 6379 -j REJECT --reject-with tcp-reset

iptables -D INPUT -p tcp -m tcp --dport 6379 -j DROP
iptables -D INPUT -p tcp -m tcp --dport 6379 -j REJECT --reject-with icmp-port-unreachable
iptables -D INPUT -p tcp -m tcp --dport 6379 -j REJECT --reject-with tcp-reset

iptables -I INPUT -p tcp -m tcp --dport 6379 -j REJECT --reject-with icmp-port-unreachable


iptables -I OUTPUT -j REJECT --reject-with icmp-port-unreachable && iptables -I INPUT -j REJECT --reject-with icmp-port-unreachable && iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -I INPUT -p tcp --dport 22 -j ACCEPT
iptables -D INPUT -j REJECT --reject-with icmp-port-unreachable && iptables -D OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -D INPUT -p tcp --dport 22 -j ACCEPT

// 没有加上Output，就是返回Connection Refused，不会夯住(这边说的是telnet)，而ping的话也会有对应--reject-with的返回，不会夯住
iptables -I INPUT -j REJECT --reject-with icmp-port-unreachable && iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -I INPUT -p tcp --dport 22 -j ACCEPT

// 没有加上Output，但加上了Forward呢，也是Connection Refused，不会夯住
iptables -I INPUT -j REJECT --reject-with icmp-port-unreachable && iptables -I FORWARD -j REJECT --reject-with icmp-port-unreachable && iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -I INPUT -p tcp --dport 22 -j ACCEPT

// icmp-port-unreachable是都支持的，因为Output上有东西，所以不能返回Connection Refused，而是夯住
iptables -D OUTPUT -j REJECT --reject-with icmp-port-unreachable && iptables -D FORWARD -j REJECT --reject-with icmp-port-unreachable && iptables -D INPUT -j REJECT --reject-with icmp-port-unreachable && iptables -D OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -D INPUT -p tcp --dport 22 -j ACCEPT

// Forward不支持tcp-reset，改成tcp-reset下面的命令会报错
iptables -I OUTPUT -j REJECT --reject-with icmp-port-unreachable && iptables -I FORWARD -j REJECT --reject-with icmp-port-unreachable && iptables -I INPUT -j REJECT --reject-with icmp-port-unreachable && iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -I INPUT -p tcp --dport 22 -j ACCEPT

// Output也不支持tcp-reset
iptables -I OUTPUT -j REJECT --reject-with tcp-reset && iptables -I FORWARD -j REJECT --reject-with icmp-port-unreachable && iptables -I INPUT -j REJECT --reject-with tcp-reset && iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -I INPUT -p tcp --dport 22 -j ACCEPT

// 只有Input上面可以，直接挂掉了
iptables -I OUTPUT -j REJECT --reject-with icmp-port-unreachable && iptables -I FORWARD -j REJECT --reject-with icmp-port-unreachable && iptables -I INPUT -j REJECT --reject-with tcp-reset && iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -I INPUT -p tcp --dport 22 -j ACCEPT

如果是已存在的连接，会断吗（ESTABLISHED）

会断的，tcpdump抓包的时候发现没有 入包和出包了

iptables -I OUTPUT -j REJECT && iptables -I FORWARD -j REJECT && iptables -I INPUT -j REJECT && iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -I INPUT -p tcp --dport 22 -j ACCEPT && iptables -I OUTPUT -p icmp -j ACCEPT && iptables -I INPUT -p icmp -j ACCEPT

上面的语句会让icmp协议通过，也就是ping检测正常了，

// 解除
iptables -D OUTPUT -j REJECT && iptables -D FORWARD -j REJECT && iptables -D INPUT -j REJECT && iptables -D OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -D INPUT -p tcp --dport 22 -j ACCEPT && iptables -D OUTPUT -p icmp -j ACCEPT && iptables -D INPUT -p icmp -j ACCEPT

// 只对Input的icmp放行，还是会夯住
iptables -I OUTPUT -j REJECT && iptables -I FORWARD -j REJECT && iptables -I INPUT -j REJECT && iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -I INPUT -p tcp --dport 22 -j ACCEPT  && iptables -I INPUT -p icmp -j ACCEPT

// 只对Output的icmp放行，还是会夯住
iptables -I OUTPUT -j REJECT && iptables -I FORWARD -j REJECT && iptables -I INPUT -j REJECT && iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -I INPUT -p tcp --dport 22 -j ACCEPT && iptables -I OUTPUT -p icmp -j ACCEPT
报错是默认 Destination Port Unreachable，也就是总要有outport出去才行

但是让icmp的协议通过了，客户端telnet的返回也变成了Connection Refused了，而不是夯住了，nc命令也是，这个可能需要看一下了！！

icmp放通和nc，telnet的检测是有什么关系？个人猜测的话，先看下icmp的用途

在RFC，将ICMP 大致分成两种功能：差错通知和信息查询。[1]给送信者的错误通知；[2]送信者的信息查询。icmp是工作在网络层，ICMP 协议是为了辅助IP 协议，交换各种各样的控制信息而被制造出来的

[1]是到IP 数据包被对方的计算机处理的过程中，发生了什么错误时被使用。不仅传送发生了错误这个事实，也传送错误原因等消息。
[2]的信息询问是在送信方的计算机向对方计算机询问信息时被使用。被询问内容的种类非常丰富，他们有目标IP 地址的机器是否存在这种基本确认，调查自己网络的子网掩码，取得对方机器的时间信息等。

ICMP作为IP的上层协议在工作，ICMP 的内容是放在IP 数据包的数据部分里来互相交流的。也就是，从ICMP的报文格式来说，ICMP 是IP 的上层协议。但是，正如RFC 所记载的，ICMP 是分担了IP 的一部分功能。所以，被认为是与IP 同层的协议
更加详细地看一下数据包的格式吧。用来传送ICMP 报文的IP 数据包上实际上有不少字段。但是实际上与ICMP 协议相关的只有7 个子段。

1）协议；2）源IP 地址；3）目的IP 地址；4）生存时间；这四个包含在IP 首部的字段。

5）类型；6）代码；7）选项数据；这三个包含在ICMP数据部分的字段。

ping使用的是icmp协议，traceroute使用的是icmp协议，以Windows 上的tracert 命令为例看了原理，有些别的操作系统的traceroute 命令的原理略微不同。具体来说，也有用向目标发送UDP 数据包代替ICMP 回送请求报文来实现的。虽说是用UDP，但途中的路由器的处理与図 8完全相同。只是UDP 数据包到达目标后的处理不同。目标计算机突然收到与通信无关的数据包，就返回ICMP 错误，因此根据返回数据包的内容来判断命令的中止。

端口扫描也会用到icmp，端口扫描大致分为“UDP 的端口扫描”和“TCP 的端口扫描”两种。这里面，与ICMP 相关的是UDP一边。使用TCP 的通信，通信之前必定要先遵循三向握手的程序。因此，只要边错开端口号边尝试TCP连接就能调查端口的开闭。不特别需要ICMP。与此相对，UDP 没有这样的连接程序。因此，调查端口是否打开需要想点办法。
这样，被使用的是ICMP。根据ICMP 规格，UDP 数据包到达不存在的端口时，服务器需要返回ICMP 的“终点不可达”之一的“端口不可达”报文。

UDP 端口扫描一边一个一个错开端口号，一边持续着这个通信。这样，就知道了哪个端口是“好象开着的”了。但是，UDP 端口扫描与TCP 端口扫描有很大区别的地方。那就是，即使ICMP 端口不可达报文没有返回，也不能断定端口开着。端口扫描除了被管理员用来检查服务器上是否有开着的漏洞，作为黑客非法访问的事先调查，对服务器实施的情况也是很多的。需要非常小心地来使用。

telnet使用的是tcp协议，是不是检测有问题的时候，会返回icmp的回送回答（）

ping 基于ICMP协议是属于ip层协议，通信不需要端口所以无法测试 tcp udp 传输层的端口，tcping可以测试端口的

yum install tcping

tcping使用的是tcp协议

首先要明白两个概念：黑名单和白名单。

黑名单：把所有人当做好人，只拒绝坏人。
白名单：把所有人当做坏人，只放行好人。

看起来似乎白名单安全一点，黑名单接受的范围大一点。

对于iptables来说本来存在默认规则，通常默认规则设置为ACCEPT或者DROP。

如果默认规则是ACCEPT，那就是把所有人当做好人了，如此可以建立黑名单机制了。

如果默认规则是DROP，即是把所有人当做坏人，可以建立白名单机制。（最好不要这样做，原因后文会讲到）

如果是Drop的默认策略，不小心用了 iptables -F（iptables -F会清除掉除了默认规则外的所有规则），你就掉了，22也是默认被拒绝的，所以一般是用的ACCEPT，就是把所有人当做好人，只拒绝坏人，黑名单机制对吧！

我们也可以给他加一个条件，让他可以建立白名单机制。具体来看如何操作：

iptables -I INPUT -p tcp -m multiport --dports 22,80 -j ACCEPT
iptables -A INPUT -j REJECT // 就是兜底一个reject，其他都放行，相当于模拟白名单了

对于上面一段的总结：

iptables -I OUTPUT -p tcp -d 10.6.10.0/24 -j REJECT // telnet 10.6.10.232 8080 不通（很快返回telnet: connect to address 10.6.10.232: Connection refused） ping 10.6.10.232 可以
iptables -I OUTPUT -p icmp -d 10.6.10.0/24 -j REJECT // telnet 10.6.10.232 8080 通 ping 10.6.10.232 不可以

iptables -I OUTPUT -p tcp -d 10.6.10.0/24 --dport 6379 -j REJECT // 只对6379端口有效

iptables -I OUTPUT -j REJECT // 只加这个，直接断连了，ping不通。。显示的是 请求超时
iptables -I INPUT -j REJECT // 只加这个，直接断连了，ping不通。。显示的是 无法连到端口，跟上面还是有区别的，进去没进去是不一样的

再来

iptables -I OUTPUT -p tcp -d 10.12.0.0/17,10.12.128.0/19,10.50.0.0/19,10.52.0.0/14,10.56.0.0/13,10.96.0.0/15,10.98.0.0/16,10.232.48.0/20,100.69.192.0/20 --dport 6379 -j REJECT
iptables -I INPUT -p tcp -s 10.12.0.0/17,10.12.128.0/19,10.50.0.0/19,10.52.0.0/14,10.56.0.0/13,10.96.0.0/15,10.98.0.0/16,10.232.48.0/20,100.69.192.0/20 -j REJECT
iptables -I INPUT -p tcp --dport 22 -j ACCEPT

这样能满足出向只到6379不通，入向全拒绝，但放开22，22也是可以出的，因为出只拒绝6379

iptables -I OUTPUT -p tcp -d 10.12.0.0/17,10.12.128.0/19,10.50.0.0/19,10.52.0.0/14,10.56.0.0/13,10.96.0.0/15,10.98.0.0/16,10.232.48.0/20,100.69.192.0/20 -j REJECT
iptables -I INPUT -p tcp -s 10.12.0.0/17,10.12.128.0/19,10.50.0.0/19,10.52.0.0/14,10.56.0.0/13,10.96.0.0/15,10.98.0.0/16,10.232.48.0/20,100.69.192.0/20 -j REJECT
iptables -I INPUT -p tcp --dport 22 -j ACCEPT

这样出向所有都不通，但入向22进来的话（telnet），表现出的是：不是很快telnet: connect to address 10.6.10.232: Connection refused，而是会夯住等，注意和上面的比对，这个时候要加上
iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT
这样22也会通了


iptables -L INPUT --line-numbers

iptables -D INPUT 7


iptables -I OUTPUT -p tcp -d 10.12.0.0/17,10.12.128.0/19,10.50.0.0/19,10.52.0.0/14,10.56.0.0/13,10.96.0.0/15,10.98.0.0/16,10.232.48.0/20,100.69.192.0/20 -j DROP
iptables -I INPUT -p tcp -s 10.12.0.0/17,10.12.128.0/19,10.50.0.0/19,10.52.0.0/14,10.56.0.0/13,10.96.0.0/15,10.98.0.0/16,10.232.48.0/20,100.69.192.0/20 -j DROP
iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT
iptables -I INPUT -p tcp --dport 22 -j ACCEPT

iptables -I OUTPUT -p tcp -d 10.12.160.0/20,10.14.0.0/18,10.14.192.0/18,10.15.0.0/16,10.25.0.0/16,10.40.0.0/13,10.232.1.0/20,10.0.110.0/23,10.0.112.0/24,10.1.6.0/24,10.1.7.0/28,10.9.0.0/16,10.12.253.0/24,10.12.254.0/24,10.19.128.0/17,10.51.16.0/24,10.99.0.0/18,192.168.0.0/16 -j DROP
iptables -I INPUT -p tcp -d 10.12.160.0/20,10.14.0.0/18,10.14.192.0/18,10.15.0.0/16,10.25.0.0/16,10.40.0.0/13,10.232.1.0/20,10.0.110.0/23,10.0.112.0/24,10.1.6.0/24,10.1.7.0/28,10.9.0.0/16,10.12.253.0/24,10.12.254.0/24,10.19.128.0/17,10.51.16.0/24,10.99.0.0/18,192.168.0.0/16 -j DROP
iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT
iptables -I INPUT -p tcp --dport 22 -j ACCEPT


// 注入UY | HQ IDC
iptables -I OUTPUT -p tcp -d 10.12.0.0/17,10.12.128.0/19,10.50.0.0/19,10.52.0.0/14,10.56.0.0/13,10.96.0.0/15,10.98.0.0/16,10.232.48.0/20,100.69.192.0/20 -j DROP && iptables -I INPUT -p tcp -s 10.12.0.0/17,10.12.128.0/19,10.50.0.0/19,10.52.0.0/14,10.56.0.0/13,10.96.0.0/15,10.98.0.0/16,10.232.48.0/20,100.69.192.0/20 -j DROP && iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -I INPUT -p tcp --dport 22 -j ACCEPT && iptables -I INPUT -p icmp --icmp-type 8 -s 10.62.130.101 -j DROP
// 恢复UY | HQ IDC
iptables -D OUTPUT -p tcp -d 10.12.0.0/17,10.12.128.0/19,10.50.0.0/19,10.52.0.0/14,10.56.0.0/13,10.96.0.0/15,10.98.0.0/16,10.232.48.0/20,100.69.192.0/20 -j DROP && iptables -D INPUT -p tcp -s 10.12.0.0/17,10.12.128.0/19,10.50.0.0/19,10.52.0.0/14,10.56.0.0/13,10.96.0.0/15,10.98.0.0/16,10.232.48.0/20,100.69.192.0/20 -j DROP && iptables -D OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -D INPUT -p tcp --dport 22 -j ACCEPT && iptables -D INPUT -p icmp --icmp-type 8 -s 10.62.130.101 -j DROP

// 注入LB IDC
iptables -I OUTPUT -p tcp -d 10.12.160.0/20,10.14.0.0/18,10.14.192.0/18,10.15.0.0/16,10.25.0.0/16,10.40.0.0/13,10.232.1.0/20,10.0.110.0/23,10.0.112.0/24,10.1.6.0/24,10.1.7.0/28,10.9.0.0/16,10.12.253.0/24,10.12.254.0/24,10.19.128.0/17,10.51.16.0/24,10.99.0.0/18,192.168.0.0/16 -j DROP && iptables -I INPUT -p tcp -d 10.12.160.0/20,10.14.0.0/18,10.14.192.0/18,10.15.0.0/16,10.25.0.0/16,10.40.0.0/13,10.232.1.0/20,10.0.110.0/23,10.0.112.0/24,10.1.6.0/24,10.1.7.0/28,10.9.0.0/16,10.12.253.0/24,10.12.254.0/24,10.19.128.0/17,10.51.16.0/24,10.99.0.0/18,192.168.0.0/16 -j DROP && iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -I INPUT -p tcp --dport 22 -j ACCEPT && iptables -I INPUT -p icmp
// 恢复LB IDC
iptables -D OUTPUT -p tcp -d 10.12.160.0/20,10.14.0.0/18,10.14.192.0/18,10.15.0.0/16,10.25.0.0/16,10.40.0.0/13,10.232.1.0/20,10.0.110.0/23,10.0.112.0/24,10.1.6.0/24,10.1.7.0/28,10.9.0.0/16,10.12.253.0/24,10.12.254.0/24,10.19.128.0/17,10.51.16.0/24,10.99.0.0/18,192.168.0.0/16 -j DROP && iptables -D INPUT -p tcp -d 10.12.160.0/20,10.14.0.0/18,10.14.192.0/18,10.15.0.0/16,10.25.0.0/16,10.40.0.0/13,10.232.1.0/20,10.0.110.0/23,10.0.112.0/24,10.1.6.0/24,10.1.7.0/28,10.9.0.0/16,10.12.253.0/24,10.12.254.0/24,10.19.128.0/17,10.51.16.0/24,10.99.0.0/18,192.168.0.0/16 -j DROP && iptables -D OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -D INPUT -p tcp --dport 22 -j ACCEPT


再来，REJECT和DROP的区别，REJETC默认有with的，直接就是telnet: connect to address 10.6.10.232: Connection refused， DROP的话会夯住telnet: connect to address 10.6.10.232: Connection timed out

看哪种符合你的真实预期

再来，超时时间怎么设置？

SSH：
在下面的文件里面，
/etc/ssh/sshd_config
有下面的记述，
KeepAlive
ClientAliveInterval
ClientAlivecountMax

ssh连接超时问题解决方案：

1.修改server端的etc/ssh/sshd_config

ClientAliveInterval 60 ＃server每隔60秒发送一次请求给client，然后client响应，从而保持连接
ClientAliveCountMax 3 ＃server发出请求后，客户端没有响应得次数达到3，就自动断开连接，正常情况下，client不会不响应

2.修改client端的etc/ssh/ssh_config添加以下：（在没有权限改server配置的情形下，注意两个是不同的文件）

ServerAliveInterval 60 ＃client每隔60秒发送一次请求给server，然后server响应，从而保持连接
ServerAliveCountMax 3  ＃client发出请求后，服务器端没有响应得次数达到3，就自动断开连接，正常情况下，server不会不响应

3.另一种方式：

不修改配置文件

在命令参数里ssh -o ServerAliveInterval=60 这样子只会在需要的连接中保持持久连接， 毕竟不是所有连接都要保持持久的

4.可以编辑/etc/profile文件，在里面加上一句TMOUT=300之后重新登录就可以了.

HOSTNAME='/bin/hostname'
HISTIZE=30
TMOUT=300

# 将以上300修改为0就是设置不超时
source /etc/profile

从一些网页中得知可在一些特定文件中设置TMOUT变量来控制 telnet 的超时时间，也有网页中提到可以使用nc命令代替 telnet 实现同类效果并可以设置超时时间

telnet连接超时问题解决方案：

https://testerhome.com/topics/8859

127秒的返回

time telnet 123.59.208.201 62715

也就是大概会夯住2分多钟，然后返回 telnet: connect to address 10.6.10.232: Connection timed out


下面这两条策略，是不是可以只用一条就够了？

iptables -I OUTPUT -p tcp -d 10.12.0.0/17,10.12.128.0/19,10.50.0.0/19,10.52.0.0/14,10.56.0.0/13,10.96.0.0/15,10.98.0.0/16,10.232.48.0/20,100.69.192.0/20 -j DROP
iptables -I INPUT -p tcp -s 10.12.0.0/17,10.12.128.0/19,10.50.0.0/19,10.52.0.0/14,10.56.0.0/13,10.96.0.0/15,10.98.0.0/16,10.232.48.0/20,100.69.192.0/20 -j DROP

如果只有OUTPUT，本及telnet访问日版是不通的，日版telnet访问本机也是不通的，换个简单实验

10.2.85.27 上 telnet 10.6.10.232 8080
10.6.10.232 上 telnet 10.2.85.27 6379

10.2.85.27 上做一条 iptables -I OUTPUT -p tcp -d 10.6.10.232 -j DROP，那么上面两条命令都不行了，同样，换成入向的
10.2.85.27 上做一条 iptables -I INPUT -s tcp -d 10.6.10.232 -j DROP，上面两条命令也不行

tcp一定是双向建连，任意一侧不行就不行（telnet的探测用的是tcp），所以只做一条策略应该也能达到效果

再来，如果LB机器就是阻断LB的网络呢

iptables -I OUTPUT -p tcp -d 10.12.0.0/17,10.12.128.0/19,10.50.0.0/19,10.52.0.0/14,10.56.0.0/13,10.96.0.0/15,10.98.0.0/16,10.232.48.0/20,100.69.192.0/20 -j DROP
iptables -I INPUT -p tcp -s 10.12.0.0/17,10.12.128.0/19,10.50.0.0/19,10.52.0.0/14,10.56.0.0/13,10.96.0.0/15,10.98.0.0/16,10.232.48.0/20,100.69.192.0/20 -j DROP
iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT
iptables -I INPUT -p tcp --dport 22 -j ACCEPT

10.12.0.0/17,10.12.128.0/19,10.50.0.0/19,10.52.0.0/14,10.56.0.0/13,10.96.0.0/15,10.98.0.0/16,10.232.48.0/20,100.69.192.0/20

iptables -I OUTPUT -p tcp -d 10.12.0.0/17,10.12.128.0/19,10.50.0.0/19,10.52.0.0/14,10.56.0.0/13,10.96.0.0/15,10.98.0.0/16,10.232.48.0/20,100.69.192.0/20 -j DROP && iptables -I INPUT -p tcp -s 10.12.0.0/17,10.12.128.0/19,10.50.0.0/19,10.52.0.0/14,10.56.0.0/13,10.96.0.0/15,10.98.0.0/16,10.232.48.0/20,100.69.192.0/20 -j DROP && iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT && iptables -I INPUT -p tcp --dport 22 -j ACCEPT

显然对LB的所有访问会不通，但是对UY或者HQ还是通的

！！！！！聊聊模拟网络故障的问题，当然是iptables的：！！！！！

我们知道在TCP建立连接的时候有3次握手的规则

1. 客户端发送’SYN’给服务端

2. 服务端返回确认’SYN_ACK’给客户端

3. 客户端最终确定’ACK’

在这3次握手的时间内，每一次都有可能网络会掉包,我们分析一下每一种掉包的情况：

1. SYN丢失第一次握手客户端发送SYN掉包的情况：这种情况下，客户端发送的SYN丢失在网络中，没有得到确认，客户端的TCP会超时重发SYN。发送7个SYN后等待一个超时时间（例如：127秒），如果在这段时间内仍然没有收到ACK，则connect返回超时。

2. SYN-ACK丢失从客户端的角度来讲以前面一种情况类似。从服务端的角度来讲，由LISTEN状态进入SYN_REVD状态。服务端的TCP会重发SYN-ACK，直到超时。SYN攻击正是利用这一原理，攻击方伪造大量的SYN包发送到服务器，服务器对收到的SYN包不断回应SYN-ACK，直到超时。这会浪费服务器大量的资源，甚至导致奔溃。对服务端的应用层来讲，什么也没有发生。因为TCP只有在经过3次握手之后才回通知应用层，有新的连接到来。

3. ACK丢失这对服务端来讲与2相同。对于客户端来讲，由SYN_SENT状态进入了ESTABLISED状态，即连接成功了。连接成功后客户端就可以发送数据了。

但实际上数据是发送不到服务端的（我们假设客户端收到SYN-ACK之后，客户端与服务端之间的网络就断开了），客户端发送出去的数据得不到确认，一般重发3次左右就会处于等待ACK的状态（win7）。而ubuntu 12.10下，调用send会返回成功，直到TCP的缓冲被填满（测试环境：局域网，感觉这个不是很合理，按照书上所说：应该是使用“指数退避”进行重传 -- TCP/IP协议详解，大概是我的测试环境中有NAT所致

吧）。最终，客户端产生一个复位信号并终止连接。返回给应用程序的结果是Connection time out（errno: 110）

我们可以在端口上控制服务端无法与客户端握手成功来让超时的情况发生

具体的实现要用到 iptables 这个命令

iptables -A OUTPUT -p tcp -m tcp --tcp-flags SYN SYN --sport 9090 -j DROP

这个命令是用来drop 掉响应SYN的返回，之前我们看到第一次客服端向服务器请求SYN的握手信息，而这个命令就是阻止服务器返回SYN_ACK的确认握手信息，这样客户端就无法收到服务端的握手确认信息了.

下面还有一种情况，就是连接已经成功了，但是在传输数据的时候，服务端没有及时返回数据，我们来看看这种情况是如何模拟的：

iptables -A OUTPUT -p tcp -m tcp --tcp-flags PSH PSH --sport 9090 -j DROP

这里用到的flags 是PSH ,对，PSH的意思是控制信息是可以正常传送的，也就是说握手是正常成功的，然后传输数据的时候，我们限制了服务器无法给客户端传送数据内容，这样就模拟了连接是成功的，但是无法正常读取到服务端的数据的超时情况了

HttpClient有三个超时(connectionRequestTimeout,connectTimeout,socketTimeout)

1. connectTimeOut：指建立连接的超时时间，比较容易理解
2. connectionRequestTimeOut：指从连接池获取到连接的超时时间，如果是非连接池的话，该参数暂时没有发现有什么用处
3. socketTimeOut：指客户端和服务进行数据交互的时间，是指两者之间如果两个数据包之间的时间大于该时间则认为超时，而不是整个交互的整体时间，
比如如果设置1秒超时，如果每隔0.8秒传输一次数据，传输10次，总共8秒，这样是不超时的。而如果任意两个数据包之间的时间超过了1秒，则超时。

https://blog.csdn.net/z69183787/article/details/78039601

http://www.baeldung.com/httpclient-connection-management



### 十一、linux中的基本运维

a.linux开启内核转发

```html
# sysctl -w net.ipv4.ip_forward=1  临时设置
# vi /etc/sysctl.conf  设置net.ipv4.ip_forward=1，这是永久设置
```

b.command not found

linux经常会出现command not found，有时候并不是没有这个cli，可能环境变量导致

```html
# echo $PATH
```
看下现在的path是什么，有时候会漏掉`/usr/local/bin`，而某些cli会放在这个地方，临时修改下PATH


```html
# export PATH=$PATH:/usr/local/bin
```

但不永久，每个用户根目录下有`~/.bash_profile`文件，可以将上述命令写在末尾，还有全局型，即每个用户都能使用的在`/etc/profile`下面，修改完了记得`source`下文件生效

c.locate的使用

locate(locate) 命令用来查找文件或目录。 locate命令要比find -name快得多，原因在于它不搜索具体目录，而是搜索一个数据库/var/lib/mlocate/mlocate.db 。这个数据库中含有本地所有文件信息。Linux系统自动创建这个数据库，并且每天自动更新一次，因此，我们在用whereis和locate 查找文件时，有时会找到已经被删除的数据，或者刚刚建立文件，却无法查找到，原因就是因为数据库文件没有被更新。为了避免这种情况，可以在使用locate之前，先使用updatedb命令，手动更新数据库。整个locate工作其实是由四部分组成的:
1. /usr/bin/updatedb 主要用来更新数据库，通过crontab自动完成的
2. /usr/bin/locate 查询文件位置
3. /etc/updatedb.conf updatedb的配置文件
4. /var/lib/mlocate/mlocate.db 存放文件信息的文件

d.查看CPU和MEMORY的信息

物理CPU的个数，物理CPU就是计算机上实际配置的CPU个数，其中的physical id就是每个物理CPU的ID

```html
# cat /proc/cpuinfo | grep "physical id" | sort | uniq | wc -l
```
内存总量

```html
# cat /proc/meminfo | grep MemTotal
```

cpu cores数量，也是物理的

核数就是指CPU上集中的处理数据的cpu核心个数，单核指cpu核心数一个，双核则指的是两个。通常每个CPU下的核数都是固定的，比如你的计算机有两个物理CPU，每个CPU是双核，那么计算机就是四核的

linux的cpu核心总数也可以在/proc/cpuinfo里面通过指令cat /proc/cpuinfo查看的到，其中的core id指的是每个物理CPU下的cpu核的id，能找到几个core id就代表你的计算机有几个核心

```html
# cat /proc/cpuinfo | grep "cpu cores" | uniq | awk {'print $4'}
```

是这样，cpu cores有几个 core id就顺着排下来，这边的统计还是看下文件后再决定

逻辑CPU（这层才涉及到逻辑的概念，超线程）操作系统可以使用逻辑CPU来模拟出真实CPU的效果。在之前没有多核处理器的时候，一个CPU只有一个核，而现在有了多核技术，其效果就好像把多个CPU集中在一个CPU上。当计算机没有开启超线程时，逻辑CPU的个数就是计算机的核数。而当超线程开启后，逻辑CPU的个数是核数的两倍。实际上逻辑CPU的数量就是平时称呼的几核几线程中的线程数量，在linux的cpuinfo中逻辑CPU数就是processor的数量

```html

cat /proc/cpuinfo | grep "processor" | wc -l

```

知道上面这些，常说的几核几线程就好理解了。假设计算机有一个物理CPU，是双核的，支持超线程。那么这台计算机就是双核四线程的。
所以两路（两路指的是有两个物理CPU）四核超线程就有2*4*2=16个逻辑CPU。有人也把它称之为16核，实际上在linux的/proc/cpuinfo中查看只有8核。

既然计算机多核与超线程模拟相关，所以实际上计算机的核数翻倍并不意味着性能的翻倍，也不意味着核数越多计算机性能会越来越好，因为超线程只是充分利用了CPU的空闲资源，实际上在应用中基于很多原因，CPU的执行单元都没有被充分使用

自己用过的物理机

```html

processor有32个，出了两个physical id的就是两路，核数是8核，支持超分的话就是 2*8*2=32个逻辑CPU

```


K8S里面node信息的cpu数量应该是cpuinfo里面processor的数量

```html
# cat /proc/cpuinfo | grep "physical id" | wc -l
```

补充：

```html

processor　：系统中逻辑处理核的编号。对于单核处理器，则课认为是其CPU编号，对于多核处理器则可以是物理核、或者使用超线程技术虚拟的逻辑核
vendor_id　：CPU制造商
cpu family　：CPU产品系列代号
model　　　：CPU属于其系列中的哪一代的代号
model name：CPU属于的名字及其编号、标称主频
stepping　  ：CPU属于制作更新版本

microcode  : 微码　(详解见: http://blog.csdn.net/zhl1224/article/details/5836035)
cpu MHz　  ：CPU的实际使用主频
cache size   ：CPU二级缓存大小
physical id   ：单个CPU的标号
siblings       ：单个CPU逻辑的核数
core id        ：当前物理核在其所处CPU中的编号，这个编号不一定连续
cpu cores    ：该逻辑核所处CPU的物理核数
apicid          ：用来区分不同逻辑核的编号，系统中每个逻辑核的此编号必然不同，此编号不一定连续
fpu             ：是否具有浮点运算单元（Floating Point Unit）
fpu_exception  ：是否支持浮点计算异常
cpuid level   ：执行cpuid指令前，eax寄存器中的值，根据不同的值cpuid指令会返回不同的内容
wp             ：表明当前CPU是否在内核态支持对用户空间的写保护（Write Protection）
flags          ：当前CPU支持的功能
bogomips   ：在系统内核启动时粗略测算的CPU速度（Million Instructions Per Second）
clflush size  ：每次刷新缓存的大小单位
cache_alignment ：缓存地址对齐单位
address sizes     ：可访问地址空间位数
power management ：对能源管理的支持，有以下几个可选支持功能

```

/proc/cpuinfo 描述中有 6 个条目适用于多内核和超线程（HT）技术检查：processor, vendor id, physical id, siblings, core id 和 cpu cores。

processor 条目包括这一逻辑处理器的唯一标识符。
physical id 条目包括每个物理封装的唯一标识符。
core id 条目保存每个内核的唯一标识符。
siblings 条目列出了位于相同物理封装中的逻辑处理器的数量。
cpu cores 条目包含位于相同物理封装中的内核数量。

通过下面信息可以知道我的电脑只有一个cpu，该cpu里面有2个物理内核，4个逻辑内核。应用了超线程技术，使系统将每个内核当做两个内核使用。

其实只要 物理CPU数 * 核数 不等于 processor数，就用了超线程技术（HT-Hyper-Threading），一般是两倍

简便的判断方法：

如果“siblings”和“cpu cores”一致，则说明不支持超线程，或者超线程未打开。

如果“siblings”是“cpu cores”的两倍，则说明支持超线程，并且超线程已打开。

top 然后 1出来的是 processor

补充下，其实按照下面的三个说法就是正确的了，到底机器是多少C的

$ grep "model name" /proc/cpuinfo | wc -l  // 逻辑cpu数量

$ lscpu | grep '^CPU(s)'  // 逻辑cpu数量

或者执行top命令，然后按数字1，可以看到Cpu0~CpuN，共N+1个CPU。


e.linux如何查看开机自启动的服务

```html
ntsysv
```
这个命令会弹出一个界面

f.linux中lsof命令的使用

lsof命令列出所有打开的文件，一些参数解释如下

1. FD代表文件描述符，CWD代表当前工作目录，RTD代表根目录，TXT代表文本程序，还有1u，3r，4w
2. TYPE代表文件和它的鉴定，DIR目录，REG代表常规文件，CHR字符特殊文件，FIFO先进先出

// 不同的用户出来数据是不一样的，列出当前用户的！！！

列出用户特定的已打开文件

```html
lsof -u qiming
```
查找在特定端口上运行的进程

```html
lsof -i TCP:22
```
仅列出IPv4和IPv6打开的文件

```html
lsof -i 4
lsof -i 6
```
列出TCP端口范围1-1024的打开文件

```html
lsof -i TCP:1-1024
```
使用^字符排除用户

```html
lsof -i -u^qiming
```
找出谁在看什么文件和命令

```html
lsof -i -u qiming
```
列出所有网络连接
```html
lsof -i
```
终止特定用户的所有活动

```html
kill -9 `lsof -t -u qiming`
```

g.基本的cp和scp命令

cp命令的参数

-a或--archive 　此参数的效果和同时指定"-dpR"参数相同。
-b或--backup 　删除，覆盖目标文件之前的备份，备份文件会在字尾加上一个备份字符串。
-d或--no-dereference 　当复制符号连接时，把目标文件或目录也建立为符号连接，并指向与源文件或目录连接的原始文件或目录。
-f或--force 　强行复制文件或目录，不论目标文件或目录是否已存在。
-i或--interactive 　覆盖既有文件之前先询问用户。
-l或--link 　对源文件建立硬连接，而非复制文件。
-p或--preserve 　保留源文件或目录的属性。
-P或--parents 　保留源文件或目录的路径。
-r 递归处理，将指定目录下的文件与子目录一并处理。
-R或--recursive 　递归处理，将指定目录下的所有文件与子目录一并处理。
-s或--symbolic-link 　对源文件建立符号连接，而非复制文件。
-S<备份字尾字符串>或--suffix=<备份字尾字符串> 　用"-b"参数备份目标文件后，备份文件的字尾会被加上一个备份字符串，预设的备份字尾字符串是符号"~"。
-u或--update 　使用这项参数后只会在源文件的更改时间较目标文件更新时或是　名称相互对应的目标文件并不存在，才复制文件。
-v或--verbose 　显示指令执行过程。
-V<备份方式>或--version-control=<备份方式> 　用"-b"参数备份目标文件后，备份文件的字尾会被加上一个备份字符串，这字符串不仅可用"-S"参数变更，当使用"-V"参数指定不同备份方式时，也会产生不同字尾的备份字串。  
-x或--one-file-system 　复制的文件或目录存放的文件系统，必须与cp指令执行时所处的文件系统相同，否则不予复制。
--help 　在线帮助。
--sparse=<使用时机> 　设置保存稀疏文件的时机。
--version 　显示版本信息

这里要注意的是其实cp加上-f也是没有效果的，因为是有别名，`alias cp='cp -i'`，一个是去掉别名就可以了，另一个是用以下命令

```html
\cp * -rf ../../test
```

scp中是没有-f只说的，都是直接覆盖了

h.top命令

输入top后，如果打一个1，就会看到各个逻辑CPU的信息，这个逻辑CPU就是 /proc/cpuinfo中的内容

承接上面d的内容，这个出来的条目数就是/proc/cpuinfo中的processor的个数

Top相关

```html

top - 01:06:48 up  1:22,  1 user,  load average: 0.06, 0.60, 0.48
Tasks:  29 total,   1 running,  28 sleeping,   0 stopped,   0 zombie
Cpu(s):  0.3% us,  1.0% sy,  0.0% ni, 98.7% id,  0.0% wa,  0.0% hi,  0.0% si
Mem:    191272k total,   173656k used,    17616k free,    22052k buffers
Swap:   192772k total,        0k used,   192772k free,   123988k cached

PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
1379 root      16   0  7976 2456 1980 S  0.7  1.3   0:11.03 sshd
14704 root      16   0  2128  980  796 R  0.7  0.5   0:02.72 top
1 root      16   0  1992  632  544 S  0.0  0.3   0:00.90 init
2 root      34  19     0    0    0 S  0.0  0.0   0:00.00 ksoftirqd/0
3 root      RT   0     0    0    0 S  0.0  0.0   0:00.00 watchdog/0

统计信息区前五行是系统整体的统计信息。第一行是任务队列信息，同 uptime 命令的执行结果。其内容如下：

01:06:48    当前时间
up 1:22    系统运行时间，格式为时:分
1 user    当前登录用户数
load average: 0.06, 0.60, 0.48    系统负载，即任务队列的平均长度。三个数值分别为 1分钟、5分钟、15分钟前到现在的平均值。

第二、三行为进程和CPU的信息。当有多个CPU时，这些内容可能会超过两行（按1）。内容如下：

total 进程总数
running 正在运行的进程数
sleeping 睡眠的进程数
stopped 停止的进程数
zombie 僵尸进程数
Cpu(s):
0.3% us 用户空间占用CPU百分比
1.0% sy 内核空间占用CPU百分比
0.0% ni 用户进程空间内改变过优先级的进程占用CPU百分比
98.7% id 空闲CPU百分比
0.0% wa 等待输入输出的CPU时间百分比
0.0%hi：硬件CPU中断占用百分比
0.0%si：软中断占用百分比
0.0%st：虚拟机占用百分比

最后两行为内存信息。内容如下：

Mem:
191272k total    物理内存总量
173656k used    使用的物理内存总量
17616k free    空闲内存总量
22052k buffers    用作内核缓存的内存量
Swap:
192772k total    交换区总量
0k used    使用的交换区总量
192772k free    空闲交换区总量
123988k cached    缓冲的交换区总量,内存中的内容被换出到交换区，而后又被换入到内存，但使用过的交换区尚未被覆盖，该数值即为这些内容已存在于内存中的交换区的大小,相应的内存再次被换出时可不必再对交换区写入。


进程信息区统计信息区域的下方显示了各个进程的详细信息。首先来认识一下各列的含义。

序号  列名    含义
a    PID     进程id
b    PPID    父进程id
c    RUSER   Real user name
d    UID     进程所有者的用户id
e    USER    进程所有者的用户名
f    GROUP   进程所有者的组名
g    TTY     启动进程的终端名。不是从终端启动的进程则显示为 ?
h    PR      优先级
i    NI      nice值。负值表示高优先级，正值表示低优先级
j    P       最后使用的CPU，仅在多CPU环境下有意义
k    %CPU    上次更新到现在的CPU时间占用百分比
l    TIME    进程使用的CPU时间总计，单位秒
m    TIME+   进程使用的CPU时间总计，单位1/100秒
n    %MEM    进程使用的物理内存百分比
o    VIRT    进程使用的虚拟内存总量，单位kb。VIRT=SWAP+RES
p    SWAP    进程使用的虚拟内存中，被换出的大小，单位kb。
q    RES     进程使用的、未被换出的物理内存大小，单位kb。RES=CODE+DATA
r    CODE    可执行代码占用的物理内存大小，单位kb
s    DATA    可执行代码以外的部分(数据段+栈)占用的物理内存大小，单位kb
t    SHR     共享内存大小，单位kb
u    nFLT    页面错误次数
v    nDRT    最后一次写入到现在，被修改过的页面数。
w    S       进程状态(D=不可中断的睡眠状态,R=运行,S=睡眠,T=跟踪/停止,Z=僵尸进程)
x    COMMAND 命令名/命令行
y    WCHAN   若该进程在睡眠，则显示睡眠中的系统函数名
z    Flags   任务标志，参考 sched.h


更改显示内容通过 f 键可以选择显示的内容。按 f 键之后会显示列的列表，按 a-z 即可显示或隐藏对应的列，最后按回车键确定。
按 o 键可以改变列的显示顺序。按小写的 a-z 可以将相应的列向右移动，而大写的 A-Z 可以将相应的列向左移动。最后按回车键确定。
按大写的 F 或 O 键，然后按 a-z 可以将进程按照相应的列进行排序。而大写的 R 键可以将当前的排序倒转。


top的一些参数

d 指定每两次屏幕信息刷新之间的时间间隔。当然用户可以使用s交互命令来改变之。
p 通过指定监控进程ID来仅仅监控某个进程的状态。
q 该选项将使top没有任何延迟的进行刷新。如果调用程序有超级用户权限，那么top将以尽可能高的优先级运行。
S 指定累计模式
s 使top命令在安全模式中运行。这将去除交互命令所带来的潜在危险。
i 使top不显示任何闲置或者僵死进程。
c 显示整个命令行而不只是显示命令名


Ctrl+L 擦除并且重写屏幕。
h或者? 显示帮助画面，给出一些简短的命令总结说明。
k       终止一个进程。系统将提示用户输入需要终止的进程PID，以及需要发送给该进程什么样的信号。一般的终止进程可以使用15信号；如果不能正常结束那就使用信号9强制结束该进程。默认值是信号15。在安全模式中此命令被屏蔽。
i 忽略闲置和僵死进程。这是一个开关式命令。
q 退出程序。
r 重新安排一个进程的优先级别。系统提示用户输入需要改变的进程PID以及需要设置的进程优先级值。输入一个正值将使优先级降低，反之则可以使该进程拥有更高的优先权。默认值是10。
S 切换到累计模式。
s 改变两次刷新之间的延迟时间。系统将提示用户输入新的时间，单位为s。如果有小数，就换算成m s。输入0值则系统将不断刷新，默认值是5 s。需要注意的是如果设置太小的时间，很可能会引起不断刷新，从而根本来不及看清显示的情况，而且系统负载也会大大增加。
f或者F 从当前显示中添加或者删除项目。
o或者O 改变显示项目的顺序。
l 切换显示平均负载和启动时间信息。
m 切换显示内存信息。
t 切换显示进程和CPU状态信息。
c 切换显示命令名称和完整命令行。
M 根据驻留内存大小进行排序。
P 根据CPU使用百分比大小进行排序。
T 根据时间/累计时间进行排序。
W 将当前设置写入~/.toprc文件中。这是写top配置文件的推荐方法。

例子

top   //每隔5秒显式所有进程的资源占用情况
top -d 2  //每隔2秒显式所有进程的资源占用情况
top -c  //每隔5秒显式进程的资源占用情况，并显示进程的命令行参数(默认只有进程名)
top -p 12345 -p 6789//每隔5秒显示pid是12345和pid是6789的两个进程的资源占用情况
top -d 2 -c -p 123456 //每隔2秒显示pid是12345的进程的资源使用情况，并显式该进程启动的命令行参数

```

i.一些常看应用属性的方法

有的时候要记得加上sudo

```html
netstat -anp | grep 端口号

lsof -i:端口号

ps -aux | grep XXX 就是很快实现进程名和进程号的互查，这可以说是第一步

根据进程pid查端口

lsof -i | grep Pid

也可以

netstat -anp | grep Pid

这是第二步

另外

[root@Linux ~]# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.1  0.1   2032   644 ?        Ss   21:55   0:01 init [3]
root         2  0.0  0.0      0     0 ?        S    21:55   0:00 [migration/0]
root         3  0.0  0.0      0     0 ?        SN   21:55   0:00 [ksoftirqd/0]

解释如下：
VSZ–进程的虚拟大小
RSS–驻留集的大小，可以理解为内存中页的数量
TTY–控制终端的ID
STAT–也就是当前进程的状态，其中S-睡眠，s-表示该进程是会话的先导进程，N-表示进程拥有比普通优先级更低的优先级，R-正在运行，D-短期等待，Z-僵死进程，T-被跟踪或者被停止等等
STRAT–这个很简单，就是该进程启动的时间
TIME–进程已经消耗的CPU时间，注意是消耗CPU的时间
COMMOND–命令的名称和参数

```

j.查看系统的开机，关机时间

```html
last命令

查看最近一次开机时间
who -b
last -1 reboot

查看关机记录
last -x | grep shutdown

查看历史的启动时间
last reboot

查看系统从上次开机到现在已经运行多久了
uptime 或者 w
```

k.设置系统ssh超时时间

```html
vim /etc/ssh/sshd_config
ClientAliveInterval 30     每30秒往客户端发送会话请求，保持连接
ClientAliveCountMax 3      3表示重连3次失败后，重启SSH会话



ssh连接超时问题解决方案：

1.修改server端的etc/ssh/sshd_config

ClientAliveInterval 60 ＃server每隔60秒发送一次请求给client，然后client响应，从而保持连接

ClientAliveCountMax 3 ＃server发出请求后，客户端没有响应得次数达到3，就自动断开连接，正常情况下，client不会不响应


这边展开说下：

第一种：

ClientAliveInterval 10m          # 10 minutes
ClientAliveCountMax 2           # 2 次

第二种：


ClientAliveInterval 20m          # 20 minutes
ClientAliveCountMax 0            # 0 次

对于第一种方法，sshd将通过加密通道发送消息，此处称为Client Alive Messages，如果客户端处于非活动状态10分钟，则从客户端请求响应。 sshd守护程序将最多发送这些消息两次。 如果在发送客户端活动消息时达到此阈值，sshd将断开客户端的连接。对于第二种方法，如果客户端处于非活动状态20分钟，sshd将不会发送客户端活动消息并直接终止会话。


2.修改client端的etc/ssh/ssh_config添加以下：（在没有权限改server配置的情形下）

ServerAliveInterval 60 ＃client每隔60秒发送一次请求给server，然后server响应，从而保持连接

ServerAliveCountMax 3  ＃client发出请求后，服务器端没有响应得次数达到3，就自动断开连接，正常情况下，server不会不响应



3.另一种方式：

不修改配置文件

在命令参数里ssh -o ServerAliveInterval=60 这样子只会在需要的连接中保持持久连接， 毕竟不是所有连接都要保持持久的


还有直接修改超时时间
修改/etc/profile下tmout参数
如：sudo vi /etc/profile
tmout=300
就是300秒后闲置自动退出；

在未设置TMOUT或者设置TMOUT=0时，此闲置超时自动退出的功能禁用。

这个很简单。

```

l.netstat的使用

```html
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address Foreign Address State
tcp 0 2 210.34.6.89:telnet 210.34.6.96:2873 ESTABLISHED
tcp 296 0 210.34.6.89:1165 210.34.6.84:netbios-ssn ESTABLISHED
tcp 0 0 localhost.localdom:9001 localhost.localdom:1162 ESTABLISHED
tcp 0 0 localhost.localdom:1162 localhost.localdom:9001 ESTABLISHED
tcp 0 80 210.34.6.89:1161 210.34.6.10:netbios-ssn CLOSE

Active UNIX domain sockets (w/o servers)
Proto RefCnt Flags Type State I-Node Path
unix 1 [ ] STREAM CONNECTED 16178 @000000dd
unix 1 [ ] STREAM CONNECTED 16176 @000000dc
unix 9 [ ] DGRAM 5292 /dev/log
unix 1 [ ] STREAM CONNECTED 16182 @000000df

从整体上看，netstat的输出结果可以分为两个部分：

一个是Active Internet connections，称为有源TCP连接，其中"Recv-Q"和"Send-Q"指%0A的是接收队列和发送队列。这些数字一般都应该是0。如果不是则表示软件包正在队列中堆积。这种情况只能在非常少的情况见到。

另一个是Active UNIX domain sockets，称为有源Unix域套接口(和网络套接字一样，但是只能用于本机通信，性能可以提高一倍)。
Proto显示连接使用的协议,RefCnt表示连接到本套接口上的进程号,Types显示套接口的类型,State显示套接口当前的状态,Path表示连接到套接口的其它进程使用的路径名。

-a (all)显示所有选项，默认不显示LISTEN相关
-t (tcp)仅显示tcp相关选项
-u (udp)仅显示udp相关选项
-n 拒绝显示别名，能显示数字的全部转化成数字。
-l 仅列出有在 Listen (监听) 的服務状态（对于还没有建立完整连接的服务来说）

-p 显示建立相关链接的程序名
-r 显示路由信息，路由表
-e 显示扩展信息，例如uid等
-s 按各个协议进行统计
-c 每隔一个固定时间，执行该netstat命令。（观测指标，每隔一秒执行一次）
-e unix下一切皆文件，那么这个连接的iNode是多少呢？借助-e(extend)参数可以看到这些信息
-o 查看和连接相关定时器信息（Timer中，keepalive (18.69/0/0) keepalive的时间计时，on (19.97/7/0) 重发的时间计时，off 没有时间计时，timewait 等待时间计时）。定时器的理解需要对TCP协议有较多理解

提示：LISTEN和LISTENING的状态只有用-a或者-l才能看到

查看连接某服务端口最多的的IP地址
netstat -nat | grep "192.168.1.15:22" |awk '{print $5}'|awk -F: '{print $1}'|sort|uniq -c|sort -nr|head -20

TCP各种状态列表
netstat -nat |awk '{print $6}'
先把状态全都取出来,然后使用uniq -c统计，之后再进行排序。
netstat -nat |awk '{print $6}'|sort|uniq -c

分析access.log获得访问前10位的ip地址
awk '{print $1}' access.log |sort|uniq -c|sort -nr|head -10

查看连接某服务端口最多的的IP地址：
netstat -ntu | grep :80 | awk '{print $5}' | cut -d: -f1 | awk '{++ip[$1]} END {for(i in ip) print ip[i],"\t",i}' | sort -nr

TCP各种状态列表：
netstat -nt | grep -e 127.0.0.1 -e 0.0.0.0 -e ::: -v | awk '/^tcp/ {++state[$NF]} END {for(i in state) print i,"\t",state[i]}'

```

这边涉及到一个netstat 出来的状态 state

ESTABLISHED 指TCP连接已建立，双方可以进行方向数据传递

CLOSE_WAIT: 这种状态的含义其实是表示在等待关闭。当对方close一个SOCKET后发送FIN报文给自己，你系统毫无疑问地会回应一个ACK报文给对方，此时则进入到CLOSE_WAIT状态。接下来呢，实际上你真正需要考虑的事情是察看你是否还有数据发送给对方，如果没有的话， 那么你也就可以close 这个SOCKET，发送 FIN 报文给对方，也即关闭连接。所以你在CLOSE_WAIT 状态下，需要完成的事情是等待你去关闭连接。

LISTENING: 指TCP正在监听端口，可以接受连接

TIME_WAIT: 指连接已准备关闭。表示收到了对方的FIN报文，并发送出了ACK报文，就等2MSL后即可回到CLOSED可用状态了。如果FIN_WAIT_1状态下，收到了对方同时带FIN标志和ACK标志的报文时，可以直接进入到TIME_WAIT状态，而无须经过FIN_WAIT_2 状态。

FIN_WAIT_1: 这个状态要好好解释一下，其实FIN_WAIT_1和 FIN_WAIT_2状态的真正含义都是表示等待对方的FIN报 文。而这两种状态的区别是：FIN_WAIT_1状态实际上是当SOCKET在ESTABLISHED状态时，它想主动关闭连接，向对方发送了FIN 报文，此时该SOCKET即进入到FIN_WAIT_1 状态。而当对方回应ACK 报文后，则进入到FIN_WAIT_2状态，当然在实际的正常情况 下，无论对方何种情况下，都应该马上回应ACK报文，所以FIN_WAIT_1状态一般是比较难见到的，而FIN_WAIT_2 状态还有时常常可以用 netstat看到。
FIN_WAIT_2：上面已经详细解释了这种状态，实际上FIN_WAIT_2 状态下的SOCKET，表示半连接，也即有一方要求close 连接，但另外还告诉对方，我暂时还有点数据需要传送给你，稍后再关闭连接。

LAST_ACK: 是被动关闭一方在发送FIN报文后，最后等待对方的ACK报文。当收到ACK报文后，也即可以进入到CLOSED可用状态了

SYNC_RECEIVED: 收到对方的连接建立请求,

这个状态表示接受到了SYN报文，在正常情况下，这个状态是服务器端的SOCKET在建立TCP连接时的三次握手会话过程中的一个中间状态，很短暂，基本上用netstat你是很难看到这种状态的，除非你特意写了一个客户端测试程序，故意将三次TCP握手过程中最后一个ACK报文不予发送。因此这种状态时，当收到客户端的ACK报文后，它会进入到ESTABLISHED状态。

SYNC_SEND: 已经主动发出连接建立请求。与SYN_RCVD遥想呼应，当客户端SOCKET执行CONNECT连接时，它首先发送SYN报文，因此也随即它会进入到了SYN_SENT状态，并等待服务端的发送三次握手中的第2个报文。


调大tcp的连接数，和消亡时间

TCP连接断开后，会以TIME_WAIT状态保留一定的时间，然后才会释放端口。当并发请求过多的时候，就会产生大量的TIME_WAIT状态 的连接，无法及时断开的话，会占用大量的端口资源和服务器资源。这个时候我们可以优化TCP的内核参数，来及时将TIME_WAIT状态的端口清理掉

```html
netstat -n | awk ‘/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}’
这个命令会输出类似下面的结果：
LAST_ACK 16
SYN_RECV 348
ESTABLISHED 70
FIN_WAIT1 229
FIN_WAIT2 30
CLOSING 33
TIME_WAIT 18098
```

我们只用关心TIME_WAIT的个数，在这里可以看到，有18000多个TIME_WAIT，这样就占用了18000多个端口。要知道端口的数量只有 65535个，占用一个少一个，会严重的影响到后继的新连接。这种情况下，我们就有必要调整下Linux的TCP内核参数，让系统更快的释放 TIME_WAIT连接。

在这个文件中，加入下面的几行内容：
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_fin_timeout = 30

输入下面的命令，让内核参数生效：#sysctl -p

简单的说明上面的参数的含义：

net.ipv4.tcp_syncookies = 1

表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；

net.ipv4.tcp_tw_reuse = 1

表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；

net.ipv4.tcp_tw_recycle = 1

表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭；

net.ipv4.tcp_fin_timeout

修改系?默认的 TIMEOUT 时间。

在经过这样的调整之后，除了会进一步提升服务器的负载能力之外，还能够防御小流量程度的DoS、CC和SYN攻击。

此外，如果你的连接数本身就很多，我们可以再优化一下TCP的可使用端口范围，进一步提升服务器的并发能力。依然是往上面的参数文件中，加入下面这些配置：
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.ip_local_port_range = 10000 65000
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_tw_buckets = 5000
这几个参数，建议只在流量非常大的服务器上开启，会有显著的效果。一般的流量小的服务器上，没有必要去设置这几个参数。

net.ipv4.tcp_keepalive_time = 1200

表示当keepalive起用的时候，TCP发送keepalive消息的频度。缺省是2小时，改为20分钟。

net.ipv4.ip_local_port_range = 10000 65000

表示用于向外连接的端口范围。缺省情况下很小：32768到61000，改为10000到65000。（注意：这里不要将最低值设的太低，否则可能会占用掉正常的端口！）

net.ipv4.tcp_max_syn_backlog = 8192

表示SYN队列的长度，默认为1024，加大队列长度为8192，可以容纳更多等待连接的网络连接数。

net.ipv4.tcp_max_tw_buckets = 6000

表示系统同时保持TIME_WAIT的最大数量，如果超过这个数字，TIME_WAIT将立刻被清除并打印警告信息。默 认为180000，改为6000。对于Apache、Nginx等服务器，上几行的参数可以很好地减少TIME_WAIT套接字数量，但是对于 Squid，效果却不大。此项参数可以控制TIME_WAIT的最大数量，避免Squid服务器被大量的TIME_WAIT拖死。

[参考](http://zhangbin.junxilinux.com/?p=219)

问：大量的 TIME_WAIT 状态 TCP 连接存在，其本质原因是什么？

首先 TIME_WAIT 状态，在TCP 连接中，主动关闭连接的一方出现的状态；（收到 FIN 命令，进入 TIME_WAIT 状态，并返回 ACK 命令），另一端是会进入Closed状态的

[参考](https://blog.csdn.net/permike/article/details/104407774)

1.大量的短连接存在
2.特别是 HTTP 请求中，如果 connection 头部取值被设置为 close 时，基本都由「服务端」发起主动关闭连接
3.而，TCP 四次挥手关闭连接机制中，为了保证 ACK 重发和丢弃延迟数据，设置 time_wait 为 2 倍的 MSL（报文最大存活时间）,保持 2 个 MSL 时间，即，4 分钟；（MSL 为 2 分钟）

所以要看是什么样的http请求，短连接，长连接，keepalive等等的内容

问：如何解决上述 time_wait 状态大量存在，导致新连接创建失败的问题，一般解决办法：

1.客户端，HTTP 请求的头部，connection 设置为 keep-alive，保持存活一段时间：现在的浏览器，一般都这么进行了

2.服务器端

允许 time_wait 状态的 socket 被重用，上面的内核参数

缩减 time_wait 时间，设置为 1 MSL（即，2 mins,Linux 系统停留在 TIME_WAIT 的时间为固定的 60 秒）

Tips：RFC 793中规定MSL为2分钟，实际应用中常用的是30秒，1分钟和2分钟等。

2MSL，TCP 的 TIME_WAIT 状态，也称为2MSL等待状态：

当TCP的一端发起主动关闭（收到 FIN 请求），在发出最后一个ACK 响应后，即第3次握 手完成后，发送了第四次握手的ACK包后，就进入了TIME_WAIT状态。
2.必须在此状态上停留两倍的MSL时间，等待2MSL时间主要目的是怕最后一个 ACK包对方没收到，那么对方在超时后将重发第三次握手的FIN包，主动关闭端接到重发的FIN包后，可以再发一个ACK应答包。
3.在 TIME_WAIT 状态时，两端的端口不能使用，要等到2MSL时间结束，才可继续使用。（IP 层）
4.当连接处于2MSL等待阶段时，任何迟到的报文段都将被丢弃。
不过在实际应用中，可以通过设置 「SO_REUSEADDR选项」，达到不必等待2MSL时间结束，即可使用被占用的端口。


netstat中如果listen 8080，那么客户端与自身的8080连接也能看到，只是展示自身的会放到远端的前面，显示8080口

也就是说netstat看的是别的机器到本机的连接，不是本机器到别的机器的连接


ss 是 Socket Statistics 的缩写。ss 命令可以用来获取 socket 统计信息，它显示的内容和 netstat 类似。但 ss 的优势在于它能够显示更多更详细的有关 TCP 和连接状态的信息，而且比 netstat 更快。当服务器的 socket 连接数量变得非常大时，无论是使用 netstat 命令还是直接 cat /proc/net/tcp，执行速度都会很慢。ss 命令利用到了 TCP 协议栈中 tcp_diag。tcp_diag 是一个用于分析统计的模块，可以获得 Linux 内核中第一手的信息，因此 ss 命令的性能会好很多。

常用选项

 -h, –help 帮助
 -V, –version 显示版本号
 -t, –tcp 显示 TCP 协议的 sockets
 -u, –udp 显示 UDP 协议的 sockets
 -x, –unix 显示 unix domain sockets，与 -f 选项相同
 -n, –numeric 不解析服务的名称，如 “22” 端口不会显示成 “ssh”
 -l, –listening 只显示处于监听状态的端口
 -p, –processes 显示监听端口的进程(Ubuntu 上需要 sudo)
 -a, –all 对 TCP 协议来说，既包含监听的端口，也包含建立的连接
 -r, –resolve 把 IP 解释为域名，把端口号解释为协议名称

由于性能出色且功能丰富，ss 命令可以用来替代 netsate 命令成为我们日常查看 socket 相关信息的利器。其实抛弃 netstate 命令已经是大势所趋，有的 Linux 版本默认已经不再内置 netstate 而是内置了 ss 命令。

m.curl命令的使用

```html

-- 这样是通的，是在linux服务器里，不在windows服务器里
curl -X POST -H "Content-Type: application/json" -d '{"access_token":"f490c62227d596fae89bf1dff1b80969","request_body":{"query": [{"type": "APP", "data": ["XXX","XXX","XXX","XXX"]},{"type": "REDIS", "data": ["redis1","redis2"]},{"type": "DB", "data": ["db1","db2"]}]}}' http://XXX.XXX.XXX.com/api/XXX

-- 所以这个也应该是ok的，但是会报错
curl -X POST -H "Content-Type: application/json" -d '{"access_token":"8efc7aee088eda0fab04ee00500dd53e","request_body":{}}' http://XXX.XXX.XXX.com/api/XXX

n.socket

接上面l的netstat分析，要说回socket，首先socket的编程，Socket和ServerSocket，对应客户端和服务端

socket是在应用层和传输层之间的一个抽象层，socket本质是编程接口(API)，它把TCP/IP层复杂的操作抽象为几个简单的接口供应用层调用以实现进程在网络中通信。TCP/IP只是一个协议栈，必须要具体实现，同时还要提供对外的操作接口（API），这就是Socket接口，通过Socket,我们才能使用TCP/IP协议。

JDK的java.net包下有两个类：Socket和ServerSocket，在Client和Server建立连接成功后，两端都会产生一个Socket实例，操作这个实例，完成所需的会话，而我们就通过这些API进行网络编程，不需要去关心底层的实现了。 Socket连接过程分为三个步骤：服务器监听，客户端请求，连接确认。

1.服务器端先初始化Socket。（ listenfd 从名称看就是为了要监听而创建的socket描述符）

那bind 是干嘛？是为了声明说我要占用这个端口了， 你们都别用了。所以2.绑定端口（bind）

接着 3.listen函数才是真正开始对端口监听了。  

接下来是个死循环啊，啊啊也对，因为服务器端需要一直提供服务，只能坐以待命。那这个accept是干啥的呢？

4.调用accept阻塞，等待客户端来连接我。

为什么使用了listenfd , 然后返回了一个新的connfd ?  你还记得服务器要应付很多的客户端发起的连接， 所以它一定得把各个客户端区分开，怎么区分呢？  那只有用一个新的socket来表示， 可以看到后面接受/发送（写和读）消息的操作都是基于connfd 来做的。   至于之前的listenfd ， 它只起到一个大门的作用了， 意思是说，欢迎敲门， 进门之后我将为你生成一个独一无二的socket描述符！（引子张大胖的Socekt，o((⊙﹏⊙))o.）

这时有个客户端初始化一个Socket，然后5.该客户端连接服务器（connect），连接成功则建立连接。此时服务器的accept 相当于和客户端的connect 一起完成了TCP的三次握手 ！

连接建立以后6.客户端发送发送数据请求     

7.服务器接收请求并处理，然后回应数据给客户端   8.客户端读取到的数据，最后关闭连接。 这样一次完整的交互就结束了。

还有一个问题就是socket指的是 (IP, Port),   现在我已经有了一个listenfd 的socket, 端口是80  然后每次客户端发起连接还要创建新的connfd,  因为80端口已经被占用，难道服务器端会为每个连接都创建新的端口吗？

不是的，类似于mysql中的关联表，这个server的Ip + Port会有多个client的关联

所以socket得通过五元组(协议， 客户端IP, 客户端Port,  服务器端IP, 服务器端Port)来确定。


```

o.一些基础的运维指令

o1、网卡的工作模式和网卡速度

```html

ethtool eth0 | egrep 'Speed|Duplex'


Speed: 1000Mb/s
Duplex: Full

```

o2、平均负载

系统平均负载指是处于可运行状态和不可中断状态的进程的平均数量。

即单位时间内，系统处于可运行状态和不可中断状态的平均进程数，也就是平均活跃进程数，它和 CPU 使用率并没有直接关系。

-可运行状态的进程，包括正在使用CPU的进程，和正在等待CPU的进程。
对应于ps命令输出的STAT列中状态为R的进程。
状态R：running or runnable (on run queue)

-不可中断状态的进程，表示正在等待其它系统资源的进程，例如等待磁盘I/O。
对应于ps命令输出的STAT列中状态为D的进程。
状态D：uninterruptible sleep (usually IO)。
不可中断状态实际上是系统对进程和硬件设备的一种保护机制。比如，当一个进程向磁盘读写数据时，为了保证数据的一致性，在得到磁盘回复前，它是不能被其他进程或者中断打断的。

因此，在平均负载把不可中断状态的进程考虑进去之后，我们称之为系统平均负载或Linux平均负载，而不是CPU平均负载。

1分钟、5分钟、15分钟

解释1：

如果平均负载为0.0，则表示系统处于空闲状态。
如果1分钟的平均值高于5分钟或15分钟的平均值，则表示负载正在增加。
如果1分钟的平均值低于5或15分钟的平均值，则表示负载正在减少。
如果平均负载高于系统的CPU数量，那么系统可能会遇到性能问题。

解释2：

如果 1 分钟、5 分钟、15 分钟的三个值基本相同，或者相差不大，那就说明系统负载很平稳。
但如果 1 分钟的值远小于 15 分钟的值，就说明系统最近 1 分钟的负载在减少，而过去 15 分钟内却有很大的负载。
反过来，如果 1 分钟的值远大于 15 分钟的值，就说明最近 1 分钟的负载在增加，这种增加有可能只是临时性的，也有可能还会持续增加下去，所以就需要持续观察。一旦 1 分钟的平均负载接近或超过了 CPU 的个数，就意味着系统正在发生过载的问题，这时就得分析调查是哪里导致的问题，并要想办法优化了。

例如，对于单个CPU系统上的平均负载为"1.73 0.60 7.98"，表示：
在最后1分钟内，系统平均过载73%（(1.73-1)/1）。
在最近5分钟内，系统平均负载不高，还有空闲。
在最后15分钟内，系统平均过载为698%（(7.98-1)/1）。
如果平均负载为2，在只有1个CPU的系统中，那么有一半的进程可能竞争不到CPU。
因此，一般比较理想的情况，是每个CPU上面运行着一个进程。


平衡负载多少正常当平均负载高于 CPU 数量 70% 的时候，你就应该分析排查负载高的问题了。一旦负载过高，就可能导致进程响应变慢，进而影响服务的正常功能。

对于具有多个CPU的系统，通常可以先将平均负载除以处理器数量。然后再通过查看CPU的使用率、IO等待、上下文切换等进行排查问题。

既然平均负载代表的是活跃进程数，那平均负载高了，是不是就意味着 CPU 使用率高了？

回顾一下，平均负载是指单位时间内，处于可运行状态和不可中断状态的进程数。所以，它不仅包括了正在使用 CPU 的进程，还包括等待 CPU 和等待 I/O 的进程。

而 CPU 使用率，是单位时间内 CPU 繁忙情况的统计，跟平均负载并不一定完全对应。比如：

-CPU 密集型进程，使用大量 CPU 会导致平均负载升高，此时这两者是一致的；
-I/O 密集型进程，等待 I/O 也会导致平均负载升高，但 CPU 使用率不一定很高； // 注意！！
-大量等待 CPU 的进程调度也会导致平均负载升高，此时的 CPU 使用率也会比较高。


```html

uptime

top

w   // 1分钟 5分钟 15分钟

cat /proc/loadavg  // 前三个值对应上面

```

补充：
ps -ef 与 ps aux

Linux下显示系统进程的命令ps，最常用的有ps -ef 和ps aux。这两个到底有什么区别呢？两者没太大差别，讨论这个问题，要追溯到Unix系统中的两种风格，System Ｖ风格和BSD 风格，ps aux最初用到Unix Style中，而ps -ef被用在System V Style中，两者输出略有不同。现在的大部分Linux系统都是可以同时使用这两种方式的。

p.linux基本操作

```html

在用linux命令时候， 我们经常需要同时执行多条命令， 那么命令之间该如何分割呢？

分号： 顺序地独立执行各条命令， 彼此之间不关心是否失败， 所有命令都会执行。

&&  ： 顺序执行各条命令， 只有当前一个执行成功时候， 才执行后面的。

||   ： 顺序执行各条命令， 只有当前面一个执行失败的时候， 才执行后面的。


```

q.grep的使用

Linux中grep查找含有某字符串的所有文件

```html

--递归查找目录下含有该字符串的所有文件
grep -rn "data_chushou_pay_info"  /home/hadoop/nisj/automationDemand/

--查找当前目录下后缀名过滤的文件
grep -Rn "data_chushou_pay_info" *.py

--当前目录及设定子目录下的符合条件的文件
grep -Rn "data_chushou_pay_info" /home/hadoop/nisj/automationDemand/ *.py

--结合find命令过滤目录及文件名后缀
find /home/hadoop/nisj/automationDemand/ -type f -name '*.py'|xargs grep -n 'data_chushou_pay_info'


Grep选项：
* : 表示当前目录所有文件，也可以是某个文件名
-r 是递归查找
-n 是显示行号
-R 查找所有文件包含子目录
-i 忽略大小写

有意思的命令行参数：
grep -i pattern files ：不区分大小写地搜索。默认情况区分大小写
grep -l pattern files ：只列出匹配的文件名,不列出路径
grep -L pattern files ：列出不匹配的文件名
grep -w pattern files ：只匹配整个单词，而不是字符串的一部分（如匹配‘magic’，而不是‘magical’）
grep -C number pattern files ：匹配的上下文分别显示[number]行
grep pattern1 | pattern2 files ：显示匹配 pattern1 或 pattern2 的行
grep pattern1 files | grep pattern2 ：显示既匹配 pattern1 又匹配 pattern2 的行


```

### 十二、vi中的快速操作

以下是在command模式下的快捷键

| 编号 | 说明 | 快捷键 |
| :----: | :----:  | :----: |
| 1 | 撤销上一步的操作 | u |
| 2 | 恢复上一步被撤销的操作 | ctrl + u |
| 3 | 向文件首翻半屏 | ctrl + u |
| 4 | 向文件尾翻半屏 | ctrl + d |
| 5 | 向文件首翻一屏 | ctrl + b |
| 6 | 向文件尾翻一屏 | ctrl + f |
| 7 | 保存当前的修改退出 | ZZ |
| 8 | 删除光标所在行 | dd |
| 9 | 删除光标所在行及其后n-1行 | ndd |
| 10 | 将当前行及其n行的内容复制到寄存器中（不是右键的copy） | nyy |
| 11 | 粘贴文本操作，用于将缓存区的内容粘贴到当前光标所在位置的下方 | p |
| 12 | 粘贴文本操作，用于将缓存区的内容粘贴到当前光标所在位置的上方 | P |
| 13 | 在行末添加文本 | A |
| 14 | 在行首插入文本 | I |
| 15 | 在当前行后面插入一空行 | o |
| 16 | 在当前行前面插入一空行 | O |
| 17 | 转到文件末尾 | G |
| 18 | 转到第9行 | 9G |
| 19 | 删除光标所在处的字符（就是光标后面的） | x |
| 20 | 删除光标所在处的字符（就是光标前面的） | X |
| 21 | 删除到下一个单词开头 | dw |
| 22 | 删除到本单词末尾 | de |
| 23 | 删除到前一个单词（删的是后面的） | dE |
| 24 | 删除到前一个单词（删的是前面的） | db |
| 25 | 删除到前一个单词包括光标（删的是前面的） | dB |
| 26 | 删除光标位置到本行结尾 | D d$ （咋感觉不要前面这个D） |
| 27 | 删除光标位置到本行开头 | d0 |
| 28 | 方向键 | hjkl |
| 29 | 快速移动光标至行首 | shift + 6 或 0 |
| 30 | 快速移动光标至行末 | shift + 4 |
| 31 | 向后移动字符和向后移动n个字符（想要忽略标点符号，就用大写） | w / 2w |
| 32 | 向前移动字符和向前移动n个字符（想要忽略标点符号，就用大写） | b / 2b |
| 34 | 选中 | v |
| 35 | 结束选中 | y |
| 36 | 打开多个，在已经有vi的情况下 | :sp [filepath] |
| 37 | 多个vi的切换 | ctrl + 两次w 或者 ctrl + w后上下键 |


### 十三、linux搭一个nfs

使用yum -y install nfs-utils，因为centos7自带了rpcbind，所以不用安装rpc服务，rpc监听在111端口，可以使用

```html
ss -tnulp | grep 111
```

查看rpc服务是否自动启动，如果没有启动，就用下面命令启动

```html
systemctl start rpcbind
```

rpc在NFS的搭建过程中至关重要，因为rpc能够获得nfs服务器端的端口号等信息，nfs服务器端通过rpc获得这些信息后才能连接nfs服务器端

编辑`/etc/exports` 文件，文件目录待会创建，代表所有主机能访问，no_root_squash代表允许root用户连接
```html
/opt/kubernetes/redis/pv1  *(rw,no_root_squash)
/opt/kubernetes/redis/pv2  *(rw,no_root_squash)
/opt/kubernetes/redis/pv3  *(rw,no_root_squash)
```

创建文件夹并赋予权限，这步非常关键，否则客户端会被拒绝，检查下权限

```html
mkdir -p /opt/kubernetes/redis/pv{1..3}
chmod 777 -R /opt/kubernetes/redis/pv{1..3}
ls -al
```

检查下有没有问题，然后就可以启动了，并可以查看

```html
showmount -e localhost
systemctl enable nfs
systemctl start nfs
rpcinfo -p [ip]
```

接下来试下客户端，一定要测试一下的。随便找一台机器，这里还是要安装一下nfs-utils的，不然没有`showmount`命令

```html
showmount -e [服务端IP]
```
尝试挂载，并写些文件，可以在服务端也看见

```html
mkdir -p /opt/redis
mount -t nfs [IP]:/opt/kubernetes/redis/pv1 /opt/redis
touch a.txt
```
然后卸载下，umount不成功，用

```html
umount -l /opt/redis
```

### 十四、linux中使用ab进行并发测试

linux下的压力测试，很好用，其他工具java系的有jmeter

ab是apache自带的压力测试工具。ab非常实用，它不仅可以对apache服务器进行网站访问压力测试，也可以对或其它类型的服务器进行压力测试。比如nginx、tomcat、IIS等

ab可以这样安装

```html
yum -y install httpd-tools
```

一般用到的参数是-n，-c和-h，具体如下查看

```html
ab --help
```

例如

```html
ab -c 5 -n 10 http://www.myvick.cn/index.php

//返回
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 10.144.91.121 (be patient).....done

//测试服务器名字
Server Software:
Server Hostname:        10.144.91.XXX
Server Port:            301XX
//请求的URL中的根绝对路径，通过该文件的后缀名，我们一般可以了解该请求的类型
Document Path:          /v1/pools/7b8f0f5e2fbb4d9aa2d5fd55366d6vop/images?available=1
//HTTP响应数据的正文长度
Document Length:        3831 bytes
//并发用户数，这是我们设置的参数之一
Concurrency Level:      5
//所有这些请求被处理完成所花费的总时间 单位秒
Time taken for tests:   0.950 seconds
//总请求数量，这是我们设置的参数之一
Complete requests:      10
//表示失败的请求数量，这里的失败是指请求在连接服务器、发送数据等环节发生异常，以及无响应后超时的情况
Failed requests:        0
Write errors:           0
//所有请求的响应数据长度总和。包括每个HTTP响应数据的头信息和正文数据的长度
Total transferred:      45550 bytes
//所有请求的响应数据中正文数据的总和，也就是减去了Total transferred中HTTP响应数据中的头信息的长度
HTML transferred:       38310 bytes
//吞吐率，计算公式：Complete requests/Time taken for tests  总请求数/处理完成这些请求数所花费的时间
Requests per second:    10.52 [#/sec] (mean)
//用户平均请求等待时间，计算公式：Time token for tests/（Complete requests/Concurrency Level）。处理完成所有请求数所花费的时间/（总请求数/并发用户数）
Time per request:       475.110 [ms] (mean)
//服务器平均请求等待时间，计算公式：Time taken for tests/Complete requests，正好是吞吐率的倒数。也可以这么统计：Time per request/Concurrency Level
Time per request:       95.022 [ms] (mean, across all concurrent requests)
//这个统计很好的说明服务器的处理能力达到极限时，其出口宽带的需求量
Transfer rate:          46.81 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.1      0       0
Processing:   260  321  35.0    327     376
Waiting:      260  321  35.1    327     376
Total:        260  322  35.1    327     376

//这部分数据用于描述每个请求处理时间的分布情况，比如以上测试，100%的请求处理时间都不超过4ms，这个处理时间是指前面的Time per request，即对于单个用户而言，平均每个请求的处理时间
Percentage of the requests served within a certain time (ms)
  50%    327
  66%    327
  75%    331
  80%    374
  90%    376
  95%    376
  98%    376
  99%    376
 100%    376 (longest request)

```
如果是post请求的话，建一个文件，然后用ab

```html
ab -c 5 -n 10 -p /tmp/empty.txt http://localhost:8080/order
```

### 十五、linux中的TC使用

先从一个例子看起

网络延迟100ms

```html
sudo tc qdisc add dev enp2s0 root netem delay 100ms
```

改成200ms

```html
sudo tc qdisc change dev enp2s0 root netem delay 100ms
```

先删除配置，改成丢包10%

```html
sudo tc qdisc del dev enp2s0 root

sudo tc qdisc add dev enp2s0 root netem loss 50%

```
看测试结果的话用一个能ping通的地址就行

```html
ping ip -c 30

```

TC也是可以支持复合命令的，如果即需要丢包又需要延迟的话，就用复合命令

```html
sudo tc qdisc add dev enp2s0 root netem loss 50% delay 220ms

//然后可以show的看一下
sudo tc qdisc show dev eth0
```

另外Tc可以做一些复杂的指令，使用ingress队列，进行五元组的匹配

```html
tc qdisc add dev eth0 ingress && tc filter add dev eth0 parent ffff: protocol ip prio 1 basic match 'cmp(u16 at 2 layer transport gt 22) or cmp(u16 at 2 layer transport lt 22)' action drop

tc filter add dev eth0 parent ffff: protocol all prio 1 u32 match ip dport 6379 0xffff action drop

tc qdisc del dev eth0 ingress
```

该指令可以和上面指令复合使用，因为是不同的队列


### 十六、nslookup的使用

nslookup的命令在windows和linux中都有，用于查询DNS的记录，查看域名解析是否正常，在网络故障的时候用来诊断网络问题

```html
nslookup baidu.com
```

还可以

```html
nslookup -qt=type domain [dns-server]
```

其中，type可以是以下这些类型：

A 地址记录
AAAA 地址记录
AFSDB Andrew文件系统数据库服务器记录
ATMA ATM地址记录
CNAME 别名记录
HINFO 硬件配置记录，包括CPU、操作系统信息
ISDN 域名对应的ISDN号码
MB 存放指定邮箱的服务器
MG 邮件组记录
MINFO 邮件组和邮箱的信息记录
MR 改名的邮箱记录
MX 邮件服务器记录
NS 名字服务器记录
PTR 反向记录
RP 负责人记录
RT 路由穿透记录
SRV TCP服务器信息记录
TXT 域名对应的文本信息
X25 域名对应的X.25地址记录

```html
nslookup -qt=mx baidu.com 8.8.8.8
```

nslookup的规则就是在后面加.的话就是代表是根域，也就是到此结束的意思了

### 十七、linux内核升级

首先检查下kernel的[版本信息](https://www.kernel.org/)

其中有stable版本的，还有longterm版本的

在这个[网址](http://mirror.centos.org/centos/7/rt/x86_64/Packages/)可以找到包

```html
yum install kernel-rt-3.10.0-693.2.2.rt56.623.el7.x86_64.rpm

rpm -ivh kernel-rt-devel-3.10.0-693.2.2.rt56.623.el7.x86_64.rpm
```

上面的命令进行安装，然后执行

还有centos7上启用ELRepo，运行

```html
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org

rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm

yum --disablerepo="*" --enablerepo="elrepo-kernel" list available  //列出可用的内核相关包

yum --enablerepo=elrepo-kernel install kernel-ml //安装主线稳定内核
```

```html
//设置 GRUB 默认的内核版本

sed -i “s#GRUB_DEFAULT.*#GRUB_DEFAULT=0#g"  /etc/default/grub

grub2-mkconfig -o /boot_old/grub2/grub.cfg

grub2-mkconfig -o /etc/grub2.cfg

awk -F \' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg  //查看启动项中安装的内核

grub2-set-default 0

grub2-editenv list //确认默认的启动项

reboot
```
删除之前的旧内核

```html
rpm -qa | grep kernel
yum remove XXX
```

### 十八、linux的Shell

系统的 shell 有很多种, 比如 bash, sh, zsh 之类的, 如果要查看某一个用户使用的是什么 shell，可以使用

```html
echo $Shell
echo $0

对于常见的加载配置文件，有
/etc/profile
/etc/bashrc

~/.bashrc
~/.profile

这里说bash的shell，如果是sh或者其他shell，显然是不会运行bashrc的
```

这里还是有区别的，login shell 和 no-login shell

“login shell” 代表用户登入, 比如使用 “su -“ 命令, 或者用 ssh 连接到某一个服务器上, 都会使用该用户默认 shell 启动 login shell 模式.该模式下的 shell 会去自动执行 /etc/profile 和 ~/.profile 文件, 但不会执行任何的 bashrc 文件, 所以一般再 /etc/profile 或者 ~/.profile 里我们会手动去 source bashrc 文件.

而 no-login shell 的情况是我们在终端下直接输入 bash 或者 bash -c “CMD” 来启动的 shell.该模式下是不会自动去运行任何的 profile 文件。比如
```html
bash -c "ping 10.2.2.2"

sh -c "ping 10.2.2.2"
```

interactive shell 和 non-interactive shell

interactive shell 是交互式shell, 顾名思义就是用来和用户交互的, 提供了命令提示符可以输入命令.该模式下会存在一个叫 PS1 的环境变量, 如果还不是 login shell 的则会去 source /etc/bash.bashrc 和 ~/.bashrc 文件

non-interactive shell 则一般是通过 bash -c “CMD” 来执行的bash.该模式下不会执行任何的 rc 文件, 不过还存在一种特殊情况

可能存在的模式组合中 RC 文件的执行

SSH login, sudo su - [USER] 或者 mac 下开启终端，ssh 登入和 su - 是典型的 interactive login shell, 所以会有 PS1 变量, 并且会执行

```html
/etc/profile
~/.profile
```

在命令提示符状态下输入 bash 或者 ubuntu 默认设置下打开终端，这样开启的是 interactive no-login shell, 所以会有 PS1 变量, 只会执行

```html
/etc/bash.bashrc
~/.bashrc
```

通过 bash -c “CMD” 或者 bash BASHFILE 命令执行的 shell，这些命令什么都不会执行, 也就是设置 PS1 变量, 不执行任何 RC 文件

最特殊! 通过 “ssh server CMD” 执行的命令 或 通过程序执行远程的命令，这是最特殊的一种模式, 理论上应该既是 非交互 也是 非登入的, 但是实际上他不会设置 PS1, 但是还会执行
```html
/etc/bash.bashrc
bashrc
```

bashrc 和 profile 的区别

profile: 其实看名字就能了解大概了, profile 是某个用户唯一的用来设置环境变量的地方, 因为用户可以有多个 shell 比如 bash, sh, zsh 之类的, 但像环境变量这种其实只需要在统一的一个地方初始化就可以了, 而这就是 profile.

bashrc: bashrc 也是看名字就知道, 是专门用来给 bash 做初始化的比如用来初始化 bash 的设置, bash 的代码补全, bash 的别名, bash 的颜色. 以此类推也就还会有 shrc, zshrc 这样的文件存在了, 只是 bash 太常用了而已.

=> 代表 在文件内部 source, 换行的 => 代表自身执行结束以后在 source, 同一行表示先 source 在执行自身

```html
普通 login shell
/etc/profile
   => /etc/bash.bashrc

~/.profile
  => ~/.bashrc => /etc/bashrc

终端中直接运行 bash
/etc/bash.bashrc
~/.bashrc => /etc/bashrc

bash -c “CMD”
什么都不执行

ssh server “CMD”
/etc/bash.bashrc => /etc/profile
~/.bashrc => | /etc/bashrc => /etc/profile
             | ~/.profile
```

这里会有点小混乱, 因为既有 /etc/bash.bashrc 又有 /etc/bashrc, 其实是这样的 ubuntu 和 debian 有 /etc/bash.bashrc 文件但是没有 /etc/bashrc, 其他的系统基本都是只有 /etc/bashrc 没有 /etc/bash.bashrc.

### 十九、cron和crontab

cron，java类的是6-7位

crontab，linux写的是5-6位，最后一位都可以省略

cron从左往右一次是 秒 分 时 日 月 周 年 其中日和周 中可以有 ？，两者是二选一（用了这俩其中一个，另外一个要用?，如果其中一个是*，另外一个最好也得是?）， Day of week（周），用1-7表示时，1对应表示Sunday，7表示Saturday。第一天是Sunday，最后一天是Saturday。（，-*/） 4个字符在每个域都是通用的，逗号就是分割时间断，中划就是连续时间段，斜杠就是每隔的意思，星好就是匹配这个域下的所有值， Day of week（周）还可以用井号

crontab从左往右一次是 分 时 日 月 周 年
