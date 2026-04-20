---
layout: post
title: Llama-factory相关
tags: [Llama-factory]
author-id: zqmalyssa
---

Llama-factory相关

其本质是配置到代码的映射器，

```html

1、通过YAML 或 WebUI下达指令

2、解析指令，查找对应的模型模版 和 数据格式

3、调用peft包装模型

4、把数据和包装后的模型丢给TRL/Transformers 的trainer循环迭代

```

#### 使用相关

创建一个虚拟环境，python的3.11的环境，然后从源码安装

```html

git clone --depth 1 https://github.com/hiyouga/LLaMA-Factory.git
cd LLaMA-Factory
pip install -e ".[torch,metrics]" --no-build-isolation

# 装到bin目录的，
/Users/zqmalyssa/miniconda3/envs/py311/bin/llamafactory-cli train -h

# 切切env试试呢

find $(python3 -c "import sys; print(sys.prefix)") -name "llamafactory-cli"

```

检测下环境

```html

# 如果有cuda

import torch
torch.cuda.current_device()
torch.cuda.get_device_name(0)
torch.__version__

# 如果是mac

torch.backends.mps.is_available()

# 再看看下载下来的client能不能work

llamafactory-cli train -h

```

然后测试一下推理部分是否正常

```html

llamafactory-cli webchat \
    --model_name_or_path /Users/zqmalyssa/Model/llama/Llama-3.1-8B-Instruct \
    --template llama3

# template 模型回答所使用的prompt模版，不同模型不同
# http://localhost:7860/ 完成可以在浏览器中进行对话了，是一个chat界面

如果这个时候使用mlx的模型呢

llamafactory-cli webchat \
  --model_name_or_path /Users/zqmalyssa/Model/mlx/DeepSeek-R1-Distill-Qwen-32B-4bit \
  --template deepseekr1

ValueError: The model's quantization config from the arguments has no `quant_method` attribute. Make sure that the model has been correctly quantized

不支持mlx的量化模型，需要换格式，mlx不走 Transformers，不存在bitsAndBytes/ GPTQ / AWQ

```

使用llama命令行 或者 自带的webui可以进行微调

```html

llamafactory-cli webui

```

在HF或者魔塔下载一些数据集或者模型，一个是修改数据，二个是引入新的数据。

```html

# 注意mac的版本-i 后面有个 ''
sed -i '' 's/{{name}}/alyssa/g'  data/identity.json
sed -i '' 's/{{author}}/LLaMA Factory/g'  data/identity.json

```

命令行微调，注意save/eval等等参数的使用

