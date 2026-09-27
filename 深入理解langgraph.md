# 深入理解 LangGraph：State、Channel 与 Graph Runtime

## 前言

在开始学习 LangGraph 之前，很容易陷入 API 的学习：

```text
StateGraph
add_node
add_edge
ToolNode
interrupt
Command
Send
```

这些 API 单独看并不复杂，但如果没有理解它们背后的执行模型，很容易变成：

> “会用 LangGraph，但不知道 LangGraph 为什么这样设计。”

因此，这篇文章不以 API 罗列为目标，而是试图从执行模型出发，逐步建立 LangGraph 的整体认知。

可以先用一句话理解 LangGraph：

> **LangGraph 是一个以 State 为核心、由 Node 和 Edge 定义 Graph，并由 Runtime 驱动 Graph Execution 的框架。**

整个学习过程可以分成四层：

```text
第一层：Graph 是什么
    ↓
State / Node / Edge / StateGraph
    ↓
第二层：Graph 是怎么运行的
    ↓
Update / Channel / Reducer / Runtime
    ↓
第三层：Graph 如何演进为 Agent
    ↓
Cycle / Tool / HITL / Checkpoint / Memory
    ↓
第四层：复杂 Graph 如何执行
    ↓
SubGraph / Send / Fan-out / Streaming / Async
```

最终，我们会得到一条非常重要的执行链：

```text
State
  ↓
Node
  ↓
Update
  ↓
Channel
  ↓
Reducer
  ↓
State
  ↓
Edge / Routing
  ↓
Next Tasks
```

理解这条链，是后续进入 LangGraph 源码学习的基础。

---

# 第一部分：LangGraph 基础概念

## 1. LangGraph 到底是什么

我们先把 LangGraph 看成一个 Graph 系统。

一个 Graph 最基本包含三类东西：

```text
                 Graph
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
      State      Node      Edge
        │         │         │
        │         │         └── 决定执行路径
        │         └──────────── 执行业务逻辑
        └────────────────────── 保存执行状态
```

可以简单理解为：

* **State**：当前执行过程中有什么数据
* **Node**：当前需要执行什么逻辑
* **Edge**：执行完以后下一步去哪

因此，最基础的 LangGraph 并不神秘：

```text
State
  ↓
Node
  ↓
Edge
  ↓
Next Node
```

在理解这个模型之后，再去看 `StateGraph`、`compile()`、`invoke()`，就会容易很多。

---

## 2. State：Graph 的状态

`State` 是 LangGraph 最核心的概念。

可以把它理解为：

> **State 描述当前一次 Graph Execution 所需要的运行状态。**

例如：

```python
from typing import TypedDict


class State(TypedDict):
    input: str
    result: str
```

Graph 开始执行时，可以有：

```text
input = "hello"
result = ""
```

Node 执行过程中读取 State，并产生新的 State Update。

这里有一个非常重要的边界：

> **State 是当前一次 Graph Execution 的运行状态，而不是数据库，也不是整个 Agent 的所有数据。**

例如：

```text
State
├── 当前任务 ID
├── 当前处理结果
├── 当前状态
└── 当前执行所需要的数据
```

而数据库中的：

```text
用户信息
历史任务
业务实体
完整操作记录
```

并不意味着都应该塞进 State。

后面学习 Checkpoint、Store、Memory 时，这个边界会再次出现。

---

### Demo：最基础的 State

我们先从一个最简单的 Graph 开始。

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END


class State(TypedDict):
    input: str
    result: str


def process(state: State):
    return {
        "result": state["input"].upper()
    }


builder = StateGraph(State)

builder.add_node("process", process)

builder.add_edge(START, "process")
builder.add_edge("process", END)

graph = builder.compile()

result = graph.invoke({
    "input": "hello",
    "result": "",
})

print(result)
```

执行之后：

```text
input  = "hello"
result = "HELLO"
```

这里暂时只关注三个东西：

```text
State
  ↓
Node
  ↓
Result
```

至于 `compile()` 和 Runtime 到底做了什么，后面再展开。

---

## 3. Node：Graph 的执行单元

Node 是 Graph 中真正执行逻辑的地方。

例如：

```python
def node_a(state):
    return {
        "result": "hello"
    }
```

可以把 Node 理解成：

```text
读取 State
   ↓
执行逻辑
   ↓
返回 Update
```

Node 可以执行各种事情：

```text
调用 LLM
调用数据库
调用外部 API
执行 Python 逻辑
调用 Tool
执行计算
```

但 Node 有一个非常重要的职责边界：

> **Node 负责“做什么”，而不是负责整个 Graph 的流程控制。**

例如，不推荐把整个工作流都塞进一个 Node：

```python
def node(state):
    step_a()
    step_b()
    step_c()
    if xxx:
        step_d()
    else:
        step_e()
