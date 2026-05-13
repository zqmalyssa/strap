---
layout: post
title: ML/DL训练
tags: [机器学习, 训练]
author-id: zqmalyssa
---

机器学习、深度学习训练相关

#### mlx-examples

mlx-examples是python写的相关算法训练，这里面实现了到23年一些主流的算法，涉及到了多模态

Text Models

- Transformer language model training.
- Minimal examples of large scale text generation with LLaMA, Mistral, and more in the LLMs directory.
- A mixture-of-experts (MoE) language model with Mixtral 8x7B.
- Parameter efficient fine-tuning with LoRA or QLoRA.
- Text-to-text multi-task Transformers with T5.
- Bidirectional language understanding with BERT.

Image Models

- Generating images
  - FLUX
  - Stable Diffusion or SDXL

- Image classification using ResNets on CIFAR-10.
- Convolutional variational autoencoder (CVAE) on MNIST.

Audio Models

- Speech recognition with OpenAI's Whisper.
- Audio compression and generation with Meta's EnCodec.
- Music generation with Meta's MusicGen.

Multimodal models

- Joint text and image embeddings with CLIP.
- Text generation from image and text inputs with LLaVA.
- Image segmentation with Segment Anything (SAM).

Other Models

- Semi-supervised learning on graph-structured data with GCN.
- Real NVP normalizing flow for density estimation and sampling.

#### 训练技巧

1、对于一个未知的数据，假设输入128，输出是个2，其实可以先试下线性的，然后增加一个隐藏层的（比如神经元是16），再试32，再试64，再试128都行，比如16、128都不行的话，32，64还可以，那么会加入第二个隐藏层，32后面 加个8

2、如何设计超参数，是不是需要搜索，搜索用贝叶斯还是网格、随机，推荐的话是靠专家经验，推荐随机，就是选一些参数的组合，然后试着去炼丹，然后选一个最好的，hpo的领域，感觉可以在验证阶段进行参数的调整

3、权重衰退使用的地方可能还更多，droupout只能用于全连接层，权重衰退那个lambd不是很好调，dropout这个0.1，0.5，0.9还是很好设置的，假设我一个模型单隐藏层64神经元，感觉训练下来没什么问题（不会过拟合），那么我们就能换成128神经元然后开dropout等于0.5，就相当于一半被丢弃掉了呗，也就是dropout调起来比较方便。dropout会影响lr么？？还是期望的问题，不太会影响，但是会影响收敛速度，速度会变慢

4、epoch数、batch_size、学习率是相互 相关的，根据你的内存情况调batch_size，epoch观察一下，收敛的话就停掉。调batch_size，看你的处理样本数能否维持到某个数目

5、预训练bert的两个任务，遮蔽语言模型loss一直压不下去，还缓慢上升了，到4.9左右，而nsp任务的不降，一直0.69

