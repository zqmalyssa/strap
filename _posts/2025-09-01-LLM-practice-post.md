---
layout: post
title: llm-practice
tags: [llm-practice]
author-id: zqmalyssa
---

深度学习、强化学习、LLM的一些实践

#### 搞个model下来，做实验

1、可以在hugging face上models里面找开源的模型，在colab上跑一些免费的GPU，GPU相关的，GPT-3 规模模型训练：H100 比 A100 快 3–6 倍。大模型推理（70B）：H100 比 A100 快 10–30 倍，尤其是 FP8 训练能力，让 H100 专为大模型时代设计。免费的一般就是T4（4000-8000 人民币），用T4跑，把8B的模型换成1B的模型

NVIDIA A100 80GB：约 US$ 15,000–17,000 左右。
NVIDIA H100 80GB：约 US$ 25,000–30,000+，高端配置（如SXM版）可能达到 US$ 35,000–40,000

huggingface 的 transformer 是一个统一框架，规避了底层是用pytorch还是啥训练的模型，虽然可能没有ollema效率，但弹性高、广泛使用。所有上传到huggingface的模型，都能用它的transformer框架执行。获取huggingface的token

```html

from transformers import AutoTokenizer, AutoModelForCausalLM

# deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B
# deepseek-ai/DeepSeek-R1-Distill-Llama-8B
# 特么下载一个llama 还要授权来着。。试试上面的 1.5B 和 8B的版本，不需要授权，直接下载
model_id = "meta-llama/Llama-3.2-3B-Instruct"
#只要更換 model ID 就可以換成其他模型了
#假設 3B 模型太大，你可能會想要換成 1B 的模型 (https://huggingface.co/meta-llama/Llama-3.2-1B-Instruct)
#你只需要把上面的 "meta-llama/Llama-3.2-3B-Instruct" 換成 "meta-llama/Llama-3.2-1B-Instruct" 即可
#或是如果你想要用 Google 的 gemma (https://huggingface.co/google/gemma-3-4b-it)
#你只需要把上面的 "meta-llama/Llama-3.2-3B-Instruct" 換成 "google/gemma-3-4b-it" 即可
#總之，從今天開始，HuggingFace 上的模型隨便你使用 :)

# 记录模型所使用的token，vocabulary
tokenizer = AutoTokenizer.from_pretrained(model_id)
# 模型的参数
model = AutoModelForCausalLM.from_pretrained(model_id)

```

add_special_tokens=False 会使得句子前面不会加特殊token，直接映射到tokenizer的索引，

```html

# "good morning" 和 "i am good" 中的 good 編號一樣嗎？為什麼不一樣？
print("good morning" ,"->", tokenizer.encode("good morning",add_special_tokens=False))
print("i am good" ,"->", tokenizer.encode("i am good",add_special_tokens=False))

# 放在句首和前面有个空白，good对应的token的索引也不一样

#我們用 tokenizer.encode 把文字變成一串 id，再用 tokenizer.decode 把 id 轉回文字

text = "大家好"
# 就是一定加一个add_special_tokens=False，不然会有begin_of_text
tokens = tokenizer.encode(text,add_special_tokens=False) #add_special_tokens=False 可以避免加上代表起始的符號
text_after_encodedecode = tokenizer.decode(tokens)
print("原始文字:",text)
print("編碼在解碼後:",text_after_encodedecode)

```

现在开始用下model

```html

import torch #接下來需要用到 torch 這個套件

prompt = "1+1=" #試試看: "在二進位中，1+1="、"你是誰?"
print("輸入的 prompt 是:", prompt)

# model 不能直接輸入文字，model 只能輸入以 PyTorch tensor 格式儲存的 token IDs
# 把要輸入 prompt 轉成 model 可以處理的格式
input_ids = tokenizer.encode(prompt, return_tensors="pt") # return_tensors="pt" 表示回傳 PyTorch tensor 格式
print("這是 model 可以讀的輸入：",input_ids)

# model 以 input_ids (根據 prompt 產生) 作為輸入，產生 outputs，
outputs = model(input_ids)
# outputs 裡面包含了大量的資訊
# 我們在往後的課程還會看到 outputs 中還有甚麼
# 在這裡我們只需要 "根據輸入的 prompt ，下一個 token 的機率分布" (也就是每一個 token 接在 prompt 之後的機率)

# outputs.logits 是模型對輸入每個位置、每個 token 的信心分數（還沒轉成機率）
# outputs.logits shape: (batch_size, sequence_length, vocab_size)
last_logits = outputs.logits[:, -1, :] #得到一個 token 接在 prompt 後面的信心分數 (至於為什麼是這樣寫，留給各位同學自己研究)
probabilities = torch.softmax(last_logits, dim=-1) #softmax 可以把原始信心分數轉換成 0~1 之間的機率值

# 印出機率最高的前 top_k 名 token
top_k = 10
top_p, top_indices = torch.topk(probabilities, top_k)
print(f"機率最高的前 {top_k} 名 token:")
for i in range(top_k):
    token_id = top_indices[0][i].item() # 取得第 i 名的 token ID
    probability = top_p[0][i].item() # 對應的機率
    token_str = tokenizer.decode(token_id) # 將 token ID 解碼成文字
    print(f"Token ID: {token_id}, Token: '{token_str}', 機率: {probability:.4f}")


```

用上chatTemplate，让模型好好回答，等于把 "AI答：" 加入到你的prompt后面，model只是下一个token，model.generate就可以句子型的回复了

```html

prompt = "你是誰?"
print("現在的 prompt 是:", prompt)
prompt_with_chat_template = "使用者說：" + prompt + "\nAI回答：" #加上一個自己隨便想的 Chat Template
print("實際上模型看到的 prompt 是:", prompt_with_chat_template)
input_ids = tokenizer.encode(prompt_with_chat_template, return_tensors="pt")

outputs = model.generate(
    input_ids,
    max_length=50,
    do_sample=True,
    top_k=3,
    pad_token_id=tokenizer.eos_token_id,
    attention_mask=torch.ones_like(input_ids)
)

# 將產生的 token ids 轉回文字
generated_text = tokenizer.decode(outputs[0]) # skip_special_tokens=True 跳過特殊 token

print("生成的文字是：\n", generated_text)

#加上Chat Template，語言模型突然可以對話了， 模型一直是同一個，沒有改變喔!
#不過還是有問題，模型回答完問題後，常常繼續自己提問，這是因為這裡的 Chat Template 是自己亂想的

```

上面那种自己加Chat Template的方法不一定可以看懂，尾部还有接龙，可以使用tokenizer.apply_chat_template加上官方的chat_template。可以看下这个

