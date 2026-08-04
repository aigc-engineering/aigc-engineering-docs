# State Machine：让 Runtime 拥有生命周期。

## 前言
上一篇，我们对 Runtime 做了第一次重构。

执行 Agent 的职责被抽离到了 Execution Engine 中，Runtime 开始拥有了真正的执行内核。

整个执行流程变成了：
```
Runtime
    │
    ▼
Execution Engine
    │
    ▼
Agent
```
虽然架构更加清晰了，但 Execution 仍然只是一个普通的数据结构。
```Go
type Execution struct {
    ID     string
    Status Status
}
```
状态只是一个字段。

它并不会约束 Execution 的行为。

例如下面这段代码：
```Go
exec.Status = StatusCompleted
exec.Status = StatusRunning
```
编译不会报错。

但是对于一个 Runtime 来说，这显然是不合理的。

Execution 一旦完成，就不应该再回到 Running。

这说明，我们真正需要的不是一个状态字段，而是一个**状态机**。

## 1. 状态不是数据，而是生命周期
很多系统都会保存一个 Status，例如：

订单：
```Text
Created
Paid
Shipped
Completed
```

任务：
```Text
Pending
Running
Success
Failed
```

Workflow：
```Text
Pending
Running
Paused
Completed
```

但是，真正重要的不是状态本身，而是：
> 状态之间允许如何流转。

例如：
```
Created
    │
    ▼
Running
    │
 ┌──┴───────┐
 ▼          ▼
Completed  Failed
```

允许：
```
Created → Running
Running → Completed
Running → Failed
```

但是绝不能：
```
Completed → Running
Failed → Created
```
真正约束 Runtime 的，从来不是 Status，而是 Transition（状态转换）。

## 2. 为什么 Runtime 必须管理状态转换？
假设现在仍然使用上一篇的实现：
```Go
exec.Status = StatusRunning

err := agent.Execute(sess)

if err != nil {
    exec.Status = StatusFailed
    return err
}

exec.Status = StatusCompleted
```
目前没有问题。

但是下一篇加入 Checkpoint 呢？

或者，Humain-in-the-loop:
```
Running
↓
WaitingHuman
↓
Running
```

或者，Scheduler：
```
Running
↓
Paused
↓
Running
```
Execution 的生命周期开始越来越复杂。

如果所有地方都可以直接修改：
```Go
exec.Status = ...
```
那么整个 Runtime 很快就会失控。

因此，我们需要把所有状态修改统一管理。

## 3. 让 Execution 自己管理生命周期
从这一篇开始，我们不再直接修改 Status。

而是把状态变化封装到 Execution 中。

目录结构：
```Go
internal/
├── execution/
│   ├── execution.go
│   └── engine.go
```
----------------------
**execution.go**
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

func New(id string) *Execution {
	return &Execution{
		ID:     id,
		Status: StatusCreated,
	}
}

func (e *Execution) Start() {
	e.Status = StatusRunning
}

func (e *Execution) Complete() {
	e.Status = StatusCompleted
}

func (e *Execution) Fail() {
	e.Status = StatusFailed
}
```
相比上一篇：
```Go
exec.Status = StatusRunning
```
现在变成：
```Go
exec.Start()
```
Execution 不再暴露生命周期细节。

它开始自己管理自己的状态。

## 4. Engine 不再关心状态细节
Engine 的职责也开始变得更加简单。
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

	exec := New("run-001")

	exec.Start()

	if err := agt.Execute(sess); err != nil {
		exec.Fail()
		return err
	}

	exec.Complete()

	return nil
}
```
整个 Engine 已经不再关心：
```Go
StatusRunning
StatusCompleted
StatusFailed
```
它只负责：
```
Start()
↓
Execute()
↓
Complete()
```
生命周期交给 Execution。

执行流程交给 Engine。

职责开始真正分离。

## 5. 为什么现在还不是完整的状态机？
看到这里，很多人可能会问：

这不就是几个方法吗？

为什么叫状态机？

因为我们目前只是完成了第一步：

> 把状态修改收口。

真正的状态机，还需要管理：

* 合法状态转换
* 非法状态校验
* Transition
* Hook
* Event

例如, 以后：
```Go
exec.Complete()
```
内部可能变成：
```Go
func (e *Execution) Complete() error {

	if e.Status != StatusRunning {
		return ErrInvalidTransition
	}

	e.Status = StatusCompleted

	return nil
}
```
再往后：
```
Running
↓
Completed
↓
Publish Event
↓
Save Checkpoint
↓
Notify Observer
```
所有这些能力，都可以在状态转换时自动完成。

所以，本篇实现的是：

> Execution 生命周期的第一版。

真正的企业级状态机，我们将在后面的 Event 和 Checkpoint 中继续完善。

## 总结
第一季我们讨论了：

> 为什么 Agent 必须是状态机。

那是设计思想。

而这一篇，我们开始把这种思想落到代码中。

Execution 不再只是一个保存状态的数据结构。

它开始拥有自己的生命周期。

从这一刻开始：
```
Created
    │
    ▼
Running
    │
 ┌──┴────────┐
 ▼           ▼
Completed   Failed
```
Runtime 也终于开始具备真正的生命周期管理能力。

这也是后续：

* Event
* Checkpoint
* Scheduler
* Human-in-the-loop
* Memory

能够建立起来的基础。

## 本章 Git Tag
`v0.3-state-machine`