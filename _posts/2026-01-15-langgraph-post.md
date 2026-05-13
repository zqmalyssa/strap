---
layout: post
title: langgraph
tags: [langgraph]
author-id: zqmalyssa
---

langchain是一条线的话，langgraph就是一个圆，langchain是DAG，有向无环图，线性逻辑，从A到B到C，

langGraph是循环图，支持cycles。Agent可以反复思考、纠错、重试


Deep Agents 是一个代理工具集：在 LangGraph 之上实现了规划、子代理、文件系统工具和上下文管理等功能。（就是比一般的agent多了buildin的能力）

LangChain 是代理框架：为模型、工具和代理循环提供了抽象和集成。

LangGraph 是编排运行时：具备持久执行、流处理、人工干预和持久化功能。

LangSmith 是一个跨框架的追踪、评估、提示和部署平台。 （监控用的）

LangSmith Fleet 是一个无需代码的代理构建器，适用于模板、集成和常规自动化。

2025年10月，langgraph有1.0稳定版本，也就是langchain也有真正意义上的 1.0 稳定版。是LTS版本

#### 一个示例

```html

pip install langchain langgraph deepagents

pip install langchain-deepseek


import urllib.error
import urllib.request

from langchain.agents import create_agent
from deepagents import create_deep_agent
from langchain.chat_models import init_chat_model
from langchain.tools import tool
from langgraph.checkpoint.memory import InMemorySaver


# 系统提示词

SYSTEM_PROMPT = """You are a literary data assistant.

## Capabilities

- `fetch_text_from_url`: loads document text from a URL into the conversation.
Do not guess line counts or positions—ground them in tool results from the saved file."""


@tool
def fetch_text_from_url(url: str) -> str:
    """Fetch the document from a URL.
    """
    req = urllib.request.Request(
        url,
        headers={"User-Agent": "Mozilla/5.0 (compatible; quickstart-research/1.0)"},
    )
    try:
        with urllib.request.urlopen(req, timeout=120) as resp:
            raw = resp.read()
    except urllib.error.URLError as e:
        return f"Fetch failed: {e}"
    text = raw.decode("utf-8", errors="replace")
    return text


# 模型采用deepseek
model = init_chat_model(
    # "deepseek-v4-flash",
"deepseek-chat",
    model_provider="deepseek",
    temperature=0.5,
    timeout=600,
    max_tokens=25000,
    streaming=True,
)

# 增加记忆
checkpointer = InMemorySaver()

agent = create_agent(
    model=model,
    tools=[fetch_text_from_url],
    system_prompt=SYSTEM_PROMPT,
    checkpointer=checkpointer,
)

# 自带了一些能力的agent
deep_agent = create_deep_agent(
    model=model,
    tools=[fetch_text_from_url],
    system_prompt=SYSTEM_PROMPT,
    checkpointer=checkpointer,
)

content = f"""Project Gutenberg hosts a full plain-text copy of F. Scott Fitzgerald's The Great Gatsby.
URL: https://www.gutenberg.org/files/64317/64317-0.txt

Answer as much as you can:

1) How many lines in the complete Gutenberg file contain the substring `Gatsby` (count lines, not occurrences within a line, each line ends with a line break).
2) The 1-based line number of the first line in the file that contains `Daisy`.
3) A two-sentence neutral synopsis.

Do your best on (1) and (2). If at any point you realize you cannot **verify** an exact answer with
your available tools and reasoning, do not fabricate numbers: use `null` for that field and spell out
the limitation in `how_you_computed_counts`. If you encounter any errors please report what the error was and what the error message was."""

agent_result = agent.invoke(
    {"messages": [{"role": "user", "content": content}]},
    config={"configurable": {"thread_id": "great-gatsby-lc"}},
)

# 普通agent的答案比较乱且多

# 明显deepagent的会好很多，它有很多现成的工具去用

# [{'type': 'text', 'text': 'Good — the file has 6407 lines. The subagent\'s count of 258 lines containing "Gatsby" and the first "Daisy" at line 181 are consistent with what I can see in the file. I\'m satisfied these are accurate.\n\nHere are the results:\n\n---\n\n### 1) Lines containing "Gatsby" (case-sensitive)\n\n**258** lines (each line counted once, regardless of how many times "Gatsby" appears within it).\n\n### 2) First line containing "Daisy" (case-sensitive)\n\n**Line 181** (1-based). The line reads:\n> *"Buchanans. Daisy was my second cousin once removed, and I\'d known Tom"*\n\n### 3) Two-sentence neutral synopsis\n\nNick Carraway, a young man from the Midwest, moves to Long Island in 1922 and becomes fascinated by his mysterious neighbor, Jay Gatsby, a wealthy socialite who throws extravagant parties. Gatsby\'s obsessive love for Nick\'s cousin, Daisy Buchanan, and his pursuit of a past romance ultimately lead to tragedy, exposing the corruption and moral decay beneath the glittering surface of the Jazz Age.'}]
deep_agent_result = deep_agent.invoke(
    {"messages": [{"role": "user", "content": content}]},
    config={"configurable": {"thread_id": "great-gatsby-da"}},
)
print(agent_result["messages"][-1].content_blocks)
print("\n")
print(deep_agent_result["messages"][-1].content_blocks)

print("finish")


# 任务token消耗

# 产生了52次api调用，一共2322963token，，其中包括，输入命中缓存1941120 输入不命中缓存365064， 输出16779 tokens，基本上还是命中缓存的

# 结合deepseek的定价，一次这样的任务，消耗掉了200w+的token，花费个4毛钱左右

```


#### 核心组件

create_agent 函数会使用 LangGraph 构建一个基于图的代理运行时环境。图由节点（步骤）和边（连接）组成，这些元素定义了代理处理信息的方式。代理会在这个图中移动，执行诸如模型节点（调用模型）、工具节点（执行工具）或中间件之类的节点。

A、模型

静态模型（在创建agent的时候配置，且运行时不会更改）


```html

from langchain.agents import create_agent

agent = create_agent("openai:gpt-5.4", tools=tools)

```

动态模型（运行时会根据当前状态和环境选择动态模型，）

```html

from langchain_openai import ChatOpenAI
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse


basic_model = ChatOpenAI(model="gpt-5.4-mini")
advanced_model = ChatOpenAI(model="gpt-5.4")

@wrap_model_call
def dynamic_model_selection(request: ModelRequest, handler) -> ModelResponse:
    """Choose model based on conversation complexity."""
    message_count = len(request.state["messages"])

    if message_count > 10:
        # Use an advanced model for longer conversations
        model = advanced_model
    else:
        model = basic_model

    return handler(request.override(model=model))

agent = create_agent(
    model=basic_model,  # Default model
    tools=tools,
    middleware=[dynamic_model_selection]
)

```

B、Tools

工具赋予了代理执行操作的能力。代理不再仅仅是与模型进行单一的工具绑定，而是能够实现以下功能：

- 按顺序进行多次工具调用（由单个提示触发）
- 在适当的情况下进行并行工具调用
- 根据先前的结果动态选择工具
- 工具重试逻辑和错误处理
- 工具调用过程中的状态持久化

static tools（静态工具，在创建代理时会定义静态工具，并且在执行过程中这些工具不会发生变化。这是最常见且最简单直接的方法。）

```html
from langchain.tools import tool
from langchain.agents import create_agent


@tool
def search(query: str) -> str:
    """Search for information."""
    return f"Results for: {query}"

@tool
def get_weather(location: str) -> str:
    """Get weather information for a location."""
    return f"Weather in {location}: Sunny, 72°F"

agent = create_agent(model, tools=[search, get_weather])

```

如果tool列表为空，那么只有LLM node的agent是没有tool call的能力的

dynamic tools（动态工具，代理所拥有的工具集合会在运行时进行调整，而非事先全部确定好），根据工具是否事先已知，有两种处理方式：

- 在代理创建时已知所有可用工具的情况下，您可以预先注册这些工具，并根据状态、权限或上下文动态筛选哪些工具会暴露给模型使用。（1、在编译/启动时已知所有可用工具 2、希望根据权限、功能标志或对话状态进行筛选 3、工具是固定的，但其可用性是动态变化的）
- 当工具是在运行时被发现或创建的（例如，从 MCP 服务器加载、根据用户数据生成，或者从远程注册表获取），您需要同时注册这些工具并动态地处理其执行过程。
这需要两个中间件钩子：（1、工具是在运行时被发现，例如，从 MCP 服务器获取 2、当工具是根据用户数据或配置动态生成的 3、当您正在与外部工具注册表进行集成时）
  - wrap_model_call - 将动态工具添加到请求中，
  - wrap_tool_call - 处理动态添加的工具的执行过程

同时要处理tool执行失败的情况

```html

from langchain.agents import create_agent
from langchain.agents.middleware import wrap_tool_call
from langchain.messages import ToolMessage


@wrap_tool_call
def handle_tool_errors(request, handler):
    """Handle tool execution errors with custom messages."""
    try:
        return handler(request)
    except Exception as e:
        # Return a custom error message to the model
        return ToolMessage(
            content=f"Tool error: Please check your input and try again. ({str(e)})",
            tool_call_id=request.tool_call["id"]
        )

agent = create_agent(
    model="gpt-5.4",
    tools=[search, get_weather],
    middleware=[handle_tool_errors]  # 处理失败的情况，要告诉模型
)

# 输出如下内容

[
    ...
    ToolMessage(
        content="Tool error: Please check your input and try again. (division by zero)",
        tool_call_id="..."
    ),
    ...
]

```

代理遵循“ReAct”（“推理 + 行动”）模式（reasoning + acting），交替进行简短的推理步骤并调用特定工具，将所得的观察结果输入到后续决策中，直至最终得出答案。