[不同模型对应的模版](https://github.com/hiyouga/LLaMA-Factory?tab=readme-ov-file#supported-models)

```html

prompt = "你是誰?"
print("現在的 prompt 是:", prompt)
messages = [
    {"role": "user", "content": prompt},
]
print("現在的 messages 是:", messages)

input_ids = tokenizer.apply_chat_template(  #不只加上Chat Template，順便幫你 encode 了
    messages,
   add_generation_prompt=True,
    # add_generation_prompt=True 表示在最後一個訊息後加上一個特殊的 token (e.g., <|assistant|>)
   # 這會告訴模型現在輪到它回答了。
    return_tensors="pt"
)


print("tokenizer.apply_chat_template 的輸出：\n", input_ids)
print("===============================================\n")
print("用 tokenizer.decode 轉回文字：\n", tokenizer.decode(input_ids[0]))
print("===============================================\n")

### 以下程式碼跟前一段程式碼相同 ###

outputs = model.generate(
    input_ids,
    max_length=100,
    do_sample=True,
    top_k=3,
    pad_token_id=tokenizer.eos_token_id,
    attention_mask=torch.ones_like(input_ids)
)

# 將產生的 token ids 轉回文字
generated_text = tokenizer.decode(outputs[0])

print("生成的文字是：\n", generated_text)

```

model 和 model.generate 都不需要使用 gpu，transformers中的pipeline需要使用gpu，生成会变得很慢很慢，pipeline省略掉将文字转成token_id 再转回去的过程。

```html
from transformers import pipeline

# 建立一個pipeline，設定要使用的模型
# 换模型会自动下载
model_id = "deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B"
#model_id = "google/gemma-3-4b-it"
pipe = pipeline(
    "text-generation",
   model_id
)

messages = [{"role": "system", "content": "你是 DeepSeek，你都用中文回答我，開頭都說哈哈哈"}]

while True:
    # 1️⃣ 使用者輸入訊息
    user_prompt = input("😊 你說： ")

    # 如果輸入 "exit" 就跳出聊天
    if user_prompt.lower() == "exit":
        #print("聊天結束啦，下次再聊喔！👋")
        break

    # 將使用者訊息加進對話紀錄
    messages.append({"role": "user", "content": user_prompt})

    '''
    # 2️⃣ 將歷史訊息轉換為模型可以理解的格式
    # add_generation_prompt=True 會在訊息後面加入一個特殊標記 (<|assistant|>)，
    # 告訴模型現在輪到它講話了！
    input_ids = tokenizer.apply_chat_template(
        messages,
        add_generation_prompt=True,
        return_tensors="pt"
    )

    # 3️⃣ 生成模型的回覆
    outputs = model.generate(
        input_ids,
        max_length=2000, #這個數值需要設定大一點
        do_sample=True,
        top_k=10,
        pad_token_id=tokenizer.eos_token_id,
        attention_mask=torch.ones_like(input_ids)
    )

    # 將模型的輸出轉換為文字
    generated_text = tokenizer.decode(outputs[0], skip_special_tokens=False)

    # 🔎 從生成結果中取出模型真正的回覆內容（去除特殊token）
    # Llama 模型會用特殊的 token 區隔訊息頭尾，格式通常是這樣的：
    # [訊息頭部]<|end_header_id|> 模型的回覆內容 <|eot_id|>
    response = generated_text.split("<|end_header_id|>")[-1].split("<|eot_id|>")[0].strip()
    '''

    ### 上述註解中的程式碼所做的事情，可以僅用以下幾行程式碼完成。
    #=============================
    outputs = pipe(  # 呼叫模型生成回應
      messages,
      max_new_tokens=2000,
      pad_token_id=pipe.tokenizer.eos_token_id
    )
    response = outputs[0]["generated_text"][-1]['content'] # 從輸出內容取出模型生成的回應
    #=============================

    # 4️⃣ 顯示模型的回覆
    print("🤖 助理說：", response)

    # 將模型回覆加進對話紀錄，讓下次模型知道之前的對話內容
    messages.append({"role": "assistant", "content": response})

```

另外，用T4加载 deepseek-ai/DeepSeek-R1-Distill-Llama-8B，直接OutOfMemoryError，描述：OutOfMemoryError: CUDA out of memory. Tried to allocate 1002.00 MiB. GPU 0 has a total capacity of 14.74 GiB of which 548.12 MiB is free. Process 8165 has 14.20 GiB memory in use. Of the allocated memory 13.98 GiB is allocated by PyTorch, and 129.49 MiB is reserved by PyTorch but unallocated. If reserved but unallocated memory is large try setting PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True to avoid fragmentation.  See documentation for Memory Management  

#### 模型利用RAG，使用工具

常规的，调用函数

还有的tool use，操控鼠标和键盘（computer use），gpt处于代理人模式，可以试试看

在思考模式 和 非思考模式下 都支持工具调用（deepseek-v3.2），减小开源和闭源之间的差距

在思考模式下，https://api-docs.deepseek.com/zh-cn/news/news251201

回答问题1，模型进行多次思考 + 工具调用给出答案，在这个过程中，用户需要回传思维链内容（reasoning_content）给api，让模型继续思考

在下一个问题开始时候，问题2，需删除思维链并保留其他内容发送给api

思考模式也增加了对claude code的支持，改名字为deepseek-reasoner，或在claude code cli中开启tab键开启思考模式使用，


#### 看模型内部

看看模型的内部

```html

from transformers import AutoTokenizer, AutoModelForCausalLM

tokenizer = AutoTokenizer.from_pretrained("deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B")
model = AutoModelForCausalLM.from_pretrained("deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B")

model.num_parameters() #得到参数的输出

for name, param in model.named_parameters():
    print(f"{name:80}  |  shape: {tuple(param.shape)}")

#根據輸出，deepseek 有幾層呢？

model.embed_tokens.weight                                                         |  shape: (151936, 1536)

embedding_table，151936的voca数量，每个1536的维度。

可以看到一共28层。

```

最后可以看到一个lm_head.weigh，这个就跟llama-3b不一样了，llama-3b是没有lm_head.weigh的，gemma前面的一些参数是用来处理图片的。attention层都有q,k,v,o。

```html
# 可以把embedding table拿出来看看。

input_embedding = model.state_dict()["model.embed_tokens.weight"].numpy()

```

可以看看跟embedding最像的其他token，比如apple（苹果）

```html

top_k = 20 #自己設定一個數值

# 1️⃣ 讓使用者輸入一個 token
token = input('請輸入一個 token：') #輸入: apple, Apple, 李, 王 等等

# 2️⃣ 轉換成 token ID
token_id = tokenizer.encode(token)[1]
# 為什麼是 [1]？
# tokenizer.encode() 回傳的是: [BOS_token_id, token_id ...]
# print (tokenizer.encode(token)) <- 跑這一行試試看
# 第一個元素 [0] 是特殊起始符號 (BOS)，
# 我們真正想要的是輸入的那個 token 本身 → 所以取 index 1
print("token id 是 ",token_id)

# 3️⃣ 取得 token 的 embedding
embbeding = [input_embedding[token_id]]

# 4️⃣ 計算餘弦相似度
from sklearn.metrics.pairwise import cosine_similarity
sims = cosine_similarity(embbeding, input_embedding)[0]

# 5️⃣ 排序並取最相近 top_k,並輸出結果
nearest = sims.argsort()[::-1][1: top_k+1] #排除自己本身
print(f'和 {token} 最相近的 {top_k} 個 token：')
for idx in nearest:
  print(f'{tokenizer.decode(idx)} (score: {sims[idx]:.4f})')

```

也可以看下每层的representation

```html

inputs = tokenizer.encode("大家好", return_tensors="pt")

print("編碼後的 Token IDs：", inputs)

outputs = model(inputs, output_hidden_states=True) # output_hidden_states=True 才會回傳每一層的 representation (hidden states)

hidden_states = outputs.hidden_states
# hidden_states[0] -> embedding （把 token 轉成 token embedding 的結果)
# hidden_states[1] ~ hidden_states[N] -> 每一層 Transformer block 的輸出
print(f"一共拿到 {len(hidden_states)} 層 representation（包含 token embedding）。")

# 列出每層輸出的形狀
for idx, h in enumerate(hidden_states):
    print(f"Layer {idx:2d} 輸出形狀: {h.shape}")
    # h.shape = [batch_size, seq_len, hidden_size]
    # batch_size → 一次處理的句子數
    # sequence_length → 句子被切成多少 token
    # hidden_size → 每個 token 的向量長度


print("\n=== Token Embedding 輸出 ===")
print(hidden_states[0])

print("\n=== 第一個 Transformer Layer 的輸出 ===")
print(hidden_states[1])

```

看看模型内部的参数，主要是embedding层 和 transformer层

```html

# deepseek 1.5B的模型

print(model.num_parameters())

for name, param in model.named_parameters():
    print(f"{name:80}  |  shape: {tuple(param.shape)}")

151643  这个vocab_size的大小，151643是不是比151936（留一些空间，可以自定义）
1777088000 参数 17亿的参数量，差不多

model.embed_tokens.weight                                                         |  shape: (151936, 1536)
model.layers.0.self_attn.q_proj.weight                                            |  shape: (1536, 1536)
model.layers.0.self_attn.q_proj.bias                                              |  shape: (1536,)
model.layers.0.self_attn.k_proj.weight                                            |  shape: (256, 1536)
model.layers.0.self_attn.k_proj.bias                                              |  shape: (256,)
model.layers.0.self_attn.v_proj.weight                                            |  shape: (256, 1536)
model.layers.0.self_attn.v_proj.bias                                              |  shape: (256,)
model.layers.0.self_attn.o_proj.weight                                            |  shape: (1536, 1536)
model.layers.0.mlp.gate_proj.weight                                               |  shape: (8960, 1536)
model.layers.0.mlp.up_proj.weight                                                 |  shape: (8960, 1536)
model.layers.0.mlp.down_proj.weight                                               |  shape: (1536, 8960)
model.layers.0.input_layernorm.weight                                             |  shape: (1536,)
model.layers.0.post_attention_layernorm.weight                                    |  shape: (1536,)
model.layers.1.self_attn.q_proj.weight                                            |  shape: (1536, 1536)
model.layers.1.self_attn.q_proj.bias                                              |  shape: (1536,)
model.layers.1.self_attn.k_proj.weight                                            |  shape: (256, 1536)
model.layers.1.self_attn.k_proj.bias                                              |  shape: (256,)
model.layers.1.self_attn.v_proj.weight                                            |  shape: (256, 1536)
model.layers.1.self_attn.v_proj.bias                                              |  shape: (256,)
model.layers.1.self_attn.o_proj.weight                                            |  shape: (1536, 1536)
model.layers.1.mlp.gate_proj.weight                                               |  shape: (8960, 1536)
model.layers.1.mlp.up_proj.weight                                                 |  shape: (8960, 1536)
model.layers.1.mlp.down_proj.weight                                               |  shape: (1536, 8960)
model.layers.1.input_layernorm.weight                                             |  shape: (1536,)
model.layers.1.post_attention_layernorm.weight                                    |  shape: (1536,)
model.layers.2.self_attn.q_proj.weight                                            |  shape: (1536, 1536)
model.layers.2.self_attn.q_proj.bias                                              |  shape: (1536,)
model.layers.2.self_attn.k_proj.weight                                            |  shape: (256, 1536)
model.layers.2.self_attn.k_proj.bias                                              |  shape: (256,)
model.layers.2.self_attn.v_proj.weight                                            |  shape: (256, 1536)
model.layers.2.self_attn.v_proj.bias                                              |  shape: (256,)
model.layers.2.self_attn.o_proj.weight                                            |  shape: (1536, 1536)
model.layers.2.mlp.gate_proj.weight                                               |  shape: (8960, 1536)
model.layers.2.mlp.up_proj.weight                                                 |  shape: (8960, 1536)
model.layers.2.mlp.down_proj.weight                                               |  shape: (1536, 8960)
model.layers.2.input_layernorm.weight                                             |  shape: (1536,)
model.layers.2.post_attention_layernorm.weight                                    |  shape: (1536,)
model.layers.3.self_attn.q_proj.weight                                            |  shape: (1536, 1536)
model.layers.3.self_attn.q_proj.bias                                              |  shape: (1536,)
model.layers.3.self_attn.k_proj.weight                                            |  shape: (256, 1536)
model.layers.3.self_attn.k_proj.bias                                              |  shape: (256,)
model.layers.3.self_attn.v_proj.weight                                            |  shape: (256, 1536)
model.layers.3.self_attn.v_proj.bias                                              |  shape: (256,)
model.layers.3.self_attn.o_proj.weight                                            |  shape: (1536, 1536)
model.layers.3.mlp.gate_proj.weight                                               |  shape: (8960, 1536)
model.layers.3.mlp.up_proj.weight                                                 |  shape: (8960, 1536)
model.layers.3.mlp.down_proj.weight                                               |  shape: (1536, 8960)
model.layers.3.input_layernorm.weight                                             |  shape: (1536,)
model.layers.3.post_attention_layernorm.weight                                    |  shape: (1536,)
model.layers.4.self_attn.q_proj.weight                                            |  shape: (1536, 1536)
model.layers.4.self_attn.q_proj.bias                                              |  shape: (1536,)
model.layers.4.self_attn.k_proj.weight                                            |  shape: (256, 1536)
model.layers.4.self_attn.k_proj.bias                                              |  shape: (256,)
model.layers.4.self_attn.v_proj.weight                                            |  shape: (256, 1536)
model.layers.4.self_attn.v_proj.bias                                              |  shape: (256,)
model.layers.4.self_attn.o_proj.weight                                            |  shape: (1536, 1536)
model.layers.4.mlp.gate_proj.weight                                               |  shape: (8960, 1536)
model.layers.4.mlp.up_proj.weight                                                 |  shape: (8960, 1536)
model.layers.4.mlp.down_proj.weight                                               |  shape: (1536, 8960)
model.layers.4.input_layernorm.weight                                             |  shape: (1536,)
model.layers.4.post_attention_layernorm.weight                                    |  shape: (1536,)
model.layers.5.self_attn.q_proj.weight                                            |  shape: (1536, 1536)
model.layers.5.self_attn.q_proj.bias                                              |  shape: (1536,)
model.layers.5.self_attn.k_proj.weight                                            |  shape: (256, 1536)
model.layers.5.self_attn.k_proj.bias                                              |  shape: (256,)
model.layers.5.self_attn.v_proj.weight                                            |  shape: (256, 1536)
model.layers.5.self_attn.v_proj.bias                                              |  shape: (256,)
model.layers.5.self_attn.o_proj.weight                                            |  shape: (1536, 1536)
model.layers.5.mlp.gate_proj.weight                                               |  shape: (8960, 1536)
model.layers.5.mlp.up_proj.weight                                                 |  shape: (8960, 1536)
model.layers.5.mlp.down_proj.weight                                               |  shape: (1536, 8960)
model.layers.5.input_layernorm.weight                                             |  shape: (1536,)
model.layers.5.post_attention_layernorm.weight                                    |  shape: (1536,)
model.layers.6.self_attn.q_proj.weight                                            |  shape: (1536, 1536)
model.layers.6.self_attn.q_proj.bias                                              |  shape: (1536,)
model.layers.6.self_attn.k_proj.weight                                            |  shape: (256, 1536)
model.layers.6.self_attn.k_proj.bias                                              |  shape: (256,)
model.layers.6.self_attn.v_proj.weight                                            |  shape: (256, 1536)
model.layers.6.self_attn.v_proj.bias                                              |  shape: (256,)
model.layers.6.self_attn.o_proj.weight                                            |  shape: (1536, 1536)
model.layers.6.mlp.gate_proj.weight                                               |  shape: (8960, 1536)
model.layers.6.mlp.up_proj.weight                                                 |  shape: (8960, 1536)
model.layers.6.mlp.down_proj.weight                                               |  shape: (1536, 8960)
model.layers.6.input_layernorm.weight                                             |  shape: (1536,)
model.layers.6.post_attention_layernorm.weight                                    |  shape: (1536,)
model.layers.7.self_attn.q_proj.weight                                            |  shape: (1536, 1536)
model.layers.7.self_attn.q_proj.bias                                              |  shape: (1536,)
model.layers.7.self_attn.k_proj.weight                                            |  shape: (256, 1536)
model.layers.7.self_attn.k_proj.bias                                              |  shape: (256,)
model.layers.7.self_attn.v_proj.weight                                            |  shape: (256, 1536)
model.layers.7.self_attn.v_proj.bias                                              |  shape: (256,)
model.layers.7.self_attn.o_proj.weight                                            |  shape: (1536, 1536)
model.layers.7.mlp.gate_proj.weight                                               |  shape: (8960, 1536)
model.layers.7.mlp.up_proj.weight                                                 |  shape: (8960, 1536)
model.layers.7.mlp.down_proj.weight                                               |  shape: (1536, 8960)
model.layers.7.input_layernorm.weight                                             |  shape: (1536,)
model.layers.7.post_attention_layernorm.weight                                    |  shape: (1536,)
model.layers.8.self_attn.q_proj.weight                                            |  shape: (1536, 1536)
model.layers.8.self_attn.q_proj.bias                                              |  shape: (1536,)
model.layers.8.self_attn.k_proj.weight                                            |  shape: (256, 1536)
model.layers.8.self_attn.k_proj.bias                                              |  shape: (256,)
model.layers.8.self_attn.v_proj.weight                                            |  shape: (256, 1536)
model.layers.8.self_attn.v_proj.bias                                              |  shape: (256,)
model.layers.8.self_attn.o_proj.weight                                            |  shape: (1536, 1536)
model.layers.8.mlp.gate_proj.weight                                               |  shape: (8960, 1536)
model.layers.8.mlp.up_proj.weight                                                 |  shape: (8960, 1536)
model.layers.8.mlp.down_proj.weight                                               |  shape: (1536, 8960)
model.layers.8.input_layernorm.weight                                             |  shape: (1536,)
model.layers.8.post_attention_layernorm.weight                                    |  shape: (1536,)
model.layers.9.self_attn.q_proj.weight                                            |  shape: (1536, 1536)
model.layers.9.self_attn.q_proj.bias                                              |  shape: (1536,)
model.layers.9.self_attn.k_proj.weight                                            |  shape: (256, 1536)
model.layers.9.self_attn.k_proj.bias                                              |  shape: (256,)
model.layers.9.self_attn.v_proj.weight                                            |  shape: (256, 1536)
model.layers.9.self_attn.v_proj.bias                                              |  shape: (256,)
model.layers.9.self_attn.o_proj.weight                                            |  shape: (1536, 1536)
model.layers.9.mlp.gate_proj.weight                                               |  shape: (8960, 1536)
model.layers.9.mlp.up_proj.weight                                                 |  shape: (8960, 1536)
model.layers.9.mlp.down_proj.weight                                               |  shape: (1536, 8960)
model.layers.9.input_layernorm.weight                                             |  shape: (1536,)
model.layers.9.post_attention_layernorm.weight                                    |  shape: (1536,)
model.layers.10.self_attn.q_proj.weight                                           |  shape: (1536, 1536)
model.layers.10.self_attn.q_proj.bias                                             |  shape: (1536,)
model.layers.10.self_attn.k_proj.weight                                           |  shape: (256, 1536)
model.layers.10.self_attn.k_proj.bias                                             |  shape: (256,)
model.layers.10.self_attn.v_proj.weight                                           |  shape: (256, 1536)
model.layers.10.self_attn.v_proj.bias                                             |  shape: (256,)
model.layers.10.self_attn.o_proj.weight                                           |  shape: (1536, 1536)
model.layers.10.mlp.gate_proj.weight                                              |  shape: (8960, 1536)
model.layers.10.mlp.up_proj.weight                                                |  shape: (8960, 1536)
model.layers.10.mlp.down_proj.weight                                              |  shape: (1536, 8960)
model.layers.10.input_layernorm.weight                                            |  shape: (1536,)
model.layers.10.post_attention_layernorm.weight                                   |  shape: (1536,)
model.layers.11.self_attn.q_proj.weight                                           |  shape: (1536, 1536)
model.layers.11.self_attn.q_proj.bias                                             |  shape: (1536,)
model.layers.11.self_attn.k_proj.weight                                           |  shape: (256, 1536)
model.layers.11.self_attn.k_proj.bias                                             |  shape: (256,)
model.layers.11.self_attn.v_proj.weight                                           |  shape: (256, 1536)
model.layers.11.self_attn.v_proj.bias                                             |  shape: (256,)
model.layers.11.self_attn.o_proj.weight                                           |  shape: (1536, 1536)
model.layers.11.mlp.gate_proj.weight                                              |  shape: (8960, 1536)
model.layers.11.mlp.up_proj.weight                                                |  shape: (8960, 1536)
model.layers.11.mlp.down_proj.weight                                              |  shape: (1536, 8960)
model.layers.11.input_layernorm.weight                                            |  shape: (1536,)
model.layers.11.post_attention_layernorm.weight                                   |  shape: (1536,)
model.layers.12.self_attn.q_proj.weight                                           |  shape: (1536, 1536)
model.layers.12.self_attn.q_proj.bias                                             |  shape: (1536,)
model.layers.12.self_attn.k_proj.weight                                           |  shape: (256, 1536)
model.layers.12.self_attn.k_proj.bias                                             |  shape: (256,)
model.layers.12.self_attn.v_proj.weight                                           |  shape: (256, 1536)
model.layers.12.self_attn.v_proj.bias                                             |  shape: (256,)
model.layers.12.self_attn.o_proj.weight                                           |  shape: (1536, 1536)
model.layers.12.mlp.gate_proj.weight                                              |  shape: (8960, 1536)
model.layers.12.mlp.up_proj.weight                                                |  shape: (8960, 1536)
model.layers.12.mlp.down_proj.weight                                              |  shape: (1536, 8960)
model.layers.12.input_layernorm.weight                                            |  shape: (1536,)
model.layers.12.post_attention_layernorm.weight                                   |  shape: (1536,)
model.layers.13.self_attn.q_proj.weight                                           |  shape: (1536, 1536)
model.layers.13.self_attn.q_proj.bias                                             |  shape: (1536,)
model.layers.13.self_attn.k_proj.weight                                           |  shape: (256, 1536)
model.layers.13.self_attn.k_proj.bias                                             |  shape: (256,)
model.layers.13.self_attn.v_proj.weight                                           |  shape: (256, 1536)
model.layers.13.self_attn.v_proj.bias                                             |  shape: (256,)
model.layers.13.self_attn.o_proj.weight                                           |  shape: (1536, 1536)
model.layers.13.mlp.gate_proj.weight                                              |  shape: (8960, 1536)
model.layers.13.mlp.up_proj.weight                                                |  shape: (8960, 1536)
model.layers.13.mlp.down_proj.weight                                              |  shape: (1536, 8960)
model.layers.13.input_layernorm.weight                                            |  shape: (1536,)
model.layers.13.post_attention_layernorm.weight                                   |  shape: (1536,)
model.layers.14.self_attn.q_proj.weight                                           |  shape: (1536, 1536)
model.layers.14.self_attn.q_proj.bias                                             |  shape: (1536,)
model.layers.14.self_attn.k_proj.weight                                           |  shape: (256, 1536)
model.layers.14.self_attn.k_proj.bias                                             |  shape: (256,)
model.layers.14.self_attn.v_proj.weight                                           |  shape: (256, 1536)
model.layers.14.self_attn.v_proj.bias                                             |  shape: (256,)
model.layers.14.self_attn.o_proj.weight                                           |  shape: (1536, 1536)
model.layers.14.mlp.gate_proj.weight                                              |  shape: (8960, 1536)
model.layers.14.mlp.up_proj.weight                                                |  shape: (8960, 1536)
model.layers.14.mlp.down_proj.weight                                              |  shape: (1536, 8960)
model.layers.14.input_layernorm.weight                                            |  shape: (1536,)
model.layers.14.post_attention_layernorm.weight                                   |  shape: (1536,)
model.layers.15.self_attn.q_proj.weight                                           |  shape: (1536, 1536)
model.layers.15.self_attn.q_proj.bias                                             |  shape: (1536,)
model.layers.15.self_attn.k_proj.weight                                           |  shape: (256, 1536)
model.layers.15.self_attn.k_proj.bias                                             |  shape: (256,)
model.layers.15.self_attn.v_proj.weight                                           |  shape: (256, 1536)
model.layers.15.self_attn.v_proj.bias                                             |  shape: (256,)
model.layers.15.self_attn.o_proj.weight                                           |  shape: (1536, 1536)
model.layers.15.mlp.gate_proj.weight                                              |  shape: (8960, 1536)
model.layers.15.mlp.up_proj.weight                                                |  shape: (8960, 1536)
model.layers.15.mlp.down_proj.weight                                              |  shape: (1536, 8960)
model.layers.15.input_layernorm.weight                                            |  shape: (1536,)
model.layers.15.post_attention_layernorm.weight                                   |  shape: (1536,)
model.layers.16.self_attn.q_proj.weight                                           |  shape: (1536, 1536)
model.layers.16.self_attn.q_proj.bias                                             |  shape: (1536,)
model.layers.16.self_attn.k_proj.weight                                           |  shape: (256, 1536)
model.layers.16.self_attn.k_proj.bias                                             |  shape: (256,)
model.layers.16.self_attn.v_proj.weight                                           |  shape: (256, 1536)
model.layers.16.self_attn.v_proj.bias                                             |  shape: (256,)
model.layers.16.self_attn.o_proj.weight                                           |  shape: (1536, 1536)
model.layers.16.mlp.gate_proj.weight                                              |  shape: (8960, 1536)
model.layers.16.mlp.up_proj.weight                                                |  shape: (8960, 1536)
model.layers.16.mlp.down_proj.weight                                              |  shape: (1536, 8960)
model.layers.16.input_layernorm.weight                                            |  shape: (1536,)
model.layers.16.post_attention_layernorm.weight                                   |  shape: (1536,)
model.layers.17.self_attn.q_proj.weight                                           |  shape: (1536, 1536)
model.layers.17.self_attn.q_proj.bias                                             |  shape: (1536,)
model.layers.17.self_attn.k_proj.weight                                           |  shape: (256, 1536)
model.layers.17.self_attn.k_proj.bias                                             |  shape: (256,)
model.layers.17.self_attn.v_proj.weight                                           |  shape: (256, 1536)
model.layers.17.self_attn.v_proj.bias                                             |  shape: (256,)
model.layers.17.self_attn.o_proj.weight                                           |  shape: (1536, 1536)
model.layers.17.mlp.gate_proj.weight                                              |  shape: (8960, 1536)
model.layers.17.mlp.up_proj.weight                                                |  shape: (8960, 1536)
model.layers.17.mlp.down_proj.weight                                              |  shape: (1536, 8960)
model.layers.17.input_layernorm.weight                                            |  shape: (1536,)
model.layers.17.post_attention_layernorm.weight                                   |  shape: (1536,)
model.layers.18.self_attn.q_proj.weight                                           |  shape: (1536, 1536)
model.layers.18.self_attn.q_proj.bias                                             |  shape: (1536,)
model.layers.18.self_attn.k_proj.weight                                           |  shape: (256, 1536)
model.layers.18.self_attn.k_proj.bias                                             |  shape: (256,)
model.layers.18.self_attn.v_proj.weight                                           |  shape: (256, 1536)
model.layers.18.self_attn.v_proj.bias                                             |  shape: (256,)
model.layers.18.self_attn.o_proj.weight                                           |  shape: (1536, 1536)
model.layers.18.mlp.gate_proj.weight                                              |  shape: (8960, 1536)
model.layers.18.mlp.up_proj.weight                                                |  shape: (8960, 1536)
model.layers.18.mlp.down_proj.weight                                              |  shape: (1536, 8960)
model.layers.18.input_layernorm.weight                                            |  shape: (1536,)
model.layers.18.post_attention_layernorm.weight                                   |  shape: (1536,)
model.layers.19.self_attn.q_proj.weight                                           |  shape: (1536, 1536)
model.layers.19.self_attn.q_proj.bias                                             |  shape: (1536,)
model.layers.19.self_attn.k_proj.weight                                           |  shape: (256, 1536)
model.layers.19.self_attn.k_proj.bias                                             |  shape: (256,)
model.layers.19.self_attn.v_proj.weight                                           |  shape: (256, 1536)
model.layers.19.self_attn.v_proj.bias                                             |  shape: (256,)
model.layers.19.self_attn.o_proj.weight                                           |  shape: (1536, 1536)
model.layers.19.mlp.gate_proj.weight                                              |  shape: (8960, 1536)
model.layers.19.mlp.up_proj.weight                                                |  shape: (8960, 1536)
model.layers.19.mlp.down_proj.weight                                              |  shape: (1536, 8960)
model.layers.19.input_layernorm.weight                                            |  shape: (1536,)
model.layers.19.post_attention_layernorm.weight                                   |  shape: (1536,)
model.layers.20.self_attn.q_proj.weight                                           |  shape: (1536, 1536)
model.layers.20.self_attn.q_proj.bias                                             |  shape: (1536,)
model.layers.20.self_attn.k_proj.weight                                           |  shape: (256, 1536)
model.layers.20.self_attn.k_proj.bias                                             |  shape: (256,)
model.layers.20.self_attn.v_proj.weight                                           |  shape: (256, 1536)
model.layers.20.self_attn.v_proj.bias                                             |  shape: (256,)
model.layers.20.self_attn.o_proj.weight                                           |  shape: (1536, 1536)
model.layers.20.mlp.gate_proj.weight                                              |  shape: (8960, 1536)
model.layers.20.mlp.up_proj.weight                                                |  shape: (8960, 1536)
model.layers.20.mlp.down_proj.weight                                              |  shape: (1536, 8960)
model.layers.20.input_layernorm.weight                                            |  shape: (1536,)
model.layers.20.post_attention_layernorm.weight                                   |  shape: (1536,)
model.layers.21.self_attn.q_proj.weight                                           |  shape: (1536, 1536)
model.layers.21.self_attn.q_proj.bias                                             |  shape: (1536,)
model.layers.21.self_attn.k_proj.weight                                           |  shape: (256, 1536)
model.layers.21.self_attn.k_proj.bias                                             |  shape: (256,)
model.layers.21.self_attn.v_proj.weight                                           |  shape: (256, 1536)
model.layers.21.self_attn.v_proj.bias                                             |  shape: (256,)
model.layers.21.self_attn.o_proj.weight                                           |  shape: (1536, 1536)
model.layers.21.mlp.gate_proj.weight                                              |  shape: (8960, 1536)
model.layers.21.mlp.up_proj.weight                                                |  shape: (8960, 1536)
model.layers.21.mlp.down_proj.weight                                              |  shape: (1536, 8960)
model.layers.21.input_layernorm.weight                                            |  shape: (1536,)
model.layers.21.post_attention_layernorm.weight                                   |  shape: (1536,)
model.layers.22.self_attn.q_proj.weight                                           |  shape: (1536, 1536)
model.layers.22.self_attn.q_proj.bias                                             |  shape: (1536,)
model.layers.22.self_attn.k_proj.weight                                           |  shape: (256, 1536)
model.layers.22.self_attn.k_proj.bias                                             |  shape: (256,)
model.layers.22.self_attn.v_proj.weight                                           |  shape: (256, 1536)
model.layers.22.self_attn.v_proj.bias                                             |  shape: (256,)
model.layers.22.self_attn.o_proj.weight                                           |  shape: (1536, 1536)
model.layers.22.mlp.gate_proj.weight                                              |  shape: (8960, 1536)
model.layers.22.mlp.up_proj.weight                                                |  shape: (8960, 1536)
model.layers.22.mlp.down_proj.weight                                              |  shape: (1536, 8960)
model.layers.22.input_layernorm.weight                                            |  shape: (1536,)
model.layers.22.post_attention_layernorm.weight                                   |  shape: (1536,)
model.layers.23.self_attn.q_proj.weight                                           |  shape: (1536, 1536)
model.layers.23.self_attn.q_proj.bias                                             |  shape: (1536,)
model.layers.23.self_attn.k_proj.weight                                           |  shape: (256, 1536)
model.layers.23.self_attn.k_proj.bias                                             |  shape: (256,)
model.layers.23.self_attn.v_proj.weight                                           |  shape: (256, 1536)
model.layers.23.self_attn.v_proj.bias                                             |  shape: (256,)
model.layers.23.self_attn.o_proj.weight                                           |  shape: (1536, 1536)
model.layers.23.mlp.gate_proj.weight                                              |  shape: (8960, 1536)
model.layers.23.mlp.up_proj.weight                                                |  shape: (8960, 1536)
model.layers.23.mlp.down_proj.weight                                              |  shape: (1536, 8960)
model.layers.23.input_layernorm.weight                                            |  shape: (1536,)
model.layers.23.post_attention_layernorm.weight                                   |  shape: (1536,)
model.layers.24.self_attn.q_proj.weight                                           |  shape: (1536, 1536)
model.layers.24.self_attn.q_proj.bias                                             |  shape: (1536,)
model.layers.24.self_attn.k_proj.weight                                           |  shape: (256, 1536)
model.layers.24.self_attn.k_proj.bias                                             |  shape: (256,)
model.layers.24.self_attn.v_proj.weight                                           |  shape: (256, 1536)
model.layers.24.self_attn.v_proj.bias                                             |  shape: (256,)
model.layers.24.self_attn.o_proj.weight                                           |  shape: (1536, 1536)
model.layers.24.mlp.gate_proj.weight                                              |  shape: (8960, 1536)
model.layers.24.mlp.up_proj.weight                                                |  shape: (8960, 1536)
model.layers.24.mlp.down_proj.weight                                              |  shape: (1536, 8960)
model.layers.24.input_layernorm.weight                                            |  shape: (1536,)
model.layers.24.post_attention_layernorm.weight                                   |  shape: (1536,)
model.layers.25.self_attn.q_proj.weight                                           |  shape: (1536, 1536)
model.layers.25.self_attn.q_proj.bias                                             |  shape: (1536,)
model.layers.25.self_attn.k_proj.weight                                           |  shape: (256, 1536)
model.layers.25.self_attn.k_proj.bias                                             |  shape: (256,)
model.layers.25.self_attn.v_proj.weight                                           |  shape: (256, 1536)
model.layers.25.self_attn.v_proj.bias                                             |  shape: (256,)
model.layers.25.self_attn.o_proj.weight                                           |  shape: (1536, 1536)
model.layers.25.mlp.gate_proj.weight                                              |  shape: (8960, 1536)
model.layers.25.mlp.up_proj.weight                                                |  shape: (8960, 1536)
model.layers.25.mlp.down_proj.weight                                              |  shape: (1536, 8960)
model.layers.25.input_layernorm.weight                                            |  shape: (1536,)
model.layers.25.post_attention_layernorm.weight                                   |  shape: (1536,)
model.layers.26.self_attn.q_proj.weight                                           |  shape: (1536, 1536)
model.layers.26.self_attn.q_proj.bias                                             |  shape: (1536,)
model.layers.26.self_attn.k_proj.weight                                           |  shape: (256, 1536)
model.layers.26.self_attn.k_proj.bias                                             |  shape: (256,)
model.layers.26.self_attn.v_proj.weight                                           |  shape: (256, 1536)
model.layers.26.self_attn.v_proj.bias                                             |  shape: (256,)
model.layers.26.self_attn.o_proj.weight                                           |  shape: (1536, 1536)
model.layers.26.mlp.gate_proj.weight                                              |  shape: (8960, 1536)
model.layers.26.mlp.up_proj.weight                                                |  shape: (8960, 1536)
model.layers.26.mlp.down_proj.weight                                              |  shape: (1536, 8960)
model.layers.26.input_layernorm.weight                                            |  shape: (1536,)
model.layers.26.post_attention_layernorm.weight                                   |  shape: (1536,)
model.layers.27.self_attn.q_proj.weight                                           |  shape: (1536, 1536)
model.layers.27.self_attn.q_proj.bias                                             |  shape: (1536,)
model.layers.27.self_attn.k_proj.weight                                           |  shape: (256, 1536)
model.layers.27.self_attn.k_proj.bias                                             |  shape: (256,)
model.layers.27.self_attn.v_proj.weight                                           |  shape: (256, 1536)
model.layers.27.self_attn.v_proj.bias                                             |  shape: (256,)
model.layers.27.self_attn.o_proj.weight                                           |  shape: (1536, 1536)
model.layers.27.mlp.gate_proj.weight                                              |  shape: (8960, 1536)
model.layers.27.mlp.up_proj.weight                                                |  shape: (8960, 1536)
model.layers.27.mlp.down_proj.weight                                              |  shape: (1536, 8960)
model.layers.27.input_layernorm.weight                                            |  shape: (1536,)
model.layers.27.post_attention_layernorm.weight                                   |  shape: (1536,)
model.norm.weight                                                                 |  shape: (1536,)
lm_head.weight                                                                    |  shape: (151936, 1536)

weight = model.state_dict()["model.layers.27.mlp.up_proj.weight"].numpy()
input_embedding = model.state_dict()["model.embed_tokens.weight"].numpy()

# 比较明显第一层是embedding层，然后一共28个transformer_layer，然后一个 lm_head层（(151936, 1536)）

# q_proj, k_proj, v_proj：计算 Query、Key、Value。模型通过这三个矩阵来决定当前这个词应该关注句子里的哪些词。

这里k,v是256比q的1536小，类似使用了GQA（grouped query attention技术），能显著加快推理速度，o_proj输出投影，计算后的结果整合回1536维度

# FFN部分，gate_proj 和 up_proj（8960, 1536），把1536维生频到8960维，类似把信息拆解的更细，down_proj（1536, 8960），再把信息压回1536维

# input_layernorm 和 post_attention_layernorm，负责把每一层输出的数值重新拉回到一个稳定的范围内，防止训练时梯度爆炸。

# 是最终归一化层，在数据离开所有 Transformer 层、进入最后一层预测之前，再做最后一次全身检查，确保数值平稳。

# lm_head.weight (151936, 1536)，输出头，就是出口，要反正看！！！，是weight，第一位是输出的维度！！！

```

#### 利用多张GPU训练大语言模型（解决大模型内存不足问题的一些方法）

DeepSpeed、Flash attention、liger kernel 和 quantization

单个GPU基本训练不鸟LLM。DeepSpeed可以更好的利用GPU进行训练，如果输入非常长，用Flash attention 和 liger kernel，如果使用colab上的GPU去跑，small vRAM interence，就可以用到quantization。

假设模型是8B的，光模型存到GPU的话需要下面的大小，通常会把fp32转成16fp。小了也快了，大部分英伟达的GPU对16有加速。

fp32: 8B params = 8 * 10^9 * 32bit = 32GB
fp16: 8B params = 8 * 10^9 * 16bit = 16GB

缩小后，LLM的weights是16GB，训练后做反向传播的grad也是要存的，也是16GB的话，而使用adam算法后，还要有momentum 和 variance的参数，使用原始的32GB，全部加起来太大了 128GB。

另外一个大小是input，假设输入是 256 tokens，大的话可能到16k tokens。模型每一层的输出是要存下来的，8B的模型会有32层。每层有attention，ffnn，layer norm等等，也占大小

activation recomputation (也就是grad checkpoint)，意思就是在forward的时候，你不要把所有的层输出都存下来，只存一些重要的。backward要用到的时候重新算一次就行

LLM的batch size通常很大 通常会有 4-60M tokens per batch。DeepSeek V3的batch size 是1920，32K context，61M tokens。

gradient accumulation，就是再切分，1920 = 16 * 120，把16个丢进去更新，得到的grad先不更新，等到120个跑完累加后再更新。

优化分成3个部分

1、params、gradients 和 optimizer states

128GB一个GPU装不下就用多个，注意是每个组件都切分到对应的GPU中，主要考虑怎么切，微软的工具deepspeed，zero redundancy optimizer。有zero-1（切optimizer states部分，最多的）、zero-2（LLM 32 和 grad）、zero-3（LLM 16）。gpu之间传输数据的速度nvlink： 900GB/s是很快的。

假设一个8B的模型，用的H100，是训练不鸟的，开了zero-1，zero-2，zero-3后就可以训练了，切分后的占用空间减少

zero-offload，实在太大，还可以先放到cpu的ram。只要cpu的ram够大都可以放过去。但是GPU和CPU的传输速度会非常慢！4张A100，一张32GB。不会OOM。大致上的话，8B的模型，4张A100的GPU能跑了。卡多的话能训练的模型就越大了。

2、activations

attention 要在GPU上跑的，所以可以重写attention的code，跑在GPU上的function。

flash attention algorithm。 faster training & less memory by optimized fetching from cpu ram。以前space是O(n^2)，现在就是O(n)。q、k、v 和 A 放到cpu上，也是减小内存消耗。flash attention这个东西在pytorch上本身也支持了

另外一个是 liger kernel，加载模型的时候改下方式

如果训练的模型context太长，是个视频啥的，就可以使用上面的方法

3、quantization（量化）

lossy compression(有损压缩)， 32bit quantization algorithm -> 8bit int4 -> dequantization algorithm -> 32bit（近似）

存储的时候用压缩后的存，算法有GGML family、GPTQ、AWQ、BitsAndBytes。一个8B的模型压缩到 8B-Instruct-Q8_0，8bit的话，只需要8GB去载入就行了，colab上免费的T4、15GB的内存

#### 模型加载

48GB的mac，DeepSeek-R1-Distill-Qwen-1.5B 和 DeepSeek-R1-Distill-Llama-8B 可以加载

DeepSeek-R1-Distill-Qwen-32B 是无法正常加载的，加载分片checkpoint时候被OOM killer强制杀死

```html

Loading checkpoint shards: 38%|███▊      | 3/8
Process finished with exit code 137 (SIGKILL)

32B的大概率是内存不足的

AutoModelForCausalLM.from_pretrained的时候可以加上  

device_map="auto" # 让mac加载的时候不专门走cpu，而是mps

# 这个需要安装 pip install accelerate

# 这个还没有加量化，就能跑起来了，但是.generate()，时候inputIds要带上device

device = model.device

input_ids.to(device))

# 这样的结果就是推理起来很慢，32B的跑完步回来还在推，试试加上量化，但是

# bitsandbytes-macos 不支持 4bit、8bit GPU 加速，MPS 没有对应的 kernel，MPS 没有对应的 kernel，代码直接报错

# 比较好的做法就是 device_map="auto" + torch_dtype="float16"


```

mac实在不行的话，直接使用mlx，官方亲儿子，下载量化模型就行

```html

# 做量化

mlx_lm.convert --hf-path /Users/zqmalyssa/Model/deepseek/DeepSeek-R1-Distill-Qwen-32B --mlx-path /Users/zqmalyssa/Model/mlx/DeepSeek-R1-Distill-Qwen-32B-4bit -q  --q-bits 4

# pycharm 的 debug用 mlx会有bug，可以转成命令行去用

# 32B的模型转成 4-bit量化，能加载

/Users/zqmalyssa/miniconda3/envs/ai/bin/python -c "from mlx_lm import load, generate; local_mode = '/Users/zqmalyssa/Model/mlx/DeepSeek-R1-Distill-Qwen-32B-4bit'; model, tokenizer = load(local_mode); print('success')"

# user's 中的 ' 要转义一下
/Users/zqmalyssa/miniconda3/envs/ai/bin/python -c "from mlx_lm import load, generate; import time; local_mode = '/Users/zqmalyssa/Model/mlx/DeepSeek-R1-Distill-Qwen-32B-4bit'; model, tokenizer = load(local_mode); print('success'); messages = [{'role':'system', 'content':'A chat between a curious user and an artificial intelligence assistant. The assistant gives helpful, detailed, and polite answers to the user\\'s questions.'},{'role':'user','content':'who are you?'}]; chat_template = tokenizer.apply_chat_template(messages,add_generation_prompt=True,tokenize=False); start = time.perf_counter(); response = generate(model, tokenizer, prompt=chat_template, verbose=True); end = time.perf_counter(); print(f'耗时: {end - start:.6f} 秒');"

# 输出如下，
# Prompt: 37 tokens, 34.201 tokens-per-sec
# Generation: 81 tokens, 12.762 tokens-per-sec
# Peak memory: 18.566 GB
# 耗时: 9.302607 秒

# prompt是并行计算，generation是逐token串行计算，因此generation永远更慢。transformer的自注意力可以同时处理所有输入token。
# 计算量是很大，但是gpu和cpu是完全可以并行的，prefill吞吐量非常高，甚至能达到几万token/s
# 生成阶段属于 自回归模型，必须一个token一个token的生成，严格串行，不能并发，每个token都要做一次完整的forward，层数越多、模型越大就越慢
# 在generation阶段，模型会利用KV Cache（注意力缓存）prefill阶段，必须计算整个序列的所有K/V，Generate 阶段：每次生成只需计算新的 token，再跟旧 cache 做 attention
# 确实做完量化后，模型加载 和 generate的速度快了很多

/Users/zqmalyssa/miniconda3/envs/ai/bin/python -c "from mlx_lm import load, generate; local_mode = '/Users/zqmalyssa/Model/mlx/DeepSeek-R1-Distill-Qwen-32B-4bit'; model, tokenizer = load(local_mode); print(tokenizer.decode(71486))"

# 71486 是 Alright

```

#### 模型启动和调用

```html

# 对话
mlx_lm.generate --model /Users/zqmalyssa/Model/mistral/Mistral-7B-Instruct-v0.3 --prompt "hello, who are you"

# Prompt: 9 tokens, 39.017 tokens-per-sec
# Generation: 30 tokens, 17.320 tokens-per-sec
# Peak memory: 14.631 GB

# 本地启动模型
mlx_lm.server --model /Users/zqmalyssa/Model/ai/DeepSeek-R1-Distill-Qwen-32B-4bit/

# 会在 127.0.0.1:8080 启动大模型

# 查看模型名称
curl http://127.0.0.1:8080/v1/models

# 开始一些对话
curl http://127.0.0.1:8080/v1/chat/completions -H "Content-Type: application/json" -d '{"model":"default_model","messages":[{"role":"user","content":"hi"}]}'

# 不加就是默认model:default_model
curl http://127.0.0.1:8080/v1/chat/completions -H "Content-Type: application/json" -d '{"messages":[{"role":"user","content":"hi"}]}'

# "default_model" 是特殊标识，表示使用启动服务器时通过 --model 参数指定的模型（即 /Users/zqmalyssa/Model/ai/DeepSeek-R1-Distill-Qwen-32B-4bit/）
# 如果指定其他名称（如 "local"），服务器会将其视为模型路径，找不到时返回 404


```

#### 模型微调

```html

# 如果不微调
mlx_lm.generate --model /Users/zqmalyssa/Model/mistral/Mistral-7B-Instruct-v0.3 --prompt "You are an text to SQL query translator. Users will ask you questions in English and you will generate a SQL query based on the provided SCHEMA.\nSCHEMA:\nCREATE TABLE table_13505192_3 (series_number INTEGER, season_number VARCHAR)\nUser:What is the series number for season episode 24?\nAssistant:"

==========
To find the series number for season episode 24, we first need to find the row where the season_number is 24, and then extract the series_number. Here's the SQL query for that:

``sql
SELECT series_number FROM table_13505192_3 WHERE season_number = 24;
``

This query will return the series number for season 24 if it exists in the
==========

现在进行微调，具体可以看看lora文件的参数

/Users/zqmalyssa/miniconda3/envs/ai/bin/python /Users/zqmalyssa/Code/mlx-examples/lora/lora.py --model /Users/zqmalyssa/Model/mistral/Mistral-7B-Instruct-v0.3 --train --batch-size 1 --lora-layers 4 --data /Users/zqmalyssa/Code/pycharmproject/ai/mlx/my-data-text


# Loading pretrained model
# Total parameters 7248.450M
# Trainable parameters 0.426M  # --lora-layers 4 参数还是会有变化

# 主要这里的batch_size是1？

Iter 600: Val loss 0.805, Val took 8.354s
Iter 600: Saved adapter weights to adapters.npz.
Iter 610: Train loss 0.596, It/sec 2.225, Tokens/sec 266.348
Iter 620: Train loss 0.580, It/sec 2.527, Tokens/sec 275.439
Iter 630: Train loss 0.609, It/sec 2.232, Tokens/sec 279.239
Iter 640: Train loss 0.629, It/sec 2.519, Tokens/sec 281.145
Iter 650: Train loss 0.650, It/sec 2.328, Tokens/sec 279.341
Iter 660: Train loss 0.548, It/sec 2.433, Tokens/sec 264.989
Iter 670: Train loss 0.549, It/sec 2.652, Tokens/sec 267.285
Iter 680: Train loss 0.538, It/sec 2.344, Tokens/sec 269.608
Iter 690: Train loss 0.647, It/sec 2.337, Tokens/sec 279.255
Iter 700: Train loss 0.593, It/sec 2.349, Tokens/sec 273.468
Iter 700: Saved adapter weights to adapters.npz.
Iter 710: Train loss 0.507, It/sec 2.545, Tokens/sec 259.565
Iter 720: Train loss 0.527, It/sec 2.424, Tokens/sec 270.706
Iter 730: Train loss 0.549, It/sec 2.390, Tokens/sec 279.166
Iter 740: Train loss 0.575, It/sec 2.417, Tokens/sec 279.600
Iter 750: Train loss 0.528, It/sec 2.403, Tokens/sec 255.892
Iter 760: Train loss 0.662, It/sec 2.299, Tokens/sec 287.629
Iter 770: Train loss 0.633, It/sec 2.238, Tokens/sec 265.868
Iter 780: Train loss 0.540, It/sec 2.423, Tokens/sec 279.604
Iter 790: Train loss 0.525, It/sec 2.506, Tokens/sec 275.429
Iter 800: Train loss 0.637, It/sec 2.240, Tokens/sec 278.031
Iter 800: Val loss 0.806, Val took 8.350s
Iter 800: Saved adapter weights to adapters.npz.
Iter 810: Train loss 0.551, It/sec 2.428, Tokens/sec 274.631
Iter 820: Train loss 0.406, It/sec 2.556, Tokens/sec 259.207
Iter 830: Train loss 0.529, It/sec 2.347, Tokens/sec 269.943
Iter 840: Train loss 0.495, It/sec 2.303, Tokens/sec 265.725
Iter 850: Train loss 0.505, It/sec 2.635, Tokens/sec 270.928
Iter 860: Train loss 0.562, It/sec 2.525, Tokens/sec 279.214
Iter 870: Train loss 0.621, It/sec 2.411, Tokens/sec 299.894
Iter 880: Train loss 0.549, It/sec 2.420, Tokens/sec 282.908
Iter 890: Train loss 0.616, It/sec 2.197, Tokens/sec 276.618
Iter 900: Train loss 0.592, It/sec 2.253, Tokens/sec 271.471
Iter 900: Saved adapter weights to adapters.npz.
Iter 910: Train loss 0.510, It/sec 2.485, Tokens/sec 277.300
Iter 920: Train loss 0.503, It/sec 2.252, Tokens/sec 266.677
Iter 930: Train loss 0.475, It/sec 2.640, Tokens/sec 269.269
Iter 940: Train loss 0.624, It/sec 2.039, Tokens/sec 279.359
Iter 950: Train loss 0.433, It/sec 2.428, Tokens/sec 266.091
Iter 960: Train loss 0.549, It/sec 2.330, Tokens/sec 272.147
Iter 970: Train loss 0.509, It/sec 2.311, Tokens/sec 277.141
Iter 980: Train loss 0.514, It/sec 2.483, Tokens/sec 264.905
Iter 990: Train loss 0.558, It/sec 2.389, Tokens/sec 267.335
Iter 1000: Train loss 0.532, It/sec 2.615, Tokens/sec 293.167
Iter 1000: Val loss 0.838, Val took 8.349s
Iter 1000: Saved adapter weights to adapters.npz.

每两百次有一个val loss，每100次有一个weights to adapters.npz

# fuse一下得到的参数，会在my-models中得到新的模型
/Users/zqmalyssa/miniconda3/envs/ai/bin/python /Users/zqmalyssa/Code/mlx-examples/lora/fuse.py --model /Users/zqmalyssa/Model/mistral/Mistral-7B-Instruct-v0.3 --save-path my-models --adapter-file adapters.npz

# 再试下
mlx_lm.generate --model /Users/zqmalyssa/Model/mistral/Mistral-7B-Instruct-v0.3/my-models --prompt "You are an text to SQL query translator. Users will ask you questions in English and you will generate a SQL query based on the provided SCHEMA.\nSCHEMA:\nCREATE TABLE table_13505192_3 (series_number INTEGER, season_number VARCHAR)\nUser:What is the series number for season episode 24?\nAssistant:"

==========
Assuming that there is a join with a table that has the episode number and a foreign key to the series number, the SQL query would be:

```sql
SELECT series_number
FROM table_13505192_3
JOIN episodes_table_name
ON table_13505192_3.season_number = episodes_table_name.season_number
WHERE episodes_table_name.ep
==========

# 感觉直接炸了，看看训练数据是不是不在里面 dataset = dataset.shuffle().select(range(150))
# 直接全部数据，dataset = dataset.train_test_split(test_size=10000/68577)

# batch_size 换成 10，同时 数据集换成全部 78577，
/Users/zqmalyssa/miniconda3/envs/ai/bin/python /Users/zqmalyssa/Code/mlx-examples/lora/lora.py --model /Users/zqmalyssa/Model/mistral/Mistral-7B-Instruct-v0.3 --train --batch-size 10 --lora-layers 4 --data /Users/zqmalyssa/Code/pycharmproject/ai/mlx/my-data-text

# 训练的非常慢

# By default, the adapter weights are saved in adapters.npz. You can specify the output location with --adapter-file.
# You can resume fine-tuning with an existing adapter with --resume-adapter-file <path_to_adapters.npz>.
# 也就是说从微调的文件中是可以恢复微调的

Training
Iter 1: Val loss 2.823, Val took 96.562s
Iter 10: Train loss 2.533, It/sec 0.293, Tokens/sec 340.651
Iter 20: Train loss 1.848, It/sec 0.246, Tokens/sec 294.206
Iter 30: Train loss 1.419, It/sec 0.285, Tokens/sec 323.683
Iter 40: Train loss 1.133, It/sec 0.268, Tokens/sec 308.426
Iter 50: Train loss 1.016, It/sec 0.249, Tokens/sec 289.448
Iter 60: Train loss 0.944, It/sec 0.236, Tokens/sec 271.317
Iter 70: Train loss 0.913, It/sec 0.251, Tokens/sec 288.560
Iter 80: Train loss 0.891, It/sec 0.260, Tokens/sec 301.224
Iter 90: Train loss 0.888, It/sec 0.240, Tokens/sec 275.983
Iter 100: Train loss 0.850, It/sec 0.278, Tokens/sec 310.049
Iter 100: Saved adapter weights to adapters.npz.
Iter 110: Train loss 0.837, It/sec 0.270, Tokens/sec 309.868
Iter 120: Train loss 0.838, It/sec 0.264, Tokens/sec 299.644
Iter 130: Train loss 0.863, It/sec 0.229, Tokens/sec 271.044
Iter 140: Train loss 0.854, It/sec 0.232, Tokens/sec 266.619
Iter 150: Train loss 0.817, It/sec 0.255, Tokens/sec 290.562
Iter 160: Train loss 0.804, It/sec 0.252, Tokens/sec 286.116
Iter 170: Train loss 0.851, It/sec 0.240, Tokens/sec 287.483
Iter 180: Train loss 0.843, It/sec 0.231, Tokens/sec 265.071
Iter 190: Train loss 0.833, It/sec 0.223, Tokens/sec 256.570
Iter 200: Train loss 0.806, It/sec 0.241, Tokens/sec 272.924
Iter 200: Val loss 0.829, Val took 102.787s
Iter 200: Saved adapter weights to adapters.npz.

# 再试下
==========
SELECT series_number FROM table_13505192_3 WHERE season_number = "episode 24"
==========

没有什么assuming了，但是value值还是不太对

```

#### 模型部署

GGUF（GGUF，GPT-Generated Unified Format）是由 llama.cpp 的作者推出、为轻量快速推理优化过的二进制模型 + metadata 格式。Ollama 是基于 llama.cpp 的实现，因此自然用这个格式作为标准

不过，最近 Ollama 文档说明它支持从 safetensors 权重导入模型了，之前应该只是说支持。Ollama可以直接二进制包安装，性能会更好点，也可以docker安装

```html

ollama create my-local-model-sql -f /Users/zqmalyssa/Model/modelfile/Modelfile

Error: unsupported architecture "MistralForCausalLM"

# 模型路径 /Users/zqmalyssa/Model/deepseek/DeepSeek-R1-Distill-Llama-8B

# 切换一下架构，不用Mistral，用ds的llama蒸馏版

gathering model components
copying file sha256:8ac8c85fb242563c2260baec0909debd69d718af6a0b3d90e6cab62b4d341cd5 100%
copying file sha256:54acfad3cffe057640904ca8a1e83525e6551c70c7a04c641f5a9eda0bbf64bd 100%
copying file sha256:83bdf4be4bb1a054ff315cd804554c48a88036226fdfbc65bee84ff562fea32a 100%
copying file sha256:b9c9eb63a8e03059914880f918cd28a880dec8b6e15e4461e1ff677e3743dbb8 100%
copying file sha256:7e6b24744354ef4ba547547cc758339090f46ba2da917845cfc69f7d4ded9edb 100%
copying file sha256:cd5194726d1e8f7361a8c8425fc11d33ade5e69de1fd7615eb23fae5601af68b 100%
copying file sha256:19fb83b79bd0d06d49b7cf6f86b83f5183cd292aa2c028d633ca4fceac1ae742 100%
converting model
creating new layer sha256:42cd1ab0f3c728b4d17c43ebac91212eb5aa8aa8cf22f83732be60aa67ec6ca3
writing manifest
success

```

以上的模型就导入到ollama中

```html

# 可以进行cmd的对话
ollama run xxx

/bye
# 退出

# converting model
# Error: unknown data type: U32

# 对于 Ollama 来说，它的 loader 目前 不支持 safetensors / metadata 中的 U8 / U32 / unsigned-int。所以如果模型 metadata 里有这类 dtype，在 ollama create / import 等步骤就会失败 (“unknown data type”)
# 对于 llama.cpp + 转换脚本 convert_hf_to_gguf.py 来说，脚本内就在 dtype map 中注释掉了对 U32 / U16 / U64 的映射支持 —— 意味作者知道兼容性问题

# 32B 量化的会有以上问题，但32B本身没有这个问题，不过32B也难加载了(过于大了，1.5B很轻松)

# 这段转换估计类似，ollama底层用的就是lama.cpp
python3 ../../External/llama.cpp/convert.py models/sql-model \
--outfile models/converted.gguf --outtype q8_0 --vocab-type bpe
# 可以不从mlx走，直接用llama.cpp


```

#### openai-style的api

以下是使用mlx.server 拉起来的模型

```html

curl http://127.0.0.1:8080/v1/models

# responses 接口目前mlx没有适配
curl http://127.0.0.1:8080/v1/responses -H "Content-Type: application/json" -d '{"input":"Tell me a three sentence bedtime story about a unicorn."}'

# conversation 接口目前mlx没有适配

# 改用
# 聊天补全：/v1/chat/completions  （聊天，都是使用chat模式训练的）

{
  "model": "xxx",
  "messages": [
    {"role": "system", "content": "你是一个助手"},
    {"role": "user",   "content": "你好"}
  ]
}

<|im_start|>system
你是助手
<|im_end|>
<|im_start|>user
你好
<|im_end|>
<|im_start|>assistant

# 纯文本补全：/v1/completions   （纯文本，gpt3的产物）

{
  "model": "xxx",
  "prompt": "写一段关于月亮的诗"
}

# 也是 404
curl http://127.0.0.1:8080/v1/chat/completions/chatcmpl-2a2f3d9f-cd52-47ba-a24d-6d6a98757191

# 下面也没有实现
/v1/chat/completions/{completion_id}

```

以下是使用ollama拉起来的模型

```html

# 列出模型
curl http://127.0.0.1:11434/api/tags

{
  "models": [
    {
      "name": "my-local-model-llama-8B:latest",
      "model": "my-local-model-llama-8B:latest",
      "modified_at": "2025-12-08T12:51:36.537411698+08:00",
      "size": 16069406438,
      "digest": "964ebc25b0e722411a23ee0af137ec37bc76c0c23e482b1b64ac11a82b194e65",
      "details": {
        "parent_model": "",
        "format": "gguf",
        "family": "llama",
        "families": [
          "llama"
        ],
        "parameter_size": "8.0B",
        "quantization_level": "F16"
      }
    }
  ]
}


# 生成回答，并不需要ollama去run起来模型，只要ollama serve就可以了
curl http://localhost:11434/api/generate -d '{
  "model": "my-local-model-llama-8B:latest",
  "prompt": "Why is the sky blue?",
  "stream": false
}'

