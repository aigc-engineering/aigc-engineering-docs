# Scheduler：让 Runtime 真正跑起来

## 前言
在前面的几篇里，我们已经把 Agent Runtime 的几个关键能力串起来了：

- 通过 **State Machine** 定义 Agent 的生命周期
- 通过 **Event** 记录运行时的状态变化
- 通过 **Checkpoint** 保存 Agent 的执行快照

前面的能力解决的是：

> Agent 现在是什么状态，以及上一次执行到了哪里。

但这还不够。真正的 Runtime 还需要回答另一个更关键的问题：

> 下一步，哪个任务该执行，什么时候执行？

如果 Runtime 中存在大量 Execution：
```text
Execution A → Ready
Execution B → Waiting
Execution C → Ready
Execution D → Paused
Execution E → Ready
...
```
那么简单地循环调用 `execution.Run()` 显然不够用。

因为真实运行时必须考虑：

- 哪个 Execution 应该先执行？
- 哪些 Execution 当前可以被调度？
- 允许多少个 Execution 同时运行？
- 某个 Execution 失败后该如何处理？
- 是否应该重试？
- 某个 Worker 忙碌时，其他任务该怎么办？
- 如何避免单个 Execution 长时间占用资源？

这些问题都指向同一个核心组件：

> Scheduler。

Scheduler 的职责不是直接执行 Agent，
而是决定：

> 谁可以执行，以及什么时候执行。

因此，从这一篇开始，Runtime 的结构会进一步扩展：
```text
                  Runtime
                     │
             ┌───────┴───────┐
             │               │
         Scheduler          Engine
             │               │
             └───────┬───────┘
                     ↓
                 Execution
```

---
## 1. 为什么 Runtime 需要 Scheduler？

最简单的 Agent Runtime，可以直接执行：
```text
Request
   ↓
Execution
   ↓
Engine
   ↓
Run
```
这种方式在 Demo 中没有问题。

但进入生产环境之后，Execution 数量很快就会增加。

例如：
```text
Execution-001
Execution-002
Execution-003
...
Execution-1000
```
如果每一个 Execution 都直接启动一个 Goroutine：
```Go
go execution.Run()
```
很快就会遇到资源问题。

因为 Agent 执行通常并不是一个轻量级操作。
它可能涉及：
```text
LLM
Tool
HTTP
Database
File
Model Inference
External Service
```
因此 Runtime 必须控制：
```text
并发数
执行顺序
资源使用
```
这就是 Scheduler 存在的原因。

## 2. Scheduler 到底负责什么？
Scheduler 很容易和 Engine 混在一起。

因此首先要明确两者的职责。

### 2.1 Engine 负责“怎么执行”

Engine 面对的是一个具体 Execution：
```text
Execution
    ↓
Engine
    ↓
执行
```
它负责：
```text
加载 Execution
驱动 State Machine
执行 Agent
产生 Event
更新状态
```
Engine 解决的是：

> How —— 怎么执行。

### 2.2 Scheduler 负责“什么时候执行”

Scheduler 面对的是多个 Execution：
```text
Execution A
Execution B
Execution C
Execution D
```
它需要决定：
```text
谁先执行？
谁等待？
谁可以进入执行队列？
谁暂时不能执行？
```
所以 Scheduler 解决的是：

> When / Which —— 什么时候执行谁。

可以简单理解成：
```text
Scheduler
    ↓
选择 Execution
    ↓
Engine
    ↓
执行 Execution
```
因此两者之间应该保持清晰的边界：
```text
Scheduler
    │
    │ 决定执行谁
    ▼
Engine
    │
    │ 执行
    ▼
Execution
```
Scheduler 不应该直接修改 Execution 的状态。

它只负责调度。

---
## 3. Scheduler 最核心的数据结构：Queue

既然 Scheduler 要决定：

> 谁先执行？

那么最自然的数据结构就是：

> Queue。

例如：
```text
Execution A
Execution B
Execution C
```
进入 Scheduler：
```text
┌─────────────┐
│ Execution A │
├─────────────┤
│ Execution B │
├─────────────┤
│ Execution C │
└─────────────┘
```
Scheduler 每次取出一个：
```text
Execution A
    ↓
Engine
```
执行完成之后，再处理：
```text
Execution B
```
最简单的 Scheduler 就是：

> FIFO —— First In, First Out。

---
## 4. 定义 Scheduler
现在开始进入代码。

这一篇我们只在前面的 Runtime 基础上增加 Scheduler。

新增：
```text
internal/scheduler/
└── scheduler.go
```
首先定义 Scheduler：
```Go
package scheduler

type Scheduler struct {
	queue chan string
}

func New(buffer int) *Scheduler {
	return &Scheduler{
		queue: make(chan string, buffer),
	}
}

func (s *Scheduler) Submit(executionID string) {
	s.queue <- executionID
}

func (s *Scheduler) Next() string {
	return <-s.queue
}
```
这里的 Scheduler 非常简单。