```

因为这样 Graph 本身就失去了表达能力。

更符合 LangGraph 思路的是：

```text
Node A
  ↓
Node B
  ↓
Node C
```

把执行单元拆出来，再让 Edge 表达它们之间的关系。

于是自然产生下一个问题：

> Node 执行完之后，下一步执行谁？

答案就是 Edge。

---

## 4. Edge / Routing：确定执行路径

最简单的 Edge：

```python
builder.add_edge("a", "b")
```

表示：

```text
a → b
```

也就是说：

> **Edge 描述 Node 之间的执行关系。**

如果流程是固定的：

```text
A → B → C → END
```

那么直接使用普通 Edge 即可。

但很多 Agent 的执行路径并不是固定的。

例如：

```text
        ┌── B
A ──────┼── C
        └── D
```

这时候就需要 Conditional Routing：

```python
def router(state):
    if state["process_b"]:
        return "b"
    return "c"

builder.add_conditional_edges(
    "a",
    router,
)
```

Router 根据当前 State 或执行结果决定下一步。

因此：

> **Node 负责执行，Edge 负责连接，Routing 负责决定下一步。**

这三者组合起来，才构成真正的 Graph。

---

## 5. StateGraph：定义 Graph

现在回头看：

```python
builder = StateGraph(State)
```

就很好理解了。

`StateGraph` 可以理解为：

> **Graph Definition / Builder。**

我们通过它描述：

```text
Graph 有什么 State
Graph 有哪些 Node
Node 之间如何连接
哪些地方存在条件路由
```

例如：

```python
builder = StateGraph(State)

builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)

builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", END)
```

得到：

```text
START
  ↓
node_a
  ↓
node_b
  ↓
END
```

因此可以先把：

```text
StateGraph
    ↓
add_node()
add_edge()
    ↓
Graph Definition
```

理解为“构建 Graph”。

---

## 6. compile() / invoke()：从 Graph 定义到 Graph Execution

Graph 定义完成之后，还不能直接执行。

我们需要：

```python
graph = builder.compile()
```

这一步可以理解为：

> **把 Graph Definition 编译成一个可以执行的 Graph。**

于是形成：

```text
StateGraph
    ↓
compile()
    ↓
Compiled Graph
```

然后：

```python
result = graph.invoke({
    "input": "hello"
})
```

开始一次实际执行：

```text
Compiled Graph
      ↓
   invoke()
      ↓
Graph Execution
```

所以到这里，我们已经建立了第一条完整链路：

```text
State
  +
Node
  +
Edge
  ↓
StateGraph
  ↓
compile()
  ↓
Compiled Graph
  ↓
invoke()
  ↓
Graph Execution
```

但此时我们仍然没有回答一个关键问题：

> **`invoke()` 到底是怎么驱动 Node 执行的？**

这就进入第二部分。

---

# 第二部分：LangGraph 是如何运行的

前面解决的是：

> **Graph 是什么，以及 Graph 如何定义。**

接下来要解决的是：

> **Graph 到底是怎么运行起来的？**

这里会进入 LangGraph 更核心的运行时模型：

```text
Update
Channel
Reducer
Runtime
Super-step
```

---

## 7. State Update：Node 如何修改 State

先回到 Node：

```python
def process(state: State):
    return {
        "result": state["input"].upper()
    }
```

这里有一个容易忽略的问题。

Node 返回的：

```python
{
    "result": "HELLO"
}
```

并不是完整的 State。

它更准确地说是：

> **State Update。**

也就是说，Node 不需要返回：

```python
{
    "input": "hello",
    "result": "HELLO"
}
```

而只需要告诉 Runtime：

```text
我希望更新 result。
```

因此执行过程可以抽象为：

```text
Current State
      ↓
    Node
      ↓
    Update
      ↓
Next State
```

如果只有一个 Node 更新一个字段，这件事情非常简单。

但问题马上来了：

> **如果多个 Node 同时更新同一个 State 字段呢？**

例如：

```text
Node A → ["A"]
Node B → ["B"]
Node C → ["C"]
```

最终：

```text
results = ?
```

这就需要理解 Channel 和 Reducer。

---

## 8. Channel：State 更新的运行时机制

LangGraph 的 State 不能简单理解成一个普通 Python `dict`。

在 Graph Runtime 内部，State 的不同字段需要具备不同的更新语义。

可以概念性地理解为：

```text
State
 ├── name channel
 └── results channel
```

Node 产生 Update：

```text
Node
 ↓
Update
 ↓
