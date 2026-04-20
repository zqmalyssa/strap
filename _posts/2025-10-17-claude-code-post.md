---
layout: post
title: claude-code
tags: [claude-code]
author-id: zqmalyssa
---

claude-code相关

#### 基本使用

```html

npm install -g @anthropic-ai/claude-code@latest


claude --version
ollama --version

ollama大于0.14.0，之后兼容了Anthropic API

ollama pull qwen2.5-coder:14b

# 连接本地大模型

export ANTHROPIC_AUTH_TOKEN=ollama
export ANTHROPIC_BASE_URL=http://localhost:11434

# 如果不export，大概率需要登录，Not logged in · Please run /login

# 启动 Claude Code 并指定本地模型
claude --model qwen2.5-coder:14b

```