它目前只有两个核心操作：
```text
Submit
   ↓
进入队列

Next
   ↓
取出 Execution
```
例如：
```Go
scheduler.Submit("execution-001")
scheduler.Submit("execution-002")
scheduler.Submit("execution-003")
```
队列：
```text
execution-001
execution-002
execution-003
```
然后：
```Go
id := scheduler.Next()
```
得到：
```text
execution-001
```

---
## 5. 为什么 Scheduler 不直接执行 Execution？
这里有一个非常重要的设计原则。

我们可能很自然地写：
```Go
func (s *Scheduler) Run() {
	for {
		id := s.Next()
		execution.Run(id)
	}
}
```
但是这样 Scheduler 就开始依赖 Execution。

更重要的是：

> Scheduler 开始承担执行职责。

最终就会变成：
```text
Scheduler
├── Queue
├── Execution
├── State
├── Retry
├── Worker
└── ...
```
Scheduler 会越来越复杂。

因此我们应该保持：
```text
Scheduler
    ↓
选择 Execution
    ↓
Engine
    ↓
执行
```
Scheduler 只需要知道：
```text
executionID
```
而不需要知道：
```text
Execution 内部是什么
Agent 如何执行
Tool 如何调用
State 如何转换
```

---
## 6. Runtime 负责组装 Scheduler
把 Scheduler 它放进 Runtime。

Runtime：
```Go
type Runtime struct {
	engine     *execution.Engine
	scheduler  *scheduler.Scheduler
}
```
初始化：
```Go
func New() *Runtime {
	bus := event.NewBus()

	checkpointStore := checkpoint.NewMemoryStore()

	bus.Subscribe(
		checkpoint.Handler(checkpointStore),
	)

	engine := execution.NewEngine(bus)

	s := scheduler.New(100)

	return &Runtime{
		engine:    engine,
		scheduler: s,
	}
}
```
这里 Runtime 又承担了一个熟悉的职责：

> 组装组件。

最终：
```text
Runtime
├── Event Bus
├── Engine
├── Scheduler
└── Checkpoint
```

---
## 7. 一个真正可运行的 Scheduler

为了把 Runtime 真正跑起来，Scheduler 不只需要一个队列，还需要三个关键能力：

- 可以提交任务
- 可以在多个 Goroutine 中消费任务
- 任务执行时需要一个统一接口

因此，一个最小但完整的 Scheduler，至少应该包含：

```Go
type Task struct {
	ID      string
	Name    string
	Payload any
}

type Executor interface {
	Execute(task Task) error
}

type Scheduler struct {
	queue chan Task
}

func NewScheduler(buffer int) *Scheduler {
	return &Scheduler{
		queue: make(chan Task, buffer),
	}
}

func (s *Scheduler) Submit(task Task) {
	s.queue <- task
}

func (s *Scheduler) Run(workers int, executor Executor) {
	for i := 0; i < workers; i++ {
		go func() {
			for task := range s.queue {
				if err := executor.Execute(task); err != nil {
					continue
				}
			}
		}()
	}
}
```

这里的核心思路非常直接：

- `Submit` 把任务放进队列
- `Run` 启动多个 Worker Goroutine
- `Executor` 负责真正执行任务

这已经是一个真正可用的 Scheduler 结构，而不是只停留在概念层面。

---

## 8. Worker 是 Scheduler 的执行单元

一个 Scheduler 不应该自己处理业务逻辑，
而应该把任务分发给实际的执行器。

因此我们把执行动作抽象成一个统一接口：

```Go
type Executor interface {
	Execute(task Task) error
}
```

这样做的好处是：

- Scheduler 只控制任务流转
- Engine 负责具体执行
- 不同的执行器可以在同一套调度逻辑上复用

也就是说，Scheduler 的职责是：

> 把任务放进队列，并分发给 Worker。

而具体执行动作，由 `Executor` 完成。

---

## 9. Engine 作为 Executor

在 Runtime 中，真正执行任务的对象通常就是 Engine。

我们可以让 Engine 实现 `Executor` 接口：

```Go
type Engine struct {
	name string
}

func (e *Engine) Execute(task Task) error {
	fmt.Printf("[%s] executing task: %s\n", e.name, task.Name)

	if task.Name == "ping" {
		fmt.Println("pong")
		return nil
	}

	return nil
}
```

这里的 `Execute` 不是去做复杂的调度决策，
它只负责处理一个任务的具体动作。

这正是职责分离的关键：

```text
Scheduler
    ↓
决定谁执行
Engine
    ↓
负责怎么执行
```

---

## 10. Runtime 启动 Scheduler

在 Runtime 中，Scheduler 需要被真正启动。最小可运行的方式是：

```Go
type Runtime struct {
	Scheduler *Scheduler
	engine    *Engine
}

func NewRuntime() *Runtime {
	return &Runtime{
		Scheduler: NewScheduler(16),
		engine:    &Engine{name: "default-engine"},
	}
}

func (r *Runtime) Start() {
	r.Scheduler.Run(3, r.engine)
}
```

这样，Runtime 在启动时就会启动多个 Worker Goroutine，
这些 Worker 会持续从队列中取出任务并执行。

这是一种完整、干净、可运行的 Scheduler 形态：