Channel
```

Channel 可以理解为：

> **Graph Runtime 中承载某个 State 字段更新及其更新语义的机制。**

例如：

```python
class State(TypedDict):
    name: str
    results: list[str]
```

概念上可以理解为：

```text
name
  ↓
name channel

results
  ↓
results channel
```

不同字段可能需要不同的更新方式：

```text
覆盖
追加
聚合
自定义合并
```

因此，State 看起来像一个普通的数据结构，但 Runtime 并不是简单地执行：

```python
state.update(update)
```

而是需要根据字段对应的更新语义处理 Update。

这就是 Channel / Reducer 出现的原因。

---

## 9. Reducer：多个 Update 如何合并

假设我们有：

```python
from typing import Annotated
import operator


class State(TypedDict):
    results: Annotated[list[str], operator.add]
```

现在三个 Node 同时产生：

```text
Node A → ["A"]
Node B → ["B"]
Node C → ["C"]
```

如果直接覆盖，最终结果可能只剩下最后一次更新。

但我们希望：

```text
["A"]
   +
["B"]
   +
["C"]
   ↓
["A", "B", "C"]
```

这时候就需要 Reducer。

> **Reducer 定义一个 State 字段收到多个 Update 时应该如何合并。**

例如：

```python
results: Annotated[
    list[str],
    operator.add,
]
```

意味着这个字段的多个列表更新可以进行追加合并。

因此：

```text
Update A ──┐
Update B ──┼──→ Reducer → New State
Update C ──┘
```

这也是为什么 Reducer 会和后面的动态并行执行直接关联。

当我们通过 `Send` 动态创建多个执行分支时，每个分支都会产生 Update，最终仍然需要通过 Reducer 将结果汇聚起来。

---

### Demo：State + Reducer

```python
from typing import Annotated, TypedDict
import operator

from langgraph.graph import StateGraph, START, END


class State(TypedDict):
    results: Annotated[list[str], operator.add]


def node_a(state: State):
    return {"results": ["A"]}


def node_b(state: State):
    return {"results": ["B"]}


builder = StateGraph(State)

builder.add_node("a", node_a)
builder.add_node("b", node_b)

builder.add_edge(START, "a")
builder.add_edge(START, "b")

builder.add_edge("a", END)
builder.add_edge("b", END)

graph = builder.compile()

result = graph.invoke({
    "results": []
})

print(result["results"])
```

这里真正值得关注的不是代码本身，而是：

```text
START
 ├── Node A → ["A"]
 │
 └── Node B → ["B"]
          ↓
       Reducer
          ↓
    ["A", "B"]
```

这就是后面 Fan-out / Fan-in 的基础。

---

## 10. Graph Runtime / Super-step：Graph 到底怎么执行

现在我们已经有了：

```text
State
Node
Edge
Update
Channel
Reducer
```

终于可以回答：

> `invoke()` 到底在干什么？

它并不是简单地：

```python
node_a()
node_b()
node_c()
```

而更接近一个 Graph Runtime 持续驱动执行的过程：

```text
找到 Ready Tasks
       ↓
执行当前 Tasks
       ↓
收集 Updates
       ↓
Channel 接收 Updates
       ↓
Reducer 合并
       ↓
生成新的 State
       ↓
根据 Edge / Routing
确定下一批 Tasks
       ↓
继续执行
```

可以简化成：

```text
┌──────────────┐
│ Ready Tasks  │
└──────┬───────┘
       ↓
 Execute Nodes
       ↓
 Collect Updates
       ↓
   Channels
       ↓
    Reducer
       ↓
   New State
       ↓
Edge / Routing
       ↓
Next Tasks
       ↓
    Repeat
```

这就是理解 LangGraph Runtime 的基础。

这里可以形成一个非常重要的认知：

> **Graph 是静态定义，Execution 是动态发生。**

也就是说：

```text
StateGraph
    ↓
定义“有哪些 Node、有哪些 Edge”
```

而：

```text
Runtime
    ↓
决定“现在执行谁、产生什么 Update、下一步执行谁”
```

这也是后面源码学习中 `StateGraph → compile() → CompiledStateGraph → invoke() → Runtime` 这条链路的基础。

---

## 11. Cycle：Graph 不一定是 DAG

传统 Workflow 经常是：

```text
A → B → C → END
```

这是典型的 DAG。

但 Agent 经常需要：

```text
LLM
 ↓
Decision
 ↓
Tool
 ↓
LLM
 ↓
Decision
 ↓
...
```

也就是：

```text
A → B → C
    ↑   ↓
    └───┘
