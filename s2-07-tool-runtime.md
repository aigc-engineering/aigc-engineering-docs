# Tool Runtime：不仅仅是 Function Call

## 前言

在前一章里，我们已经把 Scheduler 的最小可运行版本补齐了：

- 任务可以提交到队列
- Worker 可以从队列中消费任务
- Executor 可以对任务做真正的执行
- Runtime 可以把 Scheduler、Engine 和 Checkpoint 组装起来

这意味着 Runtime 已经开始具备“跑起来”的能力。

但如果只是把任务放进队列，Agent 仍然只是一个“能跑的框架”，还没有真正发挥价值。

真正企业级 Agent 的关键，不在于“能跑”，而在于：

> 它能调用外部能力，并把这些能力的执行结果回到 Agent 的状态流中。

这就是 Tool Runtime。

Tool Runtime 的意义，不只是“函数调用”，而是：

- 管理工具注册
- 统一工具执行入口
- 让 Tool 成为 Runtime 内的一个可调度能力
- 保证工具调用结果能够流回 Agent 执行状态
- 支持失败、重试、超时和状态恢复

因此，Tool Runtime 是 Scheduler 真正开始落地的地方。

---

## 1. 为什么 Agent 需要 Tool Runtime？

最早的 Agent 运行时，很容易把所有行为都塞进一个大函数里：

```Go
func (a *Agent) Run(sess *session.Session) error {
    // 1. 分析用户输入
    // 2. 生成 prompt
    // 3. 调 LLM
    // 4. 判断是否调用函数
    // 5. 调工具
    // 6. 再次调用 LLM
    // 7. 返回结果
    return nil
}
```

这种写法在 Demo 里是可以跑的，但它不能代表真实生产能力。

因为真实 Agent 经常需要：

- 查询数据库
- 调用搜索引擎
- 访问外部 API
- 执行代码
- 查文件系统
- 调用自定义工具链

这些动作不只是“一个函数调用”，它们通常具备：

- 需要异步执行
- 可能失败
- 可能超时
- 可能需要重试
- 需要记录调用日志
- 需要把结果回写到状态机
- 需要与 Runtime 的事件流、checkpoint 和 scheduler 结合

因此，Agent 的工具能力不能停留在“函数简写”，而必须被提升为一个 Runtime 层能力：

> Tool Runtime。

---

## 2. Tool Runtime 解决的不是“谁调用”，而是“怎么执行”

很多人会把 Tool Runtime 理解为：

> 直接把函数注册成 map，然后调用它。

这看起来很简单，但它不够完整。

真正的 Tool Runtime 需要回答几个工程层面的问题：

### 2.1 工具注册与发现

系统中有哪些工具？

```text
search_tool
weather_tool
database_tool
code_exec_tool
```

这些工具需要以统一方式注册到 Runtime 中，而不是散落在 Agent 代码里。

### 2.2 工具执行时机

工具何时被调用？

并不是每个请求都直接调用工具，而是要看：

- Agent 当前状态是否允许调用工具
- 是否存在可执行工具任务
- 调度器是否有空闲 Worker
- 当前执行是否异常或暂停

这就要求 Tool Runtime 与 Scheduler 协作，而不是直接在 Agent 中调用。

### 2.3 工具执行结果如何回流

工具调用之后，结果需要回到：

- Execution 状态
- Session 输出
- Event 流
- Checkpoint
- 下一步 Agent 计划

也就是说，Tool 不是一个孤立动作，而是 Runtime 中的一个状态转换节点。

### 2.4 工具可能失败

工具调用可能因为：

- 网络错误
- 参数不合法
- 身份校验失败
- 依赖服务不可达
- 超时

这要求 Tool Runtime 有统一的错误处理层。

---

## 3. Tool Runtime 与 Function Call 的区别

最容易混淆的一点：

> Tool Runtime 不等于 Function Call。

Function Call 更偏“底层调用能力”，例如：

```Go
func search(query string) string
```

而 Tool Runtime 更偏“运行时能力”，它至少包含：

- 工具注册
- 任务封装
- 执行策略
- 调度接入
- 生命周期管理
- 结果回传

因此，Tool Runtime 的抽象层次更高。

可以用一个简单对比来理解：

