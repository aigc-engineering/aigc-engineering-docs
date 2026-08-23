# Observability：让 Runtime 可观测

## 前言

前面几篇文章里，我们已经把 Runtime 的关键能力逐步补齐：

- State Machine：定义执行生命周期
- Event Bus：连接事件流
- Checkpoint：保存执行快照
- Scheduler：决定任务执行顺序
- Tool Runtime：让 Agent 调用外部能力
- Memory：让 Runtime 拥有长期知识
- Human-in-the-loop：让执行在关键节点等待用户决策

到这里，Runtime 已经从“能跑起来”走到了“能被管理、恢复、协作”。

但工程上还有一个问题没有被彻底解决：

> 运行起来不等于可观测，尤其是长生命周期的 Agent 执行，必须能够被看懂、被追踪、被排查。

这也是第二季第10篇要讲的重点：

> Observability。

如果没有 Observability，那么 Runtime 仍然只有一堆状态和日志，而没有完整的执行视图。长任务会变成“黑盒”，错误会变成“猜测”，恢复会变成“靠运气”。

因此，这一章不讨论“怎么写一个 logger”，而讨论更工程化的问题：

- Runtime 为什么必须具备观测能力
- 观测应该落在哪些代码位置
- 事件流如何把状态变更和排查链路连接起来
- 什么时候观测能力决定了 Agent 是否能上线生产
- 一个真实的 Runtime，应该记录哪些字段，才能支撑复盘和恢复

换句话说，logger 解决的是“输出一串字符串”，而 Observability 解决的是“把运行过程解释清楚”。

在 Agent Runtime 中，这个差别非常关键，因为我们面对的不只是函数调用，而是：

- 执行状态迁移
- 用户决策插入
- 工具调用结果回流
- checkpoint 与 resume
- memory 的写入时机
- 各类失败的可追踪链路

如果没有这些结构化信号，排查时只能靠猜，这个 Runtime 就很难进入生产环境。

---

## 1. 为什么 Runtime 必须有 Observability？

很多人会以为，Agent 运行时只需要“能调用模型、能跑任务、能返回结果”就够了。

但真实生产问题远不止这些。

### 1.1 长任务不是短任务

传统服务通常是：

```text
request -> handler -> response
```

而 Agent Runtime 往往是：

```text
request -> analyze -> plan -> tool -> wait -> resume -> decide -> finish
```

这个过程可能跨越：

- 多轮工具调用
- 多次状态切换
- 若干次 checkpoint
- 若干次用户参与
- 多个失败和恢复

如果没有观测能力，就很难回答下面这些关键问题：

- 任务为什么停在某个节点
- 执行到哪里了
- 这次失败是来自模型、工具还是状态错误
- 什么时候需要重试
- 用户输入究竟在什么上下文下被消费
- 这次执行和上一轮执行有什么区别

这些问题，正是长生命周期 Agent 能否真正落地的关键。

### 1.2 Agent 的执行是非确定性的

传统程序的执行路径，通常可以通过代码分支看清楚。

但 Agent 运行时的执行过程，往往依赖：

- 用户输入语义
- 上下文状态
- 记忆内容
- 工具返回结果
- 模型判断
- 外部系统状态

因此，Runtime 不只是“跑代码”，而是在“管理一个非确定性的状态演化过程”。

这意味着，它不仅需要执行能力，还需要：

- 时间线
- 状态变更记录
- 事件链路
- 执行关联 ID
- 失败和恢复的证据

这正是 Observability 的价值所在。

---

## 2. Observability 不是日志，而是状态流的可追踪性

很多工程团队会把 Observability 理解成“打日志”。

但日志只是表层能力，真正的 Observability 是：

> 我能用一条清晰的执行轨迹，把系统从开始到结束重新串起来。

在当前代码里，最关键的可观测能力，不是随处打印 `fmt.Println()`，而是事件流。

看 [internal/event/event.go](internal/event/event.go) 里的定义：

