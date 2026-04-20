---
layout: post
title: agent/workflow
tags: [agent, workflow]
author-id: zqmalyssa
---

agent 和 workflow相关

#### 基本概念

workflow（dify、llama_index的workflow）像是一个事先规划好的剧本，设计好每一步要做的，比如客户工单系统，用户提交工单，自动分类派发给对应的部门，生成回复，然后发给用户。

agent更像一个有自主决策能力的助手，你给它一个目标，比如帮我分析这个市场报告，它会自己去思考，可能先去搜索相关资料，发现信息不够，再调用其他分析工具，而是根据实际情况，动态决策的。

1是灵活性，2适用场景，比如审批流程，数据管道这些，它稳定可控好调试，agent更适合探索任务，处理复杂的客户问题 3是成本和风险，agent有随机性，可能会有预期外的行为，

两者可以相结合，核心链路不变，在某些环节嵌入agent

llama_index中，agent基础用法是在功能性的，可以结合function call的模型，完成指定的具体任务

llama_index中，workflow是基于事件（event）的，分步骤（step）的方式，用于控制程序的执行流程，应用程序被划分为若干的步骤（Step），这些步骤由事件触发，并且它们自身还会发出事件

#### 起源

>I think AI agent workflows will drive massive AI progress this year — perhaps even more than the next generation of foundation models. This is an important trend, and I urge everyone who works in AI to pay attention to it. -- Andrew.Ng

吴恩达的一句话，及团队的工作，Human Eval是论文《Evaluating Large Language Models Trained on Code》中提到一个代码评测标准。GPT-3.5和GPT-4在zero-shot中分别获得了48%和67%左右的正确率，而通过使用Agent workflow，有些Agent在GPT-3.5上甚至达到了95%的准确率。可见通过迭代Agent workflow实现的性能提升远远超过了从GPT-3.5到GPT-4的改进。这一发现强调了Agent workflow在提高人工智能性能方面的重要性。甚至能让GPT-5提前到来。吴教授总结了当前的业界研究，提出了四种工作流设计模式。

- 反思 Reflection: LLM反思自己的工作并提出改进的方法。
- 工具使用 Tool use: 给LLM提供工具，如网络搜索、代码执行或任何其他功能，以帮助它收集信息、采取行动或处理数据。
- 规划 Planning: LLM制定并执行一个多步骤计划来实现目标（例如，为文章编写大纲，然后进行在线研究，然后写草稿，等等）。
- 多Agent协作 Multi-agent collaboration: 多个Agent协作，分配任务并讨论辩论想法以此提出比单个Agent更好的解决方案。

可以参考Langchain、llama_index框架

- Reflection工作流，self-refine（自我精炼），critic（与外部工具交互），reflexion（反思），一些工作诸如ReAct、SayCan、Toolformer、HuggingGPT以及WebGPT证明了使用基于LLM构建的自动化决策Agent是可行的，但是这些方法大多依赖于在当前情景中的例子(in-context example)来指导LLM生成内容，因为用梯度下降的强化学习需要很多计算和时间。简单说，就是他们依赖的只是短期记忆。所以作者提出一种新的叫做Reflexion的框架，它通过使用语言强化学习（verb reinforcement）来帮助Agent从之前的失败中学习。Reflexion会把环境的二进制或量化的反馈转换成文字描述，作为下次迭代中的额外信息。这种自我反思式的反馈就像是个语意上的梯度信号来提供给Agent具体的优化方向，让Agent知道怎样改正错误，这样就能更好地完成任务了。就像人类通过反复练习和反思来快速掌握复杂技能一样

从上图看出Reflexion（上面第3个）框架利用三个不同的模型：执行者（Actor）、评估者（Evaluator）和自我反思模型（Self-Reflection)。

- 执行者Actor基于状态观察生成文本和动作；
- 评估者对执行者产生的输出计算奖励分数；
- 自我反思模型生成口头自我反思提示以协助执行者自我改进。
- 该过程使用短期和长期记忆，其中轨迹历史作为短期记忆，而自我反思模型的输出存储在长期记忆中。

- 工具使用，MM-ReAct框架，

#### llama_index

LlamaIndex 是一个用于构建 LLM（大型语言模型）应用的数据框架，其核心目标是：让模型能“读懂”你的数据（文档、PDF、数据库、API 等）并基于这些私有数据生成回答 /执行任务。它最早被称为 “GPT Index”，现在是一个通用、可扩展的数据增强 + RAG（检索增强生成）框架。

- 数据摄取（data ingestion），通过reader/Connector模块，解析数据为统一内部表示
  - 文本文档、PDF、Word
  - 文件夹/本地数据
  - 数据库（SQL/NoSQL）
  - Web API/自定义数据源

