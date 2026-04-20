---
layout: post
title: openclaw相关
tags: [openclaw]
author-id: zqmalyssa
---

openclaw相关，能搭建一个像贾维斯一样的智能体，有skills，其实结合了之前的agent 和 workflow

#### 搭建过程

1、使用一个云服务搭建，如果要0成本，自己部署大模型

```html

1、更新刚拿到的ubuntu

apt update && apt upgrade -y

2、创建一个专属的用户，一些工具 和 命令无法通过root使用

adduser openclaw

usermod -aG sudo openclaw

su - openclaw

3、安装一些前置，包括nodejs，pnpm这些

# 下載並安裝 nvm：
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash

# 不想重新啟動 shell 時，執行：
\. "$HOME/.nvm/nvm.sh"

# 下載並安裝 Node.js：
nvm install 24

# Verify the Node.js version:
node -v # Should print "v24.14.0".

# 下載並安裝 pnpm：
corepack enable pnpm

# 核對 pnpm 版本：
pnpm -v

4、安装homebrew，用来装skills

/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 根据next step的指引来的
echo >> /home/openclaw/.bashrc
echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv bash)"' >> /home/openclaw/.bashrc
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv bash)"

sudo apt-get install build-essential

# 最后查看
brew --version

5、安装claude code，看看skip？？

6、正式安装openclaw，使用hackable的方式

git clone https://github.com/openclaw/openclaw.git

cd openclaw && pnpm install && pnpm run build

# You built it, now meet it
pnpm run openclaw onboard

7、配置

8、访问

http://127.0.0.1:18789/#token=zqmopenclaw

或者
ssh -N -L 18789:127.0.0.1:18789 openclaw@<host>


```

2、中文版，外版还是需要tailscale，claude code等等资源的，尝试看看中文版呢

```html

默认已经有了node环境

1、安装

a，从源码安装

git clone https://github.com/jiulingyun/openclaw-cn.git
cd openclaw-cn

pnpm install
pnpm ui:build
pnpm build

pnpm openclaw-cn onboard --install-daemon

b，npm安装

# 安装
npm install -g openclaw-cn@latest

# 运行安装向导
openclaw-cn onboard --install-daemon

# 启动网关
openclaw-cn gateway --port 18789 --verbose

2、进行ds和飞书的配置，飞书的有一步需要连接来了才能配置

ss -tulpn | grep 18789  

tcp   LISTEN 0      511                127.0.0.1:18789      0.0.0.0:*    users:(("clawdbot-gatewa",pid=21360,fd=23))
tcp   LISTEN 0      511                    [::1]:18789         [::]:*    users:(("clawdbot-gatewa",pid=21360,fd=24))

服务是绑定在本地回环地址上的（Loopback），没有监听内网网卡地址

openclaw的配置文件可以 把host的 127.0.0.1 修改成 0.0.0.0


3、把有些云服务（如aws）的systemd修复一下

systemctl --user status  （Failed to connect to bus: No medium found）

# 以上这种事有问题的，
# 注意pnpm openclaw-cn的命令需要到openclaw-cn的目录下去执行

pnpm openclaw-cn gateway status

# 上面的命令诊断的

systemd 用户服务不可用。
systemd 用户服务不可用；请安装/启用 systemd，或使用其他进程管理器运行网关。
如果在容器中运行，请使用前台模式: openclaw-cn gateway run

# 进行修复
su - openclaw
export XDG_RUNTIME_DIR=/run/user/$(id -u)
sudo systemctl start user@$(id -u).service

# 再验证
systemctl --user status

# 重新启动龙虾的gateway

pnpm openclaw-cn doctor --repair

pnpm openclaw-cn gateway start

pnpm openclaw-cn gateway restart

4、远端连接dashboard

# 在远端这样子连（aws默认不账密，用.pem，秘钥）
# Permission denied(publickey)，aws拒绝了连接

ssh -i ~/.ssh/my_awskp.pem -N -L 18789:127.0.0.1:18789 ubuntu@ip

5、如果显示1008的错误，健康状态 离线

把token加进去，在概览里面，然后连接一下

6、与飞书配对（注意这里不需要特别的网络，比如127 或者 使用tailscale等等，当时与飞书的消息连接在启动gateway的时候建立好了已经？？消息通道哦！！）

跟飞书消息通道创建的进行连接

Openclaw: 访问未配置。
ou_3c174733edcc9396f19bf089883e9271
配对码: XXXXX
请让机器人所有者执行以下命令批准:
openclaw-cn pairing approve feishu XXXXX

# 执行下面的命令
pnpm openclaw-cn pairing approve feishu D55MZJTK

# 得到下面的结果，这样飞书的私信就已经ok了
Approved feishu sender ou_3

7、飞书的群聊


```

#### 使用实践

1、在dashboard让它帮我安装skills