```html

# 通过sft微调让模型能够学会你是谁，以及一些商品信息

# 少一个数据集alpaca_gpt4_zh, 需要下载

CUDA_VISIBLE_DEVICES=0 llamafactory-cli train \
    --stage sft \
    --do_train \
    --model_name_or_path /Users/zqmalyssa/Model/llama/Llama-3.1-8B-Instruct \
    --dataset identity,adgen_local \
    --dataset_dir ./data \
    --template llama3 \
    --finetuning_type lora \
    --output_dir ./saves/LLaMA3-8B/lora/sft \
    --overwrite_cache \
    --overwrite_output_dir \
    --cutoff_len 1024 \
    --preprocessing_num_workers 16 \
    --per_device_train_batch_size 2 \
    --per_device_eval_batch_size 1 \
    --gradient_accumulation_steps 8 \
    --lr_scheduler_type cosine \
    --logging_steps 50 \
    --warmup_steps 20 \
    --save_strategy steps \
    --save_steps 100 \
    --eval_strategy steps \
    --eval_steps 50 \
    --load_best_model_at_end \
    --learning_rate 5e-5 \
    --num_train_epochs 5.0 \
    --max_samples 1000 \
    --val_size 0.1 \
    --plot_loss \
    --fp16


# 可以看到训练时候的进度


[INFO|tokenization_utils_base.py:2556] 2025-12-15 22:22:02,655 >> chat template saved in ./saves/LLaMA3-8B/lora/sft/checkpoint-310/chat_template.jinja
[INFO|tokenization_utils_base.py:2725] 2025-12-15 22:22:02,656 >> tokenizer config file saved in ./saves/LLaMA3-8B/lora/sft/checkpoint-310/tokenizer_config.json
[INFO|tokenization_utils_base.py:2734] 2025-12-15 22:22:02,656 >> Special tokens file saved in ./saves/LLaMA3-8B/lora/sft/checkpoint-310/special_tokens_map.json
[INFO|trainer.py:2810] 2025-12-15 22:22:03,085 >>

Training completed. Do not forget to share your model on huggingface.co/models =)


[INFO|trainer.py:3033] 2025-12-15 22:22:03,085 >> Loading best model from ./saves/LLaMA3-8B/lora/sft/checkpoint-200 (score: 2.1228792667388916).
{'train_runtime': 7895.9397, 'train_samples_per_second': 0.621, 'train_steps_per_second': 0.039, 'train_loss': 2.194175289523217, 'epoch': 5.0}
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 310/310 [2:11:35<00:00, 25.47s/it]
[INFO|trainer.py:4309] 2025-12-15 22:22:03,258 >> Saving model checkpoint to ./saves/LLaMA3-8B/lora/sft
[INFO|configuration_utils.py:763] 2025-12-15 22:22:03,266 >> loading configuration file /Users/zqmalyssa/Model/llama/Llama-3.1-8B-Instruct/config.json
[INFO|configuration_utils.py:839] 2025-12-15 22:22:03,266 >> Model config LlamaConfig {
  "architectures": [
    "LlamaForCausalLM"
  ],
  "attention_bias": false,
  "attention_dropout": 0.0,
  "bos_token_id": 128000,
  "dtype": "bfloat16",
  "eos_token_id": [
    128001,
    128008,
    128009
  ],
  "head_dim": 128,
  "hidden_act": "silu",
  "hidden_size": 4096,
  "initializer_range": 0.02,
  "intermediate_size": 14336,
  "max_position_embeddings": 131072,
  "mlp_bias": false,
  "model_type": "llama",
  "num_attention_heads": 32,
  "num_hidden_layers": 32,
  "num_key_value_heads": 8,
  "pretraining_tp": 1,
  "rms_norm_eps": 1e-05,
  "rope_scaling": {
    "factor": 8.0,
    "high_freq_factor": 4.0,
    "low_freq_factor": 1.0,
    "original_max_position_embeddings": 8192,
    "rope_type": "llama3"
  },
  "rope_theta": 500000.0,
  "tie_word_embeddings": false,
  "transformers_version": "4.57.3",
  "use_cache": true,
  "vocab_size": 128256
}

[INFO|tokenization_utils_base.py:2556] 2025-12-15 22:22:03,378 >> chat template saved in ./saves/LLaMA3-8B/lora/sft/chat_template.jinja
[INFO|tokenization_utils_base.py:2725] 2025-12-15 22:22:03,379 >> tokenizer config file saved in ./saves/LLaMA3-8B/lora/sft/tokenizer_config.json
[INFO|tokenization_utils_base.py:2734] 2025-12-15 22:22:03,380 >> Special tokens file saved in ./saves/LLaMA3-8B/lora/sft/special_tokens_map.json
***** train metrics *****
  epoch                    =        5.0
  total_flos               = 38886698GF
  train_loss               =     2.1942
  train_runtime            = 2:11:35.93
  train_samples_per_second =      0.621
  train_steps_per_second   =      0.039
Figure saved at: ./saves/LLaMA3-8B/lora/sft/training_loss.png
Figure saved at: ./saves/LLaMA3-8B/lora/sft/training_eval_loss.png
[WARNING|2025-12-15 22:22:03] llamafactory.extras.ploting:148 >> No metric eval_accuracy to plot.
[INFO|trainer.py:4643] 2025-12-15 22:22:03,571 >>
***** Running Evaluation *****
[INFO|trainer.py:4645] 2025-12-15 22:22:03,571 >>   Num examples = 110
[INFO|trainer.py:4648] 2025-12-15 22:22:03,571 >>   Batch size = 1
/Users/zqmalyssa/miniconda3/envs/py311/lib/python3.11/site-packages/torch/utils/data/dataloader.py:692: UserWarning: 'pin_memory' argument is set as true but not supported on MPS now, device pinned memory won't be used.
  warnings.warn(warn_msg)
100%|███████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 110/110 [00:56<00:00,  1.96it/s]
***** eval metrics *****
  epoch                   =        5.0
  eval_loss               =     2.1346
  eval_runtime            = 0:00:56.29
  eval_samples_per_second =      1.954
  eval_steps_per_second   =      1.954
[INFO|modelcard.py:456] 2025-12-15 22:22:59,864 >> Dropping the following result as it does not have all the necessary fields:
{'task': {'name': 'Causal Language Modeling', 'type': 'text-generation'}}

# 完成后一些文件的保存

adapter开头的就是 LoRA保存的结果了，后续用于模型推理融合
training_loss 和trainer_log等记录了训练的过程指标
其他是训练当时各种参数的备份

# 进行验证，adapter_name_or_path配置lora模型位置

llamafactory-cli webchat \
    --model_name_or_path /Users/zqmalyssa/Model/llama/Llama-3.1-8B-Instruct \
    --adapter_name_or_path ./saves/LLaMA3-8B/lora/sft  \
    --template llama3 \
    --finetuning_type lora


# 上面就有效果了

```