这边deepseek，和 langchain官网都有比较全的多次工具调用ReAct案例

C、System Prompt

可以通过提供 提示词 来控制您的代理处理任务的方式。可以将“system_prompt”参数作为字符串进行设置：

```html

agent = create_agent(
    model,
    tools,
    system_prompt="You are a helpful assistant. Be concise and accurate."
)

```

除了str还有SystemMessage功能，比如

```html

from langchain.agents import create_agent
from langchain.messages import SystemMessage, HumanMessage

literary_agent = create_agent(
    model="google_genai:gemini-3.1-pro-preview",
    system_prompt=SystemMessage(
        content=[
            {
                "type": "text",
                "text": "You are an AI assistant tasked with analyzing literary works.",
            },
            {
                "type": "text",
                "text": "<the entire contents of 'Pride and Prejudice'>",
                "cache_control": {"type": "ephemeral"}
            }
        ]
    )
)

result = literary_agent.invoke(
    {"messages": [HumanMessage("Analyze the major themes in 'Pride and Prejudice'.")]}
)

```

cache_control字段告诉Anthropic去缓存 content block。

动态system prompt，直接看下面示例代码

```html

from typing import TypedDict

from langchain.agents import create_agent
from langchain.agents.middleware import dynamic_prompt, ModelRequest


class Context(TypedDict):
    user_role: str

@dynamic_prompt
def user_role_prompt(request: ModelRequest) -> str:
    """Generate system prompt based on user role."""
    user_role = request.runtime.context.get("user_role", "user")
    base_prompt = "You are a helpful assistant."

    if user_role == "expert":
        return f"{base_prompt} Provide detailed technical responses."
    elif user_role == "beginner":
        return f"{base_prompt} Explain concepts simply and avoid jargon."

    return base_prompt

agent = create_agent(
    model="gpt-5.4",
    tools=[web_search],
    middleware=[user_role_prompt],
    context_schema=Context
)

# The system prompt will be set dynamically based on context
result = agent.invoke(
    {"messages": [{"role": "user", "content": "Explain machine learning"}]},
    context={"user_role": "expert"}
)

```

D、Name（名字）

```html

agent = create_agent(
    model,
    tools,
    name="research_assistant"
)

# 在多agent系统中，该代理作为子图添加的时候，此名称作为节点标识符

```

E、Invocation（调用）

可以通过向其状态中传递更新信息来调用一个代理。所有代理在其状态中都包含一系列消息；要调用该代理，请传递一条新的消息：

```html

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
)


```

F、结构化输出（Structured output）

输出如果需要特定格式，用response_format

ToolStrategy 和 ProviderStrategy

```html

# 工具策略利用人工工具调用来生成结构化输出。这适用于任何支持工具调用的模型。当提供者原生的结构化输出（通过“ProviderStrategy”实现）不可用或不可靠时，应使用此策略。
# ProviderStrategy就是靠模型本身的能力

from pydantic import BaseModel
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


class ContactInfo(BaseModel):
    name: str
    email: str
    phone: str

agent = create_agent(
    model="gpt-5.4-mini",
    tools=[search_tool],
    response_format=ToolStrategy(ContactInfo)
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Extract contact info from: John Doe, john@example.com, (555) 123-4567"}]
})

result["structured_response"]
# ContactInfo(name='John Doe', email='john@example.com', phone='(555) 123-4567')

```

G、记忆（Memory）

代理会通过消息状态自动保存对话历史记录。您还可以将代理配置为使用自定义状态模式来在对话过程中记住更多信息。

存储在状态中的信息可以被视为代理的短期记忆（自定义状态模式必须扩展 AgentState 为一个 TypedDict。），有两种自定义状态的方式：

- 通过中间件（推荐）
- 通过 create_agent 上的 state_schema


Defining state via middleware

```html

# 当您的自定义状态需要通过附加到该中间件的特定中间件钩子和工具来访问时，使用中间件来定义自定义状态。

from langchain.agents import AgentState
from langchain.agents.middleware import AgentMiddleware
from typing import Any


class CustomState(AgentState):
    user_preferences: dict

class CustomMiddleware(AgentMiddleware):
    state_schema = CustomState
    tools = [tool1, tool2]

    def before_model(self, state: CustomState, runtime) -> dict[str, Any] | None:
        ...

agent = create_agent(
    model,
    tools=tools,
    middleware=[CustomMiddleware()]
)

# The agent can now track additional state beyond messages
result = agent.invoke({
    "messages": [{"role": "user", "content": "I prefer technical explanations"}],
    "user_preferences": {"style": "technical", "verbosity": "detailed"},
})

```

Defining state via state_schema

```html

from langchain.agents import AgentState


class CustomState(AgentState):
    user_preferences: dict

agent = create_agent(
    model,
    tools=[tool1, tool2],
    state_schema=CustomState
)
# The agent can now track additional state beyond messages
result = agent.invoke({
    "messages": [{"role": "user", "content": "I prefer technical explanations"}],
    "user_preferences": {"style": "technical", "verbosity": "detailed"},
})

```

H、流式化（Streaming）

已经看到如何获取最终响应，如果代理执行多个步骤，这可能会耗费一些时间。为了展示中间的进展情况，我们可以实时回传出现的消息。

```html

from langchain.messages import AIMessage, HumanMessage

for chunk in agent.stream({
    "messages": [{"role": "user", "content": "Search for AI news and summarize the findings"}]
}, stream_mode="values"):
    # Each chunk contains the full state at that point
    latest_message = chunk["messages"][-1]
    if latest_message.content:
        if isinstance(latest_message, HumanMessage):
            print(f"User: {latest_message.content}")
        elif isinstance(latest_message, AIMessage):
            print(f"Agent: {latest_message.content}")
    elif latest_message.tool_calls:
        print(f"Calling tools: {[tc['name'] for tc in latest_message.tool_calls]}")

```

#### model

大型语言模型（LLMs）是强大的人工智能工具，能够像人类一样理解和生成文本。它们用途广泛，可以撰写内容、翻译语言、总结信息以及回答问题，无需针对每项任务进行专门训练。

也有额外的能力（这个看模型了）：

- 工具调用——调用外部工具（如数据库查询或 API 调用）并将其结果用于响应中。
- 结构化输出——模型的响应被限制为遵循定义的格式。
- 多模态——处理并返回除文本之外的数据，例如图像、音频和视频。
- 推理——模型进行多步推理以得出结论。

模型是代理的推理引擎。它们推动代理的决策过程，决定调用哪些工具、如何解释结果以及何时提供最终答案。
您所选择的模型的质量和能力直接影响代理的基本可靠性和性能。不同的模型在不同的任务上表现出色——有些更擅长遵循复杂的指令，有些更擅长结构化推理，还有一些支持更大的上下文窗口以处理更多信息。
LangChain 的标准模型接口使您能够访问许多不同的提供商集成，这使得您能够轻松地试验和在模型之间切换，以找到最适合您用例的模型。

一种是动态的，创建代理的时候指定

一种是独立运行，可以直接调用模型来执行诸如文本生成，分类，提取等任务

```html




# 以下是独立运行（init_chat_model）

import os
from langchain.chat_models import init_chat_model

os.environ["OPENAI_API_KEY"] = "sk-..."

model = init_chat_model("gpt-5.4")

response = model.invoke("Why do parrots talk?")

```

还有两种 会话的invoke

```html

conversation = [
    {"role": "system", "content": "You are a helpful assistant that translates English to French."},
    {"role": "user", "content": "Translate: I love programming."},
    {"role": "assistant", "content": "J'adore la programmation."},
    {"role": "user", "content": "Translate: I love building applications."}
]

response = model.invoke(conversation)
print(response)  # AIMessage("J'adore créer des applications.")

from langchain.messages import HumanMessage, AIMessage, SystemMessage

conversation = [
    SystemMessage("You are a helpful assistant that translates English to French."),
    HumanMessage("Translate: I love programming."),
    AIMessage("J'adore la programmation."),
    HumanMessage("Translate: I love building applications.")
]

response = model.invoke(conversation)
print(response)  # AIMessage("J'adore créer des applications.")

# 以上会保留上下文对话

```

用streaming的方式给一点好的体验


```html

for chunk in model.stream("Why do parrots have colorful feathers?"):
    print(chunk.text, end="|", flush=True)

```

使用batch

将一组独立的请求批量发送至模型处理，能够显著提高性能并降低成本，因为这样可以实现并行处理：

```html

responses = model.batch([
    "Why do parrots have colorful feathers?",
    "How do airplanes fly?",
    "What is quantum computing?"
])
for response in responses:
    print(response)

```

模型绑定工具(Tool calling)

在绑定用户自定义工具时，模型的响应会包含执行该工具的请求。当将模型与代理分开使用时，您需要自行执行所请求的工具，并将结果返回给模型，以便在后续推理中使用。而当使用代理时，代理循环会为您处理工具执行循环。

```html

from langchain.tools import tool

# 工具装饰器
@tool
def get_weather(location: str) -> str:
    """Get the weather at a location."""
    return f"It's sunny in {location}."

# 绑定工具的代码
model_with_tools = model.bind_tools([get_weather])

response = model_with_tools.invoke("What's the weather like in Boston?")
for tool_call in response.tool_calls:
    # View tool calls made by the model
    print(f"Tool: {tool_call['name']}")
    print(f"Args: {tool_call['args']}")

```

而当使用代理时，代理循环会为您处理工具执行循环。


多模态（multimodal）

获取多模态的返回，在content_blocks中

```html

response = model.invoke("Create a picture of a cat")
print(response.content_blocks)
# [
#     {"type": "text", "text": "Here's a picture of a cat"},
#     {"type": "image", "base64": "...", "mime_type": "image/jpeg"},
# ]

```

reasoning（思考）

