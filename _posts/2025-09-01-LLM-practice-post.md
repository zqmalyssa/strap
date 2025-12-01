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

上面那种自己加Chat Template的方法不一定可以看懂，尾部还有接龙，可以使用tokenizer.apply_chat_template加上官方的chat_template。

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

```

#### 模型编辑

#### 模型merge