```

LangGraph 支持这种 Cycle。

因此 Graph 不再只是：

> “描述一次性的任务编排。”

它还可以描述：

> **一个不断根据 State 进行决策和执行的状态机。**

这也是为什么 LangGraph 特别适合 Agent。

---

### Demo：Cyclic Agent

最简单的 Agent Loop 可以抽象成：

```text
LLM
 ↓
Decision
 ↓
Tool
 ↓
LLM
 ↑
 └──────
```

它的关键不是 Tool 本身，而是：

```text
Node
 ↓
Routing
 ↓
Node
 ↓
Routing
 ↓
Node
```

Graph 允许执行路径回到之前的 Node。

因此：

```text
Cycle
  ↓
Repeated Execution
  ↓
Agent Loop
```

---

# 第三部分：Graph 如何演进为一个 Agent

到这里，我们已经理解了一个非常重要的基础：

> **LangGraph 本质上先是一个 Graph Execution Framework。**

它并不是因为有一个叫 `Agent` 的对象，才成为 Agent。

而是：

```text
State
+
Node
+
Routing
+
Cycle
+
Tool
+
Persistence
+
HITL
```

这些能力组合起来之后，才形成一个真正可以长期运行的 Agent Execution Model。

---

## 12. Tool Calling：LLM + Tool + Graph

Agent 最常见的能力之一，就是让 LLM 根据当前状态决定是否调用 Tool。

基本结构：

```text
       START
         ↓
        LLM
         ↓
  tools_condition
 ┌───────┴───────┐
 ↓               ↓
ToolNode         END
 ↓
LLM
 ↓
...
```

关键 API：

```python
@tool
def get_weather(city: str):
    ...
```

```python
llm_with_tools = llm.bind_tools(tools)
```

```python
ToolNode(tools)
```

```python
tools_condition
```

完整生命周期：

```text
HumanMessage
     ↓
AIMessage(tool_calls)
     ↓
ToolNode
     ↓
ToolMessage
     ↓
AIMessage(final)
```

这里可以形成一个非常重要的分工：

> **LLM 负责决策，Tool 提供能力，Graph 负责执行结构。**

LangGraph 并不是简单把：

```text
LLM + Tool
```

拼起来，而是利用 Graph 把：

```text
决策
 ↓
调用
 ↓
结果
 ↓
再次决策
```

组织成一个可持续执行的状态机。

---

### Demo：Tool Calling Agent

一个最小天气 Agent：

```python
import os

from dotenv import load_dotenv
from langchain_core.messages import HumanMessage
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.prebuilt import ToolNode, tools_condition


load_dotenv()

if not os.getenv("OPENAI_API_KEY"):
    raise RuntimeError("OPENAI_API_KEY is not set")


@tool
def get_weather(city: str) -> str:
    weather_data = {
        "北京": "晴天，25°C",
        "上海": "多云，27°C",
        "深圳": "小雨，29°C",
        "台北": "多云，28°C",
    }

    return weather_data.get(
        city,
        f"{city}：暂无天气数据",
    )


tools = [get_weather]

llm = ChatOpenAI(
    model="gpt-4.1-mini",
    temperature=0,
)

llm_with_tools = llm.bind_tools(tools)


def call_llm(state: MessagesState):
    response = llm_with_tools.invoke(
        state["messages"]
    )

    return {
        "messages": [response]
    }


builder = StateGraph(MessagesState)

builder.add_node("llm", call_llm)
builder.add_node("tools", ToolNode(tools))

builder.add_edge(START, "llm")

builder.add_conditional_edges(
    "llm",
    tools_condition,
    {
        "tools": "tools",
        END: END,
    },
)

builder.add_edge("tools", "llm")

graph = builder.compile()


result = graph.invoke({
    "messages": [
        HumanMessage(
            content="北京今天天气怎么样？"
        )
    ]
})

for message in result["messages"]:
    print(
        f"{message.type}: "
        f"{message.content}"
    )
```

这个 Demo 同时把前面的几个概念串了起来：

```text
State
 ↓
Node
 ↓
Conditional Edge
 ↓
Tool
 ↓
Cycle
 ↓
State Update
```

这也是为什么不建议一开始就把 Tool Calling 单独当成一个“API 功能”学习。

---

## 13. MessagesState：Agent 的消息状态

上面的 Demo 出现了：

```python
MessagesState
```

这是 LangGraph 提供的一种特殊 State 形式，用于方便处理消息流。

典型消息包括：

```text
HumanMessage
AIMessage
ToolMessage
```

于是 Agent 的消息执行链可以表示为：

```text
User
 ↓
LLM
 ↓
Tool Call
 ↓
Tool
 ↓
Tool Result
 ↓
LLM
 ↓