```go
const (
	TypeExecutionStarted   Type = "execution.started"
	TypeExecutionPaused    Type = "execution.paused"
	TypeExecutionCompleted Type = "execution.completed"
	TypeExecutionFailed    Type = "execution.failed"
	TypeExecutionResumed   Type = "execution.resumed"
)

type Event struct {
	Type        Type
	ExecutionID string
	Status      string
	Session     session.Session
}
```

这个结构非常关键：

- 事件中带有 `ExecutionID`
- 事件中带有 `Status`
- 事件中带有 `Session`

也就是说，事件不只是“发生了某事”，而是已经记录了：

- 发生在哪里
- 发生在什么状态
- 对应哪个 Session
- 对于哪个执行任务

这正是观测能力的最小模型。

### 2.1 一个真实的事件，不只是 Type，还要有上下文

在真实 Runtime 中，单纯的 `Type` 还不够。我们还需要知道：

- 谁触发了这个事件
- 这次事件属于哪个 Session
- 当前执行的 step 是什么
- 任务是在哪个阶段暂停/恢复
- 这个状态转换的原因是什么
- 此时工具调用是否成功，返回值是否符合预期

最小实践中，事件结构可以扩展成下面这个样子：

```go
package event

type Event struct {
	Type        Type
	ExecutionID string
	SessionID   string
	Status      string
	Step        string
	ToolName    string
	Reason      string
	Actor       string
	Timestamp   string
	Metadata    map[string]any
}
```

这里的字段非常关键：

- `ExecutionID`：唯一标识一次执行
- `SessionID`：标识用户会话上下文
- `Step`：标识在执行链路中的阶段，例如 `tool_call` / `memory_write` / `user_input_wait`
- `ToolName`：标识当前工具或动作
- `Reason`：说明为什么暂停、失败或恢复
- `Actor`：标识是 `system`、`user`，还是 `agent`
- `Metadata`：允许后续扩展更多运行时数据

这样一来，观测系统就不再是“打印几行日志”，而是形成了事件链路。

### 2.2 一条真实执行链路，应该长什么样

一个真实的 Agent Runtime 可能会发生这样的时序：

```text
execution.started
  -> plan.step_created
  -> tool.search.called
  -> tool.search.succeeded
  -> memory.summary_written
  -> execution.paused (waiting user confirm)
  -> user.confirmed
  -> execution.resumed
  -> tool.database.update.called
  -> execution.completed
```

这条链路非常重要，因为它说明：

- 任务并不是一个黑盒单点状态
- 中间发生了什么，用户参与在哪一段被接入
- 哪些动作对最终结果有影响
- 失败和恢复都能沿着一条链重新追踪

这正是 Observability 在 Agent Runtime 中最有价值的地方。

---

## 3. Runtime 中的 Observability 代码落点

在当前代码中，Observability 的落点其实非常清晰。

### 3.1 Runtime 负责整合事件总线

看 [internal/runtime/runtime.go](internal/runtime/runtime.go)：

```go
func NewRuntime(
	name string,
	scheduler *scheduler.Scheduler,
	cp checkpoint.Store,
	m memory.Store,
) *Runtime {
	bus := event.NewBus()

	pauseStore := execution.NewMemoryPauseStore()
	inputHandler := execution.NewDefaultUserInputHandler(pauseStore)

	return &Runtime{
		engine:       execution.NewEngine(name, bus, pauseStore, inputHandler),
		bus:          bus,
		checkpoint:   cp,
		memory:       m,
		pauseStore:   pauseStore,
		inputHandler: inputHandler,
		Scheduler:    scheduler,
	}
}

func (r *Runtime) Subscribe(handler event.Handler) {
	r.bus.Subscribe(handler)
}
```

这里的关键在于：

> Runtime 把事件总线暴露出来，让外部模块可以订阅执行事件。

也就是说，Runtime 的观察者不是只有日志器，而是任何能够处理 `event.Event` 的监听器。