- 数据结构化（indices），不同index支持不同查询模式，例如one-shot Q&A，聊天带记忆
  - VectorStoreIndex，文本嵌入 + 向量检索，语义搜索、RAG
  - SummaryIndex，生成每个片段的摘要，索引较小或需要概览
  - TreeIndex，逐层层级检索，大规模文档检索优化
  - KeywordTableIndex，关键词检索，简单检索、关键词匹配
  - KnowledgeGraphIndex，图结构 + 实体关系，需要关系推理的任务

- 检索 + 回答引擎（retrieval / query engine），内部自动拼装prompt
  - as_query_engine() 直接回答
  - as_retriever() 只做检索，无生成
  - as_chat_engine() 多轮对话模式

- Agent和Workflow
  - FunctionAgent、ReActAgent、CodeActAgent，都实现agent.run()、自定义agent
  - WorkflowAgent，是 orchestrator：它负责组合 agents、步骤或事件，使系统能应对复杂任务或多 agent 协作
  - MultiAgent的形态，AgentWorkflow (built-in，一系列的agent，放到AgentWorkflow)、Orchestrator pattern (built-in，总的agent，协调别的agent工作)、Custom planner (DIY，写提示词，XML或者JSON，规划顺序，直接通过代码调用)

- 集成生态（intergrations）
  - 核心包core
  - 扩展包 llama-index-embeddings-huggingface, llama-index-llms-ollama

- 记忆，memory，short和long的，memory根据insert_method，是system还是user，决定memory是插入到哪里
  - short，chat_history_token_ratio=0.7,token_flush_size=3000
  - long，StaticMemoryBlock，FactExtractionMemoryBlock，VectorMemoryBlock
  - Customizing Memory Blocks（自定义内存块），自定义一个memroy，用来存储
  - remote memory。存放数据源

简单的agent代码

```html

import asyncio
from llama_index.core import VectorStoreIndex, Settings
from llama_index.llms.openai import OpenAI
from llama_index.llms.llama_cpp import LlamaCPP
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.readers.file import PyMuPDFReader
from llama_index.embeddings.huggingface import HuggingFaceEmbedding
from llama_index.core.agent.workflow import FunctionAgent
from llama_index.llms.ollama import Ollama

# Define a simple calculator tool
def multiply(a: float, b: float) -> float:
    """Useful for multiplying two numbers."""
    return a * b

# Create an agent workflow with our calculator tool
agent = FunctionAgent(
    tools=[multiply],
    # model '123-test' not found，报错还是不一样的
    # ollama._types.ResponseError: registry.ollama.ai/library/local-llama-8B-instruct:latest does not support tools (status code: 400)
    # llama3.1 是可以的
    llm=Ollama(model="llama3.1",
               # base_url="http://127.0.0.1:11434",
               request_timeout=360.0),
    system_prompt="You are a helpful assistant that can multiply two numbers.",
)

async def main():
    # Run the agent
    response = await agent.run("What is 1234 * 4567?")
    print(str(response))

# Run the agent
if __name__ == "__main__":
    asyncio.run(main())

# 发生了什么
# Agent 被赋予了一个问题：1234 * 4567 是多少？
# 在底层，这个问题以及工具的 schema（名称、docstring 和参数）被传递给了 LLM
# Agent 选择了 multiply 工具并将参数写入了工具
# Agent 收到了工具的结果并将其插入最终响应中


```

agent with rag

```html

from llama_index.core import VectorStoreIndex, SimpleDirectoryReader, Settings
from llama_index.core.agent.workflow import AgentWorkflow
from llama_index.llms.ollama import Ollama
from llama_index.embeddings.huggingface import HuggingFaceEmbedding
import asyncio
import os

# Settings control global defaults
# Settings.embed_model = HuggingFaceEmbedding(model_name="BAAI/bge-base-en-v1.5")
Settings.embed_model = HuggingFaceEmbedding(model_name="/Users/zqmalyssa/Model/embedding/bge-base-en-v1.5")
Settings.llm = Ollama(model="llama3.1", request_timeout=360.0)

# Create a RAG tool using LlamaIndex
documents = SimpleDirectoryReader("data").load_data()
index = VectorStoreIndex.from_documents(
    documents,
    # we can optionally override the embed_model here
    # embed_model=Settings.embed_model,
)
query_engine = index.as_query_engine(
    # we can optionally override the llm here
    # llm=Settings.llm,
)


def multiply(a: float, b: float) -> float:
    """Useful for multiplying two numbers."""
    return a * b


async def search_documents(query: str) -> str:
    """Useful for answering natural language questions about an personal essay written by Paul Graham."""
    response = await query_engine.aquery(query)
    return str(response)


# Create an enhanced workflow with both tools
agent = AgentWorkflow.from_tools_or_functions(
    [multiply, search_documents],
    llm=Settings.llm,
    system_prompt="""You are a helpful assistant that can perform calculations
    and search through documents to answer questions.""",
)


# Now we can ask questions about the documents or do calculations
async def main():
    response = await agent.run(
        "What did the author do in college? \nAlso, what's 7 * 8?"
    )
    print(response)


# Run the agent
if __name__ == "__main__":
    asyncio.run(main())

# 这个算是一个简答的workflow了，先检索回答了文本问题，再用tool回答了数学问题

The author's college activities are writing and program.

Now, let me try to calculate 7 * 8:
``
{"name": "multiply", "parameters": {"a": 7,"b": 8}}
``

The result of the multiplication is: 56

```