Final Answer
```

但是必须注意：

> **`MessagesState` 只是 State 的一种形式，并不意味着 Agent State 就等于 Messages。**

复杂 Agent 仍然可能需要：

```python
State:
    messages
    user_id
    task_id
    status
    tool_result
    retry_count
    ...
```

所以应该建立这样的认知：

```text
Agent State
    │
    ├── messages
    ├── business data
    ├── execution status
    └── other runtime data
```

而 `MessagesState` 只是其中消息型 State 的一种标准表达方式。

---

## 14. HITL：让 Graph 暂停

普通 Graph：

```text
A
 ↓
B
 ↓
C
```

会一直执行。

但真实 Agent 经常需要：

```text
生成内容
    ↓
等待人工审核
    ↓
审核通过
    ↓
继续执行
```

这就是 Human-in-the-loop。

LangGraph 使用：

```python
interrupt(...)
```

实现 Graph 暂停。

例如：

```text
generate_prompt
      ↓
   interrupt
      ↓
等待人工审核
```

这里非常重要的一点是：

> **`interrupt()` 不是冻结 Python 调用栈。**

也就是说，不应该把它理解成：

```text
函数停在这一行
↓
几小时后继续下一行
```

更准确的理解是：

```text
Graph Execution
      ↓
interrupt
      ↓
保存可恢复状态
      ↓
Execution 暂停
      ↓
之后重新进入执行
```

因此，包含 `interrupt()` 的 Node 在恢复时可能重新执行。

例如：

```python
def node(state):

    external_side_effect()

    interrupt()

    ...
```

恢复时：

```text
external_side_effect()
```

可能再次执行。

所以：

> **Interrupt 前的外部副作用必须设计成幂等，或者具备状态检查。**

这对于生产环境非常重要。

---

### Demo：HITL

最简单的执行过程：

```text
Node
 ↓
interrupt()
 ↓
Human Review
 ↓
Command(resume=...)
 ↓
Continue
```

它真正表达的是：

```text
Graph Execution
      ↓
   Pause
      ↓
Human Decision
      ↓
   Resume
      ↓
Graph Execution
```

---

## 15. Command：Resume / Update / Routing

人工审核之后，需要告诉 Graph：

> “现在可以继续执行了。”

可以通过：

```python
Command(
    resume=...
)
```

恢复执行。

但 `Command` 的能力并不只有 Resume。

它还可以同时表达：

```text
State Update
+
Routing
```

例如：

```python
return Command(
    update={
        "status": "approved"
    },
    goto="next_node",
)
```

这意味着：

```text
更新 State
    +
决定下一步
```

因此可以这样理解：

### Conditional Edge

更偏向：

```text
根据当前结果
决定去哪
```

### Command

更偏向：

```text
Node 做出决定
    +
更新 State
    +
决定下一步
```

这两个机制都可以实现 Routing，但表达方式和使用场景不同。

---

## 16. Checkpoint：让 Graph 可以恢复

如果 Graph 只是：

```text
invoke()
```

然后进程挂掉，那么执行状态也可能随之丢失。

对于一个可能运行：

```text
几秒
几分钟
几小时
甚至更久
```

的 Agent，这是不可接受的。

因此需要 Checkpoint。

Checkpoint 的核心并不是简单：

```text
保存 State
```

而是：

> **保存 Graph Execution 恢复所需要的状态和执行信息。**

概念上：

```text
Running
   ↓
Checkpoint
   ↓
Process Crash
   ↓
Load Checkpoint
   ↓
Resume
```

因此 Checkpoint 支撑了：

```text
Long Running
Crash Recovery
HITL
Resume
Execution Persistence
```

这时候可以把：

```text
interrupt
+
Command
+
Checkpoint
```

看成一个完整的 Long Running Execution 能力。

---

## 17. Store / Memory：State 之外的数据

到了这里，需要区分几个经常被混淆的概念。

### State

当前 Graph Execution：

```text
这一次执行正在处理什么？
```

### Checkpoint

当前 Execution：

```text
停下来之后如何恢复？
```

### Store

跨 Execution：

```text
不同执行之间共享什么数据？
```

### Memory

更大的能力概念：

```text
Agent 如何长期保存和使用信息？
```

Memory 可以基于：

```text
Store
DB
Vector DB
Profile
...
```

实现。

所以可以先建立：

```text
State ≠ Checkpoint ≠ Store ≠ Memory
```

这个区分非常重要。

例如一个用户的长期偏好：

```text
“用户喜欢使用中文回答”
```

它显然不应该只存在某一次 Graph Execution 的 State 中。

而某次任务当前执行到：

```text
image_generation
```

则属于当前 State。

两者生命周期完全不同。

---

# 第四部分：复杂 Graph 执行

前面的内容解决的是：

```text
一个 Graph 如何定义
一个 Graph 如何执行
一个 Graph 如何成为 Agent
```

接下来进入更复杂的执行模型：

```text
SubGraph
Send
Fan-out / Fan-in
Streaming
Async
```

这些能力开始真正体现 LangGraph 作为 Graph Execution Framework 的价值。

---

## 18. SubGraph：Graph inside Graph

当一个 Graph 越来越复杂时，不可能把所有 Node 都放在同一层。

例如：

```text
Main Graph
   │
   ├── Node A
   │
   ├── SubGraph
   │      ├── Node B
   │      ├── Node C
   │      └── Node D
   │
   └── Node E