```html

current step is: 1000
MLM loss 5.097, NSP loss 0.703
current step is: 2000
MLM loss 5.077, NSP loss 0.701
current step is: 3000
MLM loss 5.004, NSP loss 0.700
current step is: 4000
MLM loss 4.965, NSP loss 0.700
current step is: 5000
MLM loss 4.935, NSP loss 0.699
current step is: 6000
MLM loss 4.912, NSP loss 0.698
current step is: 7000
MLM loss 4.895, NSP loss 0.697
current step is: 8000
MLM loss 4.884, NSP loss 0.697
current step is: 9000
MLM loss 4.874, NSP loss 0.696
current step is: 10000
MLM loss 4.873, NSP loss 0.697
current step is: 11000
MLM loss 4.870, NSP loss 0.697
current step is: 12000
MLM loss 4.865, NSP loss 0.697
current step is: 13000
MLM loss 4.860, NSP loss 0.698
current step is: 14000
MLM loss 4.857, NSP loss 0.698
current step is: 15000
MLM loss 4.854, NSP loss 0.698
current step is: 16000
MLM loss 4.852, NSP loss 0.698
current step is: 17000
MLM loss 4.851, NSP loss 0.698
current step is: 18000
MLM loss 4.852, NSP loss 0.698
current step is: 19000
MLM loss 4.861, NSP loss 0.698
current step is: 20000
MLM loss 4.865, NSP loss 0.698

[Checkpoint] Step 20000 已保存至 bert_checkpoint/bert_step_20000.pth
current step is: 21000
MLM loss 4.866, NSP loss 0.698
current step is: 22000
MLM loss 4.867, NSP loss 0.698
current step is: 23000
MLM loss 4.869, NSP loss 0.698
current step is: 24000
MLM loss 4.869, NSP loss 0.698
current step is: 25000
MLM loss 4.871, NSP loss 0.698
current step is: 26000
MLM loss 4.873, NSP loss 0.698
current step is: 27000
MLM loss 4.875, NSP loss 0.698
current step is: 28000
MLM loss 4.876, NSP loss 0.698
current step is: 29000
MLM loss 4.877, NSP loss 0.698
current step is: 30000
MLM loss 4.881, NSP loss 0.698
current step is: 31000
MLM loss 4.886, NSP loss 0.698
current step is: 32000
MLM loss 4.889, NSP loss 0.698
current step is: 33000
MLM loss 4.895, NSP loss 0.698
current step is: 34000
MLM loss 4.901, NSP loss 0.698
current step is: 35000
MLM loss 4.904, NSP loss 0.698
current step is: 36000
MLM loss 4.905, NSP loss 0.698
current step is: 37000
MLM loss 4.904, NSP loss 0.698
current step is: 38000
MLM loss 4.904, NSP loss 0.698
current step is: 39000
MLM loss 4.905, NSP loss 0.698
current step is: 40000
MLM loss 4.907, NSP loss 0.698

[Checkpoint] Step 40000 已保存至 bert_checkpoint/bert_step_40000.pth
current step is: 41000
MLM loss 4.908, NSP loss 0.698
current step is: 42000
MLM loss 4.909, NSP loss 0.698
current step is: 43000
MLM loss 4.910, NSP loss 0.698
current step is: 44000
MLM loss 4.909, NSP loss 0.698
current step is: 45000
MLM loss 4.909, NSP loss 0.698
current step is: 46000
MLM loss 4.911, NSP loss 0.698
current step is: 47000
MLM loss 4.909, NSP loss 0.698
current step is: 48000
MLM loss 4.908, NSP loss 0.698
current step is: 49000
MLM loss 4.907, NSP loss 0.698
current step is: 50000
MLM loss 4.907, NSP loss 0.697
current step is: 51000
MLM loss 4.907, NSP loss 0.697

```

给的建议是模型太窄且浅，128的num_hiddens 可以调整到256试试，num_layers到4或6，放弃掉NSP的任务，学习率设置成1e-5，采用warmup

使用 AdamW： 预训练 BERT 务必使用 AdamW 优化器，它比普通的 SGD 或 Adam 在处理权重衰减（Weight Decay）上表现更好。

1、先做的调整，将lr从0.01降到1e-4

2、引入lambdaLR 实现线性的warmup和衰减

3、学习率的调度器也能同步恢复

4、修正adam成adamW

```html

[Checkpoint] Step 70000 已保存至 bert_checkpoint/bert_step_70000.pth
current step is: 71000
MLM loss 4.859, NSP loss 0.695
current step is: 72000
MLM loss 4.858, NSP loss 0.695
current step is: 73000
MLM loss 4.858, NSP loss 0.695
current step is: 74000
MLM loss 4.858, NSP loss 0.695
current step is: 75000
MLM loss 4.857, NSP loss 0.695
current step is: 76000
MLM loss 4.857, NSP loss 0.695
current step is: 77000
MLM loss 4.857, NSP loss 0.694
current step is: 78000
MLM loss 4.857, NSP loss 0.694
current step is: 79000
MLM loss 4.856, NSP loss 0.694
current step is: 80000
MLM loss 4.856, NSP loss 0.694

[Checkpoint] Step 80000 已保存至 bert_checkpoint/bert_step_80000.pth
current step is: 81000
MLM loss 4.856, NSP loss 0.694
current step is: 82000
MLM loss 4.856, NSP loss 0.694

```

4.85多已经是模型的极限了，而且nsp的loss一直不降，在0.694左右，之前的checkpoint需要删掉，现在扩大模型的容量

num_hiddens 原来 128 现在 256，num_layers原来2，现在 4，num_heads原来2，现在4

