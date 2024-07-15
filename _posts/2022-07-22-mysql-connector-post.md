---
layout: post
title: mysql-connector
tags: [connector]
author-id: zqmalyssa
---

java连接mysql的驱动

#### 相关组件

部件名称	主要接口、类	主要作用
驱动管理器	java.sql.DriverManager	注册、解除驱动。客户端通过它获取连接。
驱动层	java.sql.Driver	我们常见的用法是接收驱动管理器的命令，生成连接并返回给驱动管理器。
连接层	java.sql.Connection	形成与数据库间的连接。它的基本功能是创建我们常见的Statement、PreparedStatement然后进行一系列的数据操作。除了实现基本操作的实现类外，还有针对不同的场景有不同的实现，这包括：Failover、Loadbalance、Replication。后续文章会详细介绍它们的代码实现。
IO层	com.mysql.jdbc.MysqlIO	以java.net.Socket实现与Mysql数据管理软件的数据交互。我们应用使用到的sql命令，最终通过socket的输出流传到mysql数据库管理软件，然后通过socket的输入流获取操作结果信息。


#### 相关使用

```html

linux下直接使用

yum install -y awscli

aws configure // 设置完后 路径下就有 config和credentials文件了

awscli里面没有设置代理的地方？

```

#### 相关命令

```html

// region

aws ec2 describe-regions

（包括为账户禁用的任何区域）
aws ec2 describe-regions --all-regions

// az

aws ec2 describe-availability-zones --region ap-southeast-1

aws ec2 describe-availability-zones --all-availability-zones

// elb

aws elbv2 describe-load-balancers --name fis-test // 不带参数 列出所有的LB

aws elbv2 set-security-groups --load-balancer-arn arn:aws:elasticloadbalancing:ap-southeast-1:535461410933:loadbalancer/net/fis-test/426ef9f247be3232 --security-groups sg-08f72dec338a9acd6

LoadBalancerArn: arn:aws:elasticloadbalancing:ap-southeast-1:535461410933:loadbalancer/net/fis-test/426ef9f247be3232

An error occurred (InvalidConfigurationRequest) when calling the SetSecurityGroups operation: SetSecurityGroups is not supported for load balancers of type 'network'


aws elbv2 set-subnets --load-balancer-arn arn:aws:elasticloadbalancing:us-west-2:123456789012:loadbalancer/app/my-load-balancer/50dc6c495c0c9188 --subnets subnet-8360a9e7 subnet-b7d581c0  // set某些子网

// elbv2

aws elbv2 describe-target-groups

aws elbv2 describe-target-health --target-group-arn arn:aws:elasticloadbalancing:ap-southeast-1:535461410933:targetgroup/fis-test/54f4baf9d14356b9

// ec2

aws ec2 describe-subnets --filters Name=tag:chaos,Values=true

aws ec2 describe-subnets --filters Name=tag:chaos,Values=true Name=availability-zone,Values=ap-southeast-1b


```