```

SubGraph 可以理解成：

> **把一个完整的 Graph 封装成更大的 Graph 中的一个执行单元。**

于是：

```text
Main Graph
    ↓
SubGraph
    ↓
多个内部 Node
```

这样可以把复杂 Workflow 拆成多个模块。

例如一个实际的业务系统可能是：

```text
Main Graph
   │
   ├── Script Graph
   │
   ├── Asset Graph
   │
   └── Video Graph
```

每个 SubGraph 内部又可以拥有自己的：

```text
State
Node
Edge
Cycle
HITL
```

这使得 Graph 可以进行层次化设计。

---

## 19. Send：动态 Fan-out

普通 Edge：

```text
A → B
```

Graph 结构在定义时已经确定。

但有一种场景：

> **运行时才知道到底需要创建多少个任务。**

例如：

```text
当前资产
    ↓
发现 3 个视角
    ↓
生成 3 个任务
```

或者：

```text
发现 100 个角色
    ↓
动态生成 100 个处理任务
```

这时候普通 Edge 就不够用了。

LangGraph 提供：

```python
Send(...)
```

用于动态创建执行分支。

例如：

```python
return [
    Send(
        "process_view",
        {"view_id": "A"},
    ),
    Send(
        "process_view",
        {"view_id": "B"},
    ),
    Send(
        "process_view",
        {"view_id": "C"},
    ),
]
```

得到：

```text
          ┌── process(A)
          │
Router ───┼── process(B)
          │
          └── process(C)
```

如果有 N 个任务：

```text
Send × N
```

因此：

> **Send 是 LangGraph 动态 Fan-out 的核心机制。**

更重要的是：

> Send 不是简单在 Node 里面写一个 Python `for` 循环。

它把这些动态分支交给 Graph Runtime 管理，因此每个分支都可以成为 Graph Execution 的一部分。

---

## 20. Fan-out / Fan-in：动态并行与结果汇聚

动态创建分支之后，就会产生第二个问题：

> 多个分支执行完成之后，结果如何重新汇聚？

例如：

```text
           ┌── A ──┐
           │       │
Router ────┼── B ──┼──→ results
           │       │
           └── C ──┘
```

每个分支可能产生：

```text
A → ["A"]
B → ["B"]
C → ["C"]
```

最终通过 Reducer：

```text
["A"]
   +
["B"]
   +
["C"]
   ↓
["A", "B", "C"]
```

所以完整模型是：

```text
Send
  ↓
Fan-out
  ↓
Parallel Tasks
  ↓
Updates
  ↓
Reducer
  ↓
Fan-in
```

这里可以看到，前面学习的 Reducer 在这里再次出现。

这也是为什么：

> **Send 和 Reducer 并不是两个孤立的 API，它们共同组成 LangGraph 动态并行执行的重要机制。**

---

### Demo：Send + Fan-out / Fan-in

```python
from typing import Annotated, TypedDict
import operator

from langgraph.graph import StateGraph, START, END
from langgraph.types import Send


class ViewResult(TypedDict):
    view_id: str
    status: str
    image_url: str


class AssetState(TypedDict):
    asset_id: int
    views: list[str]
    results: Annotated[
        list[ViewResult],
        operator.add,
    ]


def extract_views(state: AssetState):
    return {
        "views": [
            "front",
            "side",
            "back",
        ]
    }


def fan_out_views(state: AssetState):
    return [
        Send(
            "process_view",
            {
                "asset_id": state["asset_id"],
                "view_id": view,
            },
        )
        for view in state["views"]
    ]


def process_view(state):
    asset_id = state["asset_id"]
    view_id = state["view_id"]

    image_url = (
        f"https://example.com/"
        f"{asset_id}/{view_id}.png"
    )

    return {
        "results": [
            {
                "view_id": view_id,
                "status": "completed",
                "image_url": image_url,
            }
        ]
    }


builder = StateGraph(AssetState)

builder.add_node(
    "extract_views",
    extract_views,
)

builder.add_node(
    "process_view",
    process_view,
)

