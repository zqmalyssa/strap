---
layout: post
title: prompt engineering
tags: [prompt engineering]
author-id: zqmalyssa
---

prompt engineering相关内容

#### 基本概念

提示工程是一个较新的学科，应用于开发和优化提示词（Prompt），帮助用户有效地将语言模型用于各种应用场景和研究领域。掌握了提示工程相关技能将有助于用户更好地了解大型语言模型的能力和局限性。研究人员可利用提示工程来提高大语言模型处理复杂任务场景的能力，如问答和算术推理能力。开发人员可通过提示工程设计和研发出强大的技术，实现和大语言模型或其他生态工具的高效接轨。



#### 模型设置

```html

Temperature：简单来说，temperature 的参数值越小，模型就会返回越确定的一个结果。如果调高该参数值，大语言模型可能会返回更随机的结果，也就是说这可能会带来更多样化或更具创造性的产出。（调小temperature）实质上，你是在增加其他可能的 token 的权重。在实际应用方面，对于质量保障（QA）等任务，我们可以设置更低的 temperature 值，以促使模型基于事实返回更真实和简洁的结果。 对于诗歌生成或其他创造性任务，适度地调高 temperature 参数值可能会更好。

Top_p：同样，使用 top_p（与 temperature 一起称为核采样（nucleus sampling）的技术），可以用来控制模型返回结果的确定性。如果你需要准确和事实的答案，就把参数值调低。如果你在寻找更多样化的响应，可以将其值调高点。

使用Top P意味着只有词元集合（tokens）中包含top_p概率质量的才会被考虑用于响应，因此较低的top_p值会选择最有信心的响应。这意味着较高的top_p值将使模型考虑更多可能的词语，包括不太可能的词语，从而导致更多样化的输出。

一般建议是改变 Temperature 和 Top P 其中一个参数就行，不用两个都调整。

Max Length：您可以通过调整 max length 来控制大模型生成的 token 数。指定 Max Length 有助于防止大模型生成冗长或不相关的响应并控制成本。

Stop Sequences：stop sequence 是一个字符串，可以阻止模型生成 token，指定 stop sequences 是控制大模型响应长度和结构的另一种方法。例如，您可以通过添加 “11” 作为 stop sequence 来告诉模型生成不超过 10 个项的列表。

Frequency Penalty：frequency penalty 是对下一个生成的 token 进行惩罚，这个惩罚和 token 在响应和提示中已出现的次数成比例， frequency penalty 越高，某个词再次出现的可能性就越小，这个设置通过给 重复数量多的 Token 设置更高的惩罚来减少响应中单词的重复。

Presence Penalty：presence penalty 也是对重复的 token 施加惩罚，但与 frequency penalty 不同的是，惩罚对于所有重复 token 都是相同的。出现两次的 token 和出现 10 次的 token 会受到相同的惩罚。 此设置可防止模型在响应中过于频繁地生成重复的词。 如果您希望模型生成多样化或创造性的文本，您可以设置更高的 presence penalty，如果您希望模型生成更专注的内容，您可以设置更低的 presence penalty。

与 temperature 和 top_p 一样，一般建议是改变 frequency penalty 和 presence penalty 其中一个参数就行，不要同时调整两个。


```

#### 提示词示例