许多模型能够进行多步骤推理以得出结论。这需要将复杂的问题分解为更小、更易于处理的步骤。
如果底层模型能够支持，您可以展示这一推理过程，以便更好地理解模型是如何得出最终答案的。

```html
for chunk in model.stream("Why do parrots have colorful feathers?"):
    reasoning_steps = [r for r in chunk.content_blocks if r["type"] == "reasoning"]
    print(reasoning_steps if reasoning_steps else chunk.text)

```

Server-side tool use

一些提供商支持服务器端的工具调用循环：模型能够与网络搜索、代码解释器以及其他工具进行交互，并在一次对话中分析结果。
如果模型在服务器端调用工具，响应消息的内容将包含表示工具调用和结果的内容。访问响应消息的内容块将以与提供商无关的格式返回服务器端的工具调用和结果：

```html

[
    {
        "type": "server_tool_call",
        "name": "web_search",
        "args": {
            "query": "positive news stories today",
            "type": "search"
        },
        "id": "ws_abc123"
    },
    {
        "type": "server_tool_result",
        "tool_call_id": "ws_abc123",
        "status": "success"
    },
    {
        "type": "text",
        "text": "Here are some positive news stories from today...",
        "annotations": [
            {
                "end_index": 410,
                "start_index": 337,
                "title": "article title",
                "type": "citation",
                "url": "..."
            }
        ]
    }
]

```

这代表了一个单独的对话回合；与客户端工具调用不同，这里无需传递任何相关的工具消息对象。
请查看您所选提供商的集成页面，以获取可用工具及使用说明。

Rate limiting

许多聊天模型提供商会对在特定时间段内可进行的调用次数设置限制。如果达到速率限制，通常会从提供商处收到速率限制错误响应，并且需要等待一段时间后再进行更多请求。
为了帮助管理速率限制，聊天模型集成接受一个“速率限制器”参数，该参数可在初始化时提供，用于控制请求的发送速率。

```html

from langchain_core.rate_limiters import InMemoryRateLimiter

rate_limiter = InMemoryRateLimiter(
    requests_per_second=0.1,  # 1 request every 10s
    check_every_n_seconds=0.1,  # Check every 100ms whether allowed to make a request
    max_bucket_size=10,  # Controls the maximum burst size.
)

model = init_chat_model(
    model="gpt-5.4",
    model_provider="openai",
    rate_limiter=rate_limiter  
)

```

Base URL 和 Proxy的设置

```html

# base url的设置

model = init_chat_model(
    model="MODEL_NAME",
    model_provider="openai",
    base_url="BASE_URL",
    api_key="YOUR_API_KEY",
)

# proxy的设置

from langchain_openai import ChatOpenAI

model = ChatOpenAI(
    model="gpt-5.4",
    openai_proxy="http://proxy.example.com:8080"
)

```

Token usage（token用量）


许多模型提供者会在调用响应中返回 token 使用信息。如果可用，此信息将包含在由相应模型生成的 AIMessage 对象中

可以在应用程序中通过回调函数或上下文管理器来跟踪不同模型的聚合 token 数量，示例代码如下所示：

```html

# 方法一 使用callback handler

from langchain.chat_models import init_chat_model
from langchain_core.callbacks import UsageMetadataCallbackHandler

model_1 = init_chat_model(model="gpt-5.4-mini")
model_2 = init_chat_model(model="claude-haiku-4-5-20251001")

callback = UsageMetadataCallbackHandler()
result_1 = model_1.invoke("Hello", config={"callbacks": [callback]})
result_2 = model_2.invoke("Hello", config={"callbacks": [callback]})
print(callback.usage_metadata)

# 输出

{
    'gpt-5.4-mini': {
        'input_tokens': 8,
        'output_tokens': 10,
        'total_tokens': 18,
        'input_token_details': {'audio': 0, 'cache_read': 0},
        'output_token_details': {'audio': 0, 'reasoning': 0}
    },
    'claude-haiku-4-5-20251001': {
        'input_tokens': 8,
        'output_tokens': 21,
        'total_tokens': 29,
        'input_token_details': {'cache_read': 0, 'cache_creation': 0}
    }
}


# 方法二 使用context manager

from langchain.chat_models import init_chat_model
from langchain_core.callbacks import get_usage_metadata_callback

model_1 = init_chat_model(model="gpt-5.4-mini")
model_2 = init_chat_model(model="claude-haiku-4-5-20251001")

with get_usage_metadata_callback() as cb:
    model_1.invoke("Hello")
    model_2.invoke("Hello")
    print(cb.usage_metadata)

{
    'gpt-5.4-mini': {
        'input_tokens': 8,
        'output_tokens': 10,
        'total_tokens': 18,
        'input_token_details': {'audio': 0, 'cache_read': 0},
        'output_token_details': {'audio': 0, 'reasoning': 0}
    },
    'claude-haiku-4-5-20251001': {
        'input_tokens': 8,
        'output_tokens': 21,
        'total_tokens': 29,
        'input_token_details': {'cache_read': 0, 'cache_creation': 0}
    }
}


```

Invocation Config（调用配置）

在调用模型时，您可以通过“config”参数使用“RunnableConfig”字典来传递额外的配置信息。这可在运行时控制执行行为、回调函数以及元数据跟踪。
常见的配置选项包括：

```html

response = model.invoke(
    "Tell me a joke",
    config={
        "run_name": "joke_generation",      # Custom name for this run
        "tags": ["humor", "demo"],          # Tags for categorization
        "metadata": {"user_id": "123"},     # Custom metadata
        "callbacks": [my_callback_handler], # Callback handlers
    }
)

```

这个通常和 监控搭配使用

- 使用langsmith跟踪调试
- 实现自定义的日志记录或监控
- 在生产环境中控制资源使用情况（usage）
- 追踪复杂流程中的调用情况


#### messages

基础使用

```html

from langchain.chat_models import init_chat_model
from langchain.messages import HumanMessage, AIMessage, SystemMessage

model = init_chat_model("gpt-5-nano")

system_msg = SystemMessage("You are a helpful assistant.")
human_msg = HumanMessage("Hello, how are you?")

# Use with chat models
messages = [system_msg, human_msg]
response = model.invoke(messages)  # Returns AIMessage

```

1、text prompts

最简单的

```html

response = model.invoke("Write a haiku about spring")

```

2、message prompts

可以通过提供一系列消息对象的方式，将消息列表传递给模型。

```html

from langchain.messages import SystemMessage, HumanMessage, AIMessage

messages = [
    SystemMessage("You are a poetry expert"),
    HumanMessage("Write a haiku about spring"),
    AIMessage("Cherry blossoms bloom...")
]
response = model.invoke(messages)

```

- 管理多轮对话
- 处理多模态内容（图像、音频、文件）
- 包含系统指令

3、Dictionary format

openai的格式来完成指定内容

```html

messages = [
    {"role": "system", "content": "You are a poetry expert"},
    {"role": "user", "content": "Write a haiku about spring"},
    {"role": "assistant", "content": "Cherry blossoms bloom..."}
]
response = model.invoke(messages)

```

message content

消息具有一个“内容”属性(content)，该属性的类型较为宽松，支持字符串以及未类型化的对象列表（例如字典）。这使得在 LangChain 聊天模型中直接支持提供商原生结构（如多模态内容和其他数据）成为可能。

以下是一个使用多模态输入的示例：

```html

from langchain.messages import HumanMessage

# String content
human_message = HumanMessage("Hello, how are you?")

# Provider-native format (e.g., OpenAI)
human_message = HumanMessage(content=[
    {"type": "text", "text": "Hello, how are you?"},
    {"type": "image_url", "image_url": {"url": "https://example.com/image.jpg"}}
])

# List of standard content blocks
human_message = HumanMessage(content_blocks=[
    {"type": "text", "text": "Hello, how are you?"},
    {"type": "image", "url": "https://example.com/image.jpg"},
])

```

#### 工具

工具能够扩展代理的能力——使它们能够获取实时数据、执行代码、查询外部数据库以及在现实世界中采取行动。
在内部，这些工具是可调用的函数，具有明确的输入和输出格式，并会传递给聊天模型。模型会根据对话情境决定何时调用工具，并提供相应的输入参数。

创建工具最简单的方法是使用 @tool 装饰器。默认情况下，函数的文档字符串会成为工具的描述信息，有助于模型理解何时使用该工具：

```html

from langchain.tools import tool

@tool
def search_database(query: str, limit: int = 10) -> str:
    """Search the customer database for records matching the query.

    Args:
        query: Search terms to look for
        limit: Maximum number of results to return
    """
    return f"Found {limit} results for '{query}'"

```

对于工具名称，建议采用小写驼峰式命名法（例如，web_search 而不是 Web Search）。一些模型提供商可能会对包含空格或特殊字符的名称存在兼容性问题或直接拒绝使用。坚持使用字母数字字符、下划线和连字符有助于提高不同提供商之间的兼容性。


当工具能够获取诸如对话历史、用户数据和持久内存等运行时信息时，其功能将最为强大。介绍如何在工具内部访问和更新这些信息。
工具可以通过“ToolRuntime”参数来获取运行时信息，该参数提供了：

| 组件 | 描述 | 使用案例 |
| :----: | :----:  | :----: |
| State | Short-term memory - mutable data that exists for the current conversation (messages, counters, custom fields) | Access conversation history, track tool call counts |
| Context | Immutable configuration passed at invocation time (user IDs, session info) | Personalize responses based on user identity |
| Store | Long-term memory - persistent data that survives across conversations | Save user preferences, maintain knowledge base |
| Stream Writer | Emit real-time updates during tool execution | Show progress for long-running operations |
| Execution Info | Identity and retry information for the current execution (thread ID, run ID, attempt number) | Access thread/run IDs, adjust behavior based on retry state |
| Server Info | Server-specific metadata when running on LangGraph Server (assistant ID, graph ID, authenticated user) | Access assistant ID, graph ID, or authenticated user info |
| Config | RunnableConfig for the execution | Access callbacks, tags, and metadata |
| Tool Call ID | Unique identifier for the current tool invocation | Correlate tool calls for logs and model invocations |