# 默认stream是true的话是一个token一个token返回的，看下面，这种就可以流式处理

{"model":"my-local-model-llama-8B:latest","created_at":"2025-12-08T07:05:00.827516Z","response":" appear","done":false}
{"model":"my-local-model-llama-8B:latest","created_at":"2025-12-08T07:05:00.896007Z","response":" blue","done":false}
{"model":"my-local-model-llama-8B:latest","created_at":"2025-12-08T07:05:00.964109Z","response":" to","done":false}
{"model":"my-local-model-llama-8B:latest","created_at":"2025-12-08T07:05:01.031439Z","response":" us","done":false}
{"model":"my-local-model-llama-8B:latest","created_at":"2025-12-08T07:05:01.100422Z","response":".","done":false}
{"model":"my-local-model-llama-8B:latest","created_at":"2025-12-08T07:05:01.167261Z","response":"","done":true,"done_reason":"stop","context":[10445,374,279,13180,6437,30,1102,596,1606,40120,1156,10385,810,30820,1022,315,35715,304,279,16975,994,814,527,520,24210,93959,11,902,8024,311,6437,323,80836,3177,13,3277,584,1427,709,389,264,40798,1938,11,1057,6548,527,810,16614,311,6437,3177,11,779,430,596,3249,279,13180,8111,6437,382,14524,11,1095,757,1781,1578,13,578,13558,64069,72916,10334,15100,420,25885,1405,24210,93959,45577,810,13750,1606,814,16681,449,9333,19252,304,279,16975,13,1115,72916,11384,279,6437,323,80836,8146,311,41836,279,13180,439,3970,505,9420,753,7479,11,1418,5129,93959,1093,2579,527,38067,2753,323,8541,311,2586,1555,505,3485,11,6968,264,63244,819,40140,520,64919,477,44084,382,4071,3604,11,3249,656,584,1518,279,13180,6437,30,1102,596,1606,6437,3177,374,38067,810,555,279,16975,13,5810,596,3249,25,40120,374,1903,709,315,2204,8146,11,449,80836,3515,279,40243,46406,323,2579,279,22807,13,3277,40120,29933,9420,596,16975,11,433,84261,449,35715,323,2678,19252,304,279,3805,13,8868,706,264,24210,46406,7863,311,1023,9621,8146,3734,369,80836,382,14524,11,719,358,3463,80836,574,3604,38067,810,1109,6437,1606,80836,706,1524,264,24210,46406,13,4452,11,1057,6548,527,2753,16614,311,80836,3177,7863,311,6437,11,779,584,45493,279,13180,439,6437,4619,315,80836,13,2100,430,2011,387,3249,279,13180,8111,6437,2391,279,1938,382,81122,11,7344,433,596,1101,5552,311,13558,64069,72916,1405,19252,304,279,16975,45577,3177,304,682,18445,13,578,24210,279,46406,11,279,810,11297,420,72916,374,11,902,11384,6437,323,80836,311,387,38067,810,11,719,2533,584,2351,2753,16614,311,80836,11,1057,6548,3820,709,389,279,9861,6437,3177,11,3339,279,13180,1427,6437,382,24220,11,358,1781,430,596,433,13,2100,11,279,13180,8111,6437,1606,24210,93959,320,12481,5574,32012,8,45577,810,304,279,16975,11,323,1418,1070,374,810,72916,369,1521,8146,11,1057,6548,527,810,16614,311,6437,11,779,584,1518,430,1933,47904,627,128014,271,791,13180,8111,6437,2391,279,1938,4245,311,264,25885,3967,439,13558,64069,72916,13,8219,4238,11,24306,315,5370,8146,449,2204,93959,11,29933,9420,596,16975,13,10928,261,93959,11,1778,439,6437,323,80836,11,16681,810,30820,449,45475,19252,11,14718,1124,311,45577,304,682,18445,13,6104,2225,6437,323,80836,527,38067,11,3823,6548,527,2753,16614,311,80836,3177,13,15636,11,279,96531,1933,584,45493,374,6437,11,3339,279,13180,5101,6437,311,603,13],"total_duration":35907127166,"load_duration":2795949541,"prompt_eval_count":7,"prompt_eval_duration":410131333,"eval_count":486,"eval_duration":30603510662}