```text
Function Call
    ↓
直接调用某个函数

Tool Runtime
    ↓
把工具当作一个可调度、可恢复、可观测的 Runtime 组件
```

在生产环境里，真正重要的不是哪个函数被调用了，而是：

> 这个工具任务怎样进入 Runtime，怎样执行，怎样记录状态，怎样在失败后恢复。

---

## 4. 最小 Tool Runtime 设计

回到当前代码结构上看，现在的 Runtime 已经有了几个关键能力：

- Runtime 负责组装组件
- Scheduler 负责任务调度
- Engine 负责执行任务
- Checkpoint 保存状态
- Event Bus 负责事件发布

这个结构非常适合继续往 Tool Runtime 上扩展。

这里我们需要明确一点：

> 代码不应该散落在 `runtime` 包里，而应该按职责拆到不同的 Go 包中。

推荐的最小目录结构如下：

```text
internal/
  tool/
    tool.go         // Tool 接口、Registry
  toolruntime/
    runtime.go      // ToolRuntime 执行入口
    task.go         // ToolTask 及工具任务的封装
  scheduler/
    scheduler.go    // 现有 Task / Scheduler，仍然保留通用任务抽象
  runtime/
    runtime.go      // Runtime 组装 Scheduler / Engine / ToolRuntime
```

这意味着：

- `internal/tool`：负责“工具能力的定义和注册”
- `internal/toolruntime`：负责“工具任务的执行和调度接入”
- `internal/scheduler`：负责“通用任务队列和 Worker”
- `internal/runtime`：负责“装配 Runtime 的最终对象”

对应的最小定义如下：

```Go
package tool

import "context"

type Tool interface {
    Name() string
    Invoke(ctx context.Context, args map[string]any) (any, error)
}

type Registry struct {
    tools map[string]Tool
}

func NewRegistry() *Registry {
    return &Registry{tools: make(map[string]Tool)}
}

func (r *Registry) Register(tool Tool) {
    r.tools[tool.Name()] = tool
}

func (r *Registry) Get(name string) (Tool, bool) {
    tool, ok := r.tools[name]
    return tool, ok
}
```

这里我们引入了两个关键抽象：

- `Tool`：描述一个工具的最小能力
- `Registry`：描述工具的注册表

注意：这已经不是单个函数，而是一个运行时内的能力容器。

---

### 代码落点：ToolTask 放在哪里？

这是本章最容易混淆的点。

结论非常明确：

- `scheduler.Task` 仍然定义在 `internal/scheduler/scheduler.go`
- `ToolTask` 这种工具相关任务结构，建议定义在 `internal/toolruntime/task.go`
- 真正提交给 Scheduler 的数据，不直接是裸 `ToolTask`，而是 `scheduler.Task`，并把工具参数放进 `Payload`

也就是说，代码结构会是：

```Go
// internal/scheduler/scheduler.go
package scheduler

type Task struct {
    ID      string
    Name    string
    Payload any
}
```

```Go
// internal/toolruntime/task.go
package toolruntime

type ToolTask struct {
    ID       string
    ToolName string
    Args     map[string]any
    Session  string
}
```

然后在提交阶段，ToolRuntime 内部把 `ToolTask` 转成通用的 `scheduler.Task`：

```Go
func (r *ToolRuntime) Submit(task ToolTask) {
    r.scheduler.Submit(scheduler.Task{
        ID:   task.ID,
        Name: "tool:" + task.ToolName,
        Payload: map[string]any{
            "tool_name": task.ToolName,
            "args":      task.Args,
            "session":   task.Session,
        },
    })
}
```

这就解释了“为什么 Scheduler 还保留通用 Task，而 ToolRuntime 里又有 ToolTask”。

它们的边界是：

- `ToolTask`：工具领域的业务模型
- `scheduler.Task`：通用的调度任务模型
- `ToolRuntime`：负责两者之间的转换

这样设计最符合 Runtime 的职责分层。
---

## 5. Tool Runtime 为什么要和 Scheduler 结合？

如果工具只是简单地直接执行：

```Go
tool.Invoke(ctx, args)
```

那它只是一个函数调用而已。