另外，如果num_hiddens 改成了256，那么后面很多参数原来是128的都要改成256，batch_size还是原来的512

```html

current step is: 1000
MLM loss 7.751, NSP loss 0.695
current step is: 2000
MLM loss 6.483, NSP loss 0.694
current step is: 3000
MLM loss 5.914, NSP loss 0.693
current step is: 4000
MLM loss 5.604, NSP loss 0.691
current step is: 5000
MLM loss 5.396, NSP loss 0.684
current step is: 6000
MLM loss 5.236, NSP loss 0.671
current step is: 7000
MLM loss 5.100, NSP loss 0.649
current step is: 8000
MLM loss 4.976, NSP loss 0.619
current step is: 9000
MLM loss 4.855, NSP loss 0.584
current step is: 10000
MLM loss 4.734, NSP loss 0.547

[Checkpoint] Step 10000 已保存至 bert_checkpoint/bert_step_10000.pth
current step is: 11000
MLM loss 4.613, NSP loss 0.511
current step is: 12000
MLM loss 4.494, NSP loss 0.478
current step is: 13000
MLM loss 4.378, NSP loss 0.448
current step is: 14000
MLM loss 4.267, NSP loss 0.422
current step is: 15000
MLM loss 4.160, NSP loss 0.398
current step is: 16000
MLM loss 4.058, NSP loss 0.377
current step is: 17000
MLM loss 3.960, NSP loss 0.358
current step is: 18000
MLM loss 3.866, NSP loss 0.340
current step is: 19000
MLM loss 3.777, NSP loss 0.325
current step is: 20000
MLM loss 3.691, NSP loss 0.310

[Checkpoint] Step 20000 已保存至 bert_checkpoint/bert_step_20000.pth
current step is: 21000
MLM loss 3.609, NSP loss 0.297
current step is: 22000
MLM loss 3.530, NSP loss 0.285
current step is: 23000
MLM loss 3.455, NSP loss 0.274
current step is: 24000
MLM loss 3.383, NSP loss 0.264
current step is: 25000
MLM loss 3.313, NSP loss 0.254
current step is: 26000
MLM loss 3.247, NSP loss 0.246
current step is: 27000
MLM loss 3.183, NSP loss 0.237
current step is: 28000
MLM loss 3.121, NSP loss 0.230
current step is: 29000
MLM loss 3.062, NSP loss 0.223
current step is: 30000
MLM loss 3.005, NSP loss 0.216

[Checkpoint] Step 30000 已保存至 bert_checkpoint/bert_step_30000.pth
current step is: 31000
MLM loss 2.951, NSP loss 0.210
current step is: 32000
MLM loss 2.898, NSP loss 0.204
current step is: 33000
MLM loss 2.847, NSP loss 0.198
current step is: 34000
MLM loss 2.798, NSP loss 0.193
current step is: 35000
MLM loss 2.750, NSP loss 0.188
current step is: 36000
MLM loss 2.704, NSP loss 0.183
current step is: 37000
MLM loss 2.660, NSP loss 0.179
current step is: 38000
MLM loss 2.617, NSP loss 0.175
current step is: 39000
MLM loss 2.576, NSP loss 0.171
current step is: 40000
MLM loss 2.535, NSP loss 0.167

[Checkpoint] Step 40000 已保存至 bert_checkpoint/bert_step_40000.pth
current step is: 41000
MLM loss 2.496, NSP loss 0.163

```

以上效果非常明显，先断掉再从checkpoint加载，但是训练有所波动，loss在往回升

```html

current step is: 41000
MLM loss 2.566, NSP loss 0.177
current step is: 42000
MLM loss 2.584, NSP loss 0.183
current step is: 43000
MLM loss 2.597, NSP loss 0.184
current step is: 44000
MLM loss 2.605, NSP loss 0.184

```

可能造成上面的原因是：

1、优化器状态的“冷启动” (Optimizer States)，虽然你加载了 trainer_state_dict，但 AdamW 记录的是过去的梯度动量（Momentum）和平方梯度均值（Variance）。当你中断训练并重启时，数据加载器（DataLoader）通常会从随机点重新开始扫数据。如果重启后的前几个 Batch 包含了一些模型之前没见过的、或者难度较大的样本，而此时优化器的动量还是基于断点前的状态，就会产生短暂的“不适应”，导致 Loss 波动。