# 一些可选项

"options": {
    "temperature": 0.8,
    "top_p": 0.9,
    "seed": 42
}

# chat
curl http://localhost:11434/api/chat -d '{
  "model": "my-local-model-llama-8B:latest",
  "messages": [
    {
      "role": "user",
      "content": "who are you?"
    }
  ],
  "stream": false
}'

# embedding
curl http://localhost:11434/api/embed -d '{
  "model": "my-local-model-llama-8B:latest",
  "input": "who are you?"
}'

# list running models
curl http://localhost:11434/api/ps


```

另外，一般会看一下返回的指标的

```html

total_duration: How long the response took to generate (生成答案的时间)
load_duration: How long the model took to load (model加载的时间)
prompt_eval_count: How many input tokens were processed ()
prompt_eval_duration: How long it took to evaluate the prompt
eval_count: How many output tokens were processes
eval_duration: How long it took to generate the output tokens

All timing values are measured in nanoseconds.

```

#### 模型开发

如果是用openai风格的sdk，调用需要花钱的情况下，经常是余额不足了，但是如果用ollama后，就可以本地搞了

```html

from openai import OpenAI

# 改成使用本地ollama部署的，然后是 OpenAI compatibility
# 完全不同改 openai的 sdk
client = OpenAI(
    base_url='http://localhost:11434/v1/',

    # required but ignored
    api_key='ollama',
)

