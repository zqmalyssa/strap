---
layout: post
title: diffusion model
tags: [diffusion model]
author-id: zqmalyssa
---

diffusion model相关

#### 基本概念

古早的时候，y是文字，x是图片，其实要看P(x|y)。在GAN还没有出来的时候，都不知道怎么生成图片。P(x|y)太复杂了，是一个高斯分布，图片模型就是把normal distribution中sample出来的东西对应到狗在奔跑图片。就是把正态分布扭曲成P(x|y)。

vae，encode图片，强迫向量是normal distribution。有encoder 和 decoder

flow-based model，通过encode输出一个向量，这个向量的分布就是normal distribution。把训练的encode反过来用，输出一张图片（从正态分布中sample一张图片）。可以限制网络的（其实只learn了encode）

diffusion model，把一张图片一直add noise，一直add noise，加到原来的图看不出是什么，看起来就像是从正态分布采样出来的一个数据一样，生图片就是learn一个denoise的图片，丢一个从normal distribution中sample出来的vector，然后denoise，denoise，图片就出来了。dall-e等等模型是用diffusion model做的，还加了一些stable diffusion。

GAN，gan相当于只learn decoder（generator），判别模型分辨两个分布。 gan跟上面三个不互斥，所以可以有 vae + gan，flow + gan，diffusion + gan。

diffusion model是什么呢，denoising diffusion probabilistic model。DDPM，dally-e啊，stable diffusion等等都是基于diffusion model。从正态分布中sample出一个vector后，不停denoise（人为定好），最终得到一个图片。denoise model是反复进行使用的，除了正常的输入，还有noise的严重程度。diffusion model里面有noise predicter。就是产生了输入图片的noise，输出的时候把这个nosie给减掉。（end to end，也可以选择这样去生成图片，但是大多数还是减去噪音，因为产生图片和产生noise的难度不一样）。noise predicter要有数据集，数据集怎么来？就是你自己从图片里面加噪音，random sample，加个1000次，叫做forward process 或者 diffusion process。就有noise predicter的训练数据了。groudtruth就是给图片加的noise。但是DDPM的论文中的算法部分的noise是一次性加进去然后训练模型，不是一次一次加。

刚才说明了生图流程，后面就要把文字加进来（text-to-image），输入文字，输出图片，所以还是需要成对的文字-图片数据集。ImageNet 1M，LAION 5.85B的数据集，训练数据中就有中英文。有文字后把文字加到denoise的model就行了。就是输入变成 输入的图片 + 一段文字的描述。noise predicter的输入也加上文字。而groud truth上也要有文字的部分

SOTA，stable diffusion （开源的文本图像生成模型）

大致的framework是 text encoder（第一个model） 和 图片的noise，共同经过generation model（第二个model，也算diffusion model），产生一个中间产物（可以没有意义，是一个图片被压缩后的结果），然后经过decoder（第三个model，还原为原始的图片）。通常三个model分开训练，再组合起来。

stable diffusion（https://arxiv.org/abs/2112.10752），论文中有个图片

dall-e系列，也是一样，只是中间的generation model可以是diffusion model 也可以是autoregressive model。文字的encode对结果又帮助（越大越好），但是diffusion model的大小对结果不显著（300M到2B）。

decoder的训练，不需要文字-图片成对的数据集，仅有图片就行了，所有的图片拿出来，downsampling 变成小图，然后就有数据集了，小图是输入，大图是ground truth。如果输入只是一个latent representation，用auto-encoder训练一个encoder，输入是图片，变成latent representation，输出也是原来的图片。训练完了就把decoder拿出来用就行了。imageN就是把小图当中间产物，stable diffusion 和 dall-e就是把latent representation当中间产物。

generation model的训练，在之前的基础上训练一个encoder，输出也是latent representation，在这上面加noise，一直一直加，而不是在原来的图片上加，输出的时候也是，1 latent representation 2 step 3 文字。noise是不需要学习的，人为设计的。