2、指标累加器（Accumulator）被重置了，metric = d2l.Accumulator(4)。如果你在恢复时，metric.data 里的数值是累加的（比如从 Step 0 到 40000 的平均值），那么 41000 步时的 Loss 是全量平均值。但由于你现在的 Loss 已经压得很低了（2.5 左右），而早期的 Loss 很高（7.0 左右），如果累加逻辑处理不当，重启后的计算可能会让均值显得稍微波动。

3、数据顺序的变化，如果你的 train_iter 没有保存随机种子状态，重启后它会重新 Shuffle 数据。断点前： 模型可能刚刚学完了一批比较容易的句子，断点后： 刚好撞上了一批比较难的、或者包含更多 [MASK] 预测难度的句子。

4、！！还有词表的ID对不上了，因为每次重新训练都要读数据，会做random.shuffle()，导致词表的对应不一致了（这个关注一下要），词表应该还行，每次对应的上，所以下面的loss也能继续压下去 （代码也加上试试吧）

观察后面的训练结果：

```html

current step is: 45000
MLM loss 2.610, NSP loss 0.182
current step is: 46000
MLM loss 2.611, NSP loss 0.180
current step is: 47000
MLM loss 2.610, NSP loss 0.177
current step is: 48000
MLM loss 2.606, NSP loss 0.175
current step is: 49000
MLM loss 2.600, NSP loss 0.172
current step is: 50000
MLM loss 2.592, NSP loss 0.170
[Checkpoint] Step 50000 已保存至 bert_checkpoint/bert_step_50000.pth
current step is: 51000
MLM loss 2.582, NSP loss 0.167
current step is: 52000
MLM loss 2.571, NSP loss 0.165
current step is: 53000
MLM loss 2.559, NSP loss 0.162
current step is: 54000
MLM loss 2.546, NSP loss 0.160
current step is: 55000
MLM loss 2.533, NSP loss 0.158
current step is: 56000
MLM loss 2.519, NSP loss 0.155
current step is: 57000
MLM loss 2.504, NSP loss 0.153
current step is: 58000
MLM loss 2.489, NSP loss 0.151
current step is: 59000
MLM loss 2.473, NSP loss 0.149
current step is: 60000
MLM loss 2.458, NSP loss 0.147
[Checkpoint] Step 60000 已保存至 bert_checkpoint/bert_step_60000.pth
current step is: 61000
MLM loss 2.442, NSP loss 0.145
current step is: 62000
MLM loss 2.426, NSP loss 0.143
current step is: 63000
MLM loss 2.410, NSP loss 0.141
current step is: 64000
MLM loss 2.394, NSP loss 0.139
current step is: 65000
MLM loss 2.378, NSP loss 0.137
current step is: 66000
MLM loss 2.361, NSP loss 0.135
current step is: 67000
MLM loss 2.345, NSP loss 0.133
current step is: 68000
MLM loss 2.329, NSP loss 0.132
current step is: 69000
MLM loss 2.313, NSP loss 0.130
current step is: 70000
MLM loss 2.298, NSP loss 0.129
[Checkpoint] Step 70000 已保存至 bert_checkpoint/bert_step_70000.pth
current step is: 71000
MLM loss 2.282, NSP loss 0.127
current step is: 72000
MLM loss 2.266, NSP loss 0.125
current step is: 73000
MLM loss 2.251, NSP loss 0.124
current step is: 74000
MLM loss 2.236, NSP loss 0.122
current step is: 75000
MLM loss 2.221, NSP loss 0.121
current step is: 76000
MLM loss 2.206, NSP loss 0.120
current step is: 77000
MLM loss 2.191, NSP loss 0.118
current step is: 78000
MLM loss 2.176, NSP loss 0.117
current step is: 79000
MLM loss 2.162, NSP loss 0.116
current step is: 80000
MLM loss 2.148, NSP loss 0.114
[Checkpoint] Step 80000 已保存至 bert_checkpoint/bert_step_80000.pth
current step is: 81000
MLM loss 2.134, NSP loss 0.113
current step is: 82000
MLM loss 2.120, NSP loss 0.112
current step is: 83000
MLM loss 2.106, NSP loss 0.111
current step is: 84000
MLM loss 2.093, NSP loss 0.110
current step is: 85000
MLM loss 2.079, NSP loss 0.108
current step is: 86000
MLM loss 2.066, NSP loss 0.107
current step is: 87000
MLM loss 2.053, NSP loss 0.106
current step is: 88000
MLM loss 2.041, NSP loss 0.105
current step is: 89000
MLM loss 2.028, NSP loss 0.104
current step is: 90000
MLM loss 2.016, NSP loss 0.103

[Checkpoint] Step 90000 已保存至 bert_checkpoint/bert_step_90000.pth


```

