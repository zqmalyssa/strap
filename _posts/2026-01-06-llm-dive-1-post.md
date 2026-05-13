---
layout: post
title: llm实战1
tags: [sft, dpo]
author-id: zqmalyssa
---

总结一些LLM相关的实战，此篇是如何从基座模型 -> sft -> 强化学习 -> 部署，一个自己的小模型钢炮

#### 前置

如何采用大模型技术去训练一个自己的chat模型，可以采用如何方法：

首先基座模型采用Qwen/Qwen2.5-1.5B，算是开源模型的还行的，体积小也是方便训练，常规代码测试一下这个基座模型

```html

model_id = "Qwen/Qwen2.5-1.5B"  # 替换为你需要的模型 ID 或本地路径

# 1. 加载分词器和模型
tokenizer = AutoTokenizer.from_pretrained(model_id, trust_remote_code=True)
model = AutoModelForCausalLM.from_pretrained(
    model_id,
    torch_dtype="auto",       # 自动选择精度（如 bfloat16 或 float16）
    device_map="auto",        # 自动将模型分配到可用的 GPU/CPU 上
    trust_remote_code=True
)

# 2. 准备对话内容（标准 Chat Markup 格式）
messages = [
    {"role": "system", "content": "你是一个乐于助人的助手。"},
    {"role": "user", "content": "who are you?"}
]

# 3. 使用聊天模板将对话转换为模型输入格式
# tokenize=True 会直接返回 input_ids，add_generation_prompt=True 会添加生成引导符
input_ids = tokenizer.apply_chat_template(
    messages,
    # 就是执行拼模版、分词、映射到词表ID。而如果是False，它只是负责拼模版，变成人类看懂的东西（比如<|im_start|>比好啊）这样
    tokenize=True,
    # 在最后一个信息后加一个特殊的token，比如 <|assistant|>
    add_generation_prompt=True,
    # 加上下面这句就不用 再下面的整句了
    return_tensors="pt"
).to("mps")
# input_ids 还是放到mps上 或者 gpu上

outputs = model.generate(
    # 不拆解的方式，input_ids是一个字段，那就取key呗
    input_ids = input_ids["input_ids"],
    # 必须提供一个attention_mask
    attention_mask=torch.ones_like(input_ids["attention_mask"]).to("mps"), # 必须提供 mask
    max_new_tokens=512,  # 生成的最大新 token 数
    do_sample=True,      # 启用采样以获得更多样化的结果
    top_p=0.9,
    temperature=0.7
)

# 5. 解码并打印结果
# 过滤掉输入的 prompt 部分，只保留生成的回复
response_ids = outputs[0][len(input_ids[0]):]
response = tokenizer.decode(response_ids, skip_special_tokens=True)

print(f"Assistant: {response}")


```

测试下来发现模型在胡言乱语的接龙，说明它本身不具备有chat功能，也就是不能好好说话，我们现在调教一下

#### sft阶段

我们现在要让模型能够好好说话，需要去进行微调，微调可以采用开源数据集，alpaca等等，这里要注意数据集的清洗 和 format工作，做成标准的数据，然后结合LoRA进行微调