### 3.2 Engine 在状态转移时发布事件

再看 [internal/execution/engine.go](internal/execution/engine.go)：

```go
func (e *Engine) Execute1(
	agent agent.Agent,
	sess *session.Session,
) error {
	exec := NewExecution(sess.ID)
	if err := exec.Start(); err != nil {
		return err
	}

	e.bus.Publish(event.Event{
		Type:        event.TypeExecutionStarted,
		ExecutionID: exec.ID,
		Status:      string(exec.Status),
		Session:     *sess,
	})

	err := agent.Execute(sess)
	if err != nil {
		if transitionErr := exec.Fail(); transitionErr != nil {
			return transitionErr
		}
		e.bus.Publish(event.Event{
			Type:        event.TypeExecutionFailed,
			ExecutionID: exec.ID,
			Status:      string(exec.Status),
			Session:     *sess,
		})
		return err
	}

	if err := exec.Complete(); err != nil {
		return err
	}

	e.bus.Publish(event.Event{
		Type:        event.TypeExecutionCompleted,
		ExecutionID: exec.ID,
		Status:      string(exec.Status),
		Session:     *sess,
	})
	return nil
}
```

这段代码非常典型：

- 执行前发布 started
- 执行中可能失败，发布 failed
- 执行成功后发布 completed
- 暂停和恢复时也有对应事件

这说明 Runtime 的 Observability 不是静态日志，而是“状态变更事件流”。

### 3.3 关键判断：Observability 不是新内核，而是 Event Bus 的消费层

这一点在整个章节里最重要：

> 我们并不需要重写 Runtime 的调度器、状态机或执行器，来实现可观测性。

在当前设计中，Runtime 已经完成了最关键的一步：在状态转换时发布事件。真正的观测层只是把这些事件消费掉，并把它们转成：

- 日志
- 监控指标
- 失败追踪
- 执行时序
- 复盘和审计数据

也就是说，最小可行的 Observability 是下面这个模式：

```text
Runtime 发布事件
   ↓
观察者订阅事件
   ↓
事件被写入日志 / 指标 / 存储 / 监控平台
```

这并不要求修改状态机或调度器，只需要让使用者充分消费 Event Bus。

如果只是想在工程里做“可观测”，并不需要新增一套 Runtime 内部机制；最核心的能力，本来就已经存在于 Event Bus 之中。

这也是为什么 Event Bus 在 Agent Runtime 里，不只是“一个消息通道”，而是“把执行事实变成审计和复盘材料的总入口”。

### 3.4 Memory 也可以成为观测的输出端

在 [internal/runtime/runtime.go](internal/runtime/runtime.go) 里，还有一个典型闭环：

```go
func (r *Runtime) SubscribeMemory() {
	r.bus.Subscribe(func(e event.Event) {
		switch e.Type {
		case event.TypeExecutionCompleted:
			_ = r.Remember(e.Session.ID, "latest_summary", e.Session.Output, "summary")
		case event.TypeExecutionFailed:
			_ = r.Remember(e.Session.ID, "last_error", e.Session.Output, "error")
		}
	})
}
```

这里非常重要：

> 事件不仅给外部系统观测，还能够写入 Memory。

这意味着 Observability 不只是“看”，还在于把一段业务事件转成可复用的状态信息。

也就是说，观测事件本身，就能驱动后续的决策与记忆更新。

---

## 4. 可观测性应该怎么设计？

从工程实践上看，Observability 的最小设计，有 4 个层次：

### 4.1 事件层：记录状态变化

这是最底层的入口，主要解决：

- 任务开始了没有
- 任务暂停了没有
- 任务恢复了没有
- 任务成功/失败了没有

对应的事件类型已经在 event.go 里定义好了。

### 4.2 上下文层：记录执行关联信息

观测必须让事件能和任务、用户、Session、工具调用绑定起来。

也就是说：