多agnet，使用agentworkflow来协调

```html

from llama_index.llms.openai import OpenAI
from tavily import AsyncTavilyClient
from llama_index.core.workflow import Context
from llama_index.core.agent.workflow import FunctionAgent, ReActAgent
from llama_index.core.agent.workflow import AgentWorkflow

# 用一下openai的
llm = OpenAI(model="gpt-4o", api_key="sk-...")

# Utilizing the Context class, we can pass state between agents, and each agent will have access to the current state of the system.

async def search_web(query: str) -> str:
    """Useful for using the web to answer questions."""
    client = AsyncTavilyClient(api_key="tvly-...")
    return str(await client.search(query))


async def record_notes(ctx: Context, notes: str, notes_title: str) -> str:
    """Useful for recording notes on a given topic. Your input should be notes with a title to save the notes under."""
    async with ctx.store.edit_state() as ctx_state:
        if "research_notes" not in ctx_state["state"]:
            ctx_state["state"]["research_notes"] = {}
        ctx_state["state"]["research_notes"][notes_title] = notes
    return "Notes recorded."


async def write_report(ctx: Context, report_content: str) -> str:
    """Useful for writing a report on a given topic. Your input should be a markdown formatted report."""
    async with ctx.store.edit_state() as ctx_state:
        ctx_state["state"]["report_content"] = report_content
    return "Report written."


async def review_report(ctx: Context, review: str) -> str:
    """Useful for reviewing a report and providing feedback. Your input should be a review of the report."""
    async with ctx.store.edit_state() as ctx_state:
        ctx_state["state"]["review"] = review
    return "Report reviewed."

# 如果模型支持tool calling，就用functionAgent，不是就用ReActAgent
#

research_agent = FunctionAgent(
    name="ResearchAgent",
    description="Useful for searching the web for information on a given topic and recording notes on the topic.",
    system_prompt=(
        "You are the ResearchAgent that can search the web for information on a given topic and record notes on the topic. "
        "Once notes are recorded and you are satisfied, you should hand off control to the WriteAgent to write a report on the topic. "
        "You should have at least some notes on a topic before handing off control to the WriteAgent."
    ),
    llm=llm,
    tools=[search_web, record_notes],
    can_handoff_to=["WriteAgent"],
)

write_agent = FunctionAgent(
    name="WriteAgent",
    description="Useful for writing a report on a given topic.",
    system_prompt=(
        "You are the WriteAgent that can write a report on a given topic. "
        "Your report should be in a markdown format. The content should be grounded in the research notes. "
        "Once the report is written, you should get feedback at least once from the ReviewAgent."
    ),
    llm=llm,
    tools=[write_report],
    can_handoff_to=["ReviewAgent", "ResearchAgent"],
)

review_agent = FunctionAgent(
    name="ReviewAgent",
    description="Useful for reviewing a report and providing feedback.",
    system_prompt=(
        "You are the ReviewAgent that can review the write report and provide feedback. "
        "Your review should either approve the current report or request changes for the WriteAgent to implement. "
        "If you have feedback that requires changes, you should hand off control to the WriteAgent to implement the changes after submitting the review."
    ),
    llm=llm,
    tools=[review_report],
    can_handoff_to=["WriteAgent"],
)

# agents 有顺序，有root_agent
agent_workflow = AgentWorkflow(
    agents=[research_agent, write_agent, review_agent],
    root_agent=research_agent.name,
    initial_state={
        "research_notes": {},
        "report_content": "Not written yet.",
        "review": "Review required.",
    },
)

from llama_index.core.agent.workflow import (
    AgentInput,
    AgentOutput,
    ToolCall,
    ToolCallResult,
    AgentStream,
)

handler = agent_workflow.run(
    user_msg=(
        "Write me a report on the history of the internet. "
        "Briefly describe the history of the internet, including the development of the internet, the development of the web, "
        "and the development of the internet in the 21st century."
    )
)

current_agent = None
current_tool_calls = ""
async for event in handler.stream_events():
    if (
        hasattr(event, "current_agent_name")
        and event.current_agent_name != current_agent
    ):
        current_agent = event.current_agent_name
        print(f"\n{'='*50}")
        print(f"🤖 Agent: {current_agent}")
        print(f"{'='*50}\n")

    # if isinstance(event, AgentStream):
    #     if event.delta:
    #         print(event.delta, end="", flush=True)
    # elif isinstance(event, AgentInput):
    #     print("📥 Input:", event.input)
    elif isinstance(event, AgentOutput):
        if event.response.content:
            print("📤 Output:", event.response.content)
        if event.tool_calls:
            print(
                "🛠️  Planning to use tools:",
                [call.tool_name for call in event.tool_calls],
            )
    elif isinstance(event, ToolCallResult):
        print(f"🔧 Tool Result ({event.tool_name}):")
        print(f"  Arguments: {event.tool_kwargs}")
        print(f"  Output: {event.tool_output}")
    elif isinstance(event, ToolCall):
        print(f"🔨 Calling Tool: {event.tool_name}")
        print(f"  With arguments: {event.tool_kwargs}")


```

