# Checkpoint：让 Runtime 记住 Agent 走到了哪里

## 前言
在前面的文章中，我们已经逐步建立了 Agent Runtime 的基础能力。

Agent 不再只是一次函数调用，而是一个具有生命周期的执行过程。

我们通过 `Execution` 表示一次执行，通过 State Machine 管理它的生命周期，再通过 Event 把 Runtime 中发生的状态变化传播出去。

到这里，Runtime 已经可以回答一个问题：

> Agent 现在正在做什么？

但对于 Long Running Agent 来说，还缺少一个非常重要的问题：

> 如果 Runtime 停了，Agent 还能知道自己刚才做到了哪里吗？

例如，一个 Agent 正在执行：
```text
分析任务
   ↓
调用工具
   ↓
处理结果
   ↓
生成下一步计划
   ↓
等待用户确认
```
执行到中间时，Runtime 可能发生：
```text
进程重启
服务发布
Worker 崩溃
节点故障
```
如果所有状态只存在于内存中：
```text
Execution
    ↓
Runtime Process
    ↓
Process Exit
    ↓
State Lost
```
那么 Agent 的生命周期也就随之丢失。

这意味着：

> State Machine 解决了 Agent“怎么运行”，但还没有解决 Agent“运行到哪里可以被记住”。

Checkpoint，就是解决这个问题的。

这一篇，我们将在前四篇 Runtime 的基础上，实现一个最简单但完整的 Checkpoint 机制：
```text
Execution
    ↓
Event
    ↓
Event Bus
    ↓
Checkpoint Handler
    ↓
Checkpoint Store
```

---
## 1. 为什么 Long Running Agent 需要 Checkpoint？
### 1.1 Runtime 进程不等于 Agent 生命周期

这是理解 Checkpoint 最重要的一点。

一个普通函数的生命周期通常是：
```text
Process
   ↓
Function Start
   ↓
Function Execute
   ↓
Function Return
```
函数依赖当前进程。

进程结束，函数也就结束了。

但 Long Running Agent 不一样。

它的生命周期可能是：
```text
Execution
   │
   ├── Running
   ├── Waiting
   ├── Paused
   └── Continue
```
甚至可能跨越多个 Runtime 进程：
```text
Runtime Process A
       │
       ▼
    Execution
       │
       ▼
   Checkpoint
       │
       ▼
Process A Exit
       │
       ▼
Runtime Process B
       │
       ▼
Continue Execution
```
因此：

> Execution 的生命周期不能依赖某一个 Runtime 进程。

### 1.2 State Machine 解决了什么？

前面的文章中，我们已经实现了 State Machine。

例如：
```text
Created
   ↓
Running
   ↓
Paused
   ↓
Completed
```
State Machine 解决的是：

> 当前状态是什么？

以及：

> 当前状态能不能转换到下一个状态？

但是它仍然存在一个问题。

假设当前：
```text
Execution.Status = Paused
```
如果这个状态只保存在内存：
```text
Execution
    │
    ▼
Memory
```
Runtime 重启之后：
```text
Memory
    ↓
Lost
```
那么这个状态就不存在了。

所以：
```text
State Machine
    +
Persistent State
```
才是一个真正能够长期运行的 Execution。

Checkpoint 就是两者之间的桥梁。

## 2. Checkpoint 到底保存什么？

Checkpoint 可以理解成：

> Execution 在某个时间点的状态快照。

但这里的“状态”不是把整个 Runtime 保存下来。

我们不需要保存：
```text
Runtime
├── Engine
├── Event Bus
├── Worker
├── Goroutine
├── Network Connection
└── ...
```
这些都是运行时对象。

真正需要保存的是：
```text
Execution State
+
Session State
```

### 2.1 Execution State

Execution 描述的是：

> 这次执行处于什么生命周期状态。

例如：
```text
Execution
├── ID
└── Status
```
状态可能是：
```text
Created
Running
Paused
Completed
Failed
```

### 2.2 Session State

Session 描述的是：

> 这次执行携带的上下文。

例如：
```text
Session
├── ID
├── Input
└── ...
```
因此，一次完整的 Checkpoint 可以表示成：
```text
Checkpoint
├── Execution ID
├── Execution Status
└── Session
```
也就是说：
```text
Execution
    +
Session
    ↓
Checkpoint
```
这也是为什么 Checkpoint 不能只保存一个 Status。

