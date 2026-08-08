# Event: 让 Runtime 从执行流程走向事件驱动
-------------------------------------------

## 前言
上一篇，我们让 Execution 拥有了自己的生命周期。

Execution 不再只是一个保存状态的结构体，而是开始主动管理状态转换：
```
Created
   │
   ▼
Running
   │
 ┌─┴─────────┐
 ▼           ▼
Completed   Failed
```
这解决了一个问题：

> Runtime 如何管理自己的生命周期？

但新的问题马上出现了。

当 Execution 从 Running 变成 Completed 时：

* Checkpoint 需要保存吗？
* Observer 需要记录吗？
* Metrics 需要更新吗？
* Memory 需要处理吗？
* 日志系统需要记录吗？

如果我们继续把这些逻辑直接写进 Execution Engine：
```Go
exec.Complete()

saveCheckpoint()
recordMetrics()
publishLog()
updateMemory()
```
那么 Engine 很快又会重新变成一个巨大的“上帝对象”。

我们刚刚通过 Execution Engine 完成了一次职责拆分。

现在不能因为加入 Event，又把所有职责重新塞回来。

所以这一篇，我们继续向前走一步：

> 让 Runtime 用 Event 表达“发生了什么”，而不是直接调用“应该做什么”。

-------------------------------------------
## 1. 为什么 Runtime 需要 Event？
先看一个最简单的执行流程。
```
Agent Execute
      │
      ▼
Execution Completed
```
如果 Runtime 中只有一个模块关心 Completed，直接调用当然没有问题。

但是随着 Runtime 不断演进，可能会出现：
```
Execution Completed
       │
       ├── Checkpoint
       ├── Observer
       ├── Memory
       ├── Metrics
       └── Logger
```
如果 Engine 直接调用它们：
```
Engine
 ├── Checkpoint
 ├── Observer
 ├── Memory
 ├── Metrics
 └── Logger
```
Engine 就开始知道越来越多的东西。

这会形成非常明显的耦合。

-----------------------------------
**Event 改变了这种关系**

我们可以把它改成：
```
Execution
    │
    ▼
  Event
    │
    ▼
 Event Bus
    │
 ├── Checkpoint
 ├── Observer
 ├── Memory
 └── Metrics
```
Execution 只需要表达：

> Execution Completed。

至于谁关心这个事件，由订阅者自己决定。

这就是 Event-driven Runtime 的核心思想。

-------------------------------------------
## 2. Event 和状态有什么关系？
Event 并不是 State 的替代品。

两者解决的是不同的问题。

**State 描述“现在是什么”**

例如：
```
Running
Completed
Failed
```
它回答的是：

> Execution 当前处于什么状态？

-------------------------------------------
**Event 描述“发生了什么”**

例如：
```
execution.started
execution.completed
execution.failed
```
它回答的是：

> Runtime 刚刚发生了什么？

因此：
```
State
    = 当前状态

Event
    = 状态变化产生的事实
```
例如：
```
Running
   │
   │ Complete()
   ▼
Completed

同时产生：

execution.completed
```
这两个概念需要同时存在。

-------------------------------------------
## 3. 先定义 Event
我们首先创建：
```
internal/event/event.go
```
Event 包只负责定义事件和事件总线。

它不应该依赖：

* Runtime
* Execution
* Agent

这样可以保证依赖关系非常简单。
```Go
package event

type Type string

const (
	TypeExecutionStarted   Type = "execution.started"
	TypeExecutionCompleted Type = "execution.completed"
	TypeExecutionFailed    Type = "execution.failed"
)

type Event struct {
	Type        Type
	ExecutionID string
}
```
这里暂时只定义三个事件。

它们对应 Execution 的三个重要生命周期变化：
```
Created
   │
   ▼
execution.started
   │
   ▼
Running
   │
   ├── execution.completed
   │
   └── execution.failed
```
以后随着 Runtime 的发展，还会出现：
```
execution.paused
execution.resumed
checkpoint.created
tool.started
tool.completed
human.input.required
```
但现在不需要提前设计。

-------------------------------------------
## 4. Event Bus
有了 Event，还需要一个地方把 Event 分发出去。

