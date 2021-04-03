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

參數：
-nn：直接以 IP 及 port number 顯示，而非主機名與服務名稱
-i ：後面接要『監聽』的網路介面，例如 eth0, lo, ppp0 等等的介面；
-w ：如果你要將監聽所得的封包資料儲存下來，用這個參數就對了！後面接檔名
-c ：監聽的封包數，如果沒有這個參數， tcpdump 會持續不斷的監聽，
     直到使用者輸入 [ctrl]-c 為止。
-A ：封包的內容以 ASCII 顯示，通常用來捉取 WWW 的網頁封包資料。
-e ：使用資料連接層 (OSI 第二層) 的 MAC 封包資料來顯示；
-q ：僅列出較為簡短的封包資訊，每一行的內容比較精簡
-X ：可以列出十六進位 (hex) 以及 ASCII 的封包內容，對於監聽封包內容很有用
-r ：從後面接的檔案將封包資料讀出來。那個『檔案』是已經存在的檔案，
     並且這個『檔案』是由 -w 所製作出來的。
所欲擷取的資料內容：我們可以專門針對某些通訊協定或者是 IP 來源進行封包擷取，
     那就可以簡化輸出的結果，並取得最有用的資訊。常見的表示方法有：
     'host foo', 'host 127.0.0.1' ：針對單部主機來進行封包擷取
     'net 192.168' ：針對某個網域來進行封包的擷取；
     'src host 127.0.0.1' 'dst net 192.168'：同時加上來源(src)或目標(dst)限制
     'tcp port 21'：還可以針對通訊協定偵測，如 tcp, udp, arp, ether 等
     還可以利用 and 與 or 來進行封包資料的整合顯示呢！

一些基本示例如下

直接启动tcpdump将监视第一个接口上所有流过的数据包

```html
# tcpdump
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
4. iptables作白名单时，将链的默认策略反过来，设置成ACCEPT，通过在链的最后设置REJECT实现白名单机制，而不是将默认策略设置成DROP
5. iptables可以自定义链，用于存一类相同的规则，方便管理
6. iptables有各种各样的扩展，方便写语句，-m参数就是使用的扩展模块（这块自行学习）

PREROUTING的规则可以存放在：raw表，mangle表，nat表
INPUT的规则可以存放在：mangle表，filter表（Centos7还有nat表，6没有）
FORWARD的规则可以存放在：mangle表，filter表
OUTPUT的规则可以存放在：raw表，mangle表，nat表，filter表
POSTROUTING的规则可以存放在：mangle表，nat表

raw表的规则可以被哪些链使用：PREROUTING，OUTPUT
mangle表的规则可以被哪些链使用：PREROUTING，INPUT，FORWARD，OUTPUT，POSTROUTING
nat表的规则可以被哪些链使用：PREROUTING，OUTPUT，POSTROUTING（Centos7还有INPUT，6没有）
filter表的规则可以被哪些链使用：INPUT，FORWARD，OUTPUT

iptables的一些基本使用

1. iptables -FXZ去清除之前所有规则

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

物理CPU的个数

```html
# cat /proc/cpuinfo | grep "physical id" | sort | uniq | wc -l
```
内存总量

```html
# cat /proc/meminfo | grep MemTotal
```

cpu cores数量，这里的cpu我觉得是逻辑，不是物理的cpu

```html
# cat /proc/cpuinfo | grep "cpu cores" | uniq | awk {'print $4'}
```
K8S里面node信息的cpu数量应该是cpuinfo里面processor的数量

```html
# cat /proc/cpuinfo | grep "physical id" | wc -l
```

e.linux如何查看开机自启动的服务

```html
ntsysv
```
这个命令会弹出一个界面

f.linux中lsof命令的使用

lsof命令列出所有打开的文件，一些参数解释如下

1. FD代表文件描述符，CWD代表当前工作目录，RTD代表根目录，TXT代表文本程序，还有1u，3r，4w
2. TYPE代表文件和它的鉴定，DIR目录，REG代表常规文件，CHR字符特殊文件，FIFO先进先出

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
-l 仅列出有在 Listen (监听) 的服務状态

-p 显示建立相关链接的程序名
-r 显示路由信息，路由表
-e 显示扩展信息，例如uid等
-s 按各个协议进行统计
-c 每隔一个固定时间，执行该netstat命令。

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
