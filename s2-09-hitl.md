# Human-in-the-Loop：等待用户参与

## 前言

在前面的几篇中，Runtime 的关键能力已经逐步完善：

- State Machine：定义执行的生命周期
- Event Bus：驱动事件流
- Checkpoint：保存执行快照  
- Scheduler：决定任务的执行顺序
- Tool Runtime：让 Agent 调用外部能力
- Memory：让 Runtime 拥有长期知识

现在出现了一个企业级 Agent 的必然需求：

> 并非所有决策都能由 Agent 自动做出，某些关键点必须由人来介入。

例如：

- Agent 准备修改重要数据，需要用户确认
- Agent 发现多条可行路径，需要用户选择
- 系统预测操作成本过高，需要用户批准
- Agent 遇到可恢复的错误，需要用户提供补充信息

这意味着 Runtime 不能是完全自动化的，而必须能够在需要的时候暂停，等待用户决策，然后基于用户的输入继续执行。

这就是 Human-in-the-Loop。

---

## 1. 为什么 Runtime 需要 Human-in-the-Loop？

从 Runtime 的架构看，前面章节已经建立了自动执行的完整闭环：

```text
Scheduler → Engine → Tool Runtime → Checkpoint
     ↑                                    ↓
     └─────── Memory ────────────────────┘
```

这套机制可以让 Agent 自动执行、调用工具、保存状态、学习历史。

但在生产环境中，自动化的风险是明确的：

- 不可逆操作需要人的确认
- 不确定的决策需要人的指导
- 异常情况需要人的判断

因此 Runtime 不能只是"能跑"，而是要"可控"。

可控的方式，就是在执行流中引入暂停点：当到达某个决策点时，Runtime 主动暂停，等待用户输入，然后基于输入恢复执行。

这样做的好处是：

- Agent 的自动能力不会被削弱
- 关键决策被提升到人的层面
- 用户可以随时干预执行过程
- 所有状态变化都被完整记录

---

## 2. Human-in-the-Loop 和 Checkpoint 有什么区别？

这是最容易混淆的地方。

当 Runtime 需要暂停时，Checkpoint 和 Human-in-the-Loop 都涉及状态保存。但它们的目的不同：

**Checkpoint** 关注的是：任务因为故障中断后，如何恢复到之前的状态继续执行。这是技术恢复。

**Human-in-the-Loop** 关注的是：任务因为需要人的决策而暂停，如何等待用户反馈，然后基于用户的输入来决定下一步。这是决策参与。

具体的区别：

| 维度 | Checkpoint | Human-in-the-Loop |
|------|-----------|-----------------|
| **触发原因** | 系统故障/定期保存 | Agent 需要人的决策 |
| **保存内容** | 当前执行状态 | 暂停点的上下文信息 |
| **谁来恢复** | 系统自动重试 | 用户手动操作 |
| **恢复参数** | 无（从快照恢复） | 用户的决策/输入 |

因此，Human-in-the-Loop 建立在 Checkpoint 的基础上，但为暂停增加了"等待输入"的语义。

---

## 3. Pause 机制的基础

Execution 的状态机已经定义了 Paused 状态。看 [internal/execution/execution.go](internal/execution/execution.go)：

### 状态机结构

[internal/execution/execution.go](internal/execution/execution.go) 定义了 Execution 的生命周期：

```go
type Status string

const (
	StatusCreated   Status = "created"
	StatusRunning   Status = "running"
	StatusPaused    Status = "paused"
	StatusCompleted Status = "completed"
	StatusFailed    Status = "failed"
)

type Execution struct {
	ID     string
	Status Status
}

func (e *Execution) Start() error {
	if e.Status != StatusCreated && e.Status != StatusPaused {
		return fmt.Errorf("cannot start execution from status %s", e.Status)
	}
	e.Status = StatusRunning
	return nil
}

func (e *Execution) Pause() error {
	if e.Status != StatusRunning {
		return fmt.Errorf("cannot pause execution from status %s", e.Status)
	}
	e.Status = StatusPaused
	return nil
}
```

