---
layout: post
title: "harbor使用及运维"
author-id: "zqmalyssa"
tags: [code, operations]
---

harbor作为docker的私服，很常用，有单机版，有集群版，也有helm版(好像不怎么灵)

### harbor的基本介绍



### harbor的运维

**1.ERROR: Failed to program FILTER chain: iptables failed: iptables --wait -I FORWARD -o br-195e8ec66d41 -j DOCKER: iptables v1.4.21: Couldn't load target `DOCKER':No such file or directory**

```html
# iptables -N DOCKER
```
另外，还有个ERROR: unable to insert jump to DOCKER-ISOLATION rule in FORWARD chain:  (iptables failed: iptables --wait -I FORWARD -j DOCKER-ISOLATION: iptables v1.4.21: Couldn't load target `DOCKER-ISOLATION':No such file or directory

```html
# iptables -N DOCKER-ISOLATION
```
手动的去创建`iptables`中的这两个链就能恢复了，很神奇为什么会没有
