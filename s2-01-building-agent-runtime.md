# 从零开始：设计我们的 Agent Runtime

## 前言
在第一季中，我们讨论了一个核心问题：

> 什么才是真正的 Agent？

我们从状态机、Workflow、Long Running、Checkpoint、Memory、Event 等角度，逐渐建立了一个完整的认知：

一个企业级 Agent，并不是一次 LLM 调用。

它更像一个长期运行的软件系统。

它需要：

* 管理状态
* 执行任务
* 响应事件
* 保存上下文
* 支持恢复
* 与外部世界交互

因此：

> Agent 的核心，不只是模型，而是 Runtime。

但是理解 Runtime 只是第一步。

真正困难的问题是：

> 如果今天让我们自己实现一个 Agent Runtime，我们应该从哪里开始？

这也是第二季要解决的问题。

## 1. 从概念走向工程
第一季我们画过这样的架构：
```
                Agent
                  |
             Agent Runtime
                  |
    +-------------+-------------+
    |             |             |
State Machine   Memory      Event System
    |
Checkpoint
    |
Tool Runtime
```
但是这只是设计思想。

真正落地时，我们需要回答：

* Runtime 的入口在哪里？
* 一次 Agent 执行如何开始？
* Runtime 如何管理执行过程？
* 状态保存在哪里？
* 模块之间如何通信？

这些问题最终都会落到代码结构上。

## 2. 我们要构建什么
第二季我们不会实现一个完整商业化 Agent 平台。

目标是：

> 构建一个最小但完整的 Agent Runtime。

我们称它：`AIGC Runtime`

最终目标：
```
                User
                 |
              Runtime
                 |
        +----------------+
        | Execution Loop |
        +----------------+
          /      |      \
         /       |       \

     State     Event    Tool

     Memory    Checkpoint
```
随着文章推进，我们会逐步加入：

| 版本   | 能力                |
| ---- | ----------------- |
| v0.1 | 基础 Runtime        |
| v0.2 | Execution         |
| v0.3 | State Machine     |
| v0.4 | Event             |
| v0.5 | Scheduler         |
| v0.6 | Tool Runtime      |
| v0.7 | Checkpoint        |
| v0.8 | Memory            |
| v0.9 | Human-in-the-loop |
| v1.0 | 完整 Runtime        |


## 3. 第一个 Runtime 应该解决什么？
很多人在设计 Agent 时，会直接想到：
```
调用 LLM
执行 Tool
返回结果
```
但是 Runtime 首先应该解决：
> 如何管理一次执行生命周期。

例如：

一次 Agent Run：
```
Created
  |
Running
  |
Completed
```
所以第一版 Runtime 只需要三个核心对象：

### 3.1 Agent
Agent 是能力提供者。

它定义：
> 我能做什么。

例如：
```Golang
type Agent interface {
    Execute(ctx Context) error
}
```

### 3.2 Execution
Execution 表示一次运行。

类似：

数据库中的:
```
Request ID
Job ID
Workflow Run
```

例如：
```Golang
type Execution struct {
    ID      string
    Status  string
}
```

### 3.3 Runtime
Runtime 是执行管理器。

它负责：
```
创建 Execution

调用 Agent

管理生命周期

返回结果
```

## 4. 第一个 Runtime 架构
第一版：
```
                 main
                  |
               Runtime
                  |
              Execution
                  |
                Agent
                  |
                Done
```
虽然非常简单。

但是后面的所有能力都会插入这里。

例如：

未来：

State Machine：
```
Runtime
  |
State Machine
  |
Execution
```

Event:
```
Runtime
  |
Event Bus
  |
Execution
```

Checkpoint:
```
Runtime
  |
Checkpoint Store
  |
Execution
```

所以：
> Runtime 是所有能力的入口。

## 5. 项目初始化
代码仓库：
```
aigc-agent-runtime
 └── runtime
      ├── cmd
      └── runtime
```
第一版：
```
runtime/
├── cmd/
│    └── main.go
└── runtime/
     ├── agent.go
     ├── execution.go
     └── runtime.go
```

## 6. 源码实现v0.1

在第一版 Runtime 中，我们只实现最核心的执行链路.

完整源码已经同步到 Github：
> https://github.com/aigc-engineering/aigc-agent-runtime

当前版本： `v0.1-runtime-kernel`

### 核心接口设计
虽然完整实现放在代码仓库中，但是这里介绍 Runtime 的核心抽象。

**Agent**

> Agent 是能力执行单元。

```Golang
type Agent interface {
    Execute(ctx Context) error
}
```

Runtime 并不关心 Agent 内部如何实现。

它只负责：

- 创建执行
- 调度执行
- 管理生命周期

**Execution**
> Execution 表示一次 Agent 运行。

```Golang
type Execution struct {
    ID string
    Status Status
}
```

它类似：

- Workflow Run
- Job Instance
- Task Execution

后续的：

- Checkpoint
- Event
- Observability

都会围绕 Execution 展开。

**Runtime**
> Runtime 是整个系统的入口。

```Golang
func (r *Runtime) Run(
    agent Agent,
    ctx Context,
) error
```

当前版本 Runtime 只负责：

```
Create Execution
        ↓
    Run Agent
        ↓
Complete Execution
```

随着后续章节推进，我们会不断增强 Runtime：
```
    v0.1
Basic Runtime
     ↓
    v0.2
State Machine
     ↓
    v0.3
Event Driven
     ↓
    v0.4+
Checkpoint / Memory / Scheduler
```

## 7. 运行结果
执行：
```
go run cmd/main.go
```

输出：
```
Execution created: run-001
Execution running
Agent executing: build runtime
Execution completed
```

## 8. 当前 Runtime 有什么？
现在：

我们已经拥有：

✅ Runtime

✅ Agent

✅ Execution

✅ 生命周期

虽然非常简单。

但是它已经具备 Runtime 最核心的结构：
```
创建执行

↓

运行 Agent

↓

结束执行
```

不过，这个 Runtime 只是一个骨架，下一篇，我们会继续加入 Runtime 最重要的能力：
> **State Machine**

让 Runtime 真正拥有生命周期管理能力。

## 总结
Agent Runtime 的构建，不应该从 Tool 或 Prompt 开始。

真正可靠的 Runtime，首先需要解决：

> 如何管理一次 Agent 执行。

因此：

第一版 Runtime 只做一件事：

管理 Execution 生命周期。

后续所有能力：

* State Machine
* Event
* Scheduler
* Checkpoint
* Memory
* Tool Runtime

都会围绕这个核心不断演进。

这就是：

> Building an Agent Runtime 的第一步。

## Code Version

Repository:
https://github.com/aigc-engineering/aigc-agent-runtime

Current Version:
v0.1-runtime-kernel