另外，不同规模 BERT 的标准参考

BERT-Base (Google 原版): 在超大规模语料（维基百科+书籍）上预训练，MLM Loss 通常能降到 1.5 - 2.0 左右。
小型 BERT / 垂直领域训练:

2.5 - 3.5： 模型开始懂语法，能填出“is/am/are”或简单的名词。

2.0 - 2.5： 模型开始懂语义，能根据上下文区分“苹果”是水果还是手机。（你现在正处于这个优秀的区间）

1.5 - 2.0： 模型极其强悍，能捕捉到非常微妙的修辞和长程逻辑。

如果你的 MLM Loss 降到了 1.0 以下（在 WikiText-2 这种小数据集上），可能会过拟合

但是做predict的时候看着有问题，调试一下

```html

# 看着是练炸了，固定词表后再试一下可以

```

6、训练DPO看看

```html

1%|          | 1/100 [00:31<52:37, 31.90s/it]{'loss': '0.6931', 'grad_norm': '3.356', 'learning_rate': '5e-05', 'entropy': '0.9337', 'num_tokens': '6504', 'logits/chosen': '-0.1965', 'logits/rejected': '-0.6364', 'mean_token_accuracy': '0.7482', 'rewards/chosen': '-2.146e-06', 'rewards/rejected': '-1.812e-06', 'rewards/accuracies': '0.625', 'rewards/margins': '-3.338e-07', 'logps/chosen': '-210', 'logps/rejected': '-186.9', 'epoch': '0.0006221'}
  2%|▏         | 2/100 [01:03<51:49, 31.73s/it]{'loss': '0.6788', 'grad_norm': '4.101', 'learning_rate': '4.95e-05', 'entropy': '0.9772', 'num_tokens': '1.195e+04', 'logits/chosen': '-0.3559', 'logits/rejected': '-0.8508', 'mean_token_accuracy': '0.6901', 'rewards/chosen': '0.003706', 'rewards/rejected': '-0.02532', 'rewards/accuracies': '1', 'rewards/margins': '0.02902', 'logps/chosen': '-285.7', 'logps/rejected': '-252.9', 'epoch': '0.001244'}
  3%|▎         | 3/100 [01:29<47:20, 29.28s/it]{'loss': '0.664', 'grad_norm': '3.695', 'learning_rate': '4.9e-05', 'entropy': '1.066', 'num_tokens': '1.809e+04', 'logits/chosen': '-0.588', 'logits/rejected': '-0.7029', 'mean_token_accuracy': '0.7232', 'rewards/chosen': '0.00286', 'rewards/rejected': '-0.05659', 'rewards/accuracies': '1', 'rewards/margins': '0.05945', 'logps/chosen': '-123.2', 'logps/rejected': '-217', 'epoch': '0.001866'}
  4%|▍         | 4/100 [02:00<47:38, 29.77s/it]{'loss': '0.6578', 'grad_norm': '3.531', 'learning_rate': '4.85e-05', 'entropy': '0.9601', 'num_tokens': '2.32e+04', 'logits/chosen': '-0.8953', 'logits/rejected': '-1.271', 'mean_token_accuracy': '0.7373', 'rewards/chosen': '0.01028', 'rewards/rejected': '-0.06271', 'rewards/accuracies': '0.875', 'rewards/margins': '0.07298', 'logps/chosen': '-205.9', 'logps/rejected': '-190.4', 'epoch': '0.002488'}
  5%|▌         | 5/100 [02:30<47:24, 29.94s/it]{'loss': '0.6296', 'grad_norm': '3.744', 'learning_rate': '4.8e-05', 'entropy': '1.061', 'num_tokens': '2.855e+04', 'logits/chosen': '-0.1584', 'logits/rejected': '-0.6586', 'mean_token_accuracy': '0.6493', 'rewards/chosen': '0.01574', 'rewards/rejected': '-0.1164', 'rewards/accuracies': '1', 'rewards/margins': '0.1321', 'logps/chosen': '-114.4', 'logps/rejected': '-273.5', 'epoch': '0.00311'}
  6%|▌         | 6/100 [03:27<1:01:05, 39.00s/it]{'loss': '0.6238', 'grad_norm': '3.764', 'learning_rate': '4.75e-05', 'entropy': '0.9966', 'num_tokens': '3.58e+04', 'logits/chosen': '-0.2525', 'logits/rejected': '-1.05', 'mean_token_accuracy': '0.6973', 'rewards/chosen': '0.005831', 'rewards/rejected': '-0.1412', 'rewards/accuracies': '1', 'rewards/margins': '0.1471', 'logps/chosen': '-129.1', 'logps/rejected': '-228.3', 'epoch': '0.003733'}
{'loss': '0.6274', 'grad_norm': '3.225', 'learning_rate': '4.7e-05', 'entropy': '0.9588', 'num_tokens': '4.29e+04', 'logits/chosen': '-0.03961', 'logits/rejected': '-0.6496', 'mean_token_accuracy': '0.6753', 'rewards/chosen': '0.01617', 'rewards/rejected': '-0.1244', 'rewards/accuracies': '0.75', 'rewards/margins': '0.1406', 'logps/chosen': '-251.4', 'logps/rejected': '-197.7', 'epoch': '0.004355'}
  8%|▊         | 8/100 [04:41<56:13, 36.67s/it]{'loss': '0.582', 'grad_norm': '3.391', 'learning_rate': '4.65e-05', 'entropy': '1.147', 'num_tokens': '4.811e+04', 'logits/chosen': '-0.4296', 'logits/rejected': '-0.9151', 'mean_token_accuracy': '0.6918', 'rewards/chosen': '-0.00167', 'rewards/rejected': '-0.2467', 'rewards/accuracies': '0.875', 'rewards/margins': '0.2451', 'logps/chosen': '-278', 'logps/rejected': '-190', 'epoch': '0.004977'}
  9%|▉         | 9/100 [05:11<52:25, 34.57s/it]{'loss': '0.5758', 'grad_norm': '3.396', 'learning_rate': '4.6e-05', 'entropy': '0.9697', 'num_tokens': '5.262e+04', 'logits/chosen': '-0.2247', 'logits/rejected': '-0.5282', 'mean_token_accuracy': '0.7139', 'rewards/chosen': '0.01361', 'rewards/rejected': '-0.2398', 'rewards/accuracies': '1', 'rewards/margins': '0.2534', 'logps/chosen': '-130.5', 'logps/rejected': '-186.8', 'epoch': '0.005599'}
 10%|█         | 10/100 [06:16<1:06:13, 44.15s/it]{'loss': '0.5639', 'grad_norm': '3.439', 'learning_rate': '4.55e-05', 'entropy': '0.9872', 'num_tokens': '5.961e+04', 'logits/chosen': '-0.5888', 'logits/rejected': '-0.8567', 'mean_token_accuracy': '0.6675', 'rewards/chosen': '0.02497', 'rewards/rejected': '-0.2622', 'rewards/accuracies': '1', 'rewards/margins': '0.2872', 'logps/chosen': '-182.7', 'logps/rejected': '-243', 'epoch': '0.006221'}


chat template saved in ./dpo_results/checkpoint-100/chat_template.jinja
tokenizer config file saved in ./dpo_results/checkpoint-100/tokenizer_config.json

{'train_runtime': '5082', 'train_samples_per_second': '0.157', 'train_steps_per_second': '0.02', 'train_loss': '0.2308', 'epoch': '0.06221'}


```