多agnet，使用agent来协调

```html

from llama_index.llms.openai import OpenAI
from tavily import AsyncTavilyClient
from llama_index.core.agent.workflow import FunctionAgent, ReActAgent
from llama_index.core.workflow import Context
import re

# function agent作为总的协调者
# sub-agent可以用不同的LLM
# 用openai作sub-agent 和 orchestrator-agent
sub_agent_llm = OpenAI(model="gpt-4.1-mini", api_key="sk-...")
orchestrator_llm = OpenAI(model="o3-mini", api_key="sk-...")


async def search_web(query: str) -> str:
    """Useful for using the web to answer questions."""
    client = AsyncTavilyClient(api_key="tvly-...")
    return str(await client.search(query))

# 3个sub agent
research_agent = FunctionAgent(
    system_prompt=(
        "You are the ResearchAgent that can search the web for information on a given topic and record notes on the topic. "
        "You should output notes on the topic in a structured format."
    ),
    llm=sub_agent_llm,
    tools=[search_web],
)

write_agent = FunctionAgent(
    system_prompt=(
        "You are the WriteAgent that can write a report on a given topic. "
        "Your report should be in a markdown format. The content should be grounded in the research notes. "
        "Return your markdown report surrounded by <report>...</report> tags."
    ),
    llm=sub_agent_llm,
    tools=[],
)

review_agent = FunctionAgent(
    system_prompt=(
        "You are the ReviewAgent that can review the write report and provide feedback. "
        "Your review should either approve the current report or request changes to be implemented."
    ),
    llm=sub_agent_llm,
    tools=[],
)

# 3个sub agent定义好后，将他们转换成可供顶级agent使用的tools
async def call_research_agent(ctx: Context, prompt: str) -> str:
    """Useful for recording research notes based on a specific prompt."""
    result = await research_agent.run(
        user_msg=f"Write some notes about the following: {prompt}"
    )

    async with ctx.store.edit_state() as ctx_state:
        ctx_state["state"]["research_notes"].append(str(result))

    return str(result)


async def call_write_agent(ctx: Context) -> str:
    """Useful for writing a report based on the research notes or revising the report based on feedback."""
    async with ctx.store.edit_state() as ctx_state:
        notes = ctx_state["state"].get("research_notes", None)
        if not notes:
            return "No research notes to write from."

        user_msg = f"Write a markdown report from the following notes. Be sure to output the report in the following format: <report>...</report>:\n\n"

        # Add the feedback to the user message if it exists
        feedback = ctx_state["state"].get("review", None)
        if feedback:
            user_msg += f"<feedback>{feedback}</feedback>\n\n"

        # Add the research notes to the user message
        notes = "\n\n".join(notes)
        user_msg += f"<research_notes>{notes}</research_notes>\n\n"

        # Run the write agent
        result = await write_agent.run(user_msg=user_msg)
        report = re.search(
            r"<report>(.*)</report>", str(result), re.DOTALL
        ).group(1)
        ctx_state["state"]["report_content"] = str(report)

    return str(report)


async def call_review_agent(ctx: Context) -> str:
    """Useful for reviewing the report and providing feedback."""
    async with ctx.store.edit_state() as ctx_state:
        report = ctx_state["state"].get("report_content", None)
        if not report:
            return "No report content to review."

        result = await review_agent.run(
            user_msg=f"Review the following report: {report}"
        )
        ctx_state["state"]["review"] = result

    return result

# Creating the Top-Level Orchestrator Agent
# 顶级agent，也是一个FunctionAgent
orchestrator = FunctionAgent(
    system_prompt=(
        "You are an expert in the field of report writing. "
        "You are given a user request and a list of tools that can help with the request. "
        "You are to orchestrate the tools to research, write, and review a report on the given topic. "
        "Once the review is positive, you should notify the user that the report is ready to be accessed."
    ),
    llm=orchestrator_llm,
    tools=[
        call_research_agent,
        call_write_agent,
        call_review_agent,
    ],
    initial_state={
        "research_notes": [],
        "report_content": None,
        "review": None,
    },
)

# 跑agent
from llama_index.core.agent.workflow import (
    AgentInput,
    AgentOutput,
    ToolCall,
    ToolCallResult,
    AgentStream,
)
from llama_index.core.workflow import Context

# Create a context for the orchestrator to hold history/state
ctx = Context(orchestrator)


async def run_orchestrator(ctx: Context, user_msg: str):
    handler = orchestrator.run(
        user_msg=user_msg,
        ctx=ctx,
    )

    async for event in handler.stream_events():
        if isinstance(event, AgentStream):
            if event.delta:
                print(event.delta, end="", flush=True)
        # elif isinstance(event, AgentInput):
        #     print("📥 Input:", event.input)
        elif isinstance(event, AgentOutput):
            # Skip printing the output since we are streaming above
            # if event.response.content:
            #     print("📤 Output:", event.response.content)
            if event.tool_calls:
                print(
                    "🛠️  Planning to use tools:",
                    [call.tool_name for call in event.tool_calls],
                )
        elif isinstance(event, ToolCallResult):
            print(f"🔧 Tool Result ({event.tool_name}):")
            print(f"  Arguments: {event.tool_kwargs}")
            print(f"  Output: {event.tool_output}")
        elif isinstance(event, ToolCall):
            print(f"🔨 Calling Tool: {event.tool_name}")
            print(f"  With arguments: {event.tool_kwargs}")



```

