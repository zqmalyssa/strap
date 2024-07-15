---
layout: post
title: excel
tags: [excel]
author-id: zqmalyssa
---

excel来个文档把

#### 一些常用的东西


=VLOOKUP(B2,O:P,2,FALSE)   // LOOKUP 和 VLOOKUP


```html

 // 正常查找

=VLOOKUP(J2,F:G,2,FALSE)

查找的值在J2内

在F - G 的区域内找，F首列必须有J2的值，

2代表选区域内的第二列

False为精确查找


// 反向查找，值在左边（不在首列）

=VLOOKUP(F5,IF({1,0},B3:B11,A3:A11),2,FALSE)

// 多条件查找，应对重复值

=VLOOKUP(F5&G5,IF({1,0},A3:A11&B3:B11,D3:D11),2,FALSE)




```


="'"&A151&"',"  // mysql包裹一个引号