从 Paused 状态可以回到 Running，这支撑了 Human-in-the-loop 的状态转换。

### Engine 中的 Pause

[internal/execution/engine.go](internal/execution/engine.go) 的 Pause 方法：

```go
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

Engine 能够将 Execution 转换为 Paused 状态，并通过 Event Bus 发布 `TypeExecutionPaused` 事件。

---

## 4. Human-in-the-Loop 的三个核心组件

基于 Pause 机制，还需要 3 个组件来完成 Human-in-the-loop 的闭环。

### 组件 1：PauseContext —— 保存暂停时的上下文

当 Execution 暂停时，需要记录暂停点的信息供外部系统展示给用户。

创建 [internal/execution/pause_context.go](internal/execution/pause_context.go)：

```go
package execution

// PauseContext 记录暂停状态的上下文信息
type PauseContext struct {
	ExecutionID string                 // 执行ID
	Reason      string                 // 暂停原因
	Prompt      string                 // 向用户展示的问题
	Options     []string               // 可选项（如果适用）
	Metadata    map[string]interface{} // 自定义元数据
}

// PauseStore 管理暂停上下文的存储
type PauseStore interface {
	SavePause(ctx PauseContext) error
	LoadPause(executionID string) (PauseContext, error)
	RemovePause(executionID string) error
}

// MemoryPauseStore 内存实现
type MemoryPauseStore struct {
	data map[string]PauseContext
}

func NewMemoryPauseStore() *MemoryPauseStore {
	return &MemoryPauseStore{
		data: make(map[string]PauseContext),
	}
}

func (s *MemoryPauseStore) SavePause(ctx PauseContext) error {
	s.data[ctx.ExecutionID] = ctx
	return nil
}

func (s *MemoryPauseStore) LoadPause(executionID string) (PauseContext, error) {
	ctx, ok := s.data[executionID]
	if !ok {
		return PauseContext{}, fmt.Errorf("pause context not found: %s", executionID)
	}
	return ctx, nil
}

func (s *MemoryPauseStore) RemovePause(executionID string) error {
	delete(s.data, executionID)
	return nil
}
```

### 组件 2：UserInputHandler —— 接收并处理用户输入

创建 [internal/execution/user_input.go](internal/execution/user_input.go)：

```go
package execution

// UserInput 用户对暂停点的输入
type UserInput struct {
	ExecutionID string                 // 执行ID
	Decision    string                 // 用户的决策或选择
	Data        map[string]interface{} // 用户输入的额外数据
}

// UserInputHandler 处理用户输入的接口
type UserInputHandler interface {
	HandleUserInput(input UserInput) error
}

// DefaultUserInputHandler 基础实现
type DefaultUserInputHandler struct {
	pauseStore PauseStore
}

func NewDefaultUserInputHandler(ps PauseStore) *DefaultUserInputHandler {
	return &DefaultUserInputHandler{
		pauseStore: ps,
	}
}

func (h *DefaultUserInputHandler) HandleUserInput(input UserInput) error {
	// 查找对应的 pause context
	pauseCtx, err := h.pauseStore.LoadPause(input.ExecutionID)
	if err != nil {
		return err
	}

	// 验证用户决策是否合法
	if !isValidOption(input.Decision, pauseCtx.Options) {
		return fmt.Errorf("invalid decision: %s", input.Decision)
	}

	// 将用户输入存储到 Memory 中供 Agent 使用
	return nil
}

func isValidOption(decision string, options []string) bool {
	for _, opt := range options {
		if opt == decision {
			return true
		}
	}
	return len(options) == 0
}
```

### 组件 3：Engine.Resume —— 从暂停状态恢复

在 [internal/execution/engine.go](internal/execution/engine.go) 中增加 Resume 方法：

```go
type Engine struct {
	name         string
	bus          *event.Bus
	pauseStore   execution.PauseStore
	inputHandler execution.UserInputHandler
}