再使用一些标准的评估方式做评估，而不只是chat

```html

pip install jieba
pip install rouge-chinese
pip install nltk

llamafactory-cli train \
    --stage sft \
    --do_predict \
    --model_name_or_path /Users/zqmalyssa/Model/llama/Llama-3.1-8B-Instruct \
    --adapter_name_or_path ./saves/LLaMA3-8B/lora/sft  \
    --eval_dataset identity,adgen_local \
    --dataset_dir ./data \
    --template llama3 \
    --finetuning_type lora \
    --output_dir ./saves/LLaMA3-8B/lora/predict \
    --overwrite_cache \
    --overwrite_output_dir \
    --cutoff_len 1024 \
    --preprocessing_num_workers 16 \
    --per_device_eval_batch_size 1 \
    --max_samples 20 \
    --predict_with_generate


# do_predict 预测模式
# predict_with_generate 现在用于生成文本
# max_samples 每个数据集采样多少用于预测对比

# generated_predictions.jsonl 文件 输出了要预测的数据集的原始label和模型predict的结果
# predict_results.json给出了原始label和模型predict的结果，用自动计算的指标数据


{
    "predict_bleu-4": 16.9900875,
    "predict_model_preparation_time": 0.0015,
    "predict_rouge-1": 38.2537975,
    "predict_rouge-2": 16.9921875,
    "predict_rouge-l": 31.847862499999998,
    "predict_runtime": 207.8544,
    "predict_samples_per_second": 0.192,
    "predict_steps_per_second": 0.192
}

BLEU-4：BLEU（Bilingual Evaluation Understudy）是一种常用的用于评估机器翻译质量的指标。BLEU-4 表示四元语法 BLEU 分数，它衡量模型生成文本与参考文本之间的 n-gram 匹配程度，其中 n=4。值越高表示生成的文本与参考文本越相似，最大值为 100。

predict_rouge-1 和 predict_rouge-2：ROUGE（Recall-Oriented Understudy for Gisting Evaluation）是一种用于评估自动摘要和文本生成模型性能的指标。ROUGE-1 表示一元 ROUGE 分数，ROUGE-2 表示二元 ROUGE 分数，分别衡量模型生成文本与参考文本之间的单个词和双词序列的匹配程度。值越高表示生成的文本与参考文本越相似，最大值为 100。

predict_rouge-l：ROUGE-L 衡量模型生成文本与参考文本之间最长公共子序列（Longest Common Subsequence）的匹配程度。值越高表示生成的文本与参考文本越相似，最大值为 100

predict_runtime：预测运行时间，表示模型生成一批样本所花费的总时间。单位通常为秒。

predict_samples_per_second：每秒生成的样本数量，表示模型每秒钟能够生成的样本数量。通常用于评估模型的推理速度。

predict_steps_per_second：每秒执行的步骤数量，表示模型每秒钟能够执行的步骤数量。对于生成模型，一般指的是每秒钟执行生成操作的次数。


# 合并lora导出模型

llamafactory-cli export \
    --model_name_or_path /Users/zqmalyssa/Model/llama/Llama-3.1-8B-Instruct \
    --adapter_name_or_path ./saves/LLaMA3-8B/lora/sft  \
    --template llama3 \
    --finetuning_type lora \
    --export_dir megred-model-path \
    --export_size 2 \
    --export_device cpu \
    --export_legacy_format False

```

也可以跑跑大模型主流评测benchmark

```html

本项目提供了mmlu，cmmlu, ceval三个常见数据集的自动评测脚本，按如下方式进行调用即可。

说明：task 目前支持 mmlu_test, ceval_validation, cmmlu_test

# 评测在未来可能被废弃

llamafactory-cli eval \
--model_name_or_path /Users/zqmalyssa/Model/llama/Llama-3.1-8B-Instruct \
--template llama3 \
--task mmlu_test \
--lang en \
--n_shot 5 \
--batch_size 1

# 和llama3的官方报告一致的

Average: 63.64                                                                                                                                     
STEM: 50.83
Social Sciences: 76.31
Humanities: 56.63
Other: 73.31

```


如果不小心中断了训练，或者爆内存等等了，可以从checkpoint恢复出来，checkpoint的存储依赖于上面的save_steps

```html

llamafactory-cli train --resume_from_checkpoint ./saves/LLaMA3-8B/lora/sft

```

进阶的话可以导出GGUF，然后部署ollama