```text
ExecutionID + SessionID + Status + Timestamp
```

不能只有“事件发生了”，而必须回答：

> 哪个任务发生了什么变化？

### 4.3 记忆层：把事件转成决策输入

观测不仅用于排查，也用于未来决策。

例如：

- 成功时保存 summary
- 失败时保存 last_error
- 暂停时保存用户选择路径

这意味着观测时序本身，就是 Agent 长期记忆的源头。

### 4.4 可视化层：把事件串成执行链路

最终的观测系统需要把事件渲染为：

- 执行时间线
- 状态流转图
- 失败原因链路
- 用户决策节点
- Resume 来源

如果没有这层，Observability 只停留在终端日志，不足以支撑企业级 Agent。

---

## 5. 为什么 Observability 是 Agent Runtime 的核心能力，而不只是“运维需求”？

这是本章最关键的一点。

很多人会把 Observability 看成运维问题：

> 监控、日志、告警，放在部署层就可以了。

但在 Agent Runtime 中，Observability 是业务能力的一部分。

因为 Agent 的核心问题，不是“任务有没有执行”，而是：

> 执行过程为什么是这个样子，以及下一步为什么该这么做。

这就要求 Runtime 必须把状态演化变成可审计对象。

否则，一旦发生以下问题：

- 任务卡住了
- 任务恢复后状态错乱
- 用户输入被错误消费
- 记忆写入了错误数据
- 工具结果未回流到执行状态

我们都无法从“现象”恢复“原因”。

对于 Agent 来说，这种问题并不只是排查问题，而是直接影响 Agent 是否能稳定运行。

---

## 6. 最小可用的 Observability 实现

在当前结构下，一个最小实现，通常可以从事件总线开始：

```go
package main

import (
	"fmt"
	"time"

	"github.com/aigc-engineering/aigc-agent-runtime/internal/event"
)

func main() {
	bus := event.NewBus()
	bus.Subscribe(func(e event.Event) {
		fmt.Printf("[%s] type=%s execution=%s status=%s session=%s\n",
			time.Now().Format(time.RFC3339),
			e.Type,
			e.ExecutionID,
			e.Status,
			e.Session.ID,
		)
	})
}
```

这段代码并不复杂，但它体现了 Observability 的本质：

- 事件被订阅
- 状态变化被记录
- 执行轨迹被保留下来

如果进一步扩展，可以在这个基础上接入：

- 结构化日志系统：把 event 以 JSON 方式落库
- 执行时间表：用 executionID 作为主键，绘制状态时序
- 任务监控仪表盘：展示 started / paused / resumed / failed / completed 的数量
- 失败回放：基于事件链路重放执行过程
- 消息中间件：把事件发到 Kafka / NATS / Redis Stream，供监控和分析平台消费
- 审计日志：把用户确认、工具执行、恢复动作保留成证据

这就是从“Demo 级事件”升级到“生产级 Observability”。

### 6.1 一个最小但更贴近生产的观察器

现在的最小示例已经说明了事件监听的思路，但在生产环境里，单纯打印到终端并不够。我们更希望收到的是：

- 结构化字段
- 可查询的 executionID
- 可过滤的 sessionID
- 可追踪的 step 和 tool
- 可以用于复盘的时间戳

因此，一个更接近工程实践的版本会像下面这样：

```go
package main

import (
	"encoding/json"
	"fmt"
	"time"

	"github.com/aigc-engineering/aigc-agent-runtime/internal/event"
)

func main() {
	bus := event.NewBus()

	bus.Subscribe(func(e event.Event) {
		payload, _ := json.Marshal(map[string]any{
			"type":      e.Type,
			"execution": e.ExecutionID,
			"session":   e.Session.ID,
			"status":    e.Status,
			"time":      time.Now().Format(time.RFC3339),
		})
		fmt.Println(string(payload))
	})

	bus.Publish(event.Event{
		Type:        event.TypeExecutionStarted,
		ExecutionID: "exec-001",
		Status:      "running",
		Session:     session.Session{ID: "sess-001"},
	})

	bus.Publish(event.Event{
		Type:        event.TypeExecutionPaused,
		ExecutionID: "exec-001",
		Status:      "paused",
		Session:     session.Session{ID: "sess-001"},
	})
}
```