这就是 Event Bus。
```Go
package event

type Handler func(Event)

type Bus struct {
	handlers []Handler
}

func NewBus() *Bus {
	return &Bus{}
}

func (b *Bus) Subscribe(handler Handler) {
	b.handlers = append(b.handlers, handler)
}

func (b *Bus) Publish(evt Event) {
	for _, handler := range b.handlers {
		handler(evt)
	}
}
```
这里我们只实现最简单的同步 Event Bus。

它只有两个核心操作：
```
Subscribe
    │
    ▼
注册事件处理器

Publish
    │
    ▼
发送事件
```
例如：
```Go
bus.Subscribe(func(evt event.Event) {
	fmt.Println("Event:", evt.Type)
})
```

然后：
```Go
bus.Publish(event.Event{
	Type:        event.TypeExecutionStarted,
	ExecutionID: "run-001",
})
```
订阅者就会收到：
```
execution.started
```

-------------------------------------------
## 5. Execution Engine 开始发布事件
接下来修改 Execution Engine。

注意一个非常重要的设计：

> Execution 负责状态转换，Engine 负责发布对应事件。

也就是说：
```
Execution
    │
    └── 管理 State

Execution Engine
    │
    └── 发布 Event
```
这样 Execution 本身不需要知道 Event Bus 的存在。

这也避免了 Execution 和 Event 之间产生不必要的耦合。
-------------------------------------------

**execution/engine.go**
```Go
package execution

import (
	"github.com/aigc-engineering/aigc-agent-runtime/internal/agent"
	"github.com/aigc-engineering/aigc-agent-runtime/internal/event"
	"github.com/aigc-engineering/aigc-agent-runtime/internal/session"
)

type Engine struct {
	bus *event.Bus
}

func NewEngine(bus *event.Bus) *Engine {
	return &Engine{
		bus: bus,
	}
}

func (e *Engine) Execute(
	agt agent.Agent,
	sess *session.Session,
) error {
	exec := New(sess.ID)

	if err := exec.Start(); err != nil {
		return err
	}

	e.bus.Publish(event.Event{
		Type:        event.TypeExecutionStarted,
		ExecutionID: exec.ID,
	})

	if err := agt.Execute(sess); err != nil {
		if transitionErr := exec.Fail(); transitionErr != nil {
			return transitionErr
		}

		e.bus.Publish(event.Event{
			Type:        event.TypeExecutionFailed,
			ExecutionID: exec.ID,
		})

		return err
	}

	if err := exec.Complete(); err != nil {
		return err
	}

	e.bus.Publish(event.Event{
		Type:        event.TypeExecutionCompleted,
		ExecutionID: exec.ID,
	})

	return nil
}
```
这里有一个值得注意的地方：
```Go
exec.Start()
```
负责状态转换。

而：
```Go
e.bus.Publish(...)
```
负责发布事实。

两者职责是分开的。

-------------------------------------------
## 6. Runtime 负责组装组件
现在 Runtime 需要创建 Event Bus，然后把它交给 Execution Engine。
```
Runtime
   │
   ├── Event Bus
   │
   └── Execution Engine
```
对应：
```Go
package runtime

import (
	"github.com/aigc-engineering/aigc-agent-runtime/internal/agent"
	"github.com/aigc-engineering/aigc-agent-runtime/internal/event"
	"github.com/aigc-engineering/aigc-agent-runtime/internal/execution"
	"github.com/aigc-engineering/aigc-agent-runtime/internal/session"
)

type Runtime struct {
	engine *execution.Engine
}

func New() *Runtime {
	bus := event.NewBus()
	engine := execution.NewEngine(bus)

	return &Runtime{
		engine: engine,
	}
}

func (r *Runtime) Run(
	agt agent.Agent,
	sess *session.Session,
) error {
	return r.engine.Execute(agt, sess)
}
```
这里出现了一个新的问题。

Runtime 创建了 Event Bus，却没有把它暴露出来。

那么外部如何订阅事件？

这就是下一阶段需要解决的问题。

不过在这一篇，我们先把 Event 的核心机制跑通。