真正需要结合 Scheduler，是因为 Tool 调用通常是“任务型”的，而不是“同步片段型”的。

例如：

```text
Agent 决定调用 search_tool
    ↓
Tool Runtime 创建 tool task
    ↓
提交给 Scheduler
    ↓
Worker 负责消费并调用工具
    ↓
执行完成后，结果回流到 Agent
```

这个链路非常重要。它说明：

- Tool Runtime 不是简单的函数库
- Tool 执行需要被调度
- Tool 任务需要被排队和消费
- Tool 结果需要回到 Runtime

这也是为什么 Scheduler 会成为 Tool Runtime 的基础支撑。

---

## 6. 从功能调用到任务调用：代码落点和职责边界

在最小设计中，Scheduler 的任务抽象已经足够简单：

```Go
// internal/scheduler/scheduler.go
package scheduler

type Task struct {
    ID      string
    Name    string
    Payload any
}
```

这是通用任务模型，属于 `internal/scheduler` 包。这一层只关心：

- 任务 ID
- 任务名称
- 任务载荷
- 能否排队和消费

它不关心工具是什么，也不关心业务语义。

但是对 Tool Runtime 来说，工具任务还需要带上更具体的信息：

```Go
// internal/toolruntime/task.go
package toolruntime

type ToolTask struct {
    ID       string
    ToolName string
    Args     map[string]any
    Session  string
}
```

然后由 `ToolRuntime` 负责把业务层的 `ToolTask` 转为通用的 `scheduler.Task`：

```Go
// internal/toolruntime/runtime.go
package toolruntime

import "github.com/aigc-engineering/aigc-agent-runtime/internal/scheduler"

type ToolRuntime struct {
    registry *tool.Registry
    scheduler *scheduler.Scheduler
}

func (r *ToolRuntime) Submit(task ToolTask) {
    r.scheduler.Submit(scheduler.Task{
        ID:   task.ID,
        Name: "tool:" + task.ToolName,
        Payload: map[string]any{
            "tool_name": task.ToolName,
            "args":      task.Args,
            "session":   task.Session,
        },
    })
}
```

这里的关键不是语法，而是语义：

> 工具调用已经从“直接函数调用”变成“可调度任务”。

这就是 Runtime 层思维的转变。

因此，最清晰的职责分层是：

- `internal/tool`：定义工具能力和它的注册表
- `internal/toolruntime/task.go`：定义工具任务结构
- `internal/scheduler/scheduler.go`：定义通用调度任务结构
- `internal/toolruntime/runtime.go`：负责把工具任务转换为调度任务并提交

一个工具调用的整体链路最终是：

```text
Agent / Execution
    ↓
ToolTask
    ↓
ToolRuntime.Submit
    ↓
scheduler.Task
    ↓
Scheduler Queue
    ↓
Worker
    ↓
ToolRuntime.Execute
    ↓
Tool.Invoke
```
---

## 7. ToolRuntime 的最小接口

我们可以把 Tool Runtime 抽象成一个统一能力层：

```Go
package toolruntime

type ToolRuntime struct {
    registry *Registry
    scheduler *scheduler.Scheduler
}

func NewToolRuntime(registry *Registry, sched *scheduler.Scheduler) *ToolRuntime {
    return &ToolRuntime{
        registry: registry,
        scheduler: sched,
    }
}
```

一个最小的执行入口可以是：

```Go
func (r *ToolRuntime) Execute(task scheduler.Task) error {
    payload, ok := task.Payload.(map[string]any)
    if !ok {
        return nil
    }

    toolName, _ := payload["tool_name"].(string)
    args, _ := payload["args"].(map[string]any)

    tool, ok := r.registry.Get(toolName)
    if !ok {
        return nil
    }

    _, err := tool.Invoke(nil, args)
    return err
}
```

这段代码是很“最小版本”的意思：

- 从 Scheduler 拿到 Tool 任务
- 在 Registry 中查找工具
- 让工具真正执行
- 返回执行结果或错误

它并不复杂，但它已经明确说明了：

> Tool Runtime 是一个运行时能力，而不是一个函数包装器。

---

## 8. Tool Runtime 与 Engine 的关系

如果说 Scheduler 决定“谁执行”，那么 Engine 负责“执行什么”，Tool Runtime 负责“工具如何被封装和使用”。