# 列出模型
list_completion = client.models.list()

# 进行对话
chat_completion = client.chat.completions.create(
    messages=[
        {
            'role': 'user',
            'content': 'Say this is a test',
        }
    ],
    model='my-local-model-llama-8B:latest',
)


```


这个跟deepseek的api是一样的，使用和openai兼容的api格式，通过修改配置，可以使用openai sdk来访问deepseek api，或使用和openai api兼容的软件

base_url：https://api.deepseek.com

出于与 OpenAI 兼容考虑，您也可以将 base_url 设置为 https://api.deepseek.com/v1 来使用，但注意，此处 v1 与模型版本无关。

deepseek-chat 和 deepseek-reasoner 对应模型版本不变，为 DeepSeek-V3.2 (128K 上下文长度)，与 APP/WEB 版不同。deepseek-chat 对应 DeepSeek-V3.2 的非思考模式，deepseek-reasoner 对应 DeepSeek-V3.2 的思考模式

尝试一下sdk，应该是很标准的一次输入输出的，可以调用本地的了

```html

from openai import OpenAI

client = OpenAI(
    base_url='http://localhost:11434/v1/',

    # required but ignored
    api_key='ollama',
)

# 可以再换成千问的了
model = "qwen3:14b"

# 提示词优化
prompt = '''请判断下面每组数中，“所有奇数之和是否为偶数”。  
每组输出 “True” 或 “False”。  

示例：  
4、8、9、15、12、2、1 → 奇数 9 + 15 + 1 = 25 → 奇数和为奇数 → 输出 False  
17、10、19、4、8、12、24 → 奇数 17 + 19 = 36 → 奇数和为偶数 → 输出 True  
16、11、14、4、8、13、24 → 奇数 11 + 13 = 24 → 奇数和为偶数 → 输出 True  
17、9、10、12、13、4、2 → 奇数 17 + 9 + 13 = 39 → 奇数和为奇数 → 输出 False  

请你判断：  
15、32、5、13、82、7、1 → （请输出 True 或 False)
'''