多agent，使用一个prompt去驱动，引入了Event、Workflow、Step

```html

from llama_index.llms.openai import OpenAI
from tavily import AsyncTavilyClient
from llama_index.core.agent.workflow import FunctionAgent, ReActAgent
from llama_index.core.workflow import Context
import re

sub_agent_llm = OpenAI(model="gpt-4.1-mini", api_key="sk-...")

async def search_web(query: str) -> str:
    """Useful for using the web to answer questions."""
    client = AsyncTavilyClient(api_key="tvly-...")
    return str(await client.search(query))

research_agent = FunctionAgent(
    name="ResearchAgent",
    description="Useful for recording research notes based on a specific prompt.",
    system_prompt=(
        "You are the ResearchAgent that can search the web for information on a given topic and record notes on the topic. "
        "You should output notes on the topic in a structured format."
    ),
    llm=sub_agent_llm,
    tools=[search_web],
)

write_agent = FunctionAgent(
    name="WriteAgent",
    description="Useful for writing a report based on the research notes or revising the report based on feedback.",
    system_prompt=(
        "You are the WriteAgent that can write a report on a given topic. "
        "Your report should be in a markdown format. The content should be grounded in the research notes. "
        "Return your markdown report surrounded by <report>...</report> tags."
    ),
    llm=sub_agent_llm,
    tools=[],
)

review_agent = FunctionAgent(
    name="ReviewAgent",
    description="Useful for reviewing a report and providing feedback.",
    system_prompt=(
        "You are the ReviewAgent that can review the write report and provide feedback. "
        "Your review should either approve the current report or request changes to be implemented."
    ),
    llm=sub_agent_llm,
    tools=[],
)

async def call_research_agent(ctx: Context, prompt: str) -> str:
    """Useful for recording research notes based on a specific prompt."""
    result = await research_agent.run(
        user_msg=f"Write some notes about the following: {prompt}"
    )

    async with ctx.store.edit_state() as ctx_state:
        ctx_state["state"]["research_notes"].append(str(result))

    return str(result)


async def call_write_agent(ctx: Context) -> str:
    """Useful for writing a report based on the research notes or revising the report based on feedback."""
    async with ctx.store.edit_state() as ctx_state:
        notes = ctx_state["state"].get("research_notes", None)
        if not notes:
            return "No research notes to write from."

        user_msg = f"Write a markdown report from the following notes. Be sure to output the report in the following format: <report>...</report>:\n\n"

        # Add the feedback to the user message if it exists
        feedback = ctx_state["state"].get("review", None)
        if feedback:
            user_msg += f"<feedback>{feedback}</feedback>\n\n"

        # Add the research notes to the user message
        notes = "\n\n".join(notes)
        user_msg += f"<research_notes>{notes}</research_notes>\n\n"

        # Run the write agent
        result = await write_agent.run(user_msg=user_msg)
        report = re.search(
            r"<report>(.*)</report>", str(result), re.DOTALL
        ).group(1)
        ctx_state["state"]["report_content"] = str(report)

    return str(report)


async def call_review_agent(ctx: Context) -> str:
    """Useful for reviewing the report and providing feedback."""
    async with ctx.store.edit_state() as ctx_state:
        report = ctx_state["state"].get("report_content", None)
        if not report:
            return "No report content to review."

        result = await review_agent.run(
            user_msg=f"Review the following report: {report}"
        )
        ctx_state["state"]["review"] = result

    return result

# define the planner workflow
# 定义计划者的 workflow
import re
import xml.etree.ElementTree as ET
from pydantic import BaseModel, Field
from typing import Any, Optional

from llama_index.core.llms import ChatMessage
from llama_index.core.workflow import (
    Context,
    Event,
    StartEvent,
    StopEvent,
    Workflow,
    step,
)

# 自定义的prompt
PLANNER_PROMPT = """You are a planner chatbot.

Given a user request and the current state, break the solution into ordered <step> blocks.  Each step must specify the agent to call and the message to send, e.g.
<plan>
  <step agent=\"ResearchAgent\">search for …</step>
  <step agent=\"WriteAgent\">draft a report …</step>
  ...
</plan>

<state>
{state}
</state>

<available_agents>
{available_agents}
</available_agents>

The general flow should be:
- Record research notes
- Write a report
- Review the report
- Write the report again if the review is not positive enough

If the user request does not require any steps, you can skip the <plan> block and respond directly.
"""


class InputEvent(StartEvent):
    user_msg: Optional[str] = Field(default=None)
    chat_history: list[ChatMessage]
    state: Optional[dict[str, Any]] = Field(default=None)


class OutputEvent(StopEvent):
    response: str
    chat_history: list[ChatMessage]
    state: dict[str, Any]


class StreamEvent(Event):
    delta: str


class PlanEvent(Event):
    step_info: str


# Modelling the plan
class PlanStep(BaseModel):
    agent_name: str
    agent_input: str


class Plan(BaseModel):
    steps: list[PlanStep]


class ExecuteEvent(Event):
    plan: Plan
    chat_history: list[ChatMessage]


class PlannerWorkflow(Workflow):
    llm: OpenAI = OpenAI(
        model="o3-mini",
        api_key="sk-...",
    )
    agents: dict[str, FunctionAgent] = {
        "ResearchAgent": research_agent,
        "WriteAgent": write_agent,
        "ReviewAgent": review_agent,
    }

    @step
    async def plan(
        self, ctx: Context, ev: InputEvent
    ) -> ExecuteEvent | OutputEvent:
        # Set initial state if it exists
        if ev.state:
            await ctx.store.set("state", ev.state)

        chat_history = ev.chat_history

        if ev.user_msg:
            user_msg = ChatMessage(
                role="user",
                content=ev.user_msg,
            )
            chat_history.append(user_msg)

        # Inject the system prompt with state and available agents
        state = await ctx.store.get("state")
        available_agents_str = "\n".join(
            [
                f'<agent name="{agent.name}">{agent.description}</agent>'
                for agent in self.agents.values()
            ]
        )
        system_prompt = ChatMessage(
            role="system",
            content=PLANNER_PROMPT.format(
                state=str(state),
                available_agents=available_agents_str,
            ),
        )

        # Stream the response from the llm
        response = await self.llm.astream_chat(
            messages=[system_prompt] + chat_history,
        )
        full_response = ""
        async for chunk in response:
            full_response += chunk.delta or ""
            if chunk.delta:
                ctx.write_event_to_stream(
                    StreamEvent(delta=chunk.delta),
                )

        # Parse the response into a plan and decide whether to execute or output
        xml_match = re.search(r"(<plan>.*</plan>)", full_response, re.DOTALL)

        if not xml_match:
            chat_history.append(
                ChatMessage(
                    role="assistant",
                    content=full_response,
                )
            )
            return OutputEvent(
                response=full_response,
                chat_history=chat_history,
                state=state,
            )
        else:
            xml_str = xml_match.group(1)
            root = ET.fromstring(xml_str)
            plan = Plan(steps=[])
            for step in root.findall("step"):
                plan.steps.append(
                    PlanStep(
                        agent_name=step.attrib["agent"],
                        agent_input=step.text.strip() if step.text else "",
                    )
                )

            return ExecuteEvent(plan=plan, chat_history=chat_history)

    @step
    async def execute(self, ctx: Context, ev: ExecuteEvent) -> InputEvent:
        chat_history = ev.chat_history
        plan = ev.plan

        for step in plan.steps:
            agent = self.agents[step.agent_name]
            agent_input = step.agent_input
            ctx.write_event_to_stream(
                PlanEvent(
                    step_info=f'<step agent="{step.agent_name}">{step.agent_input}</step>'
                ),
            )

            if step.agent_name == "ResearchAgent":
                await call_research_agent(ctx, agent_input)
            elif step.agent_name == "WriteAgent":
                # Note: we aren't passing the input from the plan since
                # we're using the state to drive the write agent
                await call_write_agent(ctx)
            elif step.agent_name == "ReviewAgent":
                await call_review_agent(ctx)

        state = await ctx.store.get("state")
        chat_history.append(
            ChatMessage(
                role="user",
                content=f"I've completed the previous steps, here's the updated state:\n\n<state>\n{state}\n</state>\n\nDo you need to continue and plan more steps?, If not, write a final response.",
            )
        )

        return InputEvent(
            chat_history=chat_history,
        )

# running the workflow

planner_workflow = PlannerWorkflow(timeout=None)

handler = planner_workflow.run(
    user_msg=(
        "Write me a report on the history of the internet. "
        "Briefly describe the history of the internet, including the development of the internet, the development of the web, "
        "and the development of the internet in the 21st century."
    ),
    chat_history=[],
    state={
        "research_notes": [],
        "report_content": "Not written yet.",
        "review": "Review required.",
    },
)

current_agent = None
current_tool_calls = ""
async for event in handler.stream_events():
    if isinstance(event, PlanEvent):
        print("Executing plan step: ", event.step_info)
    elif isinstance(event, ExecuteEvent):
        print("Executing plan: ", event.plan)

# result = await handler


```

