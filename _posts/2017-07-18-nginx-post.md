---
layout: post
title: nginx使用及运维
tags: [nginx]
author-id: zqmalyssa
---
Nginx目前常用作应用服务器，实现反向代理等操作

#### Nginx的Win简单搭建及使用

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
