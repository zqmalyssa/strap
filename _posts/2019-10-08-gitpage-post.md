---
layout: post
title: "Github Pages相关内容"
author-id: "zqmalyssa"
tags: [life, hobby]
---

因为迁移电脑，防止有遗忘，所以记录下基本内容

### 一、搭建本地测试环境

GitHub Pages使用了jekyll作为静态网页生成系统，而其是基于ruby的，因此先要安装ruby环境

[ruby安装](https://www.ruby-lang.org/en/downloads/)

安装好后commond内可以

```html
# ruby -v
```
ruby安装成功后，执行命令，热点装需要等一会

```html
# gem install jekyll bundler
```

运行`jekyll`后会有一些依赖报错，可以用`gem install`进行安装，也可以用`bundle install`，这个时候如果`jekyll`还是报错，先试下本地调试是否可用，可用可以忽略

```html
# bundle exec jekyll serve
```