ReAct Agent with RAG，使用rag检索生成报告，也用到了tools

```html

from llama_index.core.agent.workflow import ReActAgent, FunctionAgent
from llama_index.core.workflow import Context
from llama_index.llms.ollama import Ollama
from llama_index.core.agent.workflow import AgentStream, ToolCallResult
from llama_index.core import Settings, StorageContext, load_index_from_storage, SimpleDirectoryReader, VectorStoreIndex
from llama_index.embeddings.huggingface import HuggingFaceEmbedding
from llama_index.core.tools import QueryEngineTool
import asyncio

# react的agent
Settings.llm = Ollama(model="llama3.1", request_timeout=360.0)
# Settings.llm = Ollama(model="local-llama-8B-instruct", request_timeout=360.0)
Settings.embed_model = HuggingFaceEmbedding(model_name="/Users/zqmalyssa/Model/embedding/bge-base-en-v1.5")

# 声明变量
lyft_index = None
uber_index = None
index_loaded = False

# 这个跟 把rag的

try:
    storage_context = StorageContext.from_defaults(
        persist_dir="./storage/lyft"
    )
    lyft_index = load_index_from_storage(storage_context)

    storage_context = StorageContext.from_defaults(
        persist_dir="./storage/uber"
    )
    uber_index = load_index_from_storage(storage_context)

    index_loaded = True
except:
    index_loaded = False

if not index_loaded:
    # load data
    lyft_docs = SimpleDirectoryReader(
        input_files=["./data/10k/lyft_2021.pdf"]
    ).load_data()
    uber_docs = SimpleDirectoryReader(
        input_files=["./data/10k/uber_2021.pdf"]
    ).load_data()

    # build index
    lyft_index = VectorStoreIndex.from_documents(lyft_docs)
    uber_index = VectorStoreIndex.from_documents(uber_docs)

    # persist index
    lyft_index.storage_context.persist(persist_dir="./storage/lyft")
    uber_index.storage_context.persist(persist_dir="./storage/uber")

lyft_engine = lyft_index.as_query_engine(similarity_top_k=3)
uber_engine = uber_index.as_query_engine(similarity_top_k=3)

query_engine_tools = [
    QueryEngineTool.from_defaults(
        query_engine=lyft_engine,
        name="lyft_10k",
        description=(
            "Provides information about Lyft financials for year 2021. "
            "Use a detailed plain text question as input to the tool."
        ),
    ),
    QueryEngineTool.from_defaults(
        query_engine=uber_engine,
        name="uber_10k",
        description=(
            "Provides information about Uber financials for year 2021. "
            "Use a detailed plain text question as input to the tool."
        ),
    ),
]

# 如果是 function + 无tool use model
# 果然 ollama._types.ResponseError: registry.ollama.ai/library/local-llama-8B-instruct:latest does not support tools (status code: 400)
# agent = FunctionAgent(
#     tools=query_engine_tools,
#     llm=Settings.llm,
#     # system_prompt="..."
# )


agent = ReActAgent(
    tools=query_engine_tools,
    llm=Settings.llm,
    # system_prompt="..."
)

ctx = Context(agent)

async def main():
    # handler = agent.run("What was Lyft's revenue growth in 2021?", ctx=ctx)

    handler = agent.run("Compare and contrast the revenue growth of Uber and Lyft in 2021, then give an analysis", ctx=ctx)
    # Compare and contrast the revenue growth of Uber and Lyft in 2021, then give an analysis

    async for ev in handler.stream_events():
        # if isinstance(ev, ToolCallResult):
        #     print(f"\nCall {ev.tool_name} with {ev.tool_kwargs}\nReturned: {ev.tool_output}")
        if isinstance(ev, AgentStream):
            print(f"{ev.delta}", end="", flush=True)

    response = await handler
    print('\n' + str(response))

if __name__ == "__main__":
    print('program starts ...')
    asyncio.run(main())

```