```html

# 文本概括

Explain antibiotics
A:

# 信息提取

Author-contribution statements and acknowledgements in research papers should state clearly and specifically whether, and to what extent, the authors used AI technologies such as ChatGPT in the preparation of their manuscript and analysis. They should also indicate which LLMs were used. This will alert editors and reviewers to scrutinize manuscripts more carefully for potential biases, inaccuracies and improper source crediting. Likewise, scientific journals should be transparent about their use of LLMs, for example when selecting submitted manuscripts.
Mention the large language model based product mentioned in the paragraph above:

# 问答

Answer the question based on the context below. Keep the answer short and concise. Respond "Unsure about answer" if not sure about the answer.
Context: Teplizumab traces its roots to a New Jersey drug company called Ortho Pharmaceutical. There, scientists generated an early version of the antibody, dubbed OKT3. Originally sourced from mice, the molecule was able to bind to the surface of T cells and limit their cell-killing potential. In 1986, it was approved to help prevent organ rejection after kidney transplants, making it the first therapeutic antibody allowed for human use.
Question: What was OKT3 originally sourced from?
Answer:

# 文本分类

Classify the text into neutral, negative or positive.
Text: I think the food was okay.
Sentiment:

想要返回neutral 而不是Neutral，可以修改成

Classify the text into neutral, negative or positive.
Text: I think the vacation is okay.
Sentiment: neutral
Text: I think the food was okay.
Sentiment:

# 对话
The following is a conversation with an AI research assistant. The assistant tone is technical and scientific.

Human: Hello, who are you?
AI: Greeting! I am an AI research assistant. How can I help you today?
Human: Can you tell me about the creation of blackholes?
AI:

如果觉得回答过于技术，可以修改成

The following is a conversation with an AI research assistant. The assistant answers should be easy to understand even by primary school students.

Human: Hello, who are you?
AI: Greeting! I am an AI research assistant. How can I help you today?
Human: Can you tell me about the creation of black holes?
AI:

# 代码生成
/*
Ask the user for their name and say "Hello"
*/

更好一点的提示词

"""
Table departments, columns = [DepartmentId, DepartmentName]
Table students, columns = [DepartmentId, StudentId, StudentName]
Create a MySQL query for all students in the Computer Science Department
"""

# 推理
What is 9,000 * 9,000?

The odd numbers in this group add up to an even number: 15, 32, 5, 13, 82, 7, 1.

Solve by breaking the problem into steps. First, identify the odd numbers, add them, and indicate whether the result is odd or even.

```

#### 提示技术