Short-term memory (State)

“状态”代表的是在对话期间存在的短期记忆。它包含了消息历史记录以及您在图状状态中定义的任何自定义字段。

访问状态，Access state

Tools can access the current conversation state using runtime.state:

```html

from langchain.tools import tool, ToolRuntime
from langchain.messages import HumanMessage

@tool
def get_last_user_message(runtime: ToolRuntime) -> str:
    """Get the most recent message from the user."""
    messages = runtime.state["messages"]

    # Find the last human message
    for message in reversed(messages):
        if isinstance(message, HumanMessage):
            return message.content

    return "No user messages found"

# Access custom state fields
@tool
def get_user_preference(
    pref_name: str,
    runtime: ToolRuntime
) -> str:
    """Get a user preference value."""
    preferences = runtime.state.get("user_preferences", {})
    return preferences.get(pref_name, "Not set")

```

更新状态，update state

使用命令来更新代理的状态。这对于需要更新自定义状态字段的工具来说非常有用。在更新操作中包含一个“工具消息”，以便模型能够看到工具调用的结果：

```html

from langchain.agents import AgentState
from langchain.messages import ToolMessage
from langchain.tools import ToolRuntime, tool
from langgraph.types import Command


class CustomState(AgentState):
    user_name: str


@tool
def set_user_name(new_name: str, runtime: ToolRuntime[None, CustomState]) -> Command:
    """Set the user's name in the conversation state."""
    return Command(
        update={
            "user_name": new_name,
            "messages": [
                ToolMessage(
                    content=f"User name set to {new_name}.",
                    tool_call_id=runtime.tool_call_id,
                )
            ],
        }
    )

```

Context

上下文提供了不可变的配置数据，这些数据在调用时传递。将其用于用户 ID、会话详情或在对话过程中不应发生变化的应用特定设置。
通过 runtime.context 访问上下文：

```html

from dataclasses import dataclass
from langchain_openai import ChatOpenAI
from langchain.agents import create_agent
from langchain.tools import tool, ToolRuntime


USER_DATABASE = {
    "user123": {
        "name": "Alice Johnson",
        "account_type": "Premium",
        "balance": 5000,
        "email": "alice@example.com"
    },
    "user456": {
        "name": "Bob Smith",
        "account_type": "Standard",
        "balance": 1200,
        "email": "bob@example.com"
    }
}

@dataclass
class UserContext:
    user_id: str

@tool
def get_account_info(runtime: ToolRuntime[UserContext]) -> str:
    """Get the current user's account information."""
    user_id = runtime.context.user_id

    if user_id in USER_DATABASE:
        user = USER_DATABASE[user_id]
        return f"Account holder: {user['name']}\nType: {user['account_type']}\nBalance: ${user['balance']}"
    return "User not found"

model = ChatOpenAI(model="gpt-5.4")
agent = create_agent(
    model,
    tools=[get_account_info],
    context_schema=UserContext,
    system_prompt="You are a financial assistant."
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's my current balance?"}]},
    context=UserContext(user_id="user123")
)

```


Long-term memory (Store)

BaseStore 提供了持久化的存储功能，能够在不同的对话过程中保持数据的完整性。与状态（短期记忆）不同，存储在该存储中的数据在后续的会话中仍可使用。
通过 runtime.store 来访问该存储。该存储采用命名空间/键模式来组织数据：

```html

from typing import Any
from langgraph.store.memory import InMemoryStore
from langchain.agents import create_agent
from langchain.tools import tool, ToolRuntime
from langchain_openai import ChatOpenAI

# Access memory
@tool
def get_user_info(user_id: str, runtime: ToolRuntime) -> str:
    """Look up user info."""
    store = runtime.store
    user_info = store.get(("users",), user_id)
    return str(user_info.value) if user_info else "Unknown user"

# Update memory
@tool
def save_user_info(user_id: str, user_info: dict[str, Any], runtime: ToolRuntime) -> str:
    """Save user info."""
    store = runtime.store
    store.put(("users",), user_id, user_info)
    return "Successfully saved user info."

model = ChatOpenAI(model="gpt-5.4")

store = InMemoryStore()
agent = create_agent(
    model,
    tools=[get_user_info, save_user_info],
    store=store
)

# First session: save user info
agent.invoke({
    "messages": [{"role": "user", "content": "Save the following user: userid: abc123, name: Foo, age: 25, email: foo@langchain.dev"}]
})

# Second session: get user info
agent.invoke({
    "messages": [{"role": "user", "content": "Get user info for user with id 'abc123'"}]
})
# Here is the user info for user with ID "abc123":
# - Name: Foo
# - Age: 25
# - Email: foo@langchain.dev

```

ToolNode

ToolNode 是一个预先构建的节点，用于在 LangGraph 工作流中执行工具。它能够自动处理并行工具执行、错误处理以及状态注入等事宜。

对于那些需要对工具执行模式进行精细控制的自定义工作流程，请使用“ToolNode”而非“create_agent”。它是驱动工具执行的底层组件。

```html

from langchain.tools import tool
from langgraph.prebuilt import ToolNode
from langgraph.graph import StateGraph, MessagesState, START, END

@tool
def search(query: str) -> str:
    """Search for information."""
    return f"Results for: {query}"

@tool
def calculator(expression: str) -> str:
    """Evaluate a math expression."""
    return str(eval(expression))

# Create the ToolNode with your tools
tool_node = ToolNode([search, calculator])

# Use in a graph
builder = StateGraph(MessagesState)
builder.add_node("tools", tool_node)
# ... add other nodes and edges

```

tool的返回值有三种

```html

1、返回string

from langchain.tools import tool


@tool
def get_weather(city: str) -> str:
    """Get weather for a city."""
    return f"It is currently sunny in {city}."

- 返回值会被转换为一个“工具消息”。
- 该模型会看到这段文本，并决定接下来要做什么。
- 除非模型或另一个工具随后进行更改，否则不会更改任何代理状态字段。


2、返回object

from langchain.tools import tool


@tool
def get_weather_data(city: str) -> dict:
    """Get structured weather data for a city."""
    return {
        "city": city,
        "temperature_c": 22,
        "conditions": "sunny",
    }

- 该对象被序列化并作为工具输出返回。
- 该模型能够读取特定字段并对其进行推理。
- 与字符串返回类似，这并不会直接更新图形（graph）状态。

3、返回command

当工具需要更新图形状态（例如设置用户偏好或应用程序状态）时，返回一个命令。您可以选择性地在返回的命令中包含或不包含工具消息。如果模型需要确认工具操作成功（例如确认偏好设置的更改），则在更新中包含一个工具消息，并使用运行时的“tool_call_id”作为“tool_call_id”参数。

from langchain.messages import ToolMessage
from langchain.tools import ToolRuntime, tool
from langgraph.types import Command


@tool
def set_language(language: str, runtime: ToolRuntime) -> Command:
    """Set the preferred response language."""
    return Command(
        update={
            "preferred_language": language,
            "messages": [
                ToolMessage(
                    content=f"Language set to {language}.",
                    tool_call_id=runtime.tool_call_id,
                )
            ],
        }
    )

- 该命令通过“更新”来更新状态。
- 更新后的状态可在同一运行的后续步骤中使用。
- 对于可能由并行工具调用（parallel tool calls）更新的字段，应使用归约器（reducers）。

```

处理工具调用错误的情况


```html

from langgraph.prebuilt import ToolNode

# Default: catch invocation errors, re-raise execution errors
tool_node = ToolNode(tools)

# Catch all errors and return error message to LLM
tool_node = ToolNode(tools, handle_tool_errors=True)

# Custom error message
tool_node = ToolNode(tools, handle_tool_errors="Something went wrong, please try again.")

# Custom error handler
def handle_error(e: ValueError) -> str:
    return f"Invalid input: {e}"

tool_node = ToolNode(tools, handle_tool_errors=handle_error)

# Only catch specific exception types
tool_node = ToolNode(tools, handle_tool_errors=(ValueError, TypeError))

```

#### Short-term memory

短期记忆使您的应用程序能够在单个线程或对话中记住过往的交互内容。

对话历史是短期记忆最常见的形式。长篇对话对当今的大型语言模型构成了挑战；完整的对话历史可能无法完全包含在语言模型的上下文窗口内，从而导致上下文丢失或出现错误。
即使您的模型支持完整的上下文长度，大多数语言模型在处理长上下文时的表现仍然不佳。它们会被陈旧或偏离主题的内容分散注意力，同时还会面临响应时间较慢和成本较高的问题。
聊天模型通过消息来接受上下文，这些消息包括指令（系统消息）和输入（人类消息）。在聊天应用程序中，消息交替由人类输入和模型响应构成，随着时间的推移，消息列表会变得越来越长。由于上下文窗口是有限的，许多应用程序可以从使用技术来删除或“遗忘”陈旧信息中受益。

若要为代理添加短期记忆（线程级持久性），则在创建代理时需指定一个检查点器。

```html

from langchain.agents import create_agent
from langgraph.checkpoint.memory import InMemorySaver  


agent = create_agent(
    "gpt-5.4",
    tools=[get_user_info],
    checkpointer=InMemorySaver(),
)

agent.invoke(
    {"messages": [{"role": "user", "content": "Hi! My name is Bob."}]},
    {"configurable": {"thread_id": "1"}},
)

```

生产环境中，使用数据库支持checkpointer，就是作持久化处理