这三个层次是连续的：

```text
Agent
  ↓
Execution / State Machine
  ↓
Scheduler
  ↓
Tool Runtime
  ↓
Tool
```

Engine 并不直接管理工具调用细节，它更像是对执行过程的总控。

而 Tool Runtime 则负责：

- 工具注册
- 工具执行
- 参数校验
- 返回转换
- 错误封装

因此，Runtime 可以将执行流程看成：

```text
Agent 生成工具调用意图
    ↓
Execution 标记为 ToolRunning
    ↓
Scheduler 提交 ToolTask
    ↓
ToolRuntime 执行具体 tool
    ↓
结果写回执行状态
    ↓
Agent 继续下一步
```

这正是长生命周期 Agent Runtime 的真实运行方式。

---

## 9. Tool Runtime 不是“用工具”，而是“以工具为能力单元”

这个观点非常关键。

很多人把工具理解成：

> 发现一个函数，然后调用它。

但工程上的 Tool Runtime 更应该将其视为：

> 一个具备输入、输出、错误处理、状态控制的能力单元。

换句话说，Tool 的价值不是语法层面的“能被调用”，而是语义层面的“能作为 Runtime 的一个执行单元被管理”。

一个真正的 Tool 应该具备：

- 标识符：Tool Name
- 参数契约：Input Schema
- 执行逻辑：Invoke
- 执行结果：Output
- 错误模型：Error
- 运行时上下文：Context

因此，一次 tool 调用，实际上不是一个单纯函数调用，而是：

> 一个执行任务，受 Scheduler 控制，受 Runtime 状态管理，部分结果会进入 Checkpoint。

---

## 10. Tool Runtime 在 Runtime 中的实际位置

你们现在的 Runtime 架构大致可以表示成：

```text
                     Runtime
                        │
          ┌─────────────┼─────────────┐
          │             │             │
      Scheduler     Engine       Event Bus
          │             │             │
          └──────┬──────┴──────┬──────┘
                 │             │
             Tool Runtime   Checkpoint
                 │
                 ▼
               Tool Registry
```

从结构上看，Tool Runtime 是 Runtime 中最贴近“业务能力”的一层：

- Scheduler 负责按队列调度任务
- Tool Runtime 负责将“工具能力”封装为任务
- Engine 负责推进执行状态
- Checkpoint 负责保存快照
- Event Bus 负责串联状态变化

这说明 Tool Runtime 并不是独立于 Runtime 的一个容器，
它是 Runtime 组织生产能力的重要入口。

---

## 11. 一个最小但完整的 Tool Runtime 例子

下面给出一个更贴近工程实践的最小示例。这里的代码必须分成三层来写：

1. 工具定义：放在 `internal/tool` 包中
2. 工具任务与执行入口：放在 `internal/toolruntime` 包中
3. 运行时装配：放在 `internal/runtime` 或 `cmd/runtime` 中

### 代码 1：工具定义，放在 `internal/tool/search.go`

```Go
package tool

import (
    "context"
    "fmt"
)

type SearchTool struct{}

func (t *SearchTool) Name() string {
    return "search"
}

func (t *SearchTool) Invoke(ctx context.Context, args map[string]any) (any, error) {
    query, _ := args["query"].(string)
    return fmt.Sprintf("search result for: %s", query), nil
}
```

这里的意义是：

- `SearchTool` 不是业务逻辑写在 `runtime` 包里
- 它是一个标准的工具实现，属于工具层
- 这个工具只负责“实现一个搜索能力”，不负责调度

### 代码 2：注册工具，放在 `internal/runtime/runtime.go` 或 `cmd/runtime/main.go`

注册发生在 Runtime 组装阶段，不是在 `SearchTool` 自己里面做：

```Go
// internal/runtime/runtime.go
registry := tool.NewRegistry()
registry.Register(&tool.SearchTool{})
```

或者在程序入口里更直观：

```Go
// cmd/runtime/main.go
registry := tool.NewRegistry()
registry.Register(&tool.SearchTool{})
```

这里最关键的一点是：

> `SearchTool` 自己不负责注册，注册发生在 Runtime 的装配阶段。

