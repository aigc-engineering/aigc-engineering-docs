# Runtime Kernel：为什么 Runtime 不直接调用 Agent？

## 前言
上一篇，我们搭建了第一个可以运行的 Agent Runtime。

虽然它非常简单，但已经完成了 Runtime 最重要的职责：

> 管理一次 Agent 的执行。

整个执行流程如下：

```
Runtime
    │
    ▼
Agent.Execute()
    │
    ▼
Execution Finished
```
对于一个 Hello World 来说，这已经足够了。

但是，如果这是一个真正的企业级 Agent，很快就会遇到新的问题。

例如：

* 如何统计一次执行耗时？
* 如何记录日志？
* 如何支持 Checkpoint？
* 如何暂停和恢复？
* 如何进行重试？
* 如何发送 Event？
* 如何保存 Trace？

如果这些逻辑全部写进 Runtime.Run()，Runtime 很快就会变成一个几百行甚至上千行的方法。

所以，从这一篇开始，我们需要对 Runtime 做第一次架构升级。

## 1. 为什么 Runtime 不直接调用 Agent？

来看上一篇 Runtime 的核心逻辑：
```Golang
func (r *Runtime) Run(agent Agent, ctx Context) error {
    return agent.Execute(ctx)
}
```
现在它只有一行代码。

看起来没有任何问题。

很多人第一次写 Agent Runtime 时，也都是这样开始的。

但是，随着功能不断增加，代码通常会逐渐演变成这样：
```Go
func (r *Runtime) Run(agent Agent, ctx Context) error {
    startTrace()
    defer finishTrace()

    saveCheckpoint()

    publishEvent()

    metrics.Record()

    err := agent.Execute(ctx)

    if err != nil {
        retry()
    }

    saveCheckpoint()

    publishEvent()

    return err
}
```
最后，`Run()` 方法会承担越来越多的职责。

它既负责执行 Agent，又负责日志、事件、监控、重试、Checkpoint……

这违背了一个最基本的软件设计原则：

> 一个模块应该只有一个职责（Single Responsibility Principle）。

因此，我们需要把“执行 Agent”这件事情独立出来。

## 2. Runtime 真正管理的对象是什么？
很多人认为：

> Runtime 管理的是 Agent。

其实并不是。

Runtime 真正管理的是：

> Execution。

一个 Agent 可以被执行很多次。

例如：
```
Resume Agent

├── Execution 001
├── Execution 002
├── Execution 003
└── Execution 004
```

Agent 是能力的定义。

Execution 才是一次真实的运行实例。

后面我们讨论的：

* State Machine
* Event
* Checkpoint
* Memory
* Human-in-the-loop

实际上都不是挂在 Agent 上，而是挂在 Execution 上。

因此，我们需要一个专门负责管理 Execution 的组件。

这就是：

> Execution Engine。

## 3. Runtime Kernel 的第一次重构
重构之后，Runtime 的职责开始发生变化。

以前：
```
Runtime
    │
    ▼
Agent
```
现在：
```
Runtime
    │
    ▼
Execution Engine
    │
    ▼
Agent
```
Runtime 不再负责执行细节。

它只负责协调。

Execution Engine 才是真正的执行内核。

以后所有运行时能力都会逐步加入到这里。

例如：
```
Runtime
    │
    ▼
Execution Engine
    ├── State Machine
    ├── Event Bus
    ├── Checkpoint
    ├── Memory
    ├── Trace
    └── Metrics
```
这样，Runtime 本身会一直保持简洁，而 Execution Engine 则负责不断演进。

## 4. 核心实现
这一节不贴全部源码，而是讲核心结构。

完整代码请参考仓库：
`https://github.com/aigc-engineering/aigc-agent-runtime`

对应版本：
`v0.2-runtime-kernel`