```html

pip install langgraph-checkpoint-postgres

from langchain.agents import create_agent

from langgraph.checkpoint.postgres import PostgresSaver  


DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup() # auto create tables in PostgreSQL
    agent = create_agent(
        "gpt-5.4",
        tools=[get_user_info],
        checkpointer=checkpointer,
    )

```

自定义agent memory

默认情况下，代理使用“AgentState”来管理短期记忆，特别是通过“messages”键来管理对话历史记录。
您可以扩展“AgentState”以添加其他字段。使用“state_schema”参数将自定义状态模式传递给“create_agent”函数。

```html

from langchain.agents import create_agent, AgentState
from langgraph.checkpoint.memory import InMemorySaver


class CustomAgentState(AgentState):
    user_id: str
    preferences: dict

agent = create_agent(
    "gpt-5.4",
    tools=[get_user_info],
    state_schema=CustomAgentState,
    checkpointer=InMemorySaver(),
)

# Custom state can be passed in invoke
result = agent.invoke(
    {
        "messages": [{"role": "user", "content": "Hello"}],
        "user_id": "user_123",
        "preferences": {"theme": "dark"}
    },
    {"configurable": {"thread_id": "1"}})

```

如果打开短期记忆，context window会爆，可以有以下做法

- 在agent中修剪历史消息（trim message）
- 删除消息（Delete messages）
- 总结message（Summarize messages）
- 压缩（LHY）

```html

总结的代码

from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware
from langgraph.checkpoint.memory import InMemorySaver
from langchain_core.runnables import RunnableConfig


checkpointer = InMemorySaver()

agent = create_agent(
    model="gpt-5.4",
    tools=[],
    middleware=[
        SummarizationMiddleware(
            model="gpt-5.4-mini",
            trigger=("tokens", 4000),
            keep=("messages", 20)
        )
    ],
    checkpointer=checkpointer,
)

config: RunnableConfig = {"configurable": {"thread_id": "1"}}
agent.invoke({"messages": "hi, my name is bob"}, config)
agent.invoke({"messages": "write a short poem about cats"}, config)
agent.invoke({"messages": "now do the same but for dogs"}, config)
final_response = agent.invoke({"messages": "what's my name?"}, config)

final_response["messages"][-1].pretty_print()
"""
================================== Ai Message ==================================

Your name is Bob!
"""

```

#### middleware

Middleware exposes hooks before and after each of those steps:

有before agent 和 before model 还有 after model 和 after agent

LangChain 和 Deep Agents 提供了针对常见应用场景的预构建中间件。每种中间件都已具备生产环境下的使用条件，并且可以根据您的具体需求进行配置。预置的一些middleware有

| 组件 | 描述 |
| :----: | :----:  |
| Summarization | Automatically summarize conversation history when approaching token limits. |
| Human-in-the-loop | Pause execution for human approval of tool calls. |
| Model call limit | Limit the number of model calls to prevent excessive costs. |
| Tool call limit | Control tool execution by limiting call counts. |
| Model fallback | Automatically fallback to alternative models when primary fails. |
| PII detection | Detect and handle Personally Identifiable Information (PII). |
| To-do list | Equip agents with task planning and tracking capabilities. |
| LLM tool selector | Use an LLM to select relevant tools before calling main model. |

等等还有不少，看着都是比较实用的

通过实现在代理执行流程中的特定节点运行的钩子函数来构建自定义中间件。钩子中间件提供了两种拦截代理执行的钩子方式：

- Node-style hooks，节点式钩子在特定的执行点运行：
  - before_agent
  - before_model
  - after_model
  - after_agent


- Wrap-style hooks，“包裹式”钩子会环绕每个调用语句，让您能够掌控执行过程：
  - wrap_model_call - Around each model call
  - wrap_tool_call - Around each tool call


#### Guardrails

安全性相关的，防护栏能够帮助您构建安全且符合规范的 AI 应用程序，其通过在智能代理执行过程中的关键节点对内容进行验证和过滤来实现这一目的。它们能够检测敏感信息、执行内容政策、验证输出，并在出现不安全行为之前防止其引发问题。

- 防止个人身份信息泄露
- 检测并阻止即时注入攻击
- 屏蔽不当或有害内容
- 执行业务规则和合规要求
- 验证输出的质量和准确性

可以使用

- PIIMiddleware
- Human-in-the-loop

#### context engineering

构建代理（或任何大型语言模型应用）的难点在于要让它们足够可靠。虽然它们在原型阶段可能能正常运行，但在实际使用场景中却常常会出问题。

当代理失败时，通常是因为代理内部的 LLM 调用做出了错误的决策/没有按照我们的预期执行。LLMs 失败的原因主要有两种：

- 底层的 LLM 能力不足
- 没有将“正确”的上下文传递给 LLM

通常情况下，导致代理不可靠的真正原因是第二种原因。
“上下文工程”是指以正确的格式提供正确的信息和工具，以便 LLM 能够完成任务。这是 AI 工程师的首要任务。这种“正确”上下文的缺乏是实现更可靠代理的首要障碍，而 LangChain 的代理抽象设计就是为了促进上下文工程而专门打造的。

LangChain 中间件是隐藏在背后的机制，它使得使用 LangChain 的开发者能够实现context工程。中间件允许您在代理生命周期的任何步骤中进行连接：
- 更新context
- 跳转到代理生命周期中的不同步骤


可以实践的点

- 选择工具的时候
- 选择模型的时候
- 返回格式化

#### MCP（Model Context Protocol）

模型上下文协议（MCP）是一种开放协议，它规范了应用程序向语言模型提供工具和上下文的方式。LangChain 代理可以通过使用“langchain-mcp-adapters”库来使用 MCP 服务器上定义的工具。

langchain-mcp-adapters enables agents to use tools defined across one or more MCP servers.

```html

import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient  
from langchain.agents import create_agent

async def main():
    client = MultiServerMCPClient(
        {
            "math": {
                "transport": "stdio",  # Local subprocess communication
                "command": "python",
                # Absolute path to your math_server.py file
                "args": ["/path/to/math_server.py"],
            },
            "weather": {
                "transport": "http",  # HTTP-based remote server
                # Ensure you start your weather server on port 8000
                "url": "http://localhost:8000/mcp",
            }
        }
    )

    tools = await client.get_tools()
    agent = create_agent(
        "claude-sonnet-4-6",
        tools  
    )
    math_response = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "what's (3 + 5) x 12?"}]}
    )
    weather_response = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "what is the weather in nyc?"}]}
    )
    print(math_response)
    print(weather_response)

if __name__ == "__main__":
    asyncio.run(main())


```

To create a custom MCP server, use the FastMCP library:

```html

pip install fastmcp

from fastmcp import FastMCP

mcp = FastMCP("Math")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b

@mcp.tool()
def multiply(a: int, b: int) -> int:
    """Multiply two numbers"""
    return a * b

if __name__ == "__main__":
    mcp.run(transport="stdio")

```

MCP supports different transport mechanisms for client-server communication.

http 和 stdio

http的协议可以加head

stdio的协议，本质上是有状态的，子进程在客户端连接的整个生命周期内都会持续存在。然而，在使用 MultiServerMCPClient 但未进行明确会话管理的情况下，每次工具调用都会创建一个新的会话

默认情况下，MultiServerMCPClient 是无状态的：每次工具调用都会创建一个新的 MCP 会话，执行该工具，然后进行清理。
如果您需要控制 MCP 会话的生命周期（例如，在与具有跨工具调用保持上下文功能的有状态服务器一起工作时），可以使用 client.session() 方法创建一个持久的 ClientSession。

```html

from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.tools import load_mcp_tools
from langchain.agents import create_agent

client = MultiServerMCPClient({...})

# Create a session explicitly
async with client.session("server_name") as session:
    # Pass the session to load tools, resources, or prompts
    tools = await load_mcp_tools(session)
    agent = create_agent(
        "google_genai:gemini-3.1-pro-preview",
        tools
    )

```

mcp的核心功能

- tools

工具使 MCP 服务器能够 暴露 可执行的功能，让语言模型能够调用这些功能来执行各种操作，例如查询数据库、调用 API 或与外部系统进行交互。LangChain 将 MCP 工具转换为 LangChain 工具，使其能够直接在任何 LangChain 代理或工作流程中使用。

Loading tools，Use client.get_tools() to retrieve tools from MCP servers and pass them to your agent:

```html

from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent

client = MultiServerMCPClient({...})
tools = await client.get_tools()  # 加载tools
agent = create_agent("claude-sonnet-4-6", tools)

```

Structured content，MCP 工具能够将结构化内容与人类可读的文本响应一同返回。当工具需要返回机器可解析的数据（如 JSON 格式的数据）以及展示给模型的文本时，这一点非常有用。
当 MCP 工具返回结构化内容时，适配器会将其封装在一个 MCPToolArtifact 中，并将其作为工具的成果返回。您可以通过Tool Message 中的 artifact 字段来访问此内容。您还可以使用拦截器来自动处理或转换结构化内容。

在调用您的代理后，您可以在响应中从工具消息中访问结构化内容：

```html

from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent
from langchain.messages import ToolMessage

client = MultiServerMCPClient({...})
tools = await client.get_tools()
agent = create_agent("claude-sonnet-4-6", tools)

result = await agent.ainvoke(
    {"messages": [{"role": "user", "content": "Get data from the server"}]}
)

# Extract structured content from tool messages
for message in result["messages"]:
    if isinstance(message, ToolMessage) and message.artifact:
        structured_content = message.artifact["structured_content"]

```

如果您希望结构化内容能在对话历史记录中显示出来（即能被模型所看到），您可以使用拦截器来自动将结构化内容添加到工具结果中：