# sdk的示例，包含各种参数

# 这是对话补全API！！！！！！
chat_completion = client.chat.completions.create(
    messages=[
        {'role': 'user','content': prompt},
        {"role": "system", "content": "你是一个严谨、条理清楚的助手。"},
    ],
    model=model,
    # max_tokens = 10,
    temperature=0.7,
    # top_p=1.0,
    frequency_penalty=0.0,
    # presence_penalty=0.0,
)

print(chat_completion.choices[0].message.content)

```

这个api一些关键指标

```html

request部分：

thinking中

使用思考模式和不使用思考模式

frequency_penalty：-2到2之间的数字，如果该值为正，那么新 token 会根据其在已有文本中的出现频率受到相应的惩罚，降低模型重复相同内容的可能性。

max_tokens：限制一次请求中模型生成 completion 的最大 token 数。输入 token 和输出 token 的总长度受模型的上下文长度的限制。比如DS V3.2上下文长度128K，输出长度非思考默认4K，最大8K，思考默认32K，最大64K（包含思维链输出）

presence_penalty：-2到2之间的数字，如果该值为正，那么新 token 会根据其是否已在已有文本中出现受到相应的惩罚，从而增加模型谈论新主题的可能性。

stop：一个 string 或最多包含 16 个 string 的 list，在遇到这些词时，API 将停止生成更多的 token。

