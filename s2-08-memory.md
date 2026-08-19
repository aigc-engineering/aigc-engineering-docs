# Memory：让 Runtime 拥有长期记忆

## 前言

在前面的几篇里，我们已经把 Runtime 的关键能力一步步补齐：

- State Machine：定义执行状态
- Event Bus：触发事件流
- Checkpoint：保存执行快照
- Scheduler：决定执行顺序
- Tool Runtime：让 Agent 能调用外部能力

现在一个很关键的问题出现了：

> Runtime 能跑起来，并且能够执行任务，但它还不能“记住过去”。

这意味着：

- 用户的偏好需要每次重新确认
- 任务的历史信息需要每次重新推导
- 工具的成功经验不能被复用
- Agent 不能基于过去的决策持续推进目标

这在 demo 中还可以接受，但在企业级 Agent 中，这是一个明显的工程缺口。

因此，下一步我们需要增加一个新能力：

> Memory。

这篇文章不是再讲 Memory 的抽象定义，而是基于当前已经存在的 Runtime 结构，说明：

- Memory 应该放在哪个 Go 模块里
- 它和 Checkpoint、Session、Event 的边界是什么
- 它怎么接入现有 Runtime
- 它在 Long Running 场景中如何起作用

---

## 1. 为什么 Runtime 需要 Memory？

最简单的运行时，通常只关心“这次调用能不能完成”。

例如：

```text
Request
  ↓
Agent
  ↓
Tool / LLM
  ↓
Response
```

这种结构在单次请求里完全够用。

但企业级 Agent 的任务通常不是单次请求，而是长生命周期执行：

- 任务需要持续推进
- 用户会反复确认
- 目标会在多个步骤中逐步细化
- 工具调用经验会不断累积

这时就出现了一个很核心的问题：

> 任务的历史信息，是否还会影响下一次执行？

如果答案是会，那么 Runtime 就不能只保存“当前执行状态”，还必须保留“长期知识”。

这就是 Memory 的意义。

它解决的不是“本次执行是否成功”，而是：

> 未来的执行，是否还能在上一轮信息上继续推进。

---

## 2. Memory 和 Checkpoint 有什么区别？

这是最容易混淆的地方。

你们当前项目中，Checkpoint 已经在实现：

- internal/checkpoint/store.go
- internal/runtime/runtime.go

它保存的是：

- 任务当前处于什么状态
- 执行到了哪里
- 恢复时需要哪些状态

所以 Checkpoint 关注的是：

> 任务恢复。

而 Memory 关注的是：

> 任务历史和长期知识。

例如：

### Checkpoint 可能保存：

```text
Execution ID: exec-123
Status: Running
Current Step: ToolCall
Last Event: ToolSucceeded
```

### Memory 可能保存：

```text
Session ID: session-001
Kind: preference
Key: style
Value: realistic cinematic

Session ID: session-001
Kind: profile
Key: role
Value: short hair female lead
```

它们回答的是两个不同的问题：

- Checkpoint：我现在在哪？
- Memory：我以前知道了什么？

这两个维度共同构成 Long Running Agent 的状态基础。

---

## 3. Memory 不是 Session，也不是日志

Memory 也经常被误解为：

- 聊天记录
- 历史消息
- Context 缓存
- 日志归档

这些都不完全等价。

### Session

Session 是当前上下文：

```Go
type Session struct {
    ID     string
    Input  string
    Output string
}
```

它描述的是：

- 当前输入是什么
- 当前输出是什么
- 当前这次运行的上下文是什么

### Log

日志是一种运行记录：

- 调用哪些工具
- 发生了什么错误
- 任务什么时候开始
- 什么时候结束

它更偏“审计”和“排查问题”。

### Memory

Memory 是未来仍然会影响决策的信息：

- 用户偏好
- 任务事实
- 角色设定
- 工具调用经验
- 成功的参数组合

它不是单次对话，也不是临时状态，而是长期知识。

所以，Memory 应该被设计成 Runtime 中的一个独立能力层，而不是被混进 Session 或 Log。

---

## 4. Memory 应该落到哪个 Go 模块里？

在你们的项目结构中，最自然的落点是：

```text
internal/memory/
  memory.go
  store.go
```

而不是把它塞进：

- runtime
- execution
- session
- checkpoint

原因很简单：

- `runtime` 负责组装组件
- `execution` 负责状态流转
- `session` 负责当前上下文
- `checkpoint` 负责恢复快照
- `memory` 负责长期知识

这也是最清晰的层次划分。

---

## 5. 最小的 Memory 数据结构

这个阶段不需要复杂的向量库，也不需要花哨的知识图谱。

最小实践版本，先做一个非常简单的“key-value + session + type”结构即可：

