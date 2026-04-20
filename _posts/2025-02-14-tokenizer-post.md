---
layout: post
title: tokenizer
tags: [tokenizer]
author-id: zqmalyssa
---

分词器相关，NLP里面最关键的组件之一，它的核心作用是将人类可读的文本（字符串）转换为模型可以处理的数字序列（通常是整数ID）。这一过程包含两个关键步骤：

- 分词（Tokenization）：将文本拆分成更小的单元（称为token）
- 编码（Encoding）：将每个token映射为一个唯一的整数ID

分词的方式和算法

tokenizer的分词方式决定了文本如何被拆分，主流的有

| 分词方式 | 特点 | 适用场景 | 代表模型 |
| :----: | :----:  | :----: |  :----: |
| word-level | 基于空格分词 | 英文等空格分隔语言 | 早期NLP模型 |
| character-level | 按字符分词 | 适合处理未登录词| 早期RNN模型 |
| subword-level | 按子词分词（如BPE、WordPiece、SentencePiece） | 主流方案，平衡词汇表大小与OOV问题 | BERT、GPT、RoBERTa |
| byte-level | 按字节分词 | 多语言支持 | GTP2/3 |


Subword分词算法详解

(1) Byte Pair Encoding (BPE)
  - 工作原理：
    - 从字符级开始（如英文的26个字母、标点符号）
    - 统计训练语料中词频
    - 迭代合并出现频率最高的相邻字符对
    - 重复此过程直到达到预设词表大小
  - 示例："unhappiness" → ["un", "happi", "ness"]

(2) WordPiece
  - 工作原理：
    - 类似BPE，但基于概率选择合并
    - 优先合并能提高语言模型概率的词对
    - 适用于中文等没有明显空格分隔的语言
  - 示例："自然语言处理" → ["自", "然", "语", "言", "处", "理"]

(3) Unigram
  - 工作原理：
    - 从大词汇表开始
    - 逐步剔除低概率token
    - 基于最大似然估计选择最优分词
  - 能处理更复杂的语言结构

(4) SentencePiece (word piece的变种，google使用)
  - 无需空格，可处理没有空格的语言（如中文）
  - 经常与transformer模型搭配

中文分词的难点：未登录词，歧义处理，粒度选择等


BERT模型中的Tokenizer

BERT（Bidirectional Encoder Representations from Transformers）模型的Tokenizer有以下关键特征
  - 自动添加特殊标记：
    - [CLS]（ID=101）：分类任务的起点，模型最终输出的特征向量来源
    - [SEP]（ID=102）：句子结束标记，单句任务也必须存在
    - [PAD]（ID=0）：填充标记，用于批量处理时对齐序列长度
  - 输入格式：[CLS] + 文本 + [SEP]

GPT模型中的Tokenizer

GPT（Generative Pre-trained Transformer）模型的Tokenizer特点：
  - 无[SEP]标记：GPT是自回归模型（decoder-only），不需要句子分隔符（一直向前看的，不会同时看两个句子，不存在句子对的概念，bert就是非自回归的，输入是句子对，双向编码（encoder））
  - 前导空格处理：许多单词的token包含前导空格
  - 大小写敏感：对大小写敏感（如"Hello"和"hello"是不同token）

GPT的训练目标是next token，而bert的训练目标是MLM + NSP

#### 中文分词器

Jieba 是一个开源的中文分词器，支持多种分词模式，包括精确模式、全模式和搜索引擎模式。

NLTK（Natural Language Toolkit）是 Python 中的一个自然语言处理库，提供了丰富的 NLP 工具和资源。NLTK 包含多个分词器，适用于多种语言，如英文、德文等。

spaCy 是一个高性能的 NLP 库，支持多种语言的分词。spaCy 使用了预训练的模型，可以快速准确地进行分词、词性标注等任务。

Stanford CoreNLP 是斯坦福大学开发的一个 NLP 工具包，提供了多种 NLP 功能，包括分词、词性标注、命名实体识别等。

HanLP 是一个开源的中文处理工具包，提供了多种 NLP 功能，包括分词、词性标注、命名实体识别等。

#### d2l相关

在深度学习中，常用的词元化（tokenization）方法包括以下几种：

空格分词（Whitespace Tokenization）：将文本按照空格进行分割，将每个分割后的词作为一个词元。例如，"Hello world!"会被分割为[“Hello”, “world!”]。
分词器（Tokenizer）：使用专门的分词器工具，如NLTK（Natural Language Toolkit）或spaCy，在文本中识别和分割单词。这些分词器可以处理更复杂的分割规则，例如处理标点符号、缩写词等。
n-gram分词：将文本切分为连续的n个词的组合，这些组合称为n-gram。常见的是二元（bigram）和三元（trigram）分词。例如，"Hello world!"的bigram分词为[“Hello world”, “world!”]。
字符级别分词（Character-level Tokenization）：将文本按照字符进行分割，将每个字符作为一个词元。这种方法对于处理字符级别的任务（如拼写检查、机器翻译等）很有用。
子词（Subword）分词：将单词切分为更小的单元，如词根、前缀或后缀等。这种方法可以处理未登录词（out-of-vocabulary）问题，并且对于具有复杂形态的语言（如德语、芬兰语）也很有效。常见的子词分词算法有Byte-Pair Encoding（BPE）和SentencePiece。
这些方法的选择取决于特定的任务和语言，不同的词元化方法可能适用于不同的场景。在使用深度学习进行文本处理时，需要根据具体情况选择合适的词元化方法。
