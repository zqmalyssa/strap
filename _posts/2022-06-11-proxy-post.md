---
layout: post
title: 代理
tags: [代理]
author-id: zqmalyssa
---

代理来个记忆文档

#### 正向代理

用正向代理访问无法访问的资源，反向代理看看nginx之类的

临时的

```html

export  http_proxy="http://proxy-nwl.xxx.xxx.com:8080"
export  https_proxy="http://proxy-nwl.xxx.xxx.com:8080"

// 以上代理地址一样，但分别争对http和https的请求，也就是curl http://wwww.baidu.com 和 curl https://wwww.baidu.com 都好使

unset http_proxy && unset https_proxy

// curl的话这样也行

curl http://www.baidu.com -x http://proxy-nwl.xxx.xxx.com:8080

curl https://www.baidu.com -x http://proxy-nwl.xxx.xxx.com:8080

// 注意写的顺序

```