这边loss是越小越好，DPO的loss反应了模型对chosen和rejected的概率区分度。

- 偏好相关指标 (Rewards & Margins)

rewards/margins (奖励差值)：这是 chosen 的奖励减去 rejected 的奖励。它必须是正数且持续上升。你的数据从 -3.338e-07（几乎为0）升到了 0.1321，说明模型开始明确“偏爱”好回答了。

rewards/accuracies (奖励准确率)：指在一批数据中，chosen 的得分确实高于 rejected 的比例。你的数据在 0.875 到 1.0 之间波动，说明模型目前的判断非常准。

rewards/chosen & rewards/rejected：分别代表两类回答的隐式奖励分。理想状态下，前者应上升，后者应下降。

- 模型表现指标：

  - logits/chosen & logits/rejected：模型输出的未归一化原始分。

  - entropy (熵)：代表模型输出的确定性。
    - 如果熵急剧下降，说明模型变得非常“武断”，可能会出现模式崩溃（Model Collapse），即只会翻来复去说那几句话。
    - 你的熵在 0.9 到 1.0 之间，非常健康。

  - mean_token_accuracy：预测下一个 token 的准确率。

- 训练工程指标

  - grad_norm (梯度范数)：反映训练是否稳定。如果这个数值突然爆表（比如变成几百上千），说明训练炸了（梯度爆炸）。你的 3-4 左右非常正常。
  - it/s (或 s/it)：在 Mac 上约 30 秒一个 step，对于 1.5B 模型来说速度尚可。