**目录结构**
注意： 这里把context改造为了session，是为了避免和Go的标准包context冲突。
```
aigc-agent-runtime
│
├── cmd
│   └── runtime
│       └── main.go
│
├── internal
│   ├── agent
│   │   └── agent.go
│   ├── execution
│   │   ├── execution.go
│   │   └── engine.go
│   ├── runtime
│   │   └── runtime.go
│   └── session
│       └── session.go
```
----------------------
**Session**
```Go
package session

type Session struct {
	ID     string
	Input  any
	Output any
}
```
----------------------
**Agent**
```Go
package agent

import "github.com/aigc-engineering/aigc-agent-runtime/internal/session"

type Agent interface {
	Execute(sess *session.Session) error
}
```
----------------------
**Execution**
```Go
package execution

type Status string

const (
    StatusCreated   Status = "created"
    StatusRunning   Status = "running"
    StatusCompleted Status = "completed"
    StatusFailed    Status = "failed"
)

type Execution struct {
    ID     string
    Status Status
}
```
Execution 表示一次运行实例。

未来，Checkpoint、Trace、State 等信息都会逐步加入这个对象。
----------------------

**Engine**
```Go
package execution

import (
	"github.com/aigc-engineering/aigc-agent-runtime/internal/agent"
	"github.com/aigc-engineering/aigc-agent-runtime/internal/session"
)

type Engine struct{}

func New() *Engine {
	return &Engine{}
}

func (e *Engine) Execute(
	agt agent.Agent,
	sess *session.Session,
) error {

	exec := &Execution{
		ID:     "run-001",
		Status: StatusCreated,
	}

	exec.Status = StatusRunning

	err := agt.Execute(sess)

	if err != nil {
		exec.Status = StatusFailed
		return err
	}

	exec.Status = StatusCompleted

	return nil
}
```
Execution Engine 专注于一件事情：
> 驱动一次 Execution 完成整个生命周期。

----------------------
**Runtime**
Runtime则变得非常简单：
```Go
package runtime

import (
	"github.com/aigc-engineering/aigc-agent-runtime/internal/agent"
	"github.com/aigc-engineering/aigc-agent-runtime/internal/execution"
	"github.com/aigc-engineering/aigc-agent-runtime/internal/session"
)

type Runtime struct {
	engine *execution.Engine
}

func New() *Runtime {
	return &Runtime{
		engine: execution.New(),
	}
}

func (r *Runtime) Run(
	agt agent.Agent,
	sess *session.Session,
) error {

	return r.engine.Execute(agt, sess)

}
```
整个 Runtime 只保留一个职责：
> 协调 Execution Engine。

整个 Runtime 的调用链变成：
```
Runtime
    │
    ▼
Execution Engine
    │
    ▼
Agent
    │
    ▼
Session
```
可以看到，Runtime 不再直接负责 Agent 的执行，而是把执行职责委托给 Execution Engine。

## 5. Demo 演进
本篇对应 Git Tag： `v0.2-runtime-kernel`

运行：
```
git clone git@github.com:aigc-engineering/aigc-agent-runtime.git

cd aigc-agent-runtime

git checkout v0.2-runtime-kernel

go run ./cmd
```
输出：
```
Execution created: run-001
Execution running...
Agent executing:  build runtime
Execution completed.
```
## 6. 工程思考
这一篇我们并没有增加任何新功能。

甚至最终运行结果和上一篇几乎完全一样。

但这是整个 Runtime 演进过程中非常重要的一次重构。

因为：

> 优秀的软件架构，往往不是通过增加功能体现，而是通过职责划分体现。

从现在开始：

* Runtime 负责协调。
* Execution Engine 负责执行。
* Agent 负责业务。

这三个角色已经具备了清晰的边界。

未来无论增加多少能力，都不会破坏这层结构。

## 总结
很多人在实现 Agent Runtime 时，一开始就关注：

* Tool
* Prompt
* Memory
* Workflow

但真正稳定的 Runtime，并不是从功能开始设计的。

而是从：

> 执行内核（Execution Kernel）

开始设计。

当 Runtime 学会把执行职责交给 Execution Engine 时，它才真正拥有了持续演进的基础。