builder.add_edge(
    START,
    "extract_views",
)

builder.add_conditional_edges(
    "extract_views",
    fan_out_views,
)

builder.add_edge(
    "process_view",
    END,
)

graph = builder.compile()

result = graph.invoke({
    "asset_id": 1001,
    "views": [],
    "results": [],
})

print(result["results"])
```

这里完整体现了：

```text
extract_views
      ↓
     Send
   ┌──┼──┐
   A  B  C
   │  │  │
   └──┼──┘
      ↓
   Reducer
      ↓
   results
```

这就是 LangGraph 动态 Fan-out / Fan-in 的完整模型。

---

## 21. Streaming：观察 Graph Execution

前面的：

```python
graph.invoke(...)
```

最终得到的是：

```text
Final State
```

但在实际系统中，我们经常不希望等整个 Graph 执行结束之后才知道发生了什么。

例如一个长流程：

```text
Node A
 ↓
Node B
 ↓
LLM
 ↓
Tool
 ↓
Node C
```

用户可能希望实时知道：

```text
当前执行到哪里了？
哪个 Node 完成了？
LLM 输出了什么？
```

这时候可以使用：

```python
graph.stream(...)
```

与：

```python
graph.invoke(...)
```

相比：

### `invoke()`

关注：

```text
最终结果
```

### `stream()`

关注：

```text
执行过程中的中间事件 / 更新
```

因此：

```text
invoke
  ↓
Final Result

stream
  ↓
Intermediate Execution
```

需要特别区分：

```text
Graph Streaming
```

和：

```text
LLM Token Streaming
```

二者不是一回事。

前者关注：

> **Graph Execution 的过程。**

后者关注：

> **LLM 输出 Token 的过程。**

一个复杂 Agent 可以同时存在这两种 Streaming。

---

## 22. Async：异步执行

LangGraph 同时提供：

```python
ainvoke()
astream()
```

用于异步执行。

它主要解决的是：

> **异步 IO 场景下的资源利用问题。**

例如：

```text
Graph
 ↓
调用 LLM API
 ↓
等待网络响应
 ↓
继续执行
```

使用异步模型可以避免在等待 IO 时阻塞整个执行线程。

但是一定要区分：

```text
async
≠
无限并发
```

异步只是执行方式。

业务系统真正的并发控制仍然可能需要：

```text
Queue
Scheduler
Rate Limiter
Concurrency Limit
Worker
```

这些通常属于更高层的 Runtime / Infrastructure 能力。

---

# 第五部分：完整知识地图

到这里，我们已经从最简单的：

```text
State
Node
Edge
```

一路走到了：

```text
Send
Fan-out
Fan-in
Checkpoint
HITL
Tool
SubGraph
```

现在可以把整个知识体系重新压缩成一张图。

```text
                         LangGraph
                            │
                            ▼
                         StateGraph
                            │
                    add_node / add_edge
                            │
                            ▼
                     Graph Definition
                            │
                         compile()
                            │
                            ▼
                  Compiled Graph
                            │
                         invoke()
                            │
                            ▼
                     Graph Runtime
                            │
              ┌─────────────┼─────────────┐
              │             │             │
              ▼             ▼             ▼
            State          Node          Edge
              │             │             │
              │             ▼             ▼
              │          Update        Routing
              │             │
              │             ▼
              │          Channel
              │             │
              │             ▼
              │          Reducer
              │             │
              └─────────────┘
                            │
                            ▼
                        New State
                            │
                            ▼
                       Next Tasks
                            │
                            ▼
                          Repeat
                            │
             ┌──────────────┼──────────────┐
             │              │              │
             ▼              ▼              ▼
           Cycle          Tool           HITL
             │              │              │
             │          ToolNode       interrupt
             │              │              │
             │       MessagesState     Command
             │                             │
             └──────────────┬──────────────┘
                            │
                            ▼
                       Checkpoint
                            │
                            ▼
                    Long Running Agent
                            │
             ┌──────────────┼──────────────┐
             │              │              │
             ▼              ▼              ▼
          Store          SubGraph       Streaming
             │
           Memory
                            │
                            ▼
                          Send
                            │
                         Fan-out
                            │
                    Parallel Tasks
                            │
                         Reducer
                            │
                         Fan-in
```

---

## 23. 六类 Demo 对应的知识点

到目前为止，我们实际写过的 Demo 可以对应到整个知识体系。

### Demo 1：最基础 Graph

```text
START
  ↓
Node A
  ↓
Node B
  ↓