### 代码 3：把工具接入 Runtime，放在 `internal/toolruntime/runtime.go`

```Go
// internal/toolruntime/runtime.go
package toolruntime

import (
    "github.com/aigc-engineering/aigc-agent-runtime/internal/scheduler"
    "github.com/aigc-engineering/aigc-agent-runtime/internal/tool"
)

type ToolRuntime struct {
    registry  *tool.Registry
    scheduler *scheduler.Scheduler
}

func NewToolRuntime(registry *tool.Registry, sched *scheduler.Scheduler) *ToolRuntime {
    return &ToolRuntime{
        registry:  registry,
        scheduler: sched,
    }
}
```

### 代码 4：提交工具任务，仍然走 `internal/toolruntime/task.go` + `internal/scheduler/scheduler.go`

这个提交动作必须发生在工具运行时中，而不是直接在 `SearchTool` 里：

```Go
// internal/toolruntime/task.go
package toolruntime

type ToolTask struct {
    ID       string
    ToolName string
    Args     map[string]any
    Session  string
}
```

```Go
// internal/toolruntime/runtime.go
func (r *ToolRuntime) Submit(task ToolTask) {
    r.scheduler.Submit(scheduler.Task{
        ID:   task.ID,
        Name: "tool:" + task.ToolName,
        Payload: map[string]any{
            "tool_name": task.ToolName,
            "args":      task.Args,
            "session":   task.Session,
        },
    })
}
```

然后在入口里调用：

```Go
// cmd/runtime/main.go
sched := scheduler.NewScheduler(16)
registry := tool.NewRegistry()
registry.Register(&tool.SearchTool{})

tr := toolruntime.NewToolRuntime(registry, sched)
tr.Submit(toolruntime.ToolTask{
    ID:       "tool-search-001",
    ToolName: "search",
    Args: map[string]any{
        "query": "agent runtime",
    },
    Session: "session-001",
})
```

这样，最小链路就完整了：

- `SearchTool` 定义在 `internal/tool`
- `ToolTask` 定义在 `internal/toolruntime`
- `scheduler.Task` 由 `internal/scheduler` 负责
- `ToolRuntime` 把工具任务转换为调度任务
- `Runtime` 负责装配这些对象

此时我们已经完成了以下动作：

- 工具被注册到 Tool Registry
- 工具调用被转换成任务
- 任务被提交到 Scheduler
- ToolRuntime 作为执行器处理任务
- 工具执行开始真正落地

这已经不是“一个函数调用”，而是一个 Runtime 级能力。

---

## 12. Tool Runtime 与 Long Running 的结合

Tool Runtime 在长生命周期 Agent 中尤其关键，因为工具调用往往并不是瞬间完成的。

例如：

- 查询数据库可能需要 2s
- 调用外部 API 可能需要 10s
- 运行代码可能需要 30s
- 读取大文件可能需要更长时间

这意味着一个工具执行任务不能简单地看作“直接调用函数结束”，而应当具有以下状态：

```text
Queued
  ↓
Running
  ↓
Succeeded / Failed / Timeout
```

这和前面说的 State Machine 是一致的。

也就是说，Tool Runtime 并不是把工具执行当成一个孤立动作，而是把它放进整个 Execution 状态图中：

```text
Execution
    ↓
ToolPending
    ↓
ToolRunning
    ↓
ToolSucceeded
```

这里的代码落点是：

- `Execution` 状态流仍然在 `internal/execution` 包中管理
- `ToolRuntime` 只是负责发起工具执行并把结果交回状态流
- `Checkpoint` 负责保存 `Execution` 和 `Tool` 的状态快照
- `Event Bus` 负责发布 `ToolStarted`、`ToolSucceeded`、`ToolFailed` 等事件

也就是说，工具不只是“执行一次”，而是：

```text
Execution State
    ↓
ToolRuntime.Submit
    ↓
Scheduler.Queue
    ↓
Tool execute
    ↓
Result written back
    ↓
Checkpoint / Event / Next Step
```

这样才能实现：

- 工具超时后重新调度
- 失败后触发 retry
- 执行过程可观测
- 工具执行结果可恢复

这就是 Tool Runtime 与 Long Running Runtime 之间的天然连接。

