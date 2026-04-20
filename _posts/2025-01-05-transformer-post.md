---
layout: post
title: transformer相关
tags: [transformer]
author-id: zqmalyssa
---

终于到你了

#### 使用相关

| 加载器 | 用途 | 输入 | 输出 | 典型任务 |
| :----: | :----:  | :----: | :----: | :----: |
| AutoModel | 只加载基础transformer encoder / decoder | token_ids, attention mask | BaseModelOutput(hidden states) | 获取中间表示 / 做自定义任务 / feature extraction |
| AutoModelForSequenceClassification | 加载用于分类任务的模型（Encoder + 分类头） | token_ids, attention mask | logits[batch, num_labels] | 情感分析、文本分类、新闻分类等 |
| AutoModelForTokenClassification | 加载用于序列标注（NER） | token_ids, attention mask | logits[batch, seq_len, num_labels] | NER、POS tagging |
| AutoModelForQuestionClassification | 加载用于回答任务的模型 | token_ids, attention mask | start_logits, end_logits[batch, seq_len] | SQuAD QA，Span extraction |
| AutoModelForMaskedLM | 加载用于掩码语言模型（MLM） | token_ids, attention mask | logits[batch, seq_len, vocab_size] | 预训练/领域适应/填空任务 |
| AutoModelForCausalLM | 加载用于自回归语言模型（生成式） | token_ids, attention mask | logits[batch, seq_len, vocab_size] | GPT-style文本生成、续写、对话 |
| AutoModelForSeq2SeqLM | Encoder-Decoder模型（翻译/摘要） | input_ids, attention mask, decoder_input_ids | logits[batch, target_seq_len, vocab_size] | 机器翻译、文本摘要、T5/BART生成任务 |
| AutoPeftModelForCausalLM | 用于LoRA/PEFT微调的自回归生成模型 | token_ids, attention mask | logits[batch, seq_len, vocab_size] | 快速微调大模型（LoRA、Adapter、Prefix-tuning） |


AutoModel就是仅仅输出特征向量的，AutoModelForCausalLM就是有头的，如GPT、LLama、Qwen等，

深度学习一般就是主干（backbone/body）提取特征，头部（head）处理具体任务