-------------------------------------------
## 7. 一个完整的 Demo
为了让整个例子可以直接运行，我们把 Demo 放到：
```
cmd/runtime/main.go
```
```Go
package main

import (
	"fmt"

	"github.com/aigc-engineering/aigc-agent-runtime/internal/agent"
	"github.com/aigc-engineering/aigc-agent-runtime/internal/runtime"
	"github.com/aigc-engineering/aigc-agent-runtime/internal/session"
)

type HelloAgent struct{}

func (HelloAgent) Execute(sess *session.Session) error {
	fmt.Println("Agent executing...")

	sess.Output = "hello, runtime"

	return nil
}

var _ agent.Agent = HelloAgent{}

func main() {
	rt := runtime.New()

	sess := &session.Session{
		ID:    "run-001",
		Input: "hello",
	}

	if err := rt.Run(HelloAgent{}, sess); err != nil {
		panic(err)
	}

	fmt.Println("Execution completed.")
	fmt.Println("Output:", sess.Output)
}
```
运行：
```sh
go run ./cmd/runtime
```
输出：
```text
Execution created: run-001
Execution running...
Agent executing:  hello
Execution completed.
Execution completed.
Output: Hello Agent Runtime.
```
目前 Event 已经在 Runtime 内部流动，但 Demo 还看不到它。

所以，我们需要让 Runtime 提供一个订阅入口。

-------------------------------------------
## 8. 让 Runtime 暴露 Event
我们修改 Runtime。
```Go
type Runtime struct {
	engine *execution.Engine
	bus    *event.Bus
}

func New() *Runtime {
	bus := event.NewBus()

	return &Runtime{
		engine: execution.NewEngine(bus),
		bus:    bus,
	}
}

func (r *Runtime) Subscribe(handler event.Handler) {
	r.bus.Subscribe(handler)
}

func (r *Runtime) Run(
	agt agent.Agent,
	sess *session.Session,
) error {
	return r.engine.Execute(agt, sess)
}
```
这样 Runtime 对外提供：
```Go
rt.Subscribe(...)
```
Demo 就可以订阅事件：
```Go
rt.Subscribe(func(evt event.Event) {
	fmt.Println("Event:", evt.Type)
})
```
完整的 main.go：
```Go
package main

import (
	"fmt"

	"github.com/aigc-engineering/aigc-agent-runtime/internal/agent"
	"github.com/aigc-engineering/aigc-agent-runtime/internal/event"
	"github.com/aigc-engineering/aigc-agent-runtime/internal/runtime"
	"github.com/aigc-engineering/aigc-agent-runtime/internal/session"
)

type HelloAgent struct{}

func (HelloAgent) Execute(sess *session.Session) error {
	fmt.Println("Agent executing...")
	sess.Output = "hello, runtime"
	return nil
}

var _ agent.Agent = HelloAgent{}

func main() {
	rt := runtime.New()

	rt.Subscribe(func(evt event.Event) {
		fmt.Println("Event:", evt.Type)
	})

	sess := &session.Session{
		ID:    "run-001",
		Input: "hello",
	}

	if err := rt.Run(HelloAgent{}, sess); err != nil {
		panic(err)
	}

	fmt.Println("Output:", sess.Output)
}
```
运行：
```sh
go run ./cmd/runtime
```
可以看到：
```text
Execution created: run-001
Execution running...
Agent executing:  hello
Execution completed.
Execution completed.
Output: Hello Agent Runtime.
```
至此，一个最简单的事件驱动 Runtime 就建立起来了。

-------------------------------------------
## 9. 现在的 Runtime 发生了什么变化？
回顾一下整个执行过程：
```

                    Runtime
                       │
                       ▼
                Execution Engine
                       │
             ┌─────────┴─────────┐
             │                   │
             ▼                   ▼
       Execution              Event Bus
             │                   │
             ▼                   ├── Observer
           Agent                 ├── Metrics
                                 ├── Checkpoint
                                 └── Memory
```
Execution Engine 负责：

> 驱动执行。

Execution 负责：

> 管理生命周期。

Event Bus 负责：

> 分发运行时事实。

Agent 负责：

> 完成业务能力。

这四个角色开始形成清晰的边界。

-------------------------------------------
## 10. Event 真正解决的是什么？
到这里，我们已经可以看到 Event 的真正价值。

假设下一篇我们增加 Checkpoint。

如果没有 Event：
```
Execution Engine
      │
      ├── Execution
      ├── Checkpoint
      ├── Memory
      ├── Observer
      └── Metrics
```
Engine 会越来越复杂。

