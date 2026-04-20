---
layout: Pytorch
title: Pytorch相关
tags: [机器学习, Pytorch]
author-id: zqmalyssa
---

PyTorch可以单独拿出来说下

#### 基本的使用

1、permute的使用

```html

# 模拟一张图片：高=480, 宽=640, 通道=3 (RGB)
img = torch.randn(480, 640, 3)
print(img.shape)  # torch.Size([480, 640, 3])

# 维度索引对应：0->H(480), 1->W(640), 2->C(3)
# 我们想要变成 (C, H, W)，即索引顺序变成 (2, 0, 1)
new_img = img.permute(2, 0, 1)

print(new_img.shape) # torch.Size([3, 480, 640])

```

#### 使用问题

1、降版本引起

```html

pip install torch==1.12.0
pip install torchvision==0.13.0

# torch.repeat_interleave 这个算子在 MPS 上还没实现，所以系统会自动把这部分计算挪到 CPU 上跑

# 新版本 torch==2.8.0 和 torchvision==0.23.0 可以解决

```


#### 小技巧