看一下输出，大致这样

```html

Thought: The user has asked for a comparison between the revenue growth of Uber and Lyft in 2021. I need to use tools to help me answer this question.

Action: lyft_10k
Action Input: {"input": "Compare and contrast the revenue growth of Lyft in 2021"}

Observation:
Revenue increased by $11 billion from $6.93 billion to $17.92 billion for the years ended December 31, 2020, and 2021, respectively.


Thought: I need more information about Uber's revenue growth in 2021.

Action: uber_10k
Action Input: {"input": "Compare and contrast the revenue growth of Uber in 2021"}

Observation:
Revenue increased by $8.9 billion from $14.08 billion to $23 billion for the years ended December 31, 2020, and 2021, respectively.

Thought: I have gathered enough information about both companies' revenue growth in 2021.

Answer: Based on the information provided, while Lyft experienced some growth drivers such as increased demand for ridesharing services during the fourth quarter of 2021, it did not experience overall revenue growth. In contrast, Uber's revenue grew by 57% year-over-year in 2021, driven by an increase in Gross Bookings due to higher food delivery orders and expansion into new markets, as well as the acquisition of Transplace. This significant difference in revenue growth can be attributed to the various business strategies employed by each company, with Uber focusing on diversification through food delivery and freight services, while Lyft focused on maintaining its core ridesharing business.


```