func NewEngine(
	name string, 
	bus *event.Bus, 
	ps execution.PauseStore, 
	ih execution.UserInputHandler,
) *Engine {
	return &Engine{
		name:         name,
		bus:          bus,
		pauseStore:   ps,
		inputHandler: ih,
	}
}

// Resume 从 Paused 状态转换回 Running，处理用户输入
func (e *Engine) Resume(
	exec *Execution,
	sess *session.Session,
	input execution.UserInput,
) error {
	// 处理用户输入
	if err := e.inputHandler.HandleUserInput(input); err != nil {
		return fmt.Errorf("failed to handle user input: %w", err)
	}

	// 清除 pause context
	if err := e.pauseStore.RemovePause(exec.ID); err != nil {
		return fmt.Errorf("failed to remove pause context: %w", err)
	}

	// 状态转换：Paused → Running
	if err := exec.Start(); err != nil {
		return err
	}

	e.bus.Publish(event.Event{
		Type:        event.TypeExecutionResumed,
		ExecutionID: exec.ID,
		Status:      string(exec.Status),
		Session:     *sess,
	})

	return nil
}
```

需要在 [internal/event/event.go](internal/event/event.go) 中增加 `TypeExecutionResumed` 事件类型。

---

## 5. Runtime 中的集成

### 执行流程

```text
Agent 需要用户输入
     ↓
Engine.Pause() 被调用
     ↓
  ├─ Execution：Running → Paused
  ├─ PauseContext 被保存（含 Prompt / Options）
  └─ 发布 TypeExecutionPaused 事件
     ↓
外部系统收到 Paused 事件
     ↓
用户在外部系统中做出决策
     ↓
Engine.Resume() 被调用，传入用户决策
     ↓
  ├─ UserInputHandler 校验用户输入
  ├─ Execution：Paused → Running
  └─ 发布 TypeExecutionResumed 事件
     ↓
Agent 继续执行
```

### Runtime 的改动

在 [internal/runtime/runtime.go](internal/runtime/runtime.go) 中：

```go
type Runtime struct {
	engine       *execution.Engine
	bus          *event.Bus
	checkpoint   checkpoint.Store
	scheduler    *scheduler.Scheduler
	memory       memory.Store
	pauseStore   execution.PauseStore
	inputHandler execution.UserInputHandler
}

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
		scheduler:    scheduler,
		memory:       m,
		pauseStore:   pauseStore,
		inputHandler: inputHandler,
	}
}

func (r *Runtime) Pause(
	executionID string,
	reason string,
	prompt string,
	options []string,
) error {
	exec, err := r.getExecution(executionID)
	if err != nil {
		return err
	}

	pauseCtx := execution.PauseContext{
		ExecutionID: executionID,
		Reason:      reason,
		Prompt:      prompt,
		Options:     options,
	}

	if err := r.pauseStore.SavePause(pauseCtx); err != nil {
		return err
	}

	sess, _ := r.memory.Load(executionID + ":session")
	session := sess.(session.Session)
	
	return r.engine.Pause(exec, &session)
}

func (r *Runtime) Resume(
	executionID string,
	decision string,
	data map[string]interface{},
) error {
	exec, err := r.getExecution(executionID)
	if err != nil {
		return err
	}

	input := execution.UserInput{
		ExecutionID: executionID,
		Decision:    decision,
		Data:        data,
	}

	sess, _ := r.memory.Load(executionID + ":session")
	session := sess.(session.Session)
	
	return r.engine.Resume(exec, &session, input)
}
```

---

## 6. 实现示例

### 触发暂停

```go
// internal/agent/example_agent.go

type PauseHook func(executionID string, reason string, prompt string, options []string) error

type ExampleAgent struct {
	pause PauseHook
}

func NewExampleAgent(pause PauseHook) *ExampleAgent {
	return &ExampleAgent{pause: pause}
}