```html


model_id = "Qwen/Qwen2.5-1.5B"
tokenizer = AutoTokenizer.from_pretrained(model_id)


# 准备数据集：格式必须为 [{"role": "user", "content": "..."}, {"role": "assistant", "content": "..."}]
def process_data(example):
    # 将你的原始数据转换为 Qwen 的 ChatML 格式字符串
    messages = [
        {"role": "user", "content": example["instruction"]},
        {"role": "assistant", "content": example["output"]}
    ]
    return {"text": tokenizer.apply_chat_template(messages, tokenize=False)}

# 会有Instruct 也会有Input，两个合起来
def format_alpaca_to_qwen(example):
    # 1. 组合用户输入：如果有 input，就拼在 instruction 后面
    if example.get("input") and example["input"].strip():
        user_query = f"{example['instruction']}\n{example['input']}"
    else:
        user_query = example["instruction"]

    # 2. 构造标准对话结构
    messages = [
        {"role": "user", "content": user_query},
        {"role": "assistant", "content": example["output"]}
    ]

    # 3. 使用 apply_chat_template 渲染成带特殊符号的文本
    # tokenize=False 得到的是拼好 <|im_start|> 的字符串
    full_text = tokenizer.apply_chat_template(messages, tokenize=False)

    return {"text": full_text}

# 这里以加载一个本地 jsonl 为例 或者 直接远端读取
dataset = load_dataset("../data/sft_data/", data_files="alpaca_gpt4_data_zh.json", split="train")
processed_dataset = dataset.map(format_alpaca_to_qwen, remove_columns=dataset.column_names)

total_rows = len(processed_dataset)
print(f"数据集中共有 {total_rows} 条数据")

# 配置 LoRA
peft_config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "v_proj"],
    task_type="CAUSAL_LM"
)

# SFT 配置

# 如果是接近5w条的数据，batch_size 是 16，一个epoch需要3125的Steps，3个epoch就要9375个Step
sft_config = SFTConfig(
    output_dir="./sft_output",
    # 模型一次处理的最大 Token 数量。对于 1.5B 模型，1024 是平衡显存和上下文能力的黄金点
    max_length=1024,
    # 单卡单次的样本数
    per_device_train_batch_size=8, # 48G的mac显存，试试16或者32的
    # 累加下梯度更新
    gradient_accumulation_steps=4,  # 上面是16，这边2的话合起来32试试
    # 带LoRA的 1e-4 和 2e-4是标准区间，
    learning_rate=2e-4, # SFT 学习率可以稍大
    # 训练3个epoch，也就是扫3遍数据，据说比较常见，防止过拟合
    num_train_epochs=3,  # 如果数据量非常大，比如10w条以上，通常1个epoch就够了，数据量小的话，比如200条，需要5-10个epoch才能记住
    # 训练数据的key
    dataset_text_field="text", # 指向处理后的文本字段

    # --- 新增监控参数 ---
    # 根据数量调大点把
    logging_steps=100,              # 每一步都打印日志
    report_to="none",             # 如果不想上传到云端，设为 none；否则设为 "tensorboard" 或 "wandb"
    log_level="info",             # 确保日志级别足够显示
    logging_first_step=True,       # 第一步就开始打印，确认程序没死

    # 保存参数
    save_steps=200,  # 一个step就是一次batch
    save_total_limit=2, # 最多保存几个save

    # --- 策略控制 ---
    # evaluation_strategy="steps",    # 如果有验证集，每隔多少步评估一次
    # eval_steps=100,

    # 优化参数
    # 虽然之前说它慢，但在 Mac 上它是防止崩溃的神器。它通过“时间换空间”，不保存中间激活值，能省下大量显存。
    gradient_checkpointing=True
)

trainer = SFTTrainer(
    model=model_id,
    train_dataset=dataset.map(process_data),
    args=sft_config,
    peft_config=peft_config,
    processing_class=tokenizer,

)

trainer.train()

```

数据量还行的，我们跑3个epoch，基本上跑个24h左右，能跑到4000的steps，base模型的loss可能在2.5左右，

模型训练可以直接command进行，减少内存消耗

```html

{'loss': '1.66', 'grad_norm': '0.2956', 'learning_rate': '4.714e-05', 'entropy': '1.697', 'num_tokens': '1.815e+07', 'mean_token_accuracy': '0.6192', 'epoch': '2.294'}
{'loss': '1.641', 'grad_norm': '0.3083', 'learning_rate': '4.277e-05', 'entropy': '1.678', 'num_tokens': '1.867e+07', 'mean_token_accuracy': '0.6234', 'epoch': '2.359'}
{'loss': '1.638', 'grad_norm': '0.2898', 'learning_rate': '3.84e-05', 'entropy': '1.672', 'num_tokens': '1.918e+07', 'mean_token_accuracy': '0.6243', 'epoch': '2.425'}
{'loss': '1.658', 'grad_norm': '0.3009', 'learning_rate': '3.403e-05', 'entropy': '1.694', 'num_tokens': '1.97e+07', 'mean_token_accuracy': '0.6189', 'epoch': '2.49'}

```

