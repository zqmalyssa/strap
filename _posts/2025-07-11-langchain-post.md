---
layout: post
title: langchain
tags: [langchain]
author-id: zqmalyssa
---

langchain相关

#### 基本概念

#### 相关概率

llm的模块跟 chat_model的区别，当用到代理人，涉及到对话的缓存，调用聊天历史记录的话，chat_model就会比较好用。

代理人，解决一些链式模块难以处理的复杂问题，动态决策，agent。先初始化，再agent.run()。agent调用llm-math的工具去计算x的3.5次方

conversation_chain
