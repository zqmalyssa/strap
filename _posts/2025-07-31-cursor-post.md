---
layout: post
title: cursor
tags: [cursor]
author-id: zqmalyssa
---

cursor解决问题的一些总结

#### 解决的问题

1、配置image-gen-server的情况



2、配置java21启动的方式

```html

配置settings.json

配置launch.json

{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "java",
            "name": "Launch WebStarter",
            "request": "launch",
            "mainClass": "com.xxx.xxx.xxx.WebStarter",
            "projectName": "xxx-web",
            "vmArgs": [
                "-Xms512m",
                "-Xmx1024m",
                "-XX:+UseG1GC",
                "-Dspring.profiles.active=dev",
                "-Dserver.port=8080",
                "-Djdk.attach.allowAttachSelf=true",
                "-XX:+EnableDynamicAgentLoading",
                "--add-opens=java.management/sun.management=ALL-UNNAMED",
                "--add-opens=java.base/java.lang=ALL-UNNAMED",
                "--add-opens=java.base/java.lang.reflect=ALL-UNNAMED",
                "--add-opens=java.base/sun.reflect.annotation=ALL-UNNAMED",
                "--add-opens=java.base/java.math=ALL-UNNAMED",
                "--add-opens=java.base/java.util=ALL-UNNAMED",
                "--add-opens=java.base/sun.util.calendar=ALL-UNNAMED",
                "--add-opens=java.base/java.io=ALL-UNNAMED",
                "--add-opens=java.base/java.net=ALL-UNNAMED",
                "--add-opens=java.xml/com.sun.org.apache.xerces.internal.jaxp.datatype=ALL-UNNAMED"
            ],
            "args": [],
            "cwd": "${workspaceFolder}/xxx-web"
        },
        {
            "type": "java",
            "name": "Launch Job",
            "request": "launch",
            "mainClass": "com.xxx.xxx.xxx.WebStarter",
            "projectName": "xxx-job",
            "vmArgs": [
                "-Xms512m",
                "-Xmx1024m",
                "-XX:+UseG1GC",
                "-Dspring.profiles.active=fat",
                "-Dserver.port=8090",
                "-Djdk.attach.allowAttachSelf=true",
                "-XX:+EnableDynamicAgentLoading",
                "--add-opens=java.management/sun.management=ALL-UNNAMED",
                "--add-opens=java.base/java.lang=ALL-UNNAMED",
                "--add-opens=java.base/java.lang.reflect=ALL-UNNAMED",
                "--add-opens=java.base/sun.reflect.annotation=ALL-UNNAMED",
                "--add-opens=java.base/java.math=ALL-UNNAMED",
                "--add-opens=java.base/java.util=ALL-UNNAMED",
                "--add-opens=java.base/sun.util.calendar=ALL-UNNAMED",
                "--add-opens=java.base/java.io=ALL-UNNAMED",
                "--add-opens=java.base/java.net=ALL-UNNAMED",
                "--add-opens=java.xml/com.sun.org.apache.xerces.internal.jaxp.datatype=ALL-UNNAMED"
            ],
            "args": [],
            "cwd": "${workspaceFolder}/xxx-job"
        }
    ]
}

```

这个相当于帮我写好了配置文件，并且解决了格式有问题的情况

3、配置push github.com

测试需要，要push到github上的但是一直没有成功，后面发现是网络问题

```html

问题分析：
网络问题：你的网络环境屏蔽了 GitHub 的 22 端口
配置问题：虽然配置了 SSH 使用 443 端口，但远程仓库地址还是 HTTPS 格式
协议不匹配：HTTPS 地址会忽略 SSH 配置，仍然要求输入用户名密码
解决方案：
SSH 配置：已配置 ~/.ssh/config 让 github.com 走 443 端口
远程地址更改：将 https://github.com/xxx/strap.git 改为 git@github.com:xxx/strap.git
连接测试：SSH 连接成功，显示 "Hi xxx! You've successfully authenticated"
推送成功：代码已成功推送到 GitHub
关键点：
SSH 配置只对 SSH 格式的远程地址生效
HTTPS 格式的远程地址会忽略 SSH 配置
需要手动更改远程仓库地址格式
现在你的 git push 应该可以正常工作了，不会再要求输入用户名密码！

cat > ~/.ssh/config << 'EOF'
Host github.com
    HostName ssh.github.com
    Port 443
    User git
    IdentityFile ~/.ssh/id_rsa
    IdentitiesOnly yes
EOF

上面配置github.com走443端口

设置配置文件权限
chmod 600 ~/.ssh/config

添加 GitHub 主机指纹（避免交互确认）
ssh-keyscan -p 443 ssh.github.com >> ~/.ssh/known_hosts
chmod 600 ~/.ssh/known_hosts


再用 ssh -T git@github.com 试一下

git remote -v 看下远程格式

origin  https://github.com/xxx/strap.git (fetch)
origin  https://github.com/xxx/strap.git (push)

问题找到了！你的远程仓库地址是 HTTPS 格式（https://github.com/xxx/strap.git），所以即使配置了 SSH，Git 还是会使用 HTTPS 协议，需要输入用户名和密码。
需要将远程仓库地址改为 SSH 格式。执行以下命令：

git remote set-url origin git@github.com:xxx/strap.git

git remote -v 再看一下

这样就能git push了

```