```Go
package memory

type MemoryEntry struct {
    ID        string
    SessionID string
    Kind      string
    Key       string
    Value     string
    CreatedAt string
    UpdatedAt string
}

type Store interface {
    Save(entry MemoryEntry) error
    Query(sessionID string, kind string) ([]MemoryEntry, error)
    Load(sessionID string, key string) ([]MemoryEntry, error)
}
```

这几个字段是关键：

- `SessionID`：属于哪个会话
- `Kind`：偏好、事实、工具经验等分类
- `Key`：例如 `style`, `role`, `tool.search.config`
- `Value`：记忆内容
- `CreatedAt` / `UpdatedAt`：有时间线，后续可以做召回和淘汰

这已经足够支撑一个最小可用的 Memory 组件。

---

## 6. 最小的 Memory Store 实现

在当前项目里，最自然的第一版实现，是一个简单的 in-memory store，和 checkpoint 的 memory store 风格一致：

```Go
package memory

import (
    "fmt"
    "sort"
    "time"
)

type MemoryStore struct {
    data map[string][]MemoryEntry
}

func NewMemoryStore() *MemoryStore {
    return &MemoryStore{
        data: make(map[string][]MemoryEntry),
    }
}

func (s *MemoryStore) Save(entry MemoryEntry) error {
    if entry.ID == "" {
        entry.ID = fmt.Sprintf("mem-%d", time.Now().UnixNano())
    }
    if entry.CreatedAt == "" {
        entry.CreatedAt = time.Now().Format(time.RFC3339)
    }
    entry.UpdatedAt = time.Now().Format(time.RFC3339)

    s.data[entry.SessionID] = append(s.data[entry.SessionID], entry)
    return nil
}

func (s *MemoryStore) Query(sessionID string, kind string) ([]MemoryEntry, error) {
    items := s.data[sessionID]
    var result []MemoryEntry
    for _, item := range items {
        if item.Kind == kind {
            result = append(result, item)
        }
    }

    sort.Slice(result, func(i, j int) bool {
        return result[i].UpdatedAt > result[j].UpdatedAt
    })
    return result, nil
}

func (s *MemoryStore) Load(sessionID string, key string) ([]MemoryEntry, error) {
    items := s.data[sessionID]
    var result []MemoryEntry
    for _, item := range items {
        if item.Key == key {
            result = append(result, item)
        }
    }
    return result, nil
}
```

这个版本很简单，但它已经说明了 Memory 的最核心能力：

- 可以记住信息
- 可以按会话查询
- 可以按 key 回忆
- 可以按时间排序

这已经是一个“真正存在的 Memory 层”，而不是只停留在概念讨论。

---

## 7. Memory 如何接入当前 Runtime？

现在的 Runtime 在 [internal/runtime/runtime.go](internal/runtime/runtime.go) 里已经有了：

```Go
type Runtime struct {
    engine     *execution.Engine
    bus        *event.Bus
    checkpoint checkpoint.Store
    Scheduler  *scheduler.Scheduler
}
```

现在我们给它补上一层 Memory：

```Go
type Runtime struct {
    engine     *execution.Engine
    bus        *event.Bus
    checkpoint checkpoint.Store
    memory     memory.Store
    Scheduler  *scheduler.Scheduler
}
```

初始化时：

```Go
func NewRuntime() *Runtime {
    bus := event.NewBus()
    checkpointStore := checkpoint.NewMemoryStore()
    memoryStore := memory.NewMemoryStore()

    bus.Subscribe(checkpoint.Handler(checkpointStore))

    engine := execution.NewEngine("default-engine", bus)
    s := scheduler.NewScheduler(16)

    return &Runtime{
        engine:     engine,
        bus:        bus,
        checkpoint: checkpointStore,
        memory:     memoryStore,
        Scheduler:  s,
    }
}
```

这一步非常关键，因为它说明：

> Memory 是 Runtime 的一个正式能力组件，而不是临时胡乱放进一个函数里。

---

## 8. Runtime 里如何写入 Memory？

这一步应该有统一入口，而不是散落在各个业务代码里。

最小抽象：

```Go
func (r *Runtime) Remember(sessionID string, key string, value string, kind string) error {
    return r.memory.Save(memory.MemoryEntry{
        SessionID: sessionID,
        Kind:      kind,
        Key:       key,
        Value:     value,
    })
}
```

这是最小而干净的接口：

- 统一写入入口
- 允许后续扩展 TTL / filter / dedupe
- 不侵入 Agent 或 Engine 的核心逻辑

一般来说，写入 Memory 的时机可以是：

- 用户确认事实
- 工具执行成功并有可复用价值
- 一轮任务完成并总结成经验
- 失败后进行修正并保留修正知识

也就是说：

> 不是所有事件都写入 Memory，而是“有长期价值”的事实才写入 Memory。

---

## 9. Runtime 里如何读取 Memory？

读取 Memory 也应该通过统一入口：