END
```

掌握：

```python
StateGraph
add_node
add_edge
compile
invoke
```

对应：

```text
State
Node
Edge
Graph
```

---

### Demo 2：State / Reducer

```text
Node A ──→ ["A"]
Node B ──→ ["B"]
       ↓
    Reducer
       ↓
 ["A", "B"]
```

掌握：

```python
Annotated
Reducer
State Update
Channel
```

对应：

```text
Update
Channel
Reducer
```

---

### Demo 3：Cyclic Agent

```text
LLM
 ↓
Decision
 ↓
Tool
 ↓
LLM
 ↑
 └────
```

掌握：

```text
Cycle
Routing
Agent Loop
```

对应：

```text
Graph
    ↓
Cycle
    ↓
Agent
```

---

### Demo 4：Tool Calling Agent

```text
HumanMessage
      ↓
     LLM
      ↓
tools_condition
   ┌──┴──┐
   ↓     ↓
 Tool   END
   ↓
  LLM
```

掌握：

```python
@tool
bind_tools()
ToolNode
tools_condition
MessagesState
```

对应：

```text
LLM
Tool
Messages
Graph
```

---

### Demo 5：HITL

```text
Node
 ↓
interrupt()
 ↓
Human Review
 ↓
Command(resume=...)
 ↓
Continue
```

掌握：

```text
interrupt
Command
Checkpoint
Resume
```

对应：

```text
Pause
Human Decision
Persistence
Resume
```

---

### Demo 6：Send + Fan-out / Fan-in

```text
extract
   ↓
Send
 ┌─┼─┐
 A B C
 │ │ │
 └─┼─┘
   ↓
Reducer
   ↓
results
```

掌握：

```python
Send
add_conditional_edges
Reducer
Dynamic Parallelism
Fan-out
Fan-in
```

对应：

```text
Dynamic Execution
Parallel Tasks
Reducer
Aggregation
```

---

## 24. 从这里进入源码学习

如果前面的内容已经理解，那么下一阶段就不应该继续零散学习 API。

因为现在已经有了一个完整的抽象模型。

接下来真正值得研究的是：

> **LangGraph 是如何把这些抽象概念实现出来的？**

源码学习可以沿着这样一条主线展开：

```text
StateGraph
    ↓
compile()
    ↓
CompiledStateGraph
    ↓
invoke()
    ↓
Pregel
    ↓
Task
    ↓
Node Execution
    ↓
Write
    ↓
Channel
    ↓
Reducer
    ↓
Checkpoint
```

然后再分别追踪几个高级机制：

```text
Send
Command
interrupt
SubGraph
```

也就是说，前面的学习是在回答：

> **LangGraph 应该如何理解？**

下一阶段源码学习则回答：

> **LangGraph 是如何实现这个模型的？**

这两个阶段应该明确区分。

---

# 总结

如果以后忘记 LangGraph 的细节，可以先回到最基础的 Graph 模型：

```text
State
+
Node
+
Edge
```

它们定义了：

```text
Graph 是什么
```

然后再进入 Runtime：

```text
Node
 ↓
Update
 ↓
Channel
 ↓
Reducer
 ↓
State
 ↓
Edge / Routing
 ↓
Next Tasks
```

它们解释了：

```text
Graph 是怎么运行的
```

最后再在这个基础上增加：

```text
Cycle        → Agent Loop
Tool         → 外部能力
MessagesState→ 消息型 State
Interrupt    → 暂停执行
Command      → Resume / Update / Routing
Checkpoint   → 可恢复执行
Store        → 跨执行持久化
Memory       → 长期信息
SubGraph     → Graph 模块化
Streaming    → 观察执行过程
Async        → 异步执行
Send         → 动态 Fan-out
Reducer      → Fan-in / 聚合
```

最终可以把 LangGraph 压缩成一句话：

> **State 描述当前执行状态，Node 执行业务逻辑并产生 Update，Channel 承载状态更新，Reducer 合并多个 Update，Edge 决定下一步，而 Runtime 驱动整个 Graph Execution。**

而当：

```text
Graph
+
State
+
Cycle
+
Tool
+
HITL
+
Checkpoint
```

组合起来之后，它就从一个普通的 Graph Workflow，逐渐演进成了一个能够：

```text
决策
执行
循环
调用工具
暂停
等待人工
恢复
持久化
并行
```

的 **Long-Running Agent Execution Model**。

这也是理解 LangGraph 的关键。

下一步真正值得研究的，不再是“还有哪些 API”，而是：

```text
StateGraph
    ↓
compile()
    ↓
CompiledStateGraph
    ↓
invoke()
    ↓
Pregel
    ↓
Task
    ↓
Node Execution
    ↓
Channel / Write
    ↓
Reducer
    ↓
Checkpoint
```

**从这里，才正式进入 LangGraph 源码世界。**