还有就是llama_index的workflow了，采用了workflow的Event + Step去完成整个workflow的流程

```html

from llama_index.core import SimpleDirectoryReader, VectorStoreIndex
from llama_index.core.response_synthesizers import CompactAndRefine
from llama_index.core.postprocessor.llm_rerank import LLMRerank
from llama_index.core.workflow import (
    Context,
    Workflow,
    StartEvent,
    StopEvent,
    step,
)

from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding


class RAGWorkflow(Workflow):
    @step
    async def ingest(self, ctx: Context, ev: StartEvent) -> StopEvent | None:
        """Entry point to ingest a document, triggered by a StartEvent with `dirname`."""
        dirname = ev.get("dirname")
        if not dirname:
            return None

        documents = SimpleDirectoryReader(dirname).load_data()
        index = VectorStoreIndex.from_documents(
            documents=documents,
            embed_model=OpenAIEmbedding(model_name="text-embedding-3-small"),
        )
        return StopEvent(result=index)

    @step
    async def retrieve(
        self, ctx: Context, ev: StartEvent
    ) -> RetrieverEvent | None:
        "Entry point for RAG, triggered by a StartEvent with `query`."
        query = ev.get("query")
        index = ev.get("index")

        if not query:
            return None

        print(f"Query the database with: {query}")

        # store the query in the global context
        await ctx.store.set("query", query)

        # get the index from the global context
        if index is None:
            print("Index is empty, load some documents before querying!")
            return None

        retriever = index.as_retriever(similarity_top_k=2)
        nodes = await retriever.aretrieve(query)
        print(f"Retrieved {len(nodes)} nodes.")
        return RetrieverEvent(nodes=nodes)

    @step
    async def rerank(self, ctx: Context, ev: RetrieverEvent) -> RerankEvent:
        # Rerank the nodes
        ranker = LLMRerank(
            choice_batch_size=5, top_n=3, llm=OpenAI(model="gpt-4o-mini")
        )
        print(await ctx.store.get("query", default=None), flush=True)
        new_nodes = ranker.postprocess_nodes(
            ev.nodes, query_str=await ctx.store.get("query", default=None)
        )
        print(f"Reranked nodes to {len(new_nodes)}")
        return RerankEvent(nodes=new_nodes)

    @step
    async def synthesize(self, ctx: Context, ev: RerankEvent) -> StopEvent:
        """Return a streaming response using reranked nodes."""
        llm = OpenAI(model="gpt-4o-mini")
        summarizer = CompactAndRefine(llm=llm, streaming=True, verbose=True)
        query = await ctx.store.get("query", default=None)

        response = await summarizer.asynthesize(query, nodes=ev.nodes)
        return StopEvent(result=response)


```