rewards/margins 提升得很快，如果训练到后期 Loss 降得太快（比如接近 0.1），可能会导致模型过度拟合偏好数据，反而丧失了原本的通用对话能力

7、sft一个基座模型看看

用一个基座模型，去训练出一个能chat的模型，模型采用alpaca_gpt4_data_zh数据集，使用LoRA降低训练参数

```html

chat template saved in ./sft_output/checkpoint-3600/chat_template.jinja
tokenizer config file saved in ./sft_output/checkpoint-3600/tokenizer_config.json
{'loss': '1.638', 'grad_norm': '0.2898', 'learning_rate': '3.84e-05', 'entropy': '1.672', 'num_tokens': '1.918e+07', 'mean_token_accuracy': '0.6243', 'epoch': '2.425'}
{'loss': '1.658', 'grad_norm': '0.3009', 'learning_rate': '3.403e-05', 'entropy': '1.694', 'num_tokens': '1.97e+07', 'mean_token_accuracy': '0.6189', 'epoch': '2.49'}
 83%|██████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████▋                                 | 3800/4578 [22:52:14<4:50:58, 22.44s/it]Saving model checkpoint to ./sft_output/checkpoint-3800
loading configuration file config.json from cache at /Users/zqmalyssa/.cache/huggingface/hub/models--Qwen--Qwen2.5-1.5B/snapshots/8faed761d45a263340a0528343f099c05c9a4323/config.json
Model config Qwen2Config {
  "architectures": [
    "Qwen2ForCausalLM"
  ],
  "attention_dropout": 0.0,
  "bos_token_id": 151643,
  "dtype": "bfloat16",
  "eos_token_id": 151643,
  "hidden_act": "silu",
  "hidden_size": 1536,
  "initializer_range": 0.02,
  "intermediate_size": 8960,
  "layer_types": [
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention"
  ],
  "max_position_embeddings": 131072,
  "max_window_layers": 28,
  "model_type": "qwen2",
  "num_attention_heads": 12,
  "num_hidden_layers": 28,
  "num_key_value_heads": 2,
  "pad_token_id": null,
  "rms_norm_eps": 1e-06,
  "rope_parameters": {
    "rope_theta": 1000000.0,
    "rope_type": "default"
  },
  "sliding_window": null,
  "tie_word_embeddings": true,
  "transformers_version": "5.5.4",
  "use_cache": true,
  "use_mrope": false,
  "use_sliding_window": false,
  "vocab_size": 151936
}

chat template saved in ./sft_output/checkpoint-3800/chat_template.jinja
tokenizer config file saved in ./sft_output/checkpoint-3800/tokenizer_config.json
{'loss': '1.66', 'grad_norm': '0.287', 'learning_rate': '2.966e-05', 'entropy': '1.7', 'num_tokens': '2.022e+07', 'mean_token_accuracy': '0.6177', 'epoch': '2.556'}
 87%|██████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████▊                         | 3991/4578 [24:05:02<3:19:53, 20.43s/it]{'loss': '1.643', 'grad_norm': '0.2627', 'learning_rate': '2.529e-05', 'entropy': '1.683', 'num_tokens': '2.074e+07', 'mean_token_accuracy': '0.6212', 'epoch': '2.621'}
 87%|███████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████▎                        | 4000/4578 [24:08:23<3:17:30, 20.50s/it]Saving model checkpoint to ./sft_output/checkpoint-4000
loading configuration file config.json from cache at /Users/zqmalyssa/.cache/huggingface/hub/models--Qwen--Qwen2.5-1.5B/snapshots/8faed761d45a263340a0528343f099c05c9a4323/config.json
Model config Qwen2Config {
  "architectures": [
    "Qwen2ForCausalLM"
  ],
  "attention_dropout": 0.0,
  "bos_token_id": 151643,
  "dtype": "bfloat16",
  "eos_token_id": 151643,
  "hidden_act": "silu",
  "hidden_size": 1536,
  "initializer_range": 0.02,
  "intermediate_size": 8960,
  "layer_types": [
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention",
    "full_attention"
  ],
  "max_position_embeddings": 131072,
  "max_window_layers": 28,
  "model_type": "qwen2",
  "num_attention_heads": 12,
  "num_hidden_layers": 28,
  "num_key_value_heads": 2,
  "pad_token_id": null,
  "rms_norm_eps": 1e-06,
  "rope_parameters": {
    "rope_theta": 1000000.0,
    "rope_type": "default"
  },
  "sliding_window": null,
  "tie_word_embeddings": true,
  "transformers_version": "5.5.4",
  "use_cache": true,
  "use_mrope": false,
  "use_sliding_window": false,
  "vocab_size": 151936
}

chat template saved in ./sft_output/checkpoint-4000/chat_template.jinja
tokenizer config file saved in ./sft_output/checkpoint-4000/tokenizer_config.json

```