因为：
```text
Status = Paused
```
只能告诉我们：

> Agent 暂停了。

却不能告诉我们：

> 这个 Agent 是谁，以及暂停时携带了什么执行上下文。

## 3. 定义 Checkpoint

现在开始进入代码。

这里我们严格按照代码的依赖顺序来实现。

首先新增：
```text
internal/checkpoint/checkpoint.go
```
定义 Checkpoint：
```Go
package checkpoint

import "github.com/aigc-engineering/aigc-agent-runtime/internal/session"

type Checkpoint struct {
	ExecutionID string
	Status      string
	Session     session.Session
}
```
这个结构非常简单。

它只描述一个快照：
```text
ExecutionID
Status
Session
```
Checkpoint 本身不负责：
```text
保存
加载
数据库操作
Event 监听
```
它只是数据。

这和前面 Execution 的设计是一致的：

> 一个对象首先应该描述自己的状态，而不是承担所有与自己相关的行为。

## 4. 为 Checkpoint 提供存储能力

有了 Checkpoint 数据结构之后，我们还需要解决一个问题：

> Checkpoint 保存到哪里？

在生产环境中，我们可能会使用：
```text
MySQL
PostgreSQL
Redis
MongoDB
Object Storage
```
但 Runtime 本身不应该依赖某一种具体存储。

因此先定义一个接口：
```text
internal/checkpoint/store.go
```
```Go
package checkpoint

import "fmt"

type Store interface {
	Save(Checkpoint) error
	Load(string) (Checkpoint, error)
}

type MemoryStore struct {
	data map[string]Checkpoint
}

func NewMemoryStore() *MemoryStore {
	return &MemoryStore{
		data: make(map[string]Checkpoint),
	}
}

func (s *MemoryStore) Save(cp Checkpoint) error {
	s.data[cp.ExecutionID] = cp
	return nil
}

func (s *MemoryStore) Load(executionID string) (Checkpoint, error) {
	cp, ok := s.data[executionID]
	if !ok {
		return Checkpoint{}, fmt.Errorf(
			"checkpoint not found: %s",
			executionID,
		)
	}

	return cp, nil
}
```
现在 Checkpoint 已经具备了最基本的持久化抽象。

我们有：
```text
Store
 │
 ├── Save
 └── Load
```
当前提供一个：
```text
MemoryStore
```
用于 Demo。

### 4.1 为什么先使用 MemoryStore？

这里的 `MemoryStore` 并不是生产实现。

它的作用是：

> 先验证 Runtime 的 Checkpoint 架构。

因为这一篇我们真正要解决的是：
```text
Event
   ↓
Checkpoint
   ↓
Store
```
而不是数据库设计。

以后我们完全可以实现：
```Go
type MySQLStore struct {
	// ...
}
```
只要实现：
```Go
type Store interface {
	Save(Checkpoint) error
	Load(string) (Checkpoint, error)
}
```
Runtime 就不需要发生变化。

这就是接口带来的解耦。

## 5. 用 Event 驱动 Checkpoint

现在已经有：
```text
Checkpoint
Checkpoint Store
```
但是还有一个问题：

> 什么时候保存 Checkpoint？

我们当然可以在 Engine 里面直接写：
```Go
checkpointStore.Save(...)
```
但这样 Engine 就开始依赖 Checkpoint。

以后如果 Runtime 再增加：
```text
Observability
Audit
Notification
Metrics
Memory
```
Engine 就会越来越复杂。

前面的 Event 机制正好可以解决这个问题。

我们已经有：
```text
Execution
    ↓
Event
    ↓
Event Bus
```
因此 Checkpoint 可以作为一个 Event Handler：
```text
Event Bus
    │
    ├── Checkpoint Handler
    ├── Observability Handler
    └── ...
```
这样 Engine 只负责：

> 发布发生了什么。

而 Checkpoint Handler 负责：

> 发生以后，我需要保存什么。

## 6. 实现 Checkpoint Handler

现在我们创建：
```text
internal/checkpoint/handler.go
```
代码：
```Go
package checkpoint

import "github.com/aigc-engineering/aigc-agent-runtime/internal/event"

func Handler(store Store) event.Handler {
	return func(evt event.Event) {
		_ = store.Save(Checkpoint{
			ExecutionID: evt.ExecutionID,
			Status:      evt.Status,
			Session:     evt.Session,
		})
	}
}
```
这里的逻辑非常简单：
```text
Event
  ↓
创建 Checkpoint
  ↓
Store.Save()
```
注意这里没有判断：
```text
if evt.Type == ...
```
原因是我们这一版 Runtime 的设计是：