func (a *ExampleAgent) Execute(sess *session.Session) error {
	fmt.Println("Agent 完成分析，准备修改数据库")
	fmt.Println("操作影响范围：1000+ 条记录")

	return a.pause(
		sess.ID,
		"db_update_confirmation",
		"确认执行数据库更新？",
		[]string{"confirm", "cancel"},
	)
}
```

这个设计和当前项目保持一致：

- Agent 仍然只实现 `Execute(sess *session.Session) error`
- 运行时通过依赖注入把 pause 能力传进去
- Agent 不持有 `Runtime`，也不改造成一个更大的业务对象

在 Runtime 组装时：

```go
func BuildAgent(r *runtime.Runtime) agent.Agent {
	return NewExampleAgent(r.Pause)
}
```
### 处理用户输入

外部系统（如 API 服务）接收用户的确认决策，调用 Runtime.Resume()：

```go
// 示例：HTTP API 处理用户确认

func handleConfirmation(w http.ResponseWriter, r *http.Request) {
	executionID := r.URL.Query().Get("execution_id")
	decision := r.PostFormValue("decision")

	err := runtimeInstance.Resume(executionID, decision, nil)
	if err != nil {
		http.Error(w, err.Error(), 500)
		return
	}

	w.WriteHeader(200)
	w.Write([]byte("Execution resumed"))
}
```

### 订阅事件

订阅暂停和恢复事件，实现外部系统通知：

```go
r.Subscribe(func(evt event.Event) {
	if evt.Type == event.TypeExecutionPaused {
		fmt.Printf("Execution %s paused, awaiting user input\n", evt.ExecutionID)
		
		// 通知 UI
		// 发送提醒
		// 记录事件
	}
	
	if evt.Type == event.TypeExecutionResumed {
		fmt.Printf("Execution %s resumed\n", evt.ExecutionID)
	}
})
```

---

## 7. Checkpoint 的协作

当 Execution 暂停时，Checkpoint 记录当前状态供恢复使用。

### Pause 时保存快照

在 Engine 中增加方法：

```go
func (e *Engine) PauseWithCheckpoint(
	exec *Execution,
	sess *session.Session,
	cp checkpoint.Store,
) error {
	// 保存 checkpoint
	if err := cp.Save(checkpoint.Checkpoint{
		ExecutionID: exec.ID,
		Status:      string(StatusPaused),
		Session:     *sess,
	}); err != nil {
		return err
	}

	// 触发暂停和事件发布
	return e.Pause(exec, sess)
}
```

### Resume 时加载快照

```go
func (e *Engine) ResumeFromCheckpoint(
	exec *Execution,
	cp checkpoint.Store,
	input execution.UserInput,
) error {
	// 加载 checkpoint
	savedCP, err := cp.Load(exec.ID)
	if err != nil {
		return err
	}

	// 验证 checkpoint 状态
	if savedCP.Status != string(StatusPaused) {
		return fmt.Errorf("checkpoint status is not paused")
	}

	sess := &savedCP.Session

	// 处理用户输入
	if err := e.inputHandler.HandleUserInput(input); err != nil {
		return err
	}

	// 转换状态
	if err := exec.Start(); err != nil {
		return err
	}

	e.bus.Publish(event.Event{
		Type:        event.TypeExecutionResumed,
		ExecutionID: exec.ID,
		Status:      string(exec.Status),
		Session:     *sess,
	})

	return nil
}
```

---

## 8. Memory 的协作

用户的决策通过 Memory 存储，Agent 可以在恢复后访问。

### 存储决策

```go
func (h *DefaultUserInputHandler) HandleUserInput(
	input UserInput,
	memStore memory.Store,
) error {
	pauseCtx, err := h.pauseStore.LoadPause(input.ExecutionID)
	if err != nil {
		return err
	}

	if !isValidOption(input.Decision, pauseCtx.Options) {
		return fmt.Errorf("invalid decision: %s", input.Decision)
	}

	// 存储用户决策
	return memStore.Save(memory.MemoryEntry{
		Key:   fmt.Sprintf("%s:user_decision", input.ExecutionID),
		Value: input.Decision,
		Type:  memory.TypeUserInput,
	})
}
```

### 读取并应用决策

```go
type PauseHook func(executionID string, reason string, prompt string, options []string) error