模型训练到 4000 steps差不多收敛了，然后使用test的代码也能回答了，1.6多的loss最终


#### 代码相关

1、在 macOS 和 Windows 上，Python 启动多进程默认使用 spawn 模式（而不是 Linux 的 fork）。会有死循环问题

```html

# 解决方案1

# 添加 if __name__ == '__main__':

# 2. 所有的执行逻辑必须包在这个判断里
if __name__ == '__main__':
    batch_size, max_len = 512, 64
    train_iter, vocab = d2l.load_data_wiki(batch_size, max_len)

# 解决方案2

设置 num_workers=0，这会强制程序在主进程中读取数据，不再开启多进程，也就不会报错。读取速度会变慢

```

2、好的代码习惯

```html

1、在训练和推理的时候

model.eval() 和 model.train()要切换执行

2、不需要反向传播记梯度的时候

@torch.no_grad() 或者 手动一下


```

#### 训练参数

一些训练参数的总结

```html

Num examples = 48,818  # 数据集总行数
Num Epochs = 3  # 训练的epoch
Num update steps per epoch = 1,526  # 每轮更新多少次参数 48818 / 32 = 1525.56 次step的更新
Instantaneous batch size per device = 16 # batch_size 是 16
Total train batch size (w. parallel, distributed & accumulation) = 32  # 计算方式也是 16 * 2 = 32
Gradient Accumulation steps = 2  # 累加梯度step是2，模型跑2次，（16*2 = 32）才更新一次参数
Total optimization steps = 4,578  # 1526 * 3 = 4578，这个单位就是steps了
Number of trainable parameters = 2,179,072  # 模型参数部分，1.5B一般是15亿参数，而现在只要 218万参数（这是因为LoRA的优化）

```



#### 训练指标

1、为什么ln2是一个分类完全 又不完全的指标呢

```html

如果二分类问题，softmax出来后概率是 0.5的 0.5，

真实标签是1和0，根据交叉熵计算 loss都会是 -ln(0.5) = -ln(1/2) = -(ln1 - ln2) = ln2 约等于 0.693  

其实就是 -ln(0.5) = ln(2) # 倒数

```

2、loss指标相关

```html

如果词表大小是151643左右，那么，纯随机的时候是  -ln(1/151643) 约等于 12左右的loss

在预训练好后，一般loss在 2 点多，假设loss是2，对应的概率 P = e的-2次方 约等于13.5%

在sft之后，loss从2点多降到1.6，e的-1.6次方 约等于 20%，模型学会了对话结构

Loss = -ln(P)

-Loss = ln(P)

e的-loss次方 = P

```