> Event Bus 中每产生一个 Execution Event，就保存一次当前快照。

例如：
```text
execution.started
       ↓
Checkpoint

execution.paused
       ↓
Checkpoint

execution.completed
       ↓
Checkpoint
```
因此 Checkpoint 实际上形成了一系列执行快照。

在当前 Demo 中，MemoryStore 只保留最新的一份：
```text
execution-001
    ↓
latest checkpoint
```
以后如果需要支持真正的历史版本，可以进一步把 Store 设计成：
```text
execution-001
    ├── checkpoint-001
    ├── checkpoint-002
    ├── checkpoint-003
    └── checkpoint-004
```
但那是另外一个问题。

本篇首先解决：

> Runtime 能够在 Event 发生时保存当前状态。

## 7. 让 Event 携带 Checkpoint 所需的信息

Checkpoint Handler 现在需要：
```text
ExecutionID
Status
Session
```
因此 Event 必须携带这些信息。

前面的 Event 已经负责描述 Runtime 中发生的事情。

现在只需要保证 Event 中包含 Checkpoint 所需要的上下文。
```text
internal/event/event.go：
```
```Go
package event

import "github.com/aigc-engineering/aigc-agent-runtime/internal/session"

type Type string

const (
	TypeExecutionStarted   Type = "execution.started"
	TypeExecutionPaused    Type = "execution.paused"
	TypeExecutionCompleted Type = "execution.completed"
	TypeExecutionFailed    Type = "execution.failed"
)

type Event struct {
	Type        Type
	ExecutionID string
	Status      string
	Session     session.Session
}

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
这里没有引入：
```text
checkpoint
```
这是非常重要的。

Event 只描述事实：
```text
Execution ID
Status
Session
Event Type
```
它不知道：

> 谁会消费这个 Event

因此依赖关系仍然是：
```text
Event
   ↑
Checkpoint Handler
```
而不是：
```text
Event
   ↓
Checkpoint
```
这样就不会产生循环依赖。

## 8. 在 Runtime 中注册 Checkpoint Handler

现在：
```text
Checkpoint
Checkpoint Store
Checkpoint Handler
Event
Event Bus
```
都已经存在了。

最后一步才是：

> 把 Checkpoint Handler 接入 Runtime。

这时候才需要修改：
```text
internal/runtime/runtime.go
```
```Go
package runtime

import (
	"github.com/aigc-engineering/aigc-agent-runtime/internal/checkpoint"
	"github.com/aigc-engineering/aigc-agent-runtime/internal/event"
	"github.com/aigc-engineering/aigc-agent-runtime/internal/execution"
)

type Runtime struct {
	bus        *event.Bus
	engine     *execution.Engine
	checkpoint checkpoint.Store
}

func New() *Runtime {
	bus := event.NewBus()

	checkpointStore := checkpoint.NewMemoryStore()

	bus.Subscribe(
		checkpoint.Handler(checkpointStore),
	)

	engine := execution.NewEngine(
		bus,
	)

	return &Runtime{
		bus:        bus,
		engine:     engine,
		checkpoint: checkpointStore,
	}
}

func (r *Runtime) LoadCheckpoint(
	executionID string,
) (checkpoint.Checkpoint, error) {
	return r.checkpoint.Load(executionID)
}
```
这里第一次出现：
```Go
checkpoint.NewMemoryStore()
```
此时它已经在前面的章节中完整定义过。

所以代码依赖关系是完整的：
```text
checkpoint.NewMemoryStore()
        ↑
第五章已经实现
```
然后：
```Go
checkpoint.Handler(checkpointStore)
```
也已经在上一章实现。

所以 Runtime 这里只做一件事情：

> 把已经存在的组件组装起来。

## 9. Runtime 为什么负责组装？

到这里，我们可以更清楚地理解 Runtime 的职责。

Runtime 本身不负责保存 Checkpoint。

它只是负责：
```text
创建 Store
     ↓
创建 Handler
     ↓
注册到 Event Bus
```
也就是：
```text
Runtime
   │
   ├── Event Bus
   │
   ├── Checkpoint Store
   │
   ├── Checkpoint Handler
   │
   └── Execution Engine