这里的区别是：

- 事件被转换成 JSON，而不是裸字符串
- 输出结构更加稳定，能够被日志平台、ELK、监控系统消费
- execution 和 session 的关联信息得以保留
- 情况一旦扩展到多个工具、多个步骤，后续就能做聚合分析

这才更接近真正的生产 Observability，而不只是局部调试输出。

---

## 7. 一个工程上的判断：日志能告诉你“发生了什么”，观测能告诉你“为什么”

这是一条非常重要的工程原则。

### 日志

日志可以回答：

- 某个函数执行了
- 某个状态切换了
- 某个工具返回了什么

### Observability

Observability 可以回答：

- 这个状态为什么出现
- 它和前后状态有什么关系
- 哪一步是失败点
- 恢复后是否真的回到同一条执行链路
- 哪个用户输入改变了执行路径

因此，Observability 的价值，不在于记录更多文字，而在于：

> 让复杂的执行过程可以被还原和诊断。

这对 Agent Runtime 来说，是非常关键的工程能力。

---

## 8. 总结

这一章的结论很简单：

> Runtime 不是“一个会跑的盒子”，而是一个需要被观测、被诊断、被恢复、被管理的执行环境。

在当前代码中，事件总线已经为这个能力打下了基础：

- [internal/event/event.go](internal/event/event.go) 定义了事件和状态语义
- [internal/execution/engine.go](internal/execution/engine.go) 负责在状态流转时发布事件
- [internal/runtime/runtime.go](internal/runtime/runtime.go) 把事件总线暴露给外部观察者
- [internal/runtime/runtime.go](internal/runtime/runtime.go) 里的 `SubscribeMemory()` 把事件进一步转成记忆输入

这说明 Observability 不是一个后置功能，而是 Runtime 的核心架构能力之一。

如果没有它，Runtime 就只是一个黑盒，无法支持：

- 长链路任务排查
- 关键状态恢复
- 用户介入审核
- 执行历史追踪
- 任务失败分析

因此，真正可用的 Agent Runtime，必须具备两个能力：

1. 能执行
2. 能被看懂

而这两者之间的桥梁，正是 Observability。

---

## 代码版本：最小可用的观察者

下面是一版最小但完整的实现思路：

```go
package main

import (
	"fmt"
	"time"

	"github.com/aigc-engineering/aigc-agent-runtime/internal/event"
)

func main() {
	bus := event.NewBus()

	bus.Subscribe(func(e event.Event) {
		fmt.Printf("[%s] type=%s execution=%s status=%s session=%s\n",
			time.Now().Format(time.RFC3339),
			e.Type,
			e.ExecutionID,
			e.Status,
			e.Session.ID,
		)
	})

	bus.Publish(event.Event{
		Type:        event.TypeExecutionStarted,
		ExecutionID: "exec-001",
		Status:      "running",
		Session:     session.Session{ID: "sess-001"},
	})

	bus.Publish(event.Event{
		Type:        event.TypeExecutionPaused,
		ExecutionID: "exec-001",
		Status:      "paused",
		Session:     session.Session{ID: "sess-001"},
	})
}
```

如果我们再进一步，把这个观察器接到一个数据库、消息中间件或者监控系统里，就能从“本地事件流”升级为“生产级 Observability”。

这时，Runtime 不再只是一段可以运行的代码，而是一个真正可以被监控、追踪、恢复和审计的执行系统。

完整的代码示例可以在 [aigc-agent-runtime](https://github.com/aigc-engineering/aigc-agent-runtime/releases/tag/v0.10-observability) 找到。