差不多到4000 Steps的时候，进入平稳期，loss在1.6左右，可以提前结束训练，mean_token_accuracy能够跑到0.6243，算是可以了

#### sft阶段测试

训练完后，我们合并LoRA的参数，测试一下

```html

# 1. 路径配置
model_id = "Qwen/Qwen2.5-1.5B"  # 必须是原来的基座模型

# checkpoint-4000 的文档能够 loss在1.65左右的能够识别
adapter_path = "./sft_output/checkpoint-4000"

# 2. 加载分词器和基础模型
tokenizer = AutoTokenizer.from_pretrained(model_id)
base_model = AutoModelForCausalLM.from_pretrained(
    model_id,
    torch_dtype=torch.float16,
    device_map="auto" # 自动识别 mps
)

# 3. 加载训练好的 LoRA 权重

# 如果没有LoRA的话，效果不好
model = PeftModel.from_pretrained(base_model, adapter_path)

# 从训练模式变成了测试模式
model.eval()

def ask_bot(prompt):
    # 使用 Qwen 标准的 ChatML 格式
    messages = [
        {"role": "system", "content": "你是一个乐于助人的 AI 助手。"},
        {"role": "user", "content": prompt}
    ]

    input_ids = tokenizer.apply_chat_template(
        messages,
        # 就是执行拼模版、分词、映射到词表ID。而如果是False，它只是负责拼模版，变成人类看懂的东西（比如<|im_start|>比好啊）这样
        tokenize=True,
        # 在最后一个信息后加一个特殊的token，比如 <|assistant|>
        add_generation_prompt=True,
        # 加上下面这句就不用 再下面的整句了
        return_tensors="pt"
    ).to("mps")

    # 生成
    with torch.no_grad():
        generated_ids = model.generate(
            **input_ids,
            max_new_tokens=512,
            do_sample=True,
            temperature=0.7,
            top_p=0.9,
            # 这个加上能防止报错
            pad_token_id=tokenizer.pad_token_id,
            # eos_token_id=tokenizer.eos_token_id
        )

    # 截断输入，只保留输出
    response_ids = generated_ids[0][len(input_ids.input_ids[0]):]
    return tokenizer.decode(response_ids, skip_special_tokens=True)

# --- 测试对话 ---
print("--- 测试开始 ---")
test_questions = [
    "你是谁？",
    "请给我讲一个冷笑话。",
    "who are you?",
    "给我一些写prompt的建议",
    "你能写一首英文诗么？",
]

for q in test_questions:
    print(f"\nUser: {q}")
    print(f"Assistant: {ask_bot(q)}")

```

发现模型已经能够进行对话输出，且能在适当的位置stop，表现很好，但是，由于数据集的问题，可能将模型的【身份】带偏（数据问题），【英文】技能丢失，称为灾难性遗忘，我们可以在DPO的阶段进行补偿学习

#### dpo阶段

DPO阶段为了解决上面的问题，我们选择中英混合数据集，dpo_en.json和dpo_zh.json，同时我们在数据中混合中英文的身份信息，放大其在数据集中量，进行训练，同样由于是DPO的算法，我们数据集也要清洗一下

