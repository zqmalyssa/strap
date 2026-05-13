---
layout: post
title: llama系列
tags: [llama]
author-id: zqmalyssa
---

Llama相关的单独说下

#### Llama.cpp相关

Llama.cpp的主要目标是在各种硬件上（本地和云端）实现LLM推断，同时保持最小的设置和最先进的性能。

- 纯C/C++实现，没有任何依赖关系
- Apple芯片是一级的支持对象 - 通过ARM NEON、Accelerate和Metal框架进行优化
- 对x86架构的AVX、AVX2和AVX512支持
- 支持1.5位、2位、3位、4位、5位、6位和8位整数量化，以加快推断速度并减少内存使用
- 为在NVIDIA GPU上运行LLMs而定制的CUDA内核（通过HIP支持AMD GPU）
- Vulkan、SYCL和（部分）OpenCL后端支持
- CPU+GPU混合推断，部分加速超过总VRAM容量的模型

```html

# 源代码编译安装（https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md）

git clone https://github.com/ggerganov/llama.cpp.git

brew install cmake

cmake -B build

cmake --build build --config Release

# 以上build文件夹下的bin目录有东西了，如llama-cli

# 用脚本的时候需要python3.11 以上

conda create --name py311 python=3.11 -y

# ERROR: Could not install packages due to an OSError: Missing dependencies for SOCKS support(因为是一个干净的env)

/Users/zqmalyssa/miniconda3/envs/py311/bin/pip install -r requirements.txt # 如果设置了ALL_PROXY会有问题，可以先unset，在重试 或者 安装pysocks后带proxy

# 转换成gguf，8bit量化
/Users/zqmalyssa/miniconda3/envs/py311/bin/python convert_hf_to_gguf.py /Users/zqmalyssa/Model/deepseek/DeepSeek-R1-Distill-Qwen-32B --outfile /Users/zqmalyssa/Model/llama/DeepSeek-R1-Distill-Qwen-32B-8bit.gguf --outtype q8_0

# 转换后 模型从原来64G大小转换到了32G

# 但是量化后的启动起来会有乱码

/Users/zqmalyssa/Code/llama.cpp/build/bin/llama-cli -m /Users/zqmalyssa/Model/llama/DeepSeek-R1-Distill-Qwen-32B-8bit.gguf

> 你好，你谁谁
@@@@@@@@@@@@

# 直接搞一个qwen3的32B的，推理推荐，有量化版本，来个Q6_K，另外如果是safetensors的文件，llama-cli不能直接加载

/Users/zqmalyssa/Code/llama.cpp/build/bin/llama-cli -m /Users/zqmalyssa/Model/gguf/Qwen3-32B.Q6_K.gguf

# 把ds的bf16的8B转成gguf 或者 bf16，还是会有乱码问题
/Users/zqmalyssa/miniconda3/envs/py311/bin/python convert_hf_to_gguf.py /Users/zqmalyssa/Model/deepseek/DeepSeek-R1-Distill-Llama-8B --outfile /Users/zqmalyssa/Model/gguf/DeepSeek-R1-8B-fp16.gguf --outtype f16

# 直接用llama的吧
/Users/zqmalyssa/miniconda3/envs/py311/bin/python convert_hf_to_gguf.py /Users/zqmalyssa/Model/llama/Llama-3.1-8B-Instruct --outfile /Users/zqmalyssa/Model/gguf/llama-3.1-8B-fp16.gguf --outtype f16

# 这样就能用了
/Users/zqmalyssa/Code/llama.cpp/build/bin/llama-cli -m /Users/zqmalyssa/Model/gguf/llama-3.1-8B-fp16.gguf

# 正常的

ollama create local-qwen3-32B-q6 -f /Users/zqmalyssa/Model/modelfile/Modelfile

gathering model components
copying file sha256:37b91c47b55d9da66fc35f3481d940b2a922148e55f0a68ab0ec5480d68312ce 100%
parsing GGUF
using existing layer sha256:37b91c47b55d9da66fc35f3481d940b2a922148e55f0a68ab0ec5480d68312ce
writing manifest
success

# ollama可以show出模型

ollama show llama3.1:latest

ollama show llama3.1:latest --template # 这个看的效果可能更好

```

如果自己已经做出来了 gguf的话，可以再通过llama.cpp的 /build/bin下去量化

```html

./llama-quantize /Users/zqmalyssa/Code/pycharmproject/ai/sft/qwen2.5-1.5b-final.gguf /Users/zqmalyssa/Code/pycharmproject/ai/sft/qwen2.5-1.5b-final-q4_k_m.gguf Q4_K_M

# 原来3G的大小，量化下来1个G左右

ollama create my-qwen-quantize-final -f Modelfile_quantize_qwen

# Q4的量化得到的模型效果肯定还是不如原生的

```


#### 参数设置