type ExampleAgent struct {
	pause PauseHook
}

func NewExampleAgent(pause PauseHook) *ExampleAgent {
	return &ExampleAgent{pause: pause}
}

func (a *ExampleAgent) Execute(sess *session.Session) error {
	// ... Agent 逻辑 ...

	// 触发暂停
	if err := a.pause(sess.ID, "choice_required", "请选择执行策略", []string{"A", "B"}); err != nil {
		return err
	}

	// 用户做出决策后，Runtime 恢复执行
	// Agent 可以从 Memory 中读取决策
	entry, err := memStore.Load(sess.ID+":user_decision", "decision")
	if err != nil {
		return err
	}

	if len(entry) == 0 {
		return nil
	}

	decision := entry[0].Value
	if decision == "A" {
		// 执行策略 A
	} else {
		// 执行策略 B
	}

	return nil
}
```

---

## 9. 运作机制

Human-in-the-loop 包含三个协作的层面：

### 暂停层

Execution 的状态机和 Engine.Pause() 管理暂停过程。当需要用户决策时，Execution 从 Running 转换到 Paused，PauseContext 记录暂停信息。

### 交互层

PauseContext 存储暂停点的问题和可选项。UserInputHandler 验证用户输入是否符合约束。

### 恢复层

Engine.Resume() 处理用户输入，将 Execution 从 Paused 恢复到 Running。用户决策通过 Memory 传递给 Agent。

### 事件驱动

所有状态变化（Paused、Resumed）都通过 Event Bus 发布，外部系统可以订阅这些事件实现通知、审计、UI 更新等功能。

---

## 10. 设计特性

| 层面 | 组件 | 职能 |
|------|------|------|
| 状态 | Execution | Running ↔ Paused 转换 |
| 上下文 | PauseContext | 记录问题、选项、元数据 |
| 输入 | UserInput | 接收用户决策和数据 |
| 验证 | UserInputHandler | 校验用户输入合法性 |
| 恢复 | Engine.Resume() | 状态转换 + 事件发布 |
| 持久化 | Checkpoint | 保存 Paused 状态快照 |
| 决策传递 | Memory | 用户决策进入 Agent 上下文 |
| 观测 | Event Bus | 发布暂停/恢复事件供外部订阅 |

这种分层设计将暂停机制的关注点分离，Runtime 可以灵活地支持不同的 Human-in-the-loop 场景。

---

## 总结

Human-in-the-Loop 在 Runtime 中的意义，不在于让 Agent 变成一个“人工操作系统”，而在于：

> 当执行进入关键决策点时，Runtime 能安全地暂停，并在用户确认后继续执行。

从工程角度看，最关键的设计点是：

- Execution 使用 `Running` / `Paused` 状态明确表达等待用户输入的执行状态
- PauseContext 负责保存暂停时的上下文信息，供外部系统展示给用户
- UserInputHandler 负责校验用户返回的决策，保证交互输入符合约束
- Resume 负责让执行从 Paused 回到 Running，并重新继续执行流程
- Checkpoint 和 Memory 为 Human-in-the-Loop 提供恢复能力和决策持久化能力
- Event Bus 让暂停和恢复事件对外可观测

这意味着 Runtime 不再只是“自动执行容器”，而是开始具备“可控执行”和“人工参与”两种模式。

这也正是企业级 Agent 与 demo 代码差别最大的地方：

- demo 只关心一次性执行
- 企业级 Runtime 需要处理等待输入、恢复继续、审计和回溯

这也是 Human-in-the-Loop 作为企业级 Agent 关键能力的意义所在。

版本

Version: v0.9-hitl

https://github.com/aigc-engineering/aigc-agent-runtime/releases/tag/v0.9-hitl

