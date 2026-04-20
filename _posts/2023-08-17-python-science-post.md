---
layout: post
title: python相关包
tags: [python]
author-id: zqmalyssa
---

python相关的包，尤其是数科相关的总结一下

#### numpy 和 pandas

这两个基本上一起出现，Pandas 是建立在 NumPy 之上的“高级包装”

Pandas 的核心数据结构 Series 和 DataFrame，在底层其实都是由 NumPy 的 ndarray（多维数组）构成的。

NumPy (Numerical Python)：
擅长：纯数值计算、多维数组运算、线性代数。
局限：它要求数组内的所有数据类型必须一致（比如全是数字）；它没有“列名”或“行标签”的概念，只能通过索引（如 [0, 1]）访问。

Pandas (Panel Data)：
擅长：处理表格化数据、清洗数据。
优势：它可以处理混合类型的数据（一列是日期，一列是字符串，一列是数字）；它有强大的索引系统；它能非常方便地处理缺失值（NaN）。