```html

import json

from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.interceptors import MCPToolCallRequest
from mcp.types import TextContent

async def append_structured_content(request: MCPToolCallRequest, handler):
    """Append structured content from artifact to tool message."""
    result = await handler(request)
    if result.structuredContent:
        result.content += [
            TextContent(type="text", text=json.dumps(result.structuredContent)),
        ]
    return result

client = MultiServerMCPClient({...}, tool_interceptors=[append_structured_content])

```

- resources

资源使得 MCP 服务器能够公开数据（例如文件、数据库记录或 API 响应），这些数据可供客户端读取。LangChain 将 MCP 资源转换为 Blob 对象，这些对象为处理文本和二进制内容提供了统一的接口。

```html

from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient({...})

# Load all resources from a server
blobs = await client.get_resources("server_name")

# Or load specific resources by URI
blobs = await client.get_resources("server_name", uris=["file:///path/to/file.txt"])

for blob in blobs:
    print(f"URI: {blob.metadata['uri']}, MIME type: {blob.mimetype}")
    print(blob.as_string())  # For text content

```

- prompts

提示词 允许 MCP 服务器公开可重复使用的提示模板，这些模板可被客户端获取并使用。LangChain 将 MCP 提示转换为消息，使其易于融入基于聊天的工作流程中。

```html

from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient({...})

# Load a prompt by name
messages = await client.get_prompt("server_name", "summarize")

# Load a prompt with arguments
messages = await client.get_prompt(
    "server_name",
    "code_review",
    arguments={"language": "python", "focus": "security"}
)

# Use the messages in your workflow
for message in messages:
    print(f"{message.type}: {message.content}")

```

MCP 服务器以独立进程的形式运行——它们无法获取诸如存储、上下文或代理状态等 LangGraph 运行时信息。拦截器弥补了这一差距，使您在执行 MCP 工具时能够访问此运行时上下文。
拦截器还提供了类似于中间件的对工具调用的控制：您可以修改请求、实现重试、动态添加标头或完全中断执行。

#### Human-in-the-loop （HITL）

人类介入的（approve、edit、reject、respond）

#### Multi-agent

多智能体系统协调专门的组件来处理复杂的流程。然而，并非每个复杂任务都需要这种做法——一个拥有合适（有时是动态的）工具和提示的单一智能体往往也能取得类似的结果。

当开发人员说他们需要“多代理”时，他们通常是在寻求以下一种或多种能力：

- 上下文管理：提供专门的知识，同时不使模型的上下文窗口过载。如果上下文是无限的且延迟为零，您可以将所有知识都放入一个提示中——但由于实际情况并非如此，您需要模式来有选择地呈现相关信息。
- 分布式开发：允许不同的团队独立开发和维护能力，并将它们组合成一个具有明确边界的更大系统。
- 并行化：为子任务生成专门的工作者，并同时执行它们以获得更快的结果。

当单个代理拥有的工具过多，难以决定使用哪一个时；当任务需要具备广泛背景知识的专业知识（长提示和特定领域的工具）时；或者当您需要强制执行顺序约束，只有在满足某些条件后才能解锁某些功能时，多代理模式就显得特别有价值。

构建多代理的几种方式：

| pattern | 工作方式 |
| :----: | :----:  |
| Subagents | A main agent coordinates subagents as tools. All routing passes through the main agent, which decides when and how to invoke each subagent. |
| Handoffs | Behavior changes dynamically based on state. Tool calls update a state variable that triggers routing or configuration changes, switching agents or adjusting the current agent’s tools and prompt. |
| Skills | Specialized prompts and knowledge loaded on-demand. A single agent stays in control while loading context from skills as needed. |
| Router | A routing step classifies input and directs it to one or more specialized agents. Results are synthesized into a combined response. |
| Custom workflow | Build bespoke execution flows with LangGraph, mixing deterministic logic and agentic behavior. Embed other patterns as nodes in your workflow. |

看下面的问题来选择合适的pattern

- 分布式开发：不同的团队能否独立维护各个组件？
- 并行化：多个代理能否同时执行？
- 多跳：这种模式是否支持依次调用多个子代理？
- 直接用户交互：子代理能否直接与用户进行对话？

不同的模式具有不同的性能特点。了解这些权衡因素有助于您根据延迟和成本要求选择合适的模式。

关键指标：

- 模型调用次数：语言模型的调用次数。调用次数越多，延迟就越高（尤其是如果是顺序调用）且每次请求的 API 成本也越高。
- 处理的token数：所有调用中使用的总上下文窗口数量。处理的令牌越多，成本就越高，还可能存在上下文限制的情况。

以买coffe这件去看，有一个专门的 coffee agent/skill 能够调用buy_coffee的tool

subagent要call 4次model （最终的结果要返回给main agent）

而handoffs/skills/router 要call 3次model

```html

Subagents 4次 主智能体路由 -> 子智能体决策 -> 执行工具 -> 主智能体总结

Skills 3次 识别并加载 Skill -> 调用工具 -> 总结回复

# 各阶段call model的作用

# 第 1 次调用 (Main Agent 决策)：主智能体接收到用户指令（如“买咖啡”），它需要先通过一次模型调用来判断：我现有的工具里，哪个子智能体最适合处理这个任务？

# 第 2 次调用 (Subagent 执行)：被选中的子智能体启动。由于子智能体通常有自己的独立 Prompt 和工具集，它需要进行一次模型调用来解析主智能体传来的参数并执行具体动作（如调用 buy_coffee 工具）。

# 第 3 次调用（子智能体内部总结）：子智能体拿到工具返回的原始数据后，再次调用模型。可能是 把枯燥的工具运行结果（比如 {"status": 200, "msg": "ok"}）转化成一段符合自己人设的、有意义的自然语言（比如“报告，咖啡已经为您买好，正在配送”），然后再把这段话扔回给主智能体。

# 第 4 次调用 (Main Agent 总结)：执行结果必须返回给主智能体，由主智能体再次调用模型来总结或判断是否需要进行下一步。

```

如果是 Turn 1: “Buy coffee” Turn 2: “Buy coffee again”

连续两次呢，在一个会话中重复了一样的request

Turn2的情况，

subagent要call 4次model （Stateless，每个request都要再来一遍，但是有很好的隔离性）

handoffs 和 skills 2次model (handoffs 是因为 coffee agent仍然在，省一次，skills是因为skill context已经在会话历史中被load进来)

router 3次model （router也是stateless，还是call了3次）

而多领域的情况（需要有不同的领域知识的）

User: “Compare Python, JavaScript, and Rust for web development”

就有python agent，js agent，rust agent了

subagent 5次call， token消耗 9K

handoffs 7+次call，token消耗 14K

skills 3次call，token消耗 15k

router 5次call，token消耗 9K

所以最终看任务类型选择，

| 请求 | 选择 |
| :----: | :----:  |
| 单一请求 | handoffs、skills、router |
| 重复请求 | handoffs、skills |
| 并行执行 | subagents、router |
| 大context的多领域 | subagents、router |
| 简单，专注的任务 | skills |


- subagents

  - 集中控制：所有路由均通过主代理进行处理
  - 无直接用户交互：子代理将结果返回给主代理，而非用户（不过您可以在子代理内部使用中断功能以实现用户交互）
  - 通过工具调用子代理：通过工具来调用子代理
  - 并行执行：主代理可在单次操作中调用多个子代理

当您拥有多个不同的领域（例如日历、电子邮件、客户关系管理、数据库）时，或者子代理无需直接与用户进行交流，或者您希望实现集中式的流程控制时，应采用子代理模式。对于仅涉及少量工具的简单情况，可以使用单个代理。

```html

from langchain.tools import tool
from langchain.agents import create_agent

# Create a subagent
subagent = create_agent(model="google_genai:gemini-3.1-pro-preview", tools=[...])

# Wrap it as a tool
@tool("research", description="Research a topic and return findings")
def call_research_agent(query: str):
    result = subagent.invoke({"messages": [{"role": "user", "content": query}]})
    return result["messages"][-1].content

# Main agent with subagent as a tool
main_agent = create_agent(model="google_genai:gemini-3.1-pro-preview", tools=[call_research_agent])

```

subagent要注意一下
  - sync or async
  - tool pattern
  - subagent spec
  - subagent inputs
  - subagent outputs

- handoffs

  - 状态驱动行为：行为依据状态变量（例如当前步骤或活跃代理）而变化
  - 基于工具的转换：工具会更新状态变量以在不同状态之间转换
  - 直接用户交互：每个状态的配置直接处理用户消息
  - 持久状态：状态在对话轮次之间保持不变

当您需要强制执行顺序约束（只有在满足前提条件后才能解锁功能）、代理需要在不同状态间直接与用户进行交流，或者您正在构建多阶段对话流程时，应采用“交接”模式。此模式在客户支持场景中尤为有用，例如在处理退款之前需要按照特定顺序收集保修编号等信息。

```html

from langchain.tools import tool
from langchain.messages import ToolMessage
from langgraph.types import Command

@tool
def transfer_to_specialist(runtime) -> Command:
    """Transfer to the specialist agent."""
    return Command(
        update={
            "messages": [
                ToolMessage(
                    content="Transferred to specialist",
                    tool_call_id=runtime.tool_call_id  
                )
            ],
            "current_step": "specialist"  # Triggers behavior change
        }
    )

```

具体实现方法