而有了 Event：
```
Execution Engine
      │
      ▼
   Event Bus
      │
      ├── Checkpoint
      ├── Memory
      ├── Observer
      └── Metrics
```
Engine 不需要知道：

> 谁在监听这个事件。

它只需要做一件事情：

> 发布事实。

这就是事件驱动架构最核心的价值：

> 生产者只负责产生事实，消费者决定如何响应事实。

-------------------------------------------
## 11. Event Bus 现在为什么故意这么简单？
这一版 Event Bus 非常简单。

甚至只是：
```Go
type Bus struct {
	handlers []Handler
}
```
然后同步调用：
```Go
handler(evt)
```
为什么不一开始就实现：

* Kafka
* Redis Stream
* Goroutine
* Queue
* Retry
* Persistence
* Distributed Event Bus

因为这些都不是当前问题。

我们现在需要解决的是：

> Runtime 内部如何通过 Event 解耦组件？

因此第一版只需要一个：
```
In-process Event Bus
```
等 Runtime 的生命周期和模块边界真正稳定之后，再考虑：
```
Event Bus
    │
    ├── In-process
    ├── Redis
    └── Kafka
```
这是一个很重要的工程原则：

> 先解决架构问题，再解决规模问题。

-------------------------------------------
## 12. 一个需要提前注意的问题：同步还是异步？
当前实现：
```Go
Publish()
    │
    ├── handler()
    ├── handler()
    └── handler()
```
是同步的。

也就是说：
```Go
e.bus.Publish(evt)
```
只有当所有 Handler 执行完成之后，才会继续向下执行。

这样做的好处是：

* 实现简单
* 行为确定
* 错误容易调试
* 没有并发问题

但缺点也非常明显：

如果某个 Handler 很慢：
```
Execution
   │
   ▼
Event Bus
   │
   ▼
Slow Handler
   │
   ▼
Execution blocked
```
整个 Agent 执行就会被拖慢。

因此，在企业级 Runtime 中，Event 通常还需要进一步区分：
```
Synchronous Event

Asynchronous Event
```
但这属于后续 Runtime 优化的问题。

当前阶段，我们故意保持简单。

-------------------------------------------
## 13. 当前代码结构
到这一篇结束，代码仓库变成：
```text
aigc-agent-runtime/
│
├── cmd/
│   └── runtime/
│       └── main.go
│
├── internal/
│   ├── agent/
│   │   └── agent.go
│   │
│   ├── session/
│   │   └── session.go
│   │
│   ├── execution/
│   │   ├── execution.go
│   │   └── engine.go
│   │
│   ├── event/
│   │   └── event.go
│   │
│   └── runtime/
│       └── runtime.go
│
├── go.mod
└── README.md
```
依赖关系：
```
cmd
 │
 ▼
runtime
 │
 ├───────────────┐
 ▼               ▼
execution       event
 │
 ├── agent
 │
 └── session
```
这里尤其需要注意： `event`

不依赖：
```
runtime
execution
agent
session
```
而： `execution → event`

也没有直接依赖。

**Event 是独立的基础能力。**

这样可以避免随着 Runtime 演进再次出现循环依赖。

-------------------------------------------
## 总结
上一篇，我们让 Runtime 开始拥有生命周期。

这一篇，我们继续解决另一个问题：

> 生命周期发生变化之后，其他模块如何知道？

答案是 Event。

State 描述：

> 现在是什么。

Event 描述：

> 发生了什么。

于是 Runtime 的执行模型从：
```
Runtime
    │
    ▼
Execution
    │
    ▼
Agent
```
开始演进为：
```
Runtime
    │
    ▼
Execution Engine
    │
    ├── Execution
    │
    └── Event Bus
            │
            ├── Observer
            ├── Checkpoint
            ├── Memory
            └── Metrics
```
这意味着 Runtime 开始从一个简单的**执行器**，变成一个真正的**事件驱动运行时**。

而这一步非常重要。

因为从下一篇开始，我们就可以不再把所有能力硬塞进 Execution Engine，而是让新的 Runtime 能力通过 Event 接入。

-------------------------------------------
## 本章 Git Tag

`v0.4-event`

link: `https://github.com/aigc-engineering/aigc-agent-runtime/releases/tag/v0.4-event`