```Go
func (r *Runtime) Recall(sessionID string, key string) ([]memory.MemoryEntry, error) {
    return r.memory.Load(sessionID, key)
}

func (r *Runtime) RecallKind(sessionID string, kind string) ([]memory.MemoryEntry, error) {
    return r.memory.Query(sessionID, kind)
}
```

这个设计意味着：

- Runtime 在执行前可以读取历史知识
- 知识可以被注入 context 或 prompt
- Agent 不再从零开始重新判断所有事实

最经典的应用方式是：

```text
执行前：读取 Memory
    ↓
确认用户偏好 / 角色设定 / 工具经验
    ↓
启动任务
```

这正是 Memory 真正发挥价值的地方。

---

## 10. Memory 与 Event 的结合方式

Memory 的写入不应该在业务代码里散落各处，更合理的方式是利用 Event Bus。

举个例子：

```Go
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

这样的好处是：

- Memory 写入逻辑集中在事件处理器中
- 数据来源来自事件，而不是散落在各处
- 后续更容易接入日志、监控和 replay

也就是说，Memory 和 Event 是天然配合的：

```text
Event
  ↓
Memory Handler
  ↓
Memory Store
```

这比在业务逻辑里插入大量 `Remember()` 调用更自然。

---

## 11. 一个最小 Demo：用户偏好被记住

现在我们可以用最小例子说明 Memory 如何工作。

### 第一次执行：写入偏好

```Go
func main() {
    rt := runtime.NewRuntime()

    _ = rt.Remember("session-001", "style", "realistic", "preference")
    _ = rt.Remember("session-001", "role", "short hair female lead", "profile")
}
```

### 下一次执行：读取偏好

```Go
func main() {
    rt := runtime.NewRuntime()

    prefs, _ := rt.RecallKind("session-001", "preference")
    profile, _ := rt.RecallKind("session-001", "profile")

    fmt.Println(prefs)
    fmt.Println(profile)
}
```

输出大致是：

```text
[{session-001 preference style realistic}]
[{session-001 profile role short hair female lead}]
```

这很关键，因为它说明：

> Agent 在下一次执行前，可以从 Memory 中恢复长期事实，而不是从零重新判断。

这才是真正的长期记忆。

---

## 12. Memory 与 Long Running 的结合

如果只是保存用户偏好还不够，Long Running Agent 还需要保留：

- 已确认的业务事实
- 工具调用成功经验
- 上一次失败的修正方式
- 任务的执行摘要

所以真正的长生命周期状态，应该是：

```text
Checkpoint：恢复执行状态
Memory：恢复长期知识
Session：恢复当前上下文
Event：记录行为轨迹
```

最终形成：

```text
Runtime
  ├── Scheduler
  ├── Engine
  ├── Event Bus
  ├── Checkpoint
  ├── Session
  └── Memory
```

这也是为什么 Memory 是企业级 Agent 必须具备的能力。

因为它让 Runtime 不仅能“继续跑”，还能“继续知道自己在干什么”。

---

## 13. 这篇文章的工程意义

这一篇和前面的理论文章不是重复，而是延续：

- 前面讲的是 Memory 的必要性
- 这一篇讲的是 Memory 如何落地到当前 runtime

也就是说，前面回答的是：

> 为什么 Agent 需要记忆。

这一篇回答的是：

> 我们现在已经有了什么基础，而且 Memory 应该怎么加入当前 Runtime。

这种写法更适合工程实践，因为它关注的是：

- 代码落点
- 模块边界
- 事件入口
- Runtime 组装
- 最小实现

---

## 14. 这篇文章的最终结论

Memory 在当前 Runtime 中，不应该被理解成一个“额外的小功能”，而应该被视为：

> Runtime 中长期知识层的核心能力。

它的职责是：

- 保存长期事实
- 存储用户偏好
- 复用工具经验
- 提供历史上下文
- 让 Agent 在下一次执行时继续基于过去决策

而最自然的代码落点是：

```text
internal/memory/
```

并在 Runtime 中进行统一装配：

```text
internal/runtime/runtime.go
```

这样做的效果是：

- Memory 与 Checkpoint 分层明确
- Memory 与 Session 分层明确
- Memory 与 Event 串联自然
- Runtime 逐步从“执行框架”成长为“长期运行系统”

这也正是企业级 Agent 跟简单演示代码最大的差别。

---

## 总结

从工程角度看，Memory 的真正价值不是“把更多东西记住”，而是：

> 让 Runtime 能在下次执行时，继续基于过去的信息做出更稳定的决策。

而在你们现在的结构中，最合理的做法就是：

- 在 `internal/memory` 新增 Memory Store
- 在 Runtime 中装配它
- 在事件或执行完成后写入记忆
- 在执行前给 Agent 恢复历史知识

这样，Runtime 就不再只是一个“能跑的容器”，
它已经开始变成一个真正能持续推进任务的 Agent Runtime。

版本

Version: v0.8-memory
https://github.com/aigc-engineering/aigc-agent-runtime/releases/tag/v0.8-memory