```html

帮我安装一下这个skills：https://clawhub.ai/Jacky1n7/openclaw-tavily-search，里面用到的api_key 是 tvly-dev-xxx

太好了！技能已经成功安装并可以正常工作。现在让我总结一下安装情况：

技能安装完成！
我已经成功为你安装并配置了 Tavily Search 技能。以下是安装的详细信息：

安装位置
/home/openclaw/.openclaw-cn/workspace/skills/tavily-search/

包含的文件
SKILL.md - 技能使用说明
tavily_search.py - 主要的 Python 模块
tavily-cli.sh - 命令行工具
simple_test.py - API 测试脚本
README.md - 完整文档
package.json - 技能配置文件

API 配置
已预配置你的 API key: tvly-dev-xxx

能够说明配置的很成功了

# 然后在workspace的TOOL.md文件中加上一句话

# TOOLS.md - Local Notes

Skills define *how* tools work. This file is for *your* specifics — the stuff that's unique to your setup.

## Web Search
**When performing web searches, prioritize using the tavily-search skill.**

# 也就是把搜索事交给tavily去做，免费的

也可以试试直接写

Use the openclaw-tavily-search skill as the top-priority search tool when do web search.

测试：

苹果在2026年的3月份发布了哪些新的产品

我来使用刚刚安装的 Tavily Search 技能来搜索苹果在2026年3月发布的新产品。

exec
cd ~/.openclaw-cn/workspace/skills/tavily-search && ./tavily-cli.sh "苹果公司在2026年3月发布了哪些新产品"
Completed

```

2、配置一个定时任务，生成每日简报

```html

# 基于 https://github.com/AlexAnys/awesome-openclaw-usecases-zh/blob/main/usecases/custom-morning-brief.md

openclaw cron add \
  --name "morning-brief-cn" \
  --cron "30 7 * * 1-5" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --message "生成今日早间简报并发送到飞书" \
  --announce \
  --channel feishu

# command

pnpm openclaw-cn cron add \
    --name "morning-brief-cn" \
    --cron "30 8 * * 1-5" \
    --tz "Asia/Shanghai" \
    --session isolated \
    --message "生成今日早间简报并发送到飞书，关注AI、创业、科技方向，关注中美科技大厂动态，同时告诉我上海今天的天气" \
    --announce \
    --channel feishu

```

3、自己安装skills

```html

npm i -g clawhub

clawhub install "openclaw-tavily-search" # 安装skills的名字，会装到workspace中的

```

#### 进行配置

1、主要的配置入口

```html

pnpm openclaw-cn config

pnpm openclaw logs --follow # 查看日志

```

2、它是可以多agent的，每个agent一个workspace

```html

pnpm openclaw-cn agents add overseas

# 然后里面配置一个新的channel，比如telegram，

# ID是数字，token是机器人给你的

# 跟机器人对话/start，匹配一下，拿一个授权码，XXYY

pnpm openclaw-cn pairing approve telegram XXYY

# 上面跑一下 telegram 就能work了

# /model可以查看模型，切换模型

```

3、配置自定义模型

```html

# 直接 pnpm openclaw-cn config 可以配置，但是配置完后，好像默认变成custom的了

# 可以

/model
/model list
/model status

这样切换着来，就又能切换回去了

/model deepseek

# 使用custom

/model custom

在界面可以设置模型，总是掉不通，endpoint可以设置成

"custom": {
  "baseUrl": "http://custom_domain/v1",
  "apiKey": "custom",
  "api": "ollama",
  "models": [
    {
      "id": "qwen3:30b",
      "name": "qwen3:30b",
      "reasoning": false,
      "input": [
        "text"
      ],
      "cost": {
        "input": 0,
        "output": 0,
        "cacheRead": 0,
        "cacheWrite": 0
      },
      "contextWindow": 128000,
      "maxTokens": 4096
    }
  ]
}

# 一定注意api就是ollama，然后endpoint可以是http或者https，这样就通了！！ 从配置界面搞的api一直会是openai-completions，不适配远端的ollama

```

#### 安全设置

1、在每次执行exec命令的时候，都防御一下

```html

# 就是每次执行的时候直接跳窗防御

```

2、

#### 背后技术

1、sub-agent，就是子agent，可以节省context-window，而跟context有关的可以衍生出一个工种，context-engineering

```html

# 假设大龙虾要比较两篇论文的摘要，如果全程一个人和语言模型交互，context windows会很长

# 弄出两个sub-agent，分别读A 和 B的摘要，交给大龙虾比较，就节省了context-window，没有互动，没有论文全文等


```

2、比如skills，相当于各家写好的SOP，那么龙虾跟语言模型交互的时候，会把这些skill.md相关的东西放入 system prompt里面，然后让语言模型去决定选择哪一个skill，而不是把skill的内容全部贴进来，也是节省context-window。

clawhub的地方可以交换skills

3、龙虾是24小时运行的，如果上下文太长的话，其实是需要清除一下的，就是在chat的界面里面，有一个new session，就可以清除了

4、心跳机制，每隔一段时间，发文字给语言模型，让它说点什么，固定指令（heartbeat）

5、context-compression，如果快要超出 context-windows的话，启动compression，把旧的context扔给语言模型变成摘要，压缩一下，可以不断的套娃摘要

6、关于记忆，龙虾的短长期记忆是要实际写到 memory 里面，它只是一个壳子，不能确定它到底记住没有，只有真正写到memory.md的才算是记住了
