---
layout: post
title: "初识Ansible"
author-id: "zqmalyssa"
tags: [code, operations]
---

个人的主页是 [zqmalyssa](https://zqmalyssa.github.io/strap/).
选择`Github Pages`进行一些分享，主要是因为它是免费的，且Github平时用的也比较多，有足够多的前端框架进行选择，可以按照喜好定制化页面，站点的个人色彩比较浓


### Ansible的简单使用方法

之前只是听说过ansible的大名，但一直没有使用过，今天简单研究使用了一下。首先ansible是源自于一部小说`安德的游戏`，也已经被拍摄成电影了，`电影天堂`自行下载补课，废话不多，直接在自己的本地环境玩耍，三台CentOS 7服务器。

#### 一、尝试使用

[参考文献](https://docs.ansible.com/)

因为可以连接外网，直接yum install的方式安装

```html
yum install -y ansible
```

安装完后一般检查下版本

```html
ansible --version
```

随便一个文件目录配置一下Inventory文件，我们取名为hosts，标签里面写要管理的服务器地址

```html
[web]
#172.20.10.11
#172.20.10.12
#172.20.10.13

[local]
127.0.0.1

[remote]
172.20.10.12

```
先注掉[web]标签下的机器，用[local]和[remote]标签玩一玩

```html
# Run against localhost
$ ansible -i ./hosts --connection=local local -m ping

# Run against remote server
$ ansible -i ./hosts remote -m ping

```

参数内容简单说下，后面再补充，-m后面选择的是模块，这里是ping模块，测试机器的连通性，也可以是shell模块，可以使用/bin/sh进行指令下发，我们用这个指令去remote装ansible

```html
# Run against a remote server
ansible -i ./hosts remote -m shell -a 'yum install -y ansible'
```

会回显安装的过程及结果在运行ansible的机器上，ansible主要用到了ssh和python，现在一般linux的机器中都已经预置了python，只要ansible的控制机器(controller node)能ssh至被控机器(managed node)就行了

我们现在使用[web]标签，将三台机器的Ip写进去，然后运行命令，如：

```html
# Run against a remote server
ansible -i ./hosts all -m shell -a 'systemctl status mysql'
```

这样可以同时查看三台机器上mysql服务运行的情况，而不用每次一台台去查看，非常方便


整体看下来，其实从节点没有安装任何ansible的东西，但是ssh-key给了下

```html

# 在主节点上，注意不同用户不一样的
ssh-keygen
ssh-copy-id qmzhang@10.2.22.21


# 在从节点上查看
cd ~/.ssh
cat authorized_keys

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCo25fD7iDx4jInUWspXKK4aBtD20LrDaVhAM4E5q0fVDL/jGRIUIJ3lJ+ey8BF6QPvp+6mKGQJwCiP7mpPat+OMKy1hHrMg45vZsXw0Lv+cV94nH1MBTk2LMn40m7opHmv5LaTs2t6znhI/RTP1yUz6ptBFNbT5rWTFhK/CBrThvgre21CIg+8b31N/9h9kAflzo44rk7I0GE+zwcGtUnWzEtXM4EPr/Xfj+WH65Eu80BZUXTMwyH1eEa1sSTmhjyeseZiaGJYIzXckkFbzs/XLO0uFaz2+1hMPJkllsK1ZX0nhup7/T7fiT3p0xc31FuZtk2lCUkewovKtDLShhyf root@fan1
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDAcnKKnJG0D/B+b7pVlHZINh4ltD0MfWp+5/obyOB1JrzDhpP5HC+AsDXLD5qqTjaALjKTfZkyaXFo5HVSgYMi5eDluzu6dNZUKhIiwuWwND/+aTIWKTCG1qgRYbw1lV67DKkYkQtz5PN0LcY02kKk6U6OmLH1iN7ODybL/QeV7Eo+tUNYyst+MRzdw1HhvUEby9U1QDtrNoYE0D/7LQYlFe6immRvgqqg7H2E2XOnoX3v8B4AXWWgpGnChapT9gVhvSM7231uSNB38XjS9q5vCdWLlAcRawI9Hg7lFj4XWAMih1d+hzm+I0hkJPpSsglOCP5YxAki2edFQompXR7F
qmzhang@fan2

```

简单的shell命令

```html
ansible -i hosts remote -m shell -a "tc qdisc show dev eth0"

# 如果是playbook
ansible-playbook tc_show.yaml --extra-vars "{'ci_list':'fan1','ip_address': '10.2.22.21'}"

```
看下写的playbook，比较简单

```html

# 阻断的
---
- hosts: "{{ ci_list }}"
  gather_facts: no

  tasks:
  - name: find interface name by ip.
    shell: ip a | grep {{ ip_address }} | awk '{print $7}'
    register: result
    when: not "{{ ci_list }}" == "all"

  - name: inject tc block.
    shell: >
      tc qdisc add dev {{ result.stdout }} ingress && tc filter add dev {{ result.stdout }} parent ffff: protocol ip prio 1 basic match 'cmp(u16 at 2 layer transport gt 22) or cmp(u16 at 2 layer transport lt 22)' action drop
    when: not result.stdout == ""

# 恢复的
---
- hosts: "{{ ci_list }}"
  gather_facts: no

  tasks:
  - name: find interface name by ip
    shell: ip a | grep {{ ip_address }} | awk '{print $7}'
    register: result
    when: not "{{ ci_list }}" == "all"

  - name: recover tc block.
    shell: tc qdisc del dev {{ result.stdout }} ingress
    when: not result.stdout == ""

# show的
---
- hosts: "{{ ci_list }}"
  gather_facts: no

  tasks:
  - name: find interface name by ip.
    shell: ip a | grep {{ ip_address }} | awk '{print $7}'
    register: result
    when: not "{{ ci_list }}" == "all"

  - name: show tc qdisc.
    shell: tc qdisc show dev {{ result.stdout }}
    when: not result.stdout == ""

```

facts组件是用来收集被管理节点信息的，使用setup模块可以获取这些信息。
```html
ansible 10.2.22.23 -m setup

# 这条命令有的时候运行不起来，提示no match，解决是在/etc/ansible/hosts里面人工配置下，但是不应该？ 因为已经做了ssh-copy-id了
# 这边是这样的。如果运行ansible -i ./hosts remote -m ping 时候会报错了

{
    "changed": false,
    "msg": "Failed to connect to the host via ssh: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).\r\n",
    "unreachable": true
}

这个时候就要ssh-copy-id去解决了


# 可以过滤
ansible 10.2.22.23 -m setup -a "filter=*ipv4"
```
在ansible中，任何一个模块都会返回json格式的数据，即使是错误信息都是json格式的。

```html
---
    - hosts: 192.168.100.65
      tasks:
        - shell: echo hello world
          register: say_hi
        - debug: var=say_hi



# debug输出的结果如下
TASK [debug] *********************************************
ok: [192.168.100.65] => {
    "say_hi": {
        "changed": true,
        "cmd": "echo hello world",
        "delta": "0:00:00.002086",
        "end": "2017-09-20 21:03:40.484507",
        "rc": 0,
        "start": "2017-09-20 21:03:40.482421",
        "stderr": "",
        "stderr_lines": [],
        "stdout": "hello world",
        "stdout_lines": [
            "hello world"
        ]
    }
}
```
如果想要输出json数据的某一字典项，则应该使用"key.dict"或"key['dict']"的方式引用。例如最常见的stdout项"hello world"是想要输出的项，以下两种方式都能引用

```html
---
    - hosts: 192.168.100.65
      tasks:
        - shell: echo hello world
          register: say_hi
        - debug: var=say_hi.stdout
        - debug: var=sya_hi['stdout']
```
如果想要输出json数据中的某一数组项(列表项)，则应该使用"key[N]"的方式引用数组中的第N项，其中N是数组的index，从0开始计算。如果不使用index，则输出的是整个数组列表。

```html
---
    - hosts: 192.168.100.65
      tasks:
        - shell: echo hello world
          register: say_hi
        - debug: var=say_hi.stdout_lines[0]
```

显然，facts数据的顶级key为ansible_facts，在引用时应该将其包含在变量表达式中。但自动收集的facts比较特殊，它以ansible_facts作为key，ansible每次收集后会自动将其注册为变量，所以facts中的数据都可以直接通过变量引用，甚至连顶级key ansible_facts都要省略。

```html
是
ansible_eth0.ipv4.address

而不是
ansible_facts.ansible_eth0.ipv4.address

```

#### 二、非SSH访问

这边要提的是可以再ansible默认的hosts文件中进行配置`/etc/ansible/hosts`，配置一个标签后配置服务器，如果没有做互信的话，需要用账户密码登录，那么需要额外配置

```html
$ vi /etc/ansible/hosts
[test]
10.144.93.12 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass="Qiming!@#$"
```

注意密码有特殊字符，要加引号

```html
# Run easy command
ansible test -m ping
```

测试一下连通性

#### 三、集成

如果AWX需要跟Gitlab连动，获取yaml，需要在awx选择ssh的模式（也可以http），然后在`setting`中的`repository`里面找到`deploy keys`然后就是enable这个`awx_git`了，这样在update scm的时候就能拉gitlab上的yaml了

#### 四、playbook的编写

Playbook与ad-hoc相比,是一种完全不同的运用ansible的方式，类似与saltstack的state状态文件。ad-hoc无法持久使用，playbook可以持久使用。

Playbook核心元素
Hosts 执行的远程主机列表
Tasks 任务集
Varniables 内置变量或自定义变量在playbook中调用
Templates 模板，即使用模板语法的文件，比如配置文件等
Handlers 和notity结合使用，由特定条件触发的操作，满足条件方才执行，否则不执行
tags 标签，指定某条任务执行，用于选择运行playbook中的部分代码。

#### 五、问题汇总

1、如何在Ansible Shell模块中转义特殊字符

YAML不允许在一行上进行嵌套映射，例如：

```html
foo: bar: baz
```
这就是为什么YAML设计人员选择如果:与键在同一行上，则禁止在映射值中使用它。 (也可以通过简单地忽略进一步发生的情况并将其视为常规内容来解决。)

您有几种选择。您可以将整个值都放在引号中，在这种情况下，这不是一个好主意，因为您同时具有单引号和双引号，因此必须转义。

一种解决方法是在sed命令中转义空格：

```html
shell: date -s"$(curl -s --head http://google.com | grep '^Date:' | sed 's/Date:\ //g') +0530"
```
更通用的解决方案是使用折叠块标量：
```html
shell: >
  date -s"$(curl -s --head http://google.com | grep '^Date:' | sed 's/Date: //g') +0530"
```
甚至可以将其分成几行：
```html
shell: >
  date -s"$(curl -s --head http://google.com
  | grep '^Date:' | sed 's/Date: //g') +0530"
```