```text
Runtime
    ↓
Scheduler
    ├── Queue
    └── Worker Pool
        ↓
      Engine
        ↓
      Execute(task)
```

---

## 11. 只发送一个 ping，验证调度器生效

为了验证调度器确实起作用，我们可以发送一个最小任务：

```Go
func main() {
	rt := NewRuntime()
	rt.Start()

	rt.Scheduler.Submit(Task{
		ID:      "ping-001",
		Name:    "ping",
		Payload: "hello",
	})

	select {}
}
```

输出大致如下：

```text
[default-engine] executing task: ping
pong
```

这里的关键不是任务复杂度，
而是说明：

> 一个任务可以被 Scheduler 接收、分发、执行，并得到结果。

这就是一个最小可运行的 Scheduler 版本。

---

## 12. 这个版本为什么是“完整”的

这个版本已经具备最核心的调度能力：

- 任务可以加入队列
- Scheduler 可以启动 Worker
- Worker 可以从队列中拉取任务
- Executor 可以真正执行任务
- Runtime 可以把三者组装起来

因此，它已经不是一个“概念上的调度器”，
而是一个能够运行起来的调度组件。

从工程实践的角度看，这样的设计已经足够支撑后续扩展：

- 重试
- 优先级
- 任务取消
- 资源限制
- 并发控制

这些能力都可以在这个最小结构之上继续演进。

---

## 13. 继续扩展的方向

在最小可运行版本的基础上，后续还可以继续增强：

```text
Scheduler
    ↓
Queue
    ↓
Worker Pool
    ↓
Executor
    ↓
Task Context
    ↓
Retry / Priority / Backpressure
```

例如：

- 为任务增加 `RetryCount`
- 为任务增加 `Priority`
- 为任务增加 `Timeout`
- 为调度器增加 `Stop` 和 `Drain`
- 为事件流增加日志和监控

但这些都是在最小可运行的 Scheduler 之上的增强，
而不是为了掩盖当前设计不完整。

---

## 14. Scheduler 与 Checkpoint 的关系

在这个最小版本里，Scheduler 和 Checkpoint 之间也已经开始形成良好的配合关系。

Checkpoint 负责保存：

```text
Execution 当前状态
```

Scheduler 负责决定：

```text
这个任务下一步是否应该被重新调度
```

两者结合之后，可以形成一条比较完整的 Long Running Runtime 流程：

```text
Checkpoint
    ↓
恢复状态
    ↓
Scheduler
    ↓
重新入队
    ↓
Worker
    ↓
Engine
    ↓
继续执行
```

这说明 Scheduler 既关心“谁来执行”，
也关心“什么时候继续执行”。

---

## 15. 当前 Runtime 的整体结构

综合来看，当前版本可以抽象为：

```text
                         Runtime
                            │
              ┌─────────────┴─────────────┐
              │                           │
          Scheduler                    Event Bus
              │                           │
              ↓                           │
          Queue / Worker               Checkpoint Handler
              │                           │
              ↓                           ↓
            Engine                     Checkpoint Store
              │
              ↓
           Execution
```

这个结构已经具备了非常关键的运行时能力：

- Runtime 负责组装系统
- Scheduler 负责管理任务入队和分发
- Worker 负责消费和执行任务
- Engine 负责具体执行 Agent
- Event 和 Checkpoint 负责状态和恢复

这已经足够说明：

> Scheduler 已经从一个概念层组件，发展成了运行时中的基础控制单元。

---

## 总结

前面的几章，分别解决了 Agent Runtime 中几个关键能力：

- State Machine：定义生命周期
- Event：记录运行时状态变化
- Checkpoint：保存执行快照

这一章继续补充的是：

> Runtime 如何把任务纳入真正可运行的调度模型，并让它在协程中执行。

这一版 Scheduler 已经具备了最关键的基础能力：

- 任务可以被提交
- 调度器可以启动多个 Worker
- Worker 可以从队列中消费任务
- Executor 提供统一的执行接口
- Runtime 可以把这些组件组装起来

这已经是一个真实可运行的 Scheduler，而不仅仅是概念模型。

最小验证方式也非常简单：

```Go
rt := NewRuntime()
rt.Start()
rt.scheduler.Submit(Task{ID: "ping-001", Name: "ping"})
```

只发送一个 `ping`，就可以验证调度器是否真正工作。

这也是后续扩展超过简单 Queue 的基础：

```text
Scheduler
    ↓
Task Queue
    ↓
Worker Pool
    ↓
Executor
    ↓
Engine
```

它为后续继续加入重试、优先级、超时控制和资源管理奠定了基础。

版本

Version: v0.7

https://github.com/aigc-engineering/aigc-agent-runtime/releases/tag/v0.6-scheduler

本章针对 Scheduler 做了最小可运行版本的整理，重点强调：

- Scheduler 需要队列和 Worker
- 需要统一的 `Executor` 接口
- Runtime 需要启动调度器
- 最小验证可以只发送一个 `ping` 任务

这样写法更符合工程实践，也更适合从文章推进到真实代码实现。