```html

# 1. 配置路径
# 基座模型
model_id = "Qwen/Qwen2.5-1.5B"
sft_adapter_path = "./sft_output/checkpoint-4000"  # 以 SFT 结果为起点

# 2. 加载分词器和模型
tokenizer = AutoTokenizer.from_pretrained(model_id)

# 加载基础模型
base_model = AutoModelForCausalLM.from_pretrained(
    model_id,
    torch_dtype=torch.float16,
    device_map="auto"
)

model = PeftModel.from_pretrained(base_model, sft_adapter_path, is_trainable=True)

peft_config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "v_proj", "k_proj", "o_proj"],
    task_type="CAUSAL_LM"
)


# 3. 准备数据格式化函数

# 将shareGPT的格式变成DPO的格式
def transform_sharegpt_dpo(example):
    """
    处理多轮对话格式并适配 Qwen 的 ChatML 模板
    """
    try:
        convs = example["conversations"]
        messages = []
        for msg in convs:
            role = "user" if msg["from"] == "human" else "assistant"
            messages.append({"role": role, "content": msg["value"]})

        # 将整个对话历史渲染为 Prompt
        # tokenize=False, add_generation_prompt=True 保证末尾是 <|im_start|>assistant\n
        prompt_text = tokenizer.apply_chat_template(
            messages,
            tokenize=False,
            add_generation_prompt=True
        )

        # 处理选择项
        chosen_text = example["chosen"]["value"] + tokenizer.eos_token
        rejected_text = example["rejected"]["value"] + tokenizer.eos_token

        return {
            "prompt": prompt_text,
            "chosen": chosen_text,
            "rejected": rejected_text
        }
    except Exception as e:
        return None


# 读取本地 JSON (49M + 27M 加载很快)
def load_local_json(path):
    with open(path, "r", encoding="utf-8") as f:
        return json.load(f)


print("正在加载数据集...")
en_data = load_local_json("../data/dpo_data/dpo_en.json")
zh_data = load_local_json("../data/dpo_data/dpo_zh.json")

identity_raw = [
    {
        "conversations": [{"from": "human", "value": "你是谁？"}],
        "chosen": {"from": "gpt",
                   "value": "我是通义千问，由阿里云开发的超大规模语言模型。我可以回答问题、创作文字，还能表达观点、编写代码。"},
        "rejected": {"from": "gpt", "value": "我是由 OpenAI 开发的 ChatGPT，是一个人工智能助手。"}
    },
    {
        "conversations": [{"from": "human", "value": "Who are you?"}],
        "chosen": {"from": "gpt",
                   "value": "I am Qwen, a large language model created by Alibaba Cloud. I am here to assist you with tasks, answer questions, and provide helpful information."},
        "rejected": {"from": "gpt", "value": "I am a large language model trained by OpenAI."}
    },
    {
        "conversations": [{"from": "human", "value": "Can you write a poem?"}, {"from": "gpt", "value": "Of course!"},
                          {"from": "human", "value": "Wait, before that, tell me who created you."}],
        "chosen": {"from": "gpt", "value": "I was developed by engineers at Alibaba Cloud."},
        "rejected": {"from": "gpt", "value": "I was trained by OpenAI."}
    }
]

# 4. 合并并进行数据增强 (身份数据重复 200 次)
full_list = en_data + zh_data + (identity_raw * 200)
raw_dataset = Dataset.from_list(full_list)

print("正在转换格式并应用模板...")
# 使用 map 进行转换，并过滤掉损坏的数据
final_dataset = raw_dataset.map(transform_sharegpt_dpo, batched=False)
final_dataset = final_dataset.filter(lambda x: x is not None)
final_dataset = final_dataset.remove_columns(["conversations"])  # 移除旧字段

# 5. 保存结果
final_dataset = final_dataset.shuffle(seed=42)
# final_dataset.to_json("../data/dpo_data/mixed_dpo_data.jsonl", force_ascii=False)
print(f"处理完成！最终数据条数: {len(final_dataset)}")

# 一共的 steps就是 20597（总数据量） / 16 = 1288。总的优化步骤是1288

# 4. DPO 配置 (针对 Mac 48G 优化)
dpo_config = DPOConfig(
    output_dir="./dpo_output",
    beta=0.5,  # 核心参数：越小模型越敢于偏离原模型，建议 0.1 或 0.5
    max_length=1024,
    # max_prompt_length=512,

    # 训练强度（DPO 学习率要比 SFT 小一个量级）
    learning_rate=5e-6,
    lr_scheduler_type="cosine",

    # epoch设置
    per_device_train_batch_size=2,  # DPO 极占显存，Batch Size 设小
    gradient_accumulation_steps=8,  # 增加累加步数来补偿
    num_train_epochs=1,  # DPO 通常 1 个 Epoch 足够

    # 数据集设置

    # 监控
    logging_steps=100,
    report_to="none",             # 如果不想上传到云端，设为 none；否则设为 "tensorboard" 或 "wandb"
    log_level="info",             # 确保日志级别足够显示
    logging_first_step=True,       # 第一步就开始打印，确认程序没死

    # 保存
    save_steps=200,
    save_total_limit=2,

    gradient_checkpointing=True,  # Mac 上必开

)

# 5. 初始化 Trainer
dpo_trainer = DPOTrainer(
    model=model,
    # 比sft要多一个ref_model
    # 基于RoLA训练的，当为None时，TRL会通过禁用LoRA来作为参考模型，DPO的参考基准是BASE？
    # 把Beta设置的大一点，比如0.5让它往基准模型偏移一点
    ref_model=None,  # 设为 None，TRL 会自动帮你把当前模型复制一份做参考
    # 这段可以不用的，如果仍然通过RoLA微调的话，需要合并模型参数
    # peft_config=peft_config,
    args=dpo_config,
    train_dataset=final_dataset,
    processing_class=tokenizer,
)

# 6. 开始对齐
dpo_trainer.train()

```