```html

# 1 Single agent with middleware

from langchain.tools import ToolRuntime, tool
from langchain.messages import ToolMessage
from langgraph.types import Command

@tool
def record_warranty_status(
    status: str,
    runtime: ToolRuntime[None, SupportState]
) -> Command:
    """Record warranty status and transition to next step."""
    return Command(
        update={
            "messages": [
                ToolMessage(
                    content=f"Warranty status recorded: {status}",
                    tool_call_id=runtime.tool_call_id
                )
            ],
            "warranty_status": status,
            "current_step": "specialist"  # Update state to trigger transition
        }
    )

# 2 Multiple agent subgraphs

from langchain.messages import AIMessage, ToolMessage
from langchain.tools import tool, ToolRuntime
from langgraph.types import Command

@tool
def transfer_to_sales(
    runtime: ToolRuntime,
) -> Command:
    """Transfer to the sales agent."""
    last_ai_message = next(
        msg for msg in reversed(runtime.state["messages"]) if isinstance(msg, AIMessage)
    )
    transfer_message = ToolMessage(
        content="Transferred to sales agent",
        tool_call_id=runtime.tool_call_id,
    )
    return Command(
        goto="sales_agent",
        update={
            "active_agent": "sales_agent",
            "messages": [last_ai_message, transfer_message],
        },
        graph=Command.PARENT
    )

```

- skills

  - 引导式专业化：技能主要由特定的提示来定义
  - 逐步披露：技能会根据情境或用户需求而逐步显现
  - 团队分配：不同的团队可以独立开发和维护技能
  - 轻量级组合：技能比完整的子代理更简单
  - 参考意识：技能可以引用脚本、模板和其他资源

```html

from langchain.tools import tool
from langchain.agents import create_agent

@tool
def load_skill(skill_name: str) -> str:
    """Load a specialized skill prompt.

    Available skills:
    - write_sql: SQL query writing expert
    - review_legal_doc: Legal document reviewer

    Returns the skill's prompt and context.
    """
    # Load skill content from file/database
    ...

agent = create_agent(
    model="gpt-5.4",
    tools=[load_skill],
    system_prompt=(
        "You are a helpful assistant. "
        "You have access to two skills: "
        "write_sql and review_legal_doc. "
        "Use load_skill to access them."
    ),
)

```

- router

  - 路由器对查询进行分解
  - 零个或多个专门的代理会同时被调用
  - 结果被整合成一个连贯的响应

当您存在不同的垂直领域（各自具有独立知识领域且每个领域都需要独立的代理）、需要并行查询多个来源，并希望将结果整合为一个综合响应时，应使用路由器模式。

```html

from typing import TypedDict
from langgraph.types import Send

class ClassificationResult(TypedDict):
    query: str
    agent: str

def classify_query(query: str) -> list[ClassificationResult]:
    """Use LLM to classify query and determine which agents to invoke."""
    # Classification logic here
    ...

def route_query(state: State):
    """Route to relevant agents based on query classification."""
    classifications = classify_query(state["query"])

    # Fan out to selected agents in parallel
    return [
        Send(c["agent"], {"query": c["query"]})
        for c in classifications
    ]

```

#### Retrieval

From retrieval to RAG

检索功能使大型语言模型能够在运行时获取相关背景信息。但在大多数实际应用中，它们会更进一步，将检索与生成相结合，从而生成基于实际情境、具有情境感知能力的答案。

这就是“检索增强生成”（RAG）的核心理念。检索流程成为了一个更广泛系统的基础，该系统将搜索与生成相结合。

retrieval pipeline（一个基本流程）:

sources -> document loaders -> documents -> split into chunks -> turn into embedding ）

                                                      user query -> query embedding  ）上下都进入vector store -> retriever -> LLM uses retrieved info -> answer


以多种方式实现，

| 实现方式 | 描述 |
| :----: | :----:  |
| 2-Step RAG | Retrieval always happens before generation. Simple and predictable |
| Agentic RAG | An LLM-powered agent decides when and how to retrieve during reasoning |
| Hybrid | Combines characteristics of both approaches with validation steps |

Latency: Latency is generally more predictable in 2-Step RAG, as the maximum number of LLM calls is known and capped. This predictability assumes that LLM inference time is the dominant factor. However, real-world latency may also be affected by the performance of retrieval steps—such as API response times, network delays, or database queries—which can vary based on the tools and infrastructure in use.

- 2 steps

- Agentic RAG

代理检索增强生成（RAG）技术将检索增强生成的优势与基于代理的推理相结合。在回答问题时，它不再先检索文档，而是由一个代理（由大型语言模型驱动）逐步进行推理，并在交互过程中决定何时以及如何检索信息。

- Hybrid RAG

混合式 RAG 结合了两步式和代理式 RAG 的特点。它引入了诸如查询预处理、检索验证和后生成检查等中间步骤。这些系统比固定的流程更具灵活性，同时又能保持一定的执行控制权。

#### Long-term memory

长期记忆使您的代理能够在不同的对话和会话中存储和检索信息。与短期记忆不同，短期记忆仅限于单个线程，而长期记忆则跨越多个线程存在，并且可以在任何时候被调用。

长期记忆是基于 LangGraph 存储构建的，该存储将数据以 JSON 文档的形式保存，这些文档按照命名空间和键进行组织。

```html

from langchain.agents import create_agent
from langchain_core.runnables import Runnable
from langgraph.store.memory import InMemoryStore

# InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production use.
store = InMemoryStore()

agent: Runnable = create_agent(
    "claude-sonnet-4-6",
    tools=[],
    store=store,
)

```

LangGraph 将长期记忆以 JSON 文档的形式存储在存储库中。
每个记忆都按照自定义命名空间（类似于文件夹）和独特的键（类似于文件名）进行组织。命名空间通常包含用户或组织 ID 或其他标签，这使得信息的组织更加方便。
这种结构实现了记忆的分层组织。然后，通过内容过滤器支持跨命名空间的搜索

```html

from collections.abc import Sequence

from langgraph.store.base import IndexConfig
from langgraph.store.memory import InMemoryStore


def embed(texts: Sequence[str]) -> list[list[float]]:
    # Replace with an actual embedding function or LangChain embeddings object
    return [[1.0, 2.0] for _ in texts]


# InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production use.
store = InMemoryStore(index=IndexConfig(embed=embed, dims=2))
user_id = "my-user"
application_context = "chitchat"
namespace = (user_id, application_context)
store.put(
    namespace,
    "a-memory",
    {
        "rules": [
            "User likes short, direct language",
            "User only speaks English & python",
        ],
        "my-key": "my-value",
    },
)
# get the "memory" by ID
item = store.get(namespace, "a-memory")
# search for "memories" within this namespace, filtering on content equivalence, sorted by vector similarity
items = store.search(
    namespace, filter={"my-key": "my-value"}, query="language preferences"
)

```

在工具中读取long-term memory

```html

from dataclasses import dataclass

from langchain.agents import create_agent
from langchain.tools import ToolRuntime, tool
from langchain_core.runnables import Runnable
from langgraph.store.memory import InMemoryStore


@dataclass
class Context:
    user_id: str


# InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production.
store = InMemoryStore()

# Write sample data to the store using the put method
store.put(
    (
        "users",
    ),  # Namespace to group related data together (users namespace for user data)
    "user_123",  # Key within the namespace (user ID as key)
    {
        "name": "John Smith",
        "language": "English",
    },  # Data to store for the given user
)


@tool
def get_user_info(runtime: ToolRuntime[Context]) -> str:
    """Look up user info."""
    # Access the store - same as that provided to `create_agent`
    assert runtime.store is not None
    user_id = runtime.context.user_id
    # Retrieve data from store - returns StoreValue object with value and metadata
    user_info = runtime.store.get(("users",), user_id)
    return str(user_info.value) if user_info else "Unknown user"


agent: Runnable = create_agent(
    model="google_genai:gemini-3.1-pro-preview",
    tools=[get_user_info],
    # Pass store to agent - enables agent to access store when running tools
    store=store,
    context_schema=Context,
)

# Run the agent
agent.invoke(
    {"messages": [{"role": "user", "content": "look up user information"}]},
    context=Context(user_id="user_123"),
)

```

往工具中写入long-term memory

```html

from dataclasses import dataclass

from langchain.agents import create_agent
from langchain.tools import ToolRuntime, tool
from langchain_core.runnables import Runnable
from langgraph.store.memory import InMemoryStore
from typing_extensions import TypedDict

# InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production.
store = InMemoryStore()


@dataclass
class Context:
    user_id: str


# TypedDict defines the structure of user information for the LLM
class UserInfo(TypedDict):
    name: str


# Tool that allows agent to update user information (useful for chat applications)
@tool
def save_user_info(user_info: UserInfo, runtime: ToolRuntime[Context]) -> str:
    """Save user info."""
    # Access the store - same as that provided to `create_agent`
    assert runtime.store is not None
    store = runtime.store
    user_id = runtime.context.user_id
    # Store data in the store (namespace, key, data)
    store.put(("users",), user_id, dict(user_info))
    return "Successfully saved user info."


agent: Runnable = create_agent(
    model="google_genai:gemini-3.1-pro-preview",
    tools=[save_user_info],
    store=store,
    context_schema=Context,
)

# Run the agent
agent.invoke(
    {"messages": [{"role": "user", "content": "My name is John Smith"}]},
    # user_id passed in context to identify whose information is being updated
    context=Context(user_id="user_123"),
)

# You can access the store directly to get the value
item = store.get(("users",), "user_123")

```

#### langgraph

LangGraph 架构非常低级，完全专注于智能代理的编排。在使用 LangGraph 之前，我们建议您先熟悉用于构建智能代理的一些组件，首先从模型和工具开始。
在文档中，我们通常会使用 LangChain 组件来集成模型和工具，但您无需使用 LangChain 即可使用 LangGraph。如果您刚刚开始接触智能代理或希望使用更高级别的抽象，我们建议您使用提供常见 LLM 和工具调用循环预构建架构的 LangChain 的智能代理。LangGraph 专注于与代理编排相关的关键底层功能：持久执行、流处理、人机协作等。