```html

# 零样本提示

给一些回答的约束

# 少样本提示

one-shot 或者 few-shot

# CoT提示 // 列出思维的过程（加起来，减掉是多少） 或者 think step by step

shot上得列出推理的过程，或者zero-shot的 step by step


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

// ds 8B蒸馏版本
根据上述分析，统计数组中的奇数数量，如果数量为偶数，则总和为偶数；否则为奇数。
判断奇数的数量是否为偶数。  
输入：15、32、5、13、82、7、1 → 奇数有5个，为奇数。

# 自我一致性

Q：林中有15棵树。林业工人今天将在林中种树。完成后，将有21棵树。林业工人今天种了多少棵树？
A：我们从15棵树开始。后来我们有21棵树。差异必须是他们种树的数量。因此，他们必须种了21-15 = 6棵树。答案是6。

Q：停车场有3辆汽车，又来了2辆汽车，停车场有多少辆汽车？
A：停车场已经有3辆汽车。又来了2辆。现在有3 + 2 = 5辆汽车。答案是5。

Q：Leah有32块巧克力，她的姐姐有42块。如果他们吃了35块，他们总共还剩多少块？
A：Leah有32块巧克力，Leah的姐姐有42块。这意味着最初有32 + 42 = 74块巧克力。已经吃了35块。因此，他们总共还剩74-35 = 39块巧克力。答案是39。

Q：Jason有20个棒棒糖。他给Denny一些棒棒糖。现在Jason只有12个棒棒糖。Jason给Denny多少棒棒糖？
A：Jason有20个棒棒糖。因为他现在只有12个，所以他必须把剩下的给Denny。他给Denny的棒棒糖数量必须是20-12 = 8个棒棒糖。答案是8。

Q：Shawn有五个玩具。圣诞节，他从他的父母那里得到了两个玩具。他现在有多少个玩具？
A：他有5个玩具。他从妈妈那里得到了2个，所以在那之后他有5 + 2 = 7个玩具。然后他从爸爸那里得到了2个，所以总共他有7 + 2 = 9个玩具。答案是9。

Q：服务器房间里有9台计算机。从周一到周四，每天都会安装5台计算机。现在服务器房间里有多少台计算机？
A：从周一到周四有4天。每天都添加了5台计算机。这意味着总共添加了4 * 5 =
20台计算机。一开始有9台计算机，所以现在有9 + 20 = 29台计算机。答案是29。

Q：Michael有58个高尔夫球。星期二，他丢失了23个高尔夫球。星期三，他又丢失了2个。星期三结束时他还剩多少个高尔夫球？
A：Michael最初有58个球。星期二他丢失了23个，所以在那之后他有58-23 = 35个球。星期三他又丢失了2个，所以现在他有35-2 = 33个球。答案是33。

Q：Olivia有23美元。她用每个3美元的价格买了五个百吉饼。她还剩多少钱？
A：她用每个3美元的价格买了5个百吉饼。这意味着她花了15美元。她还剩8美元。

Q：当我6岁时，我的妹妹是我的一半年龄。现在我70岁了，我的妹妹多大？
A：

这样70岁的时候就不是35岁了

# 生成知识提示

输入：希腊比墨西哥大。
知识：希腊的面积约为131,957平方公里，而墨西哥的面积约为1,964,375平方公里，使墨西哥比希腊大了1,389%。
输入：眼镜总是会起雾。
知识：当你的汗水、呼吸和周围的湿度中的水蒸气落在冷的表面上，冷却并变成微小的液滴时，会在眼镜镜片上产生冷凝。你看到的是一层薄膜。你的镜片相对于你的呼吸会比较凉，尤其是当外面的空气很冷时。
输入：鱼有思考能力。
知识：鱼比它们看起来更聪明。在许多领域，如记忆力，它们的认知能力与或超过非人类灵长类动物等“更高级”的脊椎动物。鱼的长期记忆帮助它们跟踪复杂的社交关系。
输入：一个人一生中吸烟很多香烟的常见影响是患肺癌的几率高于正常水平。
知识：那些一生中平均每天吸烟不到一支香烟的人，患肺癌的风险是从不吸烟者的9倍。在每天吸烟1到10支香烟之间的人群中，死于肺癌的风险几乎是从不吸烟者的12倍。
输入：一块石头和一颗卵石大小相同。
知识：卵石是一种根据Udden-Wentworth沉积学尺度的颗粒大小为4到64毫米的岩屑。卵石通常被认为比颗粒（直径2到4毫米）大，比卵石（直径64到256毫米）小。
输入：高尔夫球的一部分是试图获得比其他人更高的得分。
知识：

问题：高尔夫球的一部分是试图获得比其他人更高的得分。是或否？
知识：高尔夫球的目标是以最少的杆数打完一组洞。一轮高尔夫球比赛通常包括18个洞。每个洞在标准高尔夫球场上一轮只打一次。每个杆计为一分，总杆数用于确定比赛的获胜者。
解释和答案：

# prompt chaining（提示链）

一个是 文档问答中的链式提示

想要更好阅读大文本文档，可以设计两个不同的提示，第一个提示负责提取相关引文以回答问题，第二个提示则以引文和原始文档为输入来回答给定的问题。换句话说，可以创建两个不同的提示来执行根据文档回答问题的任务

你是一个很有帮助的助手。你的任务是根据文档回答问题。第一步是从文档中提取与问题相关的引文，由####分隔。请使用<quotes></quotes>输出引文列表。如果没有找到相关引文，请回应“未找到相关引文！”。
####
{{文档}}
####

这一段在system的prompt中，user：what are the prompting techniques mentioned in the document?

<quotes>
- Chain-of-thought (CoT) prompting[27]
- Generated knowledge prompting[37]
- Least-to-most prompting[38]
- Self-consistency decoding[39]
- Complexity-based prompting[41]
- Self-refine[42]
- Tree-of-thought prompting[43]
- Maieutic prompting[45]
- Directional-stimulus prompting[46]
- Textual inversion and embeddings[59]
- Using gradient descent to search for prompts[61][62][63][64]
- Prompt injection[65][66][67]
</quotes>

在第一个提示中返回的引文现在可以用作下面第二个提示的输入。您可以对这些引文进行清理，比如移除引用标志。可以在提示链中新建另一个提示来移除或使用这些引用标志，但现在您可以忽略这些。然后，第二个提示接收由第一个提示提取的相关引文，并根据文档和这些提取的引文生成一个有帮助的回答。第二个提示可以是以下内容：

根据从文档中提取的相关引文（由<quotes></quotes>分隔）和原始文档（由####分隔），请构建对问题的回答。请确保答案准确、语气友好且有帮助。
####
{{文档}}
####
<quotes>
- Chain-of-thought (CoT) prompting[27]
- Generated knowledge prompting[37]
- Least-to-most prompting[38]
- Self-consistency decoding[39]
- Complexity-based prompting[41]
- Self-refine[42]
- Tree-of-thought prompting[43]
- Maieutic prompting[45]
- Directional-stimulus prompting[46]
- Textual inversion and embeddings[59]
- Using gradient descent to search for prompts[61][62][63][64]
- Prompt injection[65][66][67]
</quotes>


# 思维树（TOT）

# 检索增强生成（RAG）

# ReAct框架

# Reflexion


```