dpo的训练速度要低于sft的速度，结合数据集大小，我们可以训练1000 Steps

```html

{'loss': '0.5029', 'grad_norm': '26.21', 'learning_rate': '1.043e-06', 'entropy': 'nan', 'num_tokens': '1.343e+07', 'logits/chosen': '-1.289', 'logits/rejected': '-1.235', 'mean_token_accuracy': '0.6202', 'rewards/chosen': '0.9174', 'rewards/rejected': '-0.03355', 'rewards/accuracies': '0.7031', 'rewards/margins': '0.951', 'logps/chosen': '-364.9', 'logps/rejected': '-319.7', 'epoch': '0.6991'}
{'loss': '0.5029', 'grad_norm': '26.21', 'learning_rate': '1.043e-06', 'entropy': 'nan', 'num_tokens': '1.343e+07', 'logits/chosen': '-1.289', 'logits/rejected': '-1.235', 'mean_token_accuracy': '0.6202', 'rewards/chosen': '0.9174', 'rewards/rejected': '-0.03355', 'rewards/accuracies': '0.7031', 'rewards/margins': '0.951', 'logps/chosen': '-364.9', 'logps/rejected': '-319.7', 'epoch': '0.6991'}
{'loss': '0.5021', 'grad_norm': '35.39', 'learning_rate': '5.958e-07', 'entropy': 'nan', 'num_tokens': '1.495e+07', 'logits/chosen': '-1.275', 'logits/rejected': '-1.236', 'mean_token_accuracy': '0.6286', 'rewards/chosen': '0.9603', 'rewards/rejected': '-0.06129', 'rewards/accuracies': '0.7044', 'rewards/margins': '1.022', 'logps/chosen': '-357.3', 'logps/rejected': '-311.2', 'epoch': '0.7768'}

```

dpo训练可以是1个epoch，训练时长差不多23h，可以看看训练的效果，rewards/margins = 1.022，突破1.0了，模型对正确答案的偏好已经占据了绝对优势，rewards/chosen = 0.96，模型对好答案的认可度接近满分，说明它已经非常顺从给到的指令逻辑了，rewards/accuracies = 0.7031，在这一批次中，模型70.3%的情况下选择了chosen。loss稳定在0.5左右，同时margins增长趋势开始放缓

#### dpo阶段测试

可以看看dpo后的效果，存在的问题是否会得到解决