### 代码落点总览

到这里，最清晰的结构是：

```text
internal/
  tool/
    search.go             // SearchTool 实现
  toolruntime/
    task.go               // ToolTask
    runtime.go            // ToolRuntime + Submit + Execute
  scheduler/
    scheduler.go          // 通用 Task + Queue + Worker
  execution/
    execution.go          // Execution 状态机
    engine.go             // Engine 驱动执行
  runtime/
    runtime.go            // 组装 Runtime
  checkpoint/
    store.go              // Checkpoint
  event/
    event.go              // Event 定义
```

这里的设计原则非常重要：

- 工具实现属于工具层
- 工具任务属于工具运行时层
- 调度任务属于 Scheduler 层
- 状态机属于 Execution 层
- Runtime 只负责把它们组装起来

这也是工程上最容易把“代码写对位置”的关键。
---

## 13. Tool Runtime 真正的价值：把“外部能力”纳入 Agent Runtime

很多平台把工具调用描述成：

> 模型可以调用函数。

这只描述了“模型层面的能力”，并没有描述“系统层面的能力”。

而 Tool Runtime 的核心价值在于：

> 把外部能力纳入系统的运行时能力闭环。

也就是：

- 工具必须被注册
- 工具必须被调度
- 工具必须被状态管理
- 工具结果必须回流
- 工具失败必须可恢复
- 工具调用必须可观测

一旦工具从“函数”升级为“运行时能力”，Agent 才能真正变成企业级应用，而不只是一个功能演示系统。

---

## 14. 这章下一步怎么继续走？

这一章的重点，不是把 Tool Runtime 讲成一个大而全的工具平台，而是把它讲成：

> Runtime 中的一个关键能力层。

因此，后续继续扩展时，可以沿着这条路径继续推进：

### 14.1 Tool 参数契约

给每个工具定义输入输出 schema，便于 Agent 生成可靠调用。

### 14.2 Tool 执行状态

增加 `Queued`、`Running`、`Succeeded`、`Failed`、`Timeout` 等状态。

### 14.3 Tool Retry / Backoff

失败重试、退避策略、指数等待策略。

### 14.4 Tool Callback / Result Stream

让工具执行结果实时回流给 Agent 或上层系统。

### 14.5 Tool Observability

记录工具调用耗时、失败原因、重试次数、调用链信息。

这些都是在最小 Tool Runtime 之上的自然增强，而不是反过来把 Tool Runtime 设计得一开始就很复杂。

---

## 15. 本章的核心结论

Tool Runtime 不是简单的 Function Call，它是 Runtime 中承接 Agent 能力的关键层：

- 它让外部能力被统一管理
- 它让工具调用被调度和恢复
- 它让工具结果回流到状态机
- 它让 Agent 从“会说话”变成“会做事”

因此，Tool Runtime 是连接：

```text
Agent Decision
    ↓
Execution State
    ↓
Scheduler
    ↓
Tool Runtime
    ↓
External Capability
```

这一层的存在，正是 Agent Runtime 从“演示系统”逐步走向“真实应用系统”的关键一步。

---

## 总结

从前面的几章来看，Runtime 的关键能力已经逐步完善：

- State Machine：定义生命周期
- Event Bus：记录状态变化
- Checkpoint：保存执行快照
- Scheduler：让任务真正跑起来
- Tool Runtime：让 Agent 真正具备执行外部能力的能力

这也解释了为什么这个系列不是“单纯讲 LLM 调用”，而是：

> 把 Agent 当成一个真正长期运行、可恢复、可扩展的工程系统来设计。

Tool Runtime 正是这个系统从“模型会说话”成长为“系统会执行”的桥梁。

版本

Version: v0.7-tool-runtime

https://github.com/aigc-engineering/aigc-agent-runtime/releases/tag/v0.7-tool-runtime

本章着重解释了：

- Tool Runtime 并不仅仅是 Function Call
- Tool 要以任务和状态的形式进入 Runtime
- Scheduler 是 Tool Runtime 的关键支撑
- Agent 的真正生产力来自“工具能力被纳入 Runtime 执行闭环”

这也是后续 Memory、Human-in-the-loop 和 Observability 等章节的前置基础。
