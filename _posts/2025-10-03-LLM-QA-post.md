---
layout: post
title: LLM问题
tags: [llm问题]
author-id: zqmalyssa
---

LLM开发相关问题

#### 问题列表

1、加载embeddings模型的时候，

```html

TypeError: __init__() got an unexpected keyword argument 'word_embedding_dimension'

# 词向量模型中的 1_Pooling 文件夹下面的config.json需要也存在

```

2、chat_format相关，在program去进行模型交互的时候

```html

# 以下参数
chat_format="llama-3",

# 或者
 inputs = tokenizer.apply_chat_template(
         message,
         add_generation_prompt=True,
         return_tensors="pt"
     ).to(model.device)
需要关注

影响模型的返回

```

3、对于llama_index，如果需要用到tool use的，还是需要模型具备tool use能力（在function agent的情况下），用ReActAgent除外

[参考链接](https://developers.llamaindex.ai/python/examples/agent/react_agent_with_query_engine/?utm_source=chatgpt.com)

上面是将rag引入进来，然后作为tools给react的agent

4、rag的时候可以加上rerank，就是用更强的模型给retrieve出来的结果再筛选，看下面代码

```html

ranker = LLMRerank(
      choice_batch_size=5, top_n=3, llm=OpenAI(model="gpt-4o-mini")
)

```

5、关于量化的bitsandbytes

```html

# 如果是mac的话，是没有cuda的支持的，以前会有一些专门的版本

ImportError: The installed version of bitsandbytes (<0.43.1) requires CUDA, but CUDA is not available. You may need to install PyTorch with CUDA support or upgrade bitsandbytes to >=0.43.1.

```

6、跑qlora + SFT 和 DPO训练的时候，环境问题

```html

python3.12的环境，T4是16G的缓存

!nvidia-smi  # 看显卡
!python --version # python版本
!nvcc --version # 装的cuda的版本

NVIDIA-SMI 550.54.15              Driver Version: 550.54.15      CUDA Version: 12.4  

有个兼容问题，是想要4bit量化的

import torch
print(torch.__version__)
print(torch.version.cuda)
from bitsandbytes.nn import Linear4bit

# 用上面这四条命令debug下，cu126，cuda版本过高了，降到cu121，卸载原来的bitsandbytes，安装notebook指定的bitsandbytes==0.43.1

!pip install -U \
  torch==2.3.0+cu121 \
  torchvision==0.18.0+cu121 \
  torchaudio==2.3.0+cu121 \
  --index-url https://download.pytorch.org/whl/cu121

!pip uninstall -y bitsandbytes
!pip install bitsandbytes==0.43.1

重启环境后运行时没了

7、还是环境问题，如果在win/mac环境下遇到（[mutex.cc : 452] RAW: Lock blocking 0x600002224258   @）类似的报错，直接re-install环境把

8、sft和dpo的模型推理问题

```html

messages = [
        # 这个要加
        {"role": "system", "content": "你是一个乐于助人的 AI 助手。"},
        {"role": "user", "content": prompt}
    ]

system有的时候要加，不要问为什么，（不加可能导致回答出现 乱码）

```
