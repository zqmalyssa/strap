---
layout: post
title: tensorflow
tags: [tensorflow]
author-id: zqmalyssa
---

关于机器学习及深度学习的一些总结，代码使用tensorflow

#### 基本使用

```html

import os
os.environ['TF_CPP_MIN_LOG_LEVEL'] = '1'

// 这样可以忽略以下message
This TensorFlow binary is optimized with oneAPI Deep Neural Network Library (oneDNN)to use the following CPU instructions in performance-critical operations:  AVX AVX2

// message说的是不同的硬件会让tensor用不同的代码，支持AVX和AVX2，计算会有额外加速

```
