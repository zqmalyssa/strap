---
layout: post
title: 数据集
tags: [dataset]
author-id: zqmalyssa
---

说一些和dataset相关的

#### dataset的制作

- 第一步，清洗和标准化
- 第二步，分片与压缩
- 第三步，HF的dataset进行封装

#### 与任务相关的

- 双塔检索/向量模型的 推荐格式是 parquet（二进制格式的），推荐字段是 query，pos_answer（正例），neg_answer（负例）
- 大模型对话/指令 推荐格式是 JSONL，推荐字段是 instruction，input，output
- 纯文本预训练（如Bert），TEXT / JSONL，推荐字段是 text