```
真正发生执行的时候：
```text
Runtime
   ↓
Engine
   ↓
Execution
   ↓
Event
   ↓
Event Bus
   ↓
Checkpoint Handler
   ↓
Checkpoint Store
```
因此：

> Runtime 是组装者，Engine 是驱动者，Handler 是响应者。

这是这一篇需要建立起来的核心认识。

## 10. Engine 不需要知道 Checkpoint

现在 Checkpoint 已经接入 Runtime。

那么 Engine 需要修改吗？

答案是：

> Engine 不需要引入 Checkpoint。

它只需要像之前一样发布 Event。

例如：
```Go
func (e *Engine) Pause(
	exec *Execution,
	sess *session.Session,
) error {
	if err := exec.Pause(); err != nil {
		return err
	}

	e.bus.Publish(event.Event{
		Type:        event.TypeExecutionPaused,
		ExecutionID: exec.ID,
		Status:      string(exec.Status),
		Session:     *sess,
	})
	return nil
}
```

Engine 只做：
```text
Execution Pause
 ↓
Publish Event
```
它完全不知道：
```text
Checkpoint
```
这意味着以后增加：
```text
Observability Handler
```
或者：
```text
Audit Handler
```
Engine 都不需要修改。

## 11. 一次完整的 Checkpoint 流程

现在把整个流程串起来。

假设：
```text
Execution ID = execution-001
Status = Running
```
调用：
```Go
rt.Pause(
    &Execution{
        ID: "execution-001"
    },
    &Session{
        ID: "execution-001",
        Input: "hello",
    },
)
```
第一步：
```text
Runtime
   ↓
Engine.Pause()
```
第二步：
```text
Engine
   ↓
Execution.Pause()
```
状态变成：
```text
Running
   ↓
Paused
```
第三步：

Engine 发布：
```text
execution.paused
```
第四步：
```text
Event Bus
   ↓
Checkpoint Handler
```
第五步：

Handler 创建：
```text
Checkpoint
├── ExecutionID
├── Status = Paused
└── Session
```
第六步：
```text
Checkpoint Store
```
保存完成。

完整链路：
```text
Application
     │
     ▼
  Runtime
     │
     ▼
Execution Engine
     │
     ▼
  Execution
     │
     │ state change
     ▼
    Event
     │
     ▼
 Event Bus
     │
     ▼
Checkpoint Handler
     │
     ▼
Checkpoint Store
```
这就是本篇真正实现的能力。

## 12. 用 Demo 验证 Checkpoint

现在我们通过 Runtime API 验证整个流程。

cmd/runtime/main.go：
```Go
func main() {
	rt := runtime.NewRuntime()

	// 这里是暴漏给外部组件的event注册
	rt.Subscribe(func(evt event.Event) {
		fmt.Println("Event: ", evt.Type)
	})

	sess := &session.Session{
		ID:    "run-001",
		Input: "hello",
	}

	exec := execution.NewExecution(sess.ID)
	exec.Start() // 模拟运行中

	if err := rt.Pause(exec, sess); err != nil {
		fmt.Println("pause error: ", err)
		panic(err)
	}

	cp, err := rt.LoacCheckpoint(exec.ID)
	if err != nil {
		fmt.Println("load checkpoint error: ", err)
		return
	}

	fmt.Println("execution: ", cp.ExecutionID)
	fmt.Println("status: ", cp.Status)
}
```
但是这里还有一个前提：
```text
run-001
```
必须已经存在于 Execution Store。

而前面的 Runtime 已经提供了创建 Execution 的入口，所以正常流程应该是：
```text
Run
 ↓
Execution Started
 ↓
Event
 ↓
Checkpoint
 ↓
Pause
 ↓
Event
 ↓
Checkpoint
``
因此 Demo 应该通过 Runtime 正常启动一个 Execution，再进行后续操作。

这里我们在 main 中创建临时的 Execution 对象并使其运行。

不过这些都属于 Runtime 内部对象。

业务代码只操作：
```text
Runtime API
```
这也是整个第二季一直坚持的设计原则。

## 13. Checkpoint 并不等于 Resume

到这里，Checkpoint 已经可以保存 Execution 状态。

但我们需要注意：

> 保存状态和恢复状态，是两个不同的问题。

Checkpoint：
```text
Execution
    ↓
保存
```
Resume：
```text
Checkpoint
    ↓
加载
    ↓
恢复 Execution
    ↓
继续执行
```
因此，本篇只完成：