#### 提示应用

```html

# 生成数据

生成10个情感分析的范例。示例被归类为正面或负面。生成2个负面示例和8个正面示例。使用以下格式的示例：
Q：<句子>
A：<情感>

# 将注释转换成代码

# 提示函数

# function call

GPT-4和GPT-3.5是经过微调的LLM，能够检测函数是否被调用，随后输出包含调用函数参数的JSON。通过这一过程被调用的函数能够作为工具添加到您的AI应用中，并且您可以在单个请求中定义多个函数。


```

#### prompt engineering

提示词的一些优化，

1、提示词以固定的分隔符结尾，补全以空格开始

2、用多一点例子去做样本

3、输入数据用自然语言可以优化性能，尤其对应生成式模型，而不是structed的数据

4、在fine-tuning的时候也可以创建验证集，validation_data.json。openai api fine_tunes.create -t train_data.jsonl -v validation_data.jsonl -m <engine>，也有超参数的设置，n_epochs和batch_size，学习率。compute_classification_metrics，这个指标适用于分类任务的精调。如果设置为True，每个epoch的结束，它会在验证集上计算分类相关的指标（比如准确度和F1得分）。这个指标可以帮助你评估模型的表现，做出调整。

5、如果是分类问题，可以把类别的string换成数字1-10等等，减少token使用量，训练的数据量变小，推断的token数量也只有1个

6、api会返回给你用量的 usage。包括提示词 和 返回的。"usage": {"completion_tokens": 7,"prompt_tokens": 4,"total_tokens": 11}，输出的长度用max_token限制一下

7、1个token大约4个字符文本，100个token大约75个单词

8、\n\n 后面接目标 \n\nkeyword：

9、生成待办清单，

```html

next = openai.Completion.create(
  model="text-davinci-002",
  prompt="Todo list to create a company in US\n\n1.",
  temperature=0.3,
  max_tokens=64,
  top_p=0.1,
  frequency_penalty=0,
  presence_penalty=0.5,
  stop=["6."],
)


1. Choose a business structure.
2. Register your business with the state.
3. Obtain a federal tax ID number.
4. Open a business bank account.
5. Establish business credit.

```

10、编辑文本，Api变成Edit，参数从prompt变成input和instruction，指令是必须的，输入是可选的

```html

response = openai.Edit.create(
  model="text-davinci-edit-001",
  input="Hallo Welt",
  instruction="Translate to English",
)

print(response)

```

同样的任务可以用补全接口去完成

```html

response = openai.Edit.create(
  model="text-davinci-edit-001",
  instruction="Translate from English to French, Arabic, and Spanish.",
 input="The cat sat on the mat."
)

next = openai.Completion.create(
  model="text-davinci-003",
  prompt="""
  Translate the following sentence from English to French, Arabic, and Spanish.
  English: The cat sat on the mat.
  French:
  Arabic:
  Spanish:
  """,
  max_tokens=60,
  temperature=0
)

下面一个例子则是反过来，用编辑接口来执行补完的任务：

response = openai.Edit.create(
  model="text-davinci-edit-001",
  instruction="Complete the story",
  input="Once upon a time",
)

```