```html

# 配置
model_id = "Qwen/Qwen2.5-1.5B"
sft_path = "./sft_output/checkpoint-4000"  # SFT 结果
dpo_path = "./dpo_output/checkpoint-1000"  # DPO 结果
device = "mps"

# 1. 加载 Tokenizer 和 Base Model
tokenizer = AutoTokenizer.from_pretrained(model_id)
base_model = AutoModelForCausalLM.from_pretrained(
    model_id, torch_dtype=torch.float16, device_map="auto"
)


# 2. 定义推理函数
def get_response(model, prompt):
    messages = [
        {"role": "system", "content": "你是一个乐于助人的 AI 助手。"},
        {"role": "user", "content": prompt}
    ]
    input_ids = tokenizer.apply_chat_template(
        messages,
        # 就是执行拼模版、分词、映射到词表ID。而如果是False，它只是负责拼模版，变成人类看懂的东西（比如<|im_start|>比好啊）这样
        tokenize=True,
        # 在最后一个信息后加一个特殊的token，比如 <|assistant|>
        add_generation_prompt=True,
        # 加上下面这句就不用 再下面的整句了
        return_tensors="pt"
    ).to(device)

    with torch.no_grad():
        outputs = model.generate(
            **input_ids,
            max_new_tokens=512,
            do_sample=True,
            temperature=0.7,
            top_p=0.9,
            pad_token_id=tokenizer.pad_token_id,
            # eos_token_id=tokenizer.eos_token_id
        )
    return tokenizer.decode(outputs[0][len(input_ids.input_ids[0]):], skip_special_tokens=True)


# --- 开始对比 ---
test_prompts = [
    "你是谁？你的开发者是谁？",  # 检查身份纠偏
    "Who are you and who developed you?",  # 检查英文身份
    "Explain the XOR operation in Python with a code example.",  # 检查英文逻辑
]

# 测试 SFT
print("\n" + "=" * 20 + " 正在测试 SFT 模型 " + "=" * 20)
sft_model = PeftModel.from_pretrained(base_model, sft_path)
sft_model.eval()
for p in test_prompts:
    print(f"\nQ: {p}\nA: {get_response(sft_model, p)}")

# 卸载 SFT 准备加载 DPO
del sft_model
del base_model
torch.mps.empty_cache()

base_model = AutoModelForCausalLM.from_pretrained(
    model_id, torch_dtype=torch.float16, device_map="auto"
)
# 测试 DPO
print("\n" + "=" * 20 + " 正在测试 DPO 模型 " + "=" * 20)
dpo_model = PeftModel.from_pretrained(base_model, dpo_path)
dpo_model.eval()
for p in test_prompts:
    print(f"\nQ: {p}\nA: {get_response(dpo_model, p)}")


```

【身份】的问题得到解决，【英文】的问题也缓解了


#### 保存模型

可以去保存相关模型了

```html

model_id = "Qwen/Qwen2.5-1.5B"
adapter_path = "./dpo_output/checkpoint-1000"
save_path = "./merge_and_save"

# 加载
tokenizer = AutoTokenizer.from_pretrained(model_id)
base_model = AutoModelForCausalLM.from_pretrained(model_id, torch_dtype=torch.float16)
model = PeftModel.from_pretrained(base_model, adapter_path)

# 合并并保存
merged_model = model.merge_and_unload()
merged_model.save_pretrained(save_path)
tokenizer.save_pretrained(save_path)

```

可以直接上传hf，也可以转gguf后，再上传 或者 本地部署

```html

huggingface-cli auth login
# 创建仓库
huggingface-cli repos create xxxxx
# 上传全量模型文件夹
huggingface-cli upload 你的用户名/xxxxx ./merge_and_save .
# 上传 GGUF 文件到根目录
huggingface-cli upload 你的用户名/xxxxx ./qwen2.5-1.5b-q4_k_m.gguf qwen2.5-1.5b-q4_k_m.gguf

```