temperature：采样温度，介于 0 和 2 之间。更高的值，如 0.8，会使输出更随机，而更低的值，如 0.2，会使其更加集中和确定。 我们通常建议可以更改这个值或者更改 top_p，但不建议同时对两者进行修改。

top_p：作为调节采样温度的替代方案，模型会考虑前 top_p 概率的 token 的结果。所以 0.1 就意味着只有包括在最高 10% 概率中的 token 会被考虑。 我们通常建议修改这个值或者更改 temperature，但不建议同时对两者进行修改。

stream：如果设置为 True，将会以 SSE（server-sent events）的形式以流式发送消息增量。消息流以 data: [DONE] 结尾。

tools：模型可能会调用的 tool 的列表。目前，仅支持 function 作为工具。使用此参数来提供以 JSON 作为输入参数的 function 列表。最多支持 128 个 function。

tool_choice：

控制模型调用 tool 的行为。

none 意味着模型不会调用任何 tool，而是生成一条消息。

auto 意味着模型可以选择生成一条消息或调用一个或多个 tool。

required 意味着模型必须调用一个或多个 tool。

通过 {"type": "function", "function": {"name": "my_function"}} 指定特定 tool，会强制模型调用该 tool。

当没有 tool 时，默认值为 none。如果有 tool 存在，默认值为 auto

logprobs：是否返回所输出 token 的对数概率。如果为 true，则在 message 的 content 中返回每个输出 token 的对数概率

top_logprobs：一个介于 0 到 20 之间的整数 N，指定每个输出位置返回输出概率 top N 的 token，且返回这些 token 的对数概率。指定此参数时，logprobs 必须为 true。

responses部分：

主要关注choices和usage字段里面的内容

choices中

finish_reason：

模型停止生成 token 的原因。

stop：模型自然停止生成，或遇到 stop 序列中列出的字符串。

length ：输出长度达到了模型上下文长度限制，或达到了 max_tokens 的限制。

content_filter：输出内容因触发过滤策略而被过滤。

insufficient_system_resource：系统推理资源不足，生成被打断。

message中

模型生成的completion消息

content 是该completion的内容

reasoning_content 仅适用于 deepseek-reasoner 模型。内容为 assistant 消息中在最终答案之前的推理内容。

tool_calls，模型生成的tool调用，例如function调用

logprobs中

该choice的对数概率信息

usage中

completion_tokens： 模型completion产生的token数
prompt_tokens：用户prompt所包含的token数，该值等于prompt_cache_hit_tokens + prompt_cache_miss_tokens
prompt_cache_hit_tokens：用户prompt中，命中上下文缓存的token数
prompt_cache_miss_tokens：用户prompt中，未命中上下文缓存的token数
total_tokens：该请求中，所有token的数量（prompt + completion）
completion_tokens_details：里面有个reasoning_tokens表示推理模型所产生的思维链token数量

```

如何拼接多轮对话（去掉思维链的输出）

```html

非流式

from openai import OpenAI
client = OpenAI(api_key="<DeepSeek API Key>", base_url="https://api.deepseek.com")

# Turn 1
messages = [{"role": "user", "content": "9.11 and 9.8, which is greater?"}]
response = client.chat.completions.create(
    model="deepseek-reasoner",
    messages=messages
)

reasoning_content = response.choices[0].message.reasoning_content
content = response.choices[0].message.content

# Turn 2
messages.append({'role': 'assistant', 'content': content})
messages.append({'role': 'user', 'content': "How many Rs are there in the word 'strawberry'?"})
response = client.chat.completions.create(
    model="deepseek-reasoner",
    messages=messages
)
# ...


流式

from openai import OpenAI
client = OpenAI(api_key="<DeepSeek API Key>", base_url="https://api.deepseek.com")

# Turn 1
messages = [{"role": "user", "content": "9.11 and 9.8, which is greater?"}]
response = client.chat.completions.create(
    model="deepseek-reasoner",
    messages=messages,
    stream=True
)

reasoning_content = ""
content = ""

for chunk in response:
    if chunk.choices[0].delta.reasoning_content:
        reasoning_content += chunk.choices[0].delta.reasoning_content
    else:
        content += chunk.choices[0].delta.content

# Turn 2
messages.append({"role": "assistant", "content": content})
messages.append({'role': 'user', 'content': "How many Rs are there in the word 'strawberry'?"})
response = client.chat.completions.create(
    model="deepseek-reasoner",
    messages=messages,
    stream=True
)
# ...

```

如何使用工具调用（思考模式和非思考模式下）

```html

# 当然也要在下一轮对话的时候去掉思维链，完成多次工具调用后，完成一轮对话，再进行下一轮对话，去掉上一轮的所有思维链，上面是直接进行了多轮对话的

import os
import json
from openai import OpenAI

# The definition of the tools
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_date",
            "description": "Get the current date",
            "parameters": { "type": "object", "properties": {} },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get weather of a location, the user should supply the location and date.",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": { "type": "string", "description": "The city name" },
                    "date": { "type": "string", "description": "The date in format YYYY-mm-dd" },
                },
                "required": ["location", "date"]
            },
        }
    },
]

# The mocked version of the tool calls
def get_date_mock():
    return "2025-12-01"

def get_weather_mock(location, date):
    return "Cloudy 7~13°C"

TOOL_CALL_MAP = {
    "get_date": get_date_mock,
    "get_weather": get_weather_mock
}

def clear_reasoning_content(messages):
    for message in messages:
        if hasattr(message, 'reasoning_content'):
            message.reasoning_content = None

def run_turn(turn, messages):
    sub_turn = 1
    while True:
        response = client.chat.completions.create(
            model='deepseek-chat',
            messages=messages,
            tools=tools,
            extra_body={ "thinking": { "type": "enabled" } }
        )
        messages.append(response.choices[0].message)
        reasoning_content = response.choices[0].message.reasoning_content
        content = response.choices[0].message.content
        tool_calls = response.choices[0].message.tool_calls
        print(f"Turn {turn}.{sub_turn}\n{reasoning_content=}\n{content=}\n{tool_calls=}")
        # If there is no tool calls, then the model should get a final answer and we need to stop the loop
        if tool_calls is None:
            break
        for tool in tool_calls:
            tool_function = TOOL_CALL_MAP[tool.function.name]
            tool_result = tool_function(**json.loads(tool.function.arguments))
            print(f"tool result for {tool.function.name}: {tool_result}\n")
            messages.append({
                "role": "tool",
                "tool_call_id": tool.id,
                "content": tool_result,
            })
        sub_turn += 1

client = OpenAI(
    api_key=os.environ.get('DEEPSEEK_API_KEY'),
    base_url=os.environ.get('DEEPSEEK_BASE_URL'),
)

# The user starts a question
turn = 1
messages = [{
    "role": "user",
    "content": "How's the weather in Hangzhou Tomorrow"
}]
run_turn(turn, messages)

# The user starts a new question
turn = 2
messages.append({
    "role": "user",
    "content": "How's the weather in Hangzhou Tomorrow"
})
# We recommended to clear the reasoning_content in history messages so as to save network bandwidth
clear_reasoning_content(messages)
run_turn(turn, messages)


# 在 Turn 1 的每个子请求中，都携带了该 Turn 下产生的 reasoning_content 给 API，从而让模型继续之前的思考。response.choices[0].message 携带了 assistant 消息的所有必要字段，包括 content、reasoning_content、tool_calls。简单起见，可以直接用如下代码将消息 append 到 messages 结尾：

# messages.append(response.choices[0].message)

# 在 Turn 2 开始时，我们建议丢弃掉之前 Turn 中的 reasoning_content 来节省网络带宽：

# clear_reasoning_content(messages)

# 该代码的样例输出如下：

Turn 1.1
reasoning_content="The user is asking about the weather in Hangzhou tomorrow. I need to get the current date first, then calculate tomorrow's date, and then call the weather API. Let me start by getting the current date."
content=''
tool_calls=[ChatCompletionMessageToolCall(id='call_00_Tcek83ZQ4fFb1RfPQnsPEE5w', function=Function(arguments='{}', name='get_date'), type='function', index=0)]
tool_result(get_date): 2025-12-01