11、把补全和编辑串联起来使用，先产生一个推文，然后再翻译成西班牙语

```html

english_tweet = openai.Completion.create(
  model="text-davinci-002",
  prompt=prompt,
  temperature=0.5,
  max_tokens=20,
)

english_tweet_text = english_tweet["choices"][0]["text"].strip()
print("English Tweet:")
print(english_tweet_text)

spanish_tweet = openai.Edit.create(
  model="text-davinci-edit-001",
  input=english_tweet_text,
  instruction="Translate to Spanish",
  temperature=0.5,
)

spanish_tweet_text = spanish_tweet["choices"][0]["text"].strip()
print("Spanish Tweet:")
print(spanish_tweet_text)

```

12、不同的单词会有不同的意思，告诉模型上下文，然后再解释

```html

The light is red.
This desk is very light.
You light up my life.

prompt_a = "The light is red. Determine the part of speech of the word 'light'.\n\n"
prompt_b = "This desk is very light. Determine the part of speech of the word 'light\
'.\n\n"
prompt_c = "You light up my life. Determine the part of speech of the word 'light'.\\
n\n"

for prompt in [prompt_a, prompt_b, prompt_c]:
  result = openai.Completion.create(
  model="text-davinci-002",
  prompt=prompt,
  max_tokens=20,
  temperature=0,
)

单独问 Determine the part of speech of the word 'light' 会产生不同的变化

让它学习固定的一些语法，然后进行输出，就是指定了输出格式

prompt = """Input: Bitcoin
Output:
BTC was created in 2008, you can learn more about it here: https://bitcoin.org/en/ a\
nd get the latest price here: https://www.coingecko.com/en/coins/bitcoin.
It's all-time high is $64,895.00 and it's all-time low is $67.81.

Input: Ethereum
Output:
ETH was created in 2015, you can learn more about it here: https://ethereum.org/en/ \
and get the latest price here: https://www.coingecko.com/en/coins/ethereum
It's all-time high is $4,379.00 and it's all-time low is $0.43.

Input: Dogecoin
Output:
DOGE was created in 2013, you can learn more about it here: https://dogecoin.com/ an\
d get the latest price here: https://www.coingecko.com/en/coins/dogecoin
It's all-time high is $0.73 and it's all-time low is $0.000002.

Input: Cardano
Output:\n"""

result = openai.Completion.create(
  model="text-davinci-002",
  prompt=prompt,
  max_tokens=200,
  temperature=0,
)

print(result.choices[0]["text"].strip())


```

只希望模型进行单一的输出，就是规范化模型的输出内容

```html

prompt = """
Input: List all the files in the current directory
Output: ls -l

Input: List all the files in the current directory, including hidden files
Output: ls -la

Input: Delete all the files in the current directory
Output: rm *

Input: Count the number of occurrences of the word "sun" in the file "test.txt"
Output: grep -o "sun" test.txt | wc -l

Input:{}
Output:
"""

result = openai.Completion.create(
model="text-davinci-002",
  prompt=prompt.format("Count the number of files in the current directory"),
  max_tokens=200,
  temperature=0,
)

print(result.choices[0]["text"].strip())

输出是单一的，由于我们需要单一的回答，因此这里的temperature参数为空。我们为模型提供了足够的token来处理输出：

ls -l | wc -l

```

而在上面的例子中import click这个包，可以做悬停了，询问你的输入，你type了list all files in /tmp后就会产生ls /tmp，之后会问你是否执行等等，这些程序里面都能控制

13、数学的话，加上let's think step by step

14、比较不错的提示词结构

```html

# Role and Objective

# Instructions

## Sub-categories for more detailed instructions

# Reasoning Steps

# Output Format

# Examples
## Example 1

# Context

# Final instructions and prompt to think step by step

```

15、搞一个提示词优化多agents，逻辑矛盾提示词agent，结构化输出agent，提示词与样例不匹配agent，重写提示词agent，更新不一致的样例agent