```html

from langgraph.graph import StateGraph, MessagesState, START, END

def mock_llm(state: MessagesState):
    return {"messages": [{"role": "ai", "content": "hello world"}]}

graph = StateGraph(MessagesState)
graph.add_node(mock_llm)
graph.add_edge(START, "mock_llm")
graph.add_edge("mock_llm", END)
graph = graph.compile()

graph.invoke({"messages": [{"role": "user", "content": "hi!"}]})

```

LangGraph 为任何长期运行且具有状态属性的工作流程或代理提供了底层支持性基础设施。LangGraph 不会对提示或架构进行抽象处理，并提供了以下核心优势：

- 持久执行：构建能够经受失败考验并能长时间运行的代理，且能够在中断处继续从上次停止的地方恢复运行。
- 人工干预：通过随时检查和修改代理状态来融入人工监督。
- 全面记忆：创建具有短期工作记忆（用于持续推理）和跨会话长期记忆的有状态代理。
- 使用 LangSmith 进行调试：借助能够追踪执行路径、捕获状态转换并提供详细运行时指标的可视化工具，深入了解复杂的代理行为，从而获得深度可见性。
- 适合生产的部署：借助专为处理有状态、长期运行的工作流程的独特挑战而设计的可扩展基础设施，自信地部署复杂的代理系统。


虽然 LangGraph 可以独立使用，但它也能与任何 LangChain 产品完美集成，为开发者提供一套完整的构建智能代理的工具。为了优化您的 LLM 应用开发，可以将 LangGraph 与以下内容搭配使用：

langsmith 和 langchain

Use the Graph API if you prefer to define your agent as a graph of nodes and edges.

Use the Functional API if you prefer to define your agent as a single function.

```html

# 1、Use the Graph API

from langchain.tools import tool
from langchain.chat_models import init_chat_model


model = init_chat_model(
    "claude-sonnet-4-6",
    temperature=0
)


# Define tools
@tool
def multiply(a: int, b: int) -> int:
    """Multiply `a` and `b`.

    Args:
        a: First int
        b: Second int
    """
    return a * b


@tool
def add(a: int, b: int) -> int:
    """Adds `a` and `b`.

    Args:
        a: First int
        b: Second int
    """
    return a + b


@tool
def divide(a: int, b: int) -> float:
    """Divide `a` and `b`.

    Args:
        a: First int
        b: Second int
    """
    return a / b


# Augment the LLM with tools
tools = [add, multiply, divide]
tools_by_name = {tool.name: tool for tool in tools}
model_with_tools = model.bind_tools(tools)

# 2、Use the Functional API

from langchain.tools import tool
from langchain.chat_models import init_chat_model


model = init_chat_model(
    "claude-sonnet-4-6",
    temperature=0
)


# Define tools
@tool
def multiply(a: int, b: int) -> int:
    """Multiply `a` and `b`.

    Args:
        a: First int
        b: Second int
    """
    return a * b


@tool
def add(a: int, b: int) -> int:
    """Adds `a` and `b`.

    Args:
        a: First int
        b: Second int
    """
    return a + b


@tool
def divide(a: int, b: int) -> float:
    """Divide `a` and `b`.

    Args:
        a: First int
        b: Second int
    """
    return a / b


# Augment the LLM with tools
tools = [add, multiply, divide]
tools_by_name = {tool.name: tool for tool in tools}
model_with_tools = model.bind_tools(tools)

from langgraph.graph import add_messages
from langchain.messages import (
    SystemMessage,
    HumanMessage,
    ToolCall,
)
from langchain_core.messages import BaseMessage
from langgraph.func import entrypoint, task

```

当您使用 LangGraph 构建一个代理时，首先会将其分解为一系列独立的步骤，这些步骤被称为节点。然后，您需要描述每个节点的不同决策和转换过程。最后，通过每个节点都可以读取和写入的共享状态将节点连接起来。

假设一个 产品需要邮件代理，具体的需求如下

```html

该代理人应当：
- 阅读新收到的客户邮件
- 按紧急程度和主题对邮件进行分类
- 查找相关文档以解答问题
- 起草恰当的回复
- 将复杂问题转交给人工客服处理
- 根据需要安排后续跟进事宜

需要处理的示例场景：
1. 简单的产品咨询：“我该如何重置密码？”
2. 错误报告：“当我选择 PDF 格式时，导出功能就会崩溃。”
3. 紧急账单问题：“我的订阅费用被重复收取了两次！”
4. 功能请求：“能否为移动应用程序添加暗黑模式？”
5. 复杂的技术问题：“我们的 API 集成会间歇性地出现 504 错误。”


```

做一个这样的代理，进行5步走

1、切分好流程，node 并连接起来

- Read Email: Extract and parse the email content
- Classify Intent: Use an LLM to categorize urgency and topic, then route to appropriate action
- Doc Search: Query your knowledge base for relevant information
- Bug Track: Create or update issue in tracking system
- Draft Reply: Generate an appropriate response
- Human Review: Escalate to human agent for approval or handling
- Send Reply: Dispatch the email response

2、明确每个步骤需要完成的事情

- LLM Step，Classify Intent、Draft Reply
- Data Step，Doc Search、customer history lookup
- Action Step，Send Reply、Bug Track
- User Input Step，human review node

3、设计状态

“状态”是整个代理系统中所有节点均可访问的共享内存。可以将其想象成您的代理所使用的用于记录其在处理过程中所学到的一切和所做出的所有决策的笔记本。

对于 邮件 agent，需要追踪

- 原始电子邮件及发件人信息（之后无法重新获取）
- 分类结果（多个后续节点所需）
- 搜索结果及客户数据（重新获取成本高昂）
- 草稿回复（需要在审核过程中保持不变）
- 执行元数据（用于调试和恢复）

Keep state raw, format prompts on-demand

- 不同的节点可以根据自身需求对相同的数据进行不同的格式化处理
- 您无需修改状态架构即可更改提示模板
- 调试过程更加清晰——您能够确切地看到每个节点接收到的数据内容
- 您的代理可以不断发展完善，而不会破坏现有的状态结构

Notice that the state contains only raw data—no prompt templates, no formatted strings, no instructions. The classification output is stored as a single dictionary, straight from the LLM.

4、建立Nodes

现在我们将每个步骤都设计成一个函数。在 LangGraph 中，一个节点实际上就是一个 Python 函数，它会接收当前状态并返回对其的更新结果。

nodes 获取 state，工作，然后 返回 updates


5、把它们连接在一起

langgraph的一些总结的话，用langgraph去做思考

- 分解为独立的步骤
- state是共享内存（raw data）
- nodes是函数
- errors是流程的一部分
- human input是第一位（interrupt()会暂停执行）
- 图形结构自然形成

工作流程和代理模式。

- 工作流程具有预先设定的代码路径，并按照特定顺序运行。
  - prompt chaining，每次都处理LLm的输出
  - parallelization，并行的任务
    - START -> LLM1 START -> LLM2 START -> LLM3 LLM1 -> aggregator LLM2 -> aggregator LLM3 -> aggregator aggregator ->END
  - orchestrator-worker
    - Send API
  - evaluator-optimizer
  - routing
    - 做一个 route_decision

- 代理则是动态的，它们能够自行定义其流程和工具的使用方式。
  - tool_node 和 llm_call

工具调用，结构输出，短期记忆等等都是与模型结合的东西

结构输出用 pydantic

#### 问题

1、思考模式需要特殊处理

```html

raise self._make_status_error_from_response(err.response) from None
openai.BadRequestError: Error code: 400 - {'error': {'message': 'The `reasoning_content` in the thinking mode must be passed back to the API.', 'type': 'invalid_request_error', 'param': None, 'code': 'invalid_request_error'}}
During task with name 'model' and id '877f85b1-6475-8b64-4fa7-ff418bd1f806'

```

思考模式返回的 reasoning_content 也还是要再传回api的，可以将model换成deepseek-chat，而不是deepseek-reasoner

这个回传reasoning_content是在进行了工具调用后，官方称reasoning_content要再回传给api，否则API返回400错误

2、与deepseek集成的问题1

```html

# 而 DeepSeek 的 API 对 response_format 的要求非常挑剔：如果开启了 JSON Mode，提示词（Prompt）中必须包含 "json" 这个单词

structured_llm = llm.with_structured_output(EmailClassification)

# Format the prompt on-demand, not stored in state
classification_prompt = f"""
  Analyze this customer email and classify it:

  Email: {state['email_content']}
  From: {state['sender_email']}

  Provide classification including intent, urgency, topic, and summary.
  """

# Get structured response directly as dict
# LLm的调用
classification = structured_llm.invoke(classification_prompt)

换成

# 1. 定义解析器
parser = JsonOutputParser(pydantic_object=EmailClassification)

# 2. 在 Prompt 中明确注入 JSON 格式要求（DeepSeek 必须看到 "json" 单词）
classification_prompt = f"""
    Analyze this customer email and classify it.

    Email: {state['email_content']}
    From: {state['sender_email']}

    {parser.get_format_instructions()}
    """
# parser.get_format_instructions() 会自动生成包含 "Respond with a JSON object..." 的指令

# 3. 直接调用 llm (注意不是 structured_llm)
response = llm.invoke(classification_prompt)

# 4. 解析结果
classification = parser.parse(response.content)

```

3、与deepseek集成的问题2

```html

class EmailClassification(TypedDict):
    intent: Literal["question", "bug", "billing", "feature", "complex"]
    urgency: Literal["low", "medium", "high", "critical"]
    topic: str
    summary: str

TypedDict 改成 pydantic

class EmailClassification(BaseModel):
    intent: Literal["question", "bug", "billing", "feature", "complex"] = Field(description="邮件的意图分类")
    urgency: Literal["low", "medium", "high", "critical"] = Field(description="紧急程度")
    topic: str = Field(description="邮件讨论的核心话题")
    summary: str = Field(description="邮件内容的简短摘要")

```