Turn 1.2
reasoning_content='Today is December 1, 2025. Tomorrow is December 2, 2025. I need to format the date as YYYY-mm-dd: "2025-12-02". Now I can call get_weather with location Hangzhou and date 2025-12-02.'
content=''
tool_calls=[ChatCompletionMessageToolCall(id='call_00_V0Uwt4i63m5QnWRS1q1AO1tP', function=Function(arguments='{"location": "Hangzhou", "date": "2025-12-02"}', name='get_weather'), type='function', index=0)]
tool_result(get_weather): Cloudy 7~13°C

Turn 1.3
reasoning_content="I have the weather information: Cloudy with temperatures between 7 and 13°C. I should respond in a friendly, helpful manner. I'll mention that it's for tomorrow (December 2, 2025) and give the details. I can also ask if they need any other information. Let's craft the response."
content="Tomorrow (Tuesday, December 2, 2025) in Hangzhou will be **cloudy** with temperatures ranging from **7°C to 13°C**.  \n\nIt might be a good idea to bring a light jacket if you're heading out. Is there anything else you'd like to know about the weather?"
tool_calls=None

Turn 2.1
reasoning_content="The user wants clothing advice for tomorrow based on the weather in Hangzhou. I know tomorrow's weather: cloudy, 7-13°C. That's cool but not freezing. I should suggest layered clothing, maybe a jacket, long pants, etc. I can also mention that since it's cloudy, an umbrella might not be needed unless there's rain chance, but the forecast didn't mention rain. I should be helpful and give specific suggestions. I can also ask if they have any specific activities planned to tailor the advice. Let me respond."
content="Based on tomorrow's forecast of **cloudy weather with temperatures between 7°C and 13°C** in Hangzhou, here are some clothing suggestions:\n\n**Recommended outfit:**\n- **Upper body:** A long-sleeve shirt or sweater, plus a light to medium jacket (like a fleece, windbreaker, or light coat)\n- **Lower body:** Long pants or jeans\n- **Footwear:** Closed-toe shoes or sneakers\n- **Optional:** A scarf or light hat for extra warmth, especially in the morning and evening\n\n**Why this works:**\n- The temperature range is cool but not freezing, so layering is key\n- Since it's cloudy but no rain mentioned, you likely won't need an umbrella\n- The jacket will help with the morning chill (7°C) and can be removed if you warm up during the day\n\n**If you have specific plans:**\n- For outdoor activities: Consider adding an extra layer\n- For indoor/office settings: The layered approach allows you to adjust comfortably\n\nWould you like more specific advice based on your planned activities?"
tool_calls=None


# 如果代码中有reasoning_content字段的问题，可以

```


永远记住/chat/completion是无状态的，如果需要历史对话，需要在messages中补充，就是把回答给拼接上去

关于temperature参数的设置，ds默认是1.0

```html

代码生成/数学题解  0.0
数据抽取/分析     1.0
通用对话          1.3
翻译             1.3
创意类写作/诗歌创作  1.5


```

DeepSeek API 不限制用户并发量

在这段时间里，您的 HTTP 请求会保持连接，并持续收到如下格式的返回内容：

非流式请求：持续返回空行
流式请求：持续返回 SSE keep-alive 注释（: keep-alive）

这些内容不影响 OpenAI SDK 对响应的 JSON body 的解析。如果您在自己解析 HTTP 响应，请注意处理这些空行或注释。

#### 模型能力

还是要关注一下模型的能力的

```html

ollama show deepseek-r1:14b

Capabilities
    completion
    thinking

# 这里是没有tools的能力的，而

ollama show qwen3:14b

Capabilities
    completion
    tools
    thinking

# 这才是有thinking的能力的    

```

#### 模型缓存

deepseek的上下文硬盘缓存技术对所有用户默认开启，无需修改代码就能享用

用户的每一个请求都会触发硬盘缓存的构建，若后续请求与之前的请求在前缀上存在重复，则重复部分只需要从缓存中拉取，计入"缓存命中"

```html

# 例子一：长文本回答

第一次请求

messages: [
    {"role": "system", "content": "你是一位资深的财报分析师..."}
    {"role": "user", "content": "<财报内容>\n\n请总结一下这份财报的关键信息。"}
]

第二次请求

messages: [
    {"role": "system", "content": "你是一位资深的财报分析师..."}
    {"role": "user", "content": "<财报内容>\n\n请分析一下这份财报的盈利情况。"}
]

在上例中，两次请求都有相同的前缀，即 system 消息 + user 消息中的 <财报内容>。在第二次请求时，这部分前缀会计入“缓存命中”

# 例子二：多轮对话

第一次请求

messages: [
    {"role": "system", "content": "你是一位乐于助人的助手"},
    {"role": "user", "content": "中国的首都是哪里？"}
]

第二次请求

messages: [
    {"role": "system", "content": "你是一位乐于助人的助手"},
    {"role": "user", "content": "中国的首都是哪里？"},
    {"role": "assistant", "content": "中国的首都是北京。"},
    {"role": "user", "content": "美国的首都是哪里？"}
]

在上例中，第二次请求可以复用第一次请求开头的 system 消息和 user 消息，这部分会计入“缓存命中”。

# 例子三：使用few-shot学习

在实际应用中，用户可以通过 Few-shot 学习的方式，来提升模型的输出效果。所谓 Few-shot 学习，是指在请求中提供一些示例，让模型学习到特定的模式。由于 Few-shot 一般提供相同的上下文前缀，在硬盘缓存的加持下，Few-shot 的费用显著降低。

messages: [    
        {"role": "system", "content": "你是一位历史学专家，用户将提供一系列问题，你的回答应当简明扼要，并以`Answer:`开头"},
        {"role": "user", "content": "请问秦始皇统一六国是在哪一年？"},
        {"role": "assistant", "content": "Answer:公元前221年"},
        {"role": "user", "content": "请问汉朝的建立者是谁？"},
        {"role": "assistant", "content": "Answer:刘邦"},
        {"role": "user", "content": "请问唐朝最后一任皇帝是谁"},
        {"role": "assistant", "content": "Answer:李柷"},
        {"role": "user", "content": "请问明朝的开国皇帝是谁？"},
        {"role": "assistant", "content": "Answer:朱元璋"},
        {"role": "user", "content": "请问清朝的开国皇帝是谁？"}
]

messages: [    
        {"role": "system", "content": "你是一位历史学专家，用户将提供一系列问题，你的回答应当简明扼要，并以`Answer:`开头"},
        {"role": "user", "content": "请问秦始皇统一六国是在哪一年？"},
        {"role": "assistant", "content": "Answer:公元前221年"},
        {"role": "user", "content": "请问汉朝的建立者是谁？"},
        {"role": "assistant", "content": "Answer:刘邦"},
        {"role": "user", "content": "请问唐朝最后一任皇帝是谁"},
        {"role": "assistant", "content": "Answer:李柷"},
        {"role": "user", "content": "请问明朝的开国皇帝是谁？"},
        {"role": "assistant", "content": "Answer:朱元璋"},
        {"role": "user", "content": "请问商朝是什么时候灭亡的"},        
]

在上例中，使用了 4-shots。两次请求只有最后一个问题不一样，第二次请求可以复用第一次请求中前 4 轮对话的内容，这部分会计入“缓存命中”。

# prompt_cache_hit_tokens：本次请求的输入中，缓存命中的 tokens 数（0.1 元 / 百万 tokens）
# prompt_cache_miss_tokens：本次请求的输入中，缓存未命中的 tokens 数（1 元 / 百万 tokens）

# 硬盘缓存只匹配到用户输入的前缀部分，输出仍然是通过计算推理得到的，仍然受到 temperature 等参数的影响，从而引入随机性。其输出效果与不使用硬盘缓存相同。

# 缓存系统以 64 tokens 为一个存储单元，不足 64 tokens 的内容不会被缓存
# 缓存系统是“尽力而为”，不保证 100% 缓存命中
# 缓存构建耗时为秒级。缓存不再使用后会自动被清空，时间一般为几个小时到几天

```

#### 模型界面

可以使用开源的 open webUI来接入ollama等模型

```html

# 直接启动使用
docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always -e WEBUI_HOST=0.0.0.0 ghcr.io/open-webui/open-webui:main

#

```

有的时候模型gpu打满的情况下，ollama无法停止model运行，其实和open webui也有关系，需要先释放ui

ollama 也是，结束会话并不会stop模型，需要再stop一下，释放内存

另外，毕竟是docker启动的open webui，本身容器需要分配cpu/内存，那么占空间，可以直接宿主机本地部署试试

```html

  # 找一个干净点的 python 10+的环境，然后安装下socks支持(pysocks)，command中也要能proxy，然后安装启动

pip install open-webui

open-webui serve

# 本地端口是8080

# open-webui的联网功能也要设置的，比如使用tavily搜索这样，模型本身是没有搜索功能的

```

#### 模型编辑

#### 模型merge


#### 模型使用经验与调优

在openclaw的服务端测试下回调的链路，链路是通过cloudflare的隧道域名调回来的，返回中有

```html

{"model":"qwen3:14b","created_at":"2025-12-20T03:28:41.949859Z","message":{"role":"assistant","content":"Hello! How can I assist you today? 😊","thinking":"Okay, the user said \"hi\". I need to respond politely. Let me check the guidelines. I should be friendly and offer help. Maybe say hello back and ask how I can assist them. Keep it simple and welcoming. Alright, that should work.\n"},"done":true,"done_reason":"stop","total_duration":3849286459,"load_duration":970600375,"prompt_eval_count":11,"prompt_eval_duration":263356792,"eval_count":68,"eval_duration":2558519334}

eal	0m4.457s
user	0m0.021s
sys	0m0.007s

```

这里面总耗时4s多，load_duration是0.97s，需要接近1s把模型从内存/显存中唤醒，eval_duration是2.55s。模型生成68个token用了2.5s。网络开销等于剩余的时间，约600ms（0.6s），其实已经蛮快的了，但是thinking字段太长了，有60+token，非流式的情况下，openclaw需要等4.5秒跑完才能显示第一个字。就是首字时间被大大的推迟了。

优化点的话

```html

1、模型开启流式传输，streaming

非流式

time curl -X POST https://mingye666.xxxxx.xxx/api/chat \
-H "Content-Type: application/json" \
-d '{"model": "qwen3:14b", "messages": [{"role": "user", "content": "hi"}], "stream": false}'

流式

time curl -X POST https://mingye666.xxxxx.xxx/api/chat \
-H "Content-Type: application/json" \
-d '{
  "model": "qwen3:14b",
  "messages": [{"role": "user", "content": "写一篇关于猫的短文，100字左右。"}],
  "stream": true
}'

然后就是刷屏了，如果第一行数据在 1 秒内出来，说明隧道和网络链路极快。

推理耗时 14.966秒，386token，386 / time = 25.8 tokens/s，对于14B的模型，正常30-40t/s。

对于30B的模型，同等的硬件条件下，速度会下降到15-20t/s

而如果是MOE架构的，虽然总参数量是30B，但每次推理激活约3B参数，生成速度和4B-7B模型相当

2、彻底屏蔽思考过程

3、让模型保持在线状态，去除load的环节，常驻内存

ollama run qwen3:14b --keepalive -1

```

#### 模型的context

除了prompt engineering，现在还有context engineering，上下文随着多次agent的调用是会越来越长的，如果按照128K的上下文长度，差不多超过10w左右工作就无法继续了，所以会有各种减少context的方法

1、压缩

直接将之前的context压缩成一个summary 或者 将之前工具调用的输出替换成一句话，比如“这里曾经有一段工具输出” 或者 “这边的工具输出保存到了log.txt中”

上面两个方法可以结合起来

2、使用subagent

用agent生成新的subagent后，取回每个agent执行任务后的return结果就行，sub-agent执行的上下文丢弃掉

sub-agent需要训练模型取得，像工具调用能力一样（tool use）

sub-agent的能力可以用RL的方式去训练，惩罚主干的context太长，惩罚sub-agent做超过它职责范围的事情

3、过滤

context中可能一小部分是action，6%左右，10%左右是reasoning，80%+是observation（就是打开了一个文档，打开了一个web查找的内容）

所以observation是不是可以精简，删掉一些不要的东西，训练一个小的语言模型干这个事，比较智能的读取需要的内容

按需加载（工具 或者 skills，概念是类似的）

#### 可以say的（temp）
