# Runtime Extension：让 Runtime 可以扩展

## 前言

在前面的文章里，Runtime 的关键能力已经逐步补齐：

- State Machine：管理执行生命周期
- Event Bus：串联事件流
- Checkpoint：支持恢复与重试
- Scheduler：控制执行顺序
- Tool Runtime：连接外部能力
- Memory：提供长期知识
- Human-in-the-loop：支持人工介入
- Observability：支撑可见性与诊断

这些能力本身并不属于 AI 独有的工程范畴，但在 Agent 场景中，它们被组合成了一套完整的执行框架。

问题也随之出现：

> Runtime 看起来已经比较完整了，为什么还总觉得“还差一点”？

真正的答案不是“能力不够”，而是：

> Agent Runtime 不只是要能跑，还必须能扩展。

---

## 1. 为什么 Runtime 一定要可扩展？

很多人把 Runtime 理解成一个“固定的框架”，认为只要状态机、事件总线、调度器、memory、tool runtime 都有了，就可以直接上生产。

但真实业务远不是这样。

一个客服 Agent 可能要接入 CRM、工单系统、知识库和消息平台；
一个代码 Agent 可能要接入 GitHub、CI/CD、测试系统和代码搜索；
一个审批 Agent 可能要接入 OA、权限中心、审批流、消息通知。

同样的 Runtime，面对不同业务时，需求完全不同：

- 状态流转不同
- 审批逻辑不同
- 工具接入不同
- 权限边界不同
- 记忆组织方式不同
- 调度策略不同
- 人机协作方式不同

这说明 Runtime 的价值，不在于“有没有一套固定能力”，而在于：

> 能否在真实业务中不断装配新能力。

---

## 2. 扩展，不只是“加接口”

传统软件中的扩展，通常是插件、适配器、hook、策略模式等形式。

但 Agent Runtime 的扩展，不只是“接一个模块”这么简单。

因为这里的扩展，往往会影响执行语义本身，而不是单纯增加一个功能。

例如：

- 增加一个状态
- 增加一个审批环节
- 增加一个回滚策略
- 增加一个长时记忆机制
- 在工具调用前增加权限校验
- 在等待用户输入后定义恢复方式

这些都不是“新增一个函数”，而是改变执行过程中的关键控制点。

因此，Runtime 的扩展重点，不是“能不能插一个插件”，而是：

> 能不能在生命周期中插入可控的执行逻辑。

---

## 3. Runtime 的扩展点，应该放在哪里？

一个成熟的 Agent Runtime，至少需要在几个关键位置提供扩展点。

### 3.1 状态扩展

State Machine 不是固定状态集合，而应该允许业务按需扩展。

例如：

- accepted
- parsed
- planned
- tool_calling
- waiting_user
- reviewing
- retrying
- escalated
- completed
- failed

不同业务还会增加诸如：

- compliance_check
- contract_signing
- customer_followup
- admin_review

如果 Runtime 只允许固定状态，那么它很快会演变成一个模板框架，而不是通用运行时。

---

### 3.2 事件扩展

Event Bus 应该不只是发事件，还允许接入事件处理器。

例如：

- before_tool_call
- after_tool_result
- before_state_transition
- on_user_input
- on_checkpoint
- on_error
- on_memory_write

通过这些扩展点，可以在任务执行中插入：

- 权限校验
- 结果清洗
- 审计日志
- 人工转派
- 错误告警

这样 Runtime 才能和业务系统真正耦合。

---

### 3.3 Tool 扩展

Tool Runtime 是最容易被理解的扩展点。

但仅仅注册函数并不等于真正扩展。

一个成熟的工具扩展至少应包含：

- 参数校验
- 权限校验
- 超时控制
- 重试策略
- 返回值清洗
- 执行成本管理

否则，工具很容易变成“随意调用的能力”，失去 Runtime 的治理边界。

---

### 3.4 Memory 扩展

Memory 也不应该被固定在某一种存储实现中。

不同场景需要不同的记忆方式：

- 短期会话记忆
- 长期用户记忆
- 结构化知识记忆
- 语义记忆
- 图谱记忆
- 事件记忆

也就是说，Memory 是一个扩展接口，而不是一个单一数据库。

---

### 3.5 Scheduler 扩展

Scheduler 不只是调度“谁先执行”，还要支持：

- 优先级调度
- 资源约束
- 并发控制
- backpressure
- 失败重试
- 执行超时
- 中断与恢复

不同业务场景下，调度策略差异很大，不能写死。

---

### 3.6 Human-in-the-loop 扩展

Agent 常常不是单机自动模式，而是和人协作。

用户可能在以下场景中介入：

- 选择方案
- 填充缺失字段
- 审批结果
- 修改模型输出
- 继续执行或中止任务

因此 Human-in-the-loop 也必须是扩展点，而不是一个固定状态。

---

## 4. Runtime Extension 不是接口，而是契约

Runtime 的扩展能力真正成立的前提，是：

> 扩展必须是显式、稳定、可治理的。

换句话说，扩展不是“在代码里随便插一段”，而应该遵循明确的契约。

一个好的扩展机制，需要满足几个原则：

### 4.1 明确契约
每个扩展点必须说明：

- 输入是什么
- 输出是什么
- 触发时机
- 是否允许副作用
- 是否支持重试
- 是否参与状态流转

### 4.2 低耦合
扩展点不要直接依赖 Runtime 内部实现细节，而应依赖抽象接口。

### 4.3 可审计
所有扩展逻辑都应该具备日志、事件回溯、执行追踪能力。

### 4.4 可恢复
扩展不能破坏 checkpoint 和 resume 能力，否则 Agent 的长期执行能力会失效。

### 4.5 可治理
扩展必须能够被权限控制、版本管理和上线审批约束。

---

## 5. 一个更合理的 Runtime 结构

可以把 Runtime 理解成三层结构：

- Core Runtime：状态机、事件总线、checkpoint、scheduler、执行上下文
- Extension Layer：tool、memory、human hook、observability、workflow adapters
- Domain Layer：CRM、GitHub、ERP、ticketing、数据库、消息系统等业务适配

这种分层有一个重要意义：

> Core 稳定，Extension 灵活，Domain 可替换。

这也正是企业级 Agent Runtime 的工程价值。

---

## 6. Runtime Extension 的本质

如果说前面的文章解释了“Runtime 怎么运行”，那么这篇文章说的是：

> Runtime 不是静态框架，而是动态扩展平台。

真正成熟的 Agent，不只是一个“能调用模型的脚本”，而是一个具备执行生命周期、状态管理、事件驱动、恢复能力和扩展能力的系统。

它能够：

- 连接更多外部能力
- 适配不同业务流程
- 支持更多状态和事件
- 接入更多系统
- 具备恢复、审计和治理能力
- 随着业务演进持续扩展

这就是 Runtime Extension 的价值。

---

## 7. 结论

Runtime 的核心价值，不只是“托管执行”，而是“托管可扩展执行”。

从工程视角看，Agent Runtime 不是一套固定模块，而是一种可以持续演进的执行平台。

它必须具备：

- 明确生命周期
- 稳定状态管理
- 可靠事件驱动
- 可恢复的执行模型
- 可扩展的工具与记忆
- 人机协作扩展
- 观测与治理能力

只有在这些能力都发挥出来时，Agent 才真正从 demo 走向生产系统。

这也是 Runtime 需要扩展的根本原因。