> 保存

而没有实现：

> 恢复

这不是遗漏。

恰恰相反，我们应该有意识地把这两个问题拆开。

因为当一个 Execution 被保存下来以后，还存在一个新的问题：

> 什么时候应该恢复它？

如果：
```text
Execution A
Execution B
Execution C
```
都处于等待状态，那么 Runtime 什么时候执行哪个？

如果同时有：
```text
1000 个 Execution
```
又应该如何控制并发？

这已经不是 Checkpoint 的问题，而是 Scheduler 的问题。

## 14. Checkpoint 的工程价值

现在回头看 Checkpoint，它解决的其实不是简单的“数据保存”。

它改变的是 Agent Runtime 的生命周期模型。

没有 Checkpoint：
```text
Runtime Process
      │
      ▼
Execution
      │
      ▼
Process Exit
      │
      ▼
Execution Lost
```
有 Checkpoint：
```text
Runtime Process A
      │
      ▼
Execution
      │
      ▼
Checkpoint
      │
      ▼
Process Exit
```
未来：
```text
Runtime Process B
      │
      ▼
Load Checkpoint
      │
      ▼
Continue Execution
```
于是：

> Execution 开始拥有独立于 Runtime 进程的生命周期。

这正是 Long Running Agent 能够可靠运行的重要基础。

## 15. 当前 Runtime 的结构

到这一篇结束，我们的 Runtime 结构变成：
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
│   │   ├── engine.go
│   │   └── store.go
│   │
│   ├── event/
│   │   └── event.go
│   │
│   ├── checkpoint/
│   │   ├── checkpoint.go
│   │   ├── store.go
│   │   └── handler.go
│   │
│   └── runtime/
│       └── runtime.go
│
└── go.mod
```
相比前一篇，我们真正新增的只有：
```text
internal/checkpoint/
├── checkpoint.go
├── store.go
└── handler.go
```
然后让 Runtime 在初始化的时候：
```text
创建 Checkpoint Store
        ↓
创建 Checkpoint Handler
        ↓
注册到 Event Bus
```
这就是一个非常典型的 Runtime 能力扩展方式。

## 总结

前面的文章中，我们通过 State Machine 解决了：

> Agent 如何管理自己的生命周期。

通过 Event 解决了：

> Runtime 中发生的事情如何被其他组件感知。

这一篇，我们继续向前一步：

> 如何把 Execution 的生命周期状态保存下来。

答案就是 Checkpoint。

它的核心结构非常简单：
```text
Execution
    +
Session
    ↓
Checkpoint
```
但真正重要的是 Checkpoint 如何进入 Runtime。

我们没有让：
```text
Execution
```
自己保存状态。

也没有让：
```text
Execution Engine
```
直接调用 Checkpoint。

而是：
```text
Execution
    ↓
Engine
    ↓
Event
    ↓
Event Bus
    ↓
Checkpoint Handler
    ↓
Checkpoint Store
```
这样，Checkpoint 就成为 Runtime 中一个独立的能力。

最终：
```text
                    Runtime
                       │
                       ▼
               Execution Engine
                       │
                       ▼
                   Execution
                       │
                  State Change
                       │
                       ▼
                     Event
                       │
                       ▼
                  Event Bus
                       │
                       ▼
              Checkpoint Handler
                       │
                       ▼
               Checkpoint Store
```
Checkpoint 解决的是：

> 记住 Agent 走到了哪里。

而当我们真正拥有了这些状态之后，下一个问题自然出现：

> Runtime 什么时候应该让这些 Execution 继续运行？

这就进入了下一篇：

Scheduler：让 Runtime 决定什么时候执行什么

---
## 版本

**Version: v0.5-checkpoint**

**https://github.com/aigc-engineering/aigc-agent-runtime/releases/tag/v0.5-checkpoint**

本章在上一版本 Runtime 的基础上，新增 Checkpoint 能力。

新增目录：
```text
internal/checkpoint/
├── checkpoint.go
├── store.go
└── handler.go
```
新增能力：
```text
Event
  ↓
Checkpoint Handler
  ↓
Checkpoint Store
```
Runtime 初始化时完成 Checkpoint Handler 注册，使每次 Event 发生时都能够保存当前 Execution 的状态快照。

本章暂不实现 Checkpoint Resume，恢复与调度能力将在后续 Scheduler 章节中继续实现。