# aigc-engineering-docs

分享关于 AIGC Engineering 的内容

## 计划

### 第一季
理解企业级 Agent 的设计思想

1. 什么是 Agent
2. Agent 必须是状态机
3. Agent Workflow 与传统工作流有什么区别
4. Long Running 才是真正的 Agent
5. Checkpoint 比 Prompt 更重要
6. Human-in-the-loop 是企业级 Agent 的标配
7. Memory 是企业级 Agent 的长期记忆
8. Event 才是 Agent Runtime 的驱动力
9. Agent Runtime: 企业级 Agent 的核心

### 第二季
工程实践: Building an Agent Runtime

1. 从零开始：设计我们的 Agent Runtime
2. Runtime Kernel：：为什么 Runtime 不直接调用 Agent？
3. State Machine：让 Runtime 拥有生命周期
4. Event Bus：Runtime 的神经系统
5. Checkpoint：实现 Long Running
6. Scheduler：让 Runtime 真正跑起来
7. [Tool Runtime：不仅仅是 Function Call](s2-07-tool-runtime.md)
8. [Memory：让 Runtime 拥有长期记忆](s2-08-memory.md)
9. [Human-in-the-loop：等待用户参与](s2-09-hitl.md)
9a. [Runtime 与 AI Loop：为什么 Agent 不能只靠传统编码](s2-09a-runtime-vs-ai-loop.md)
10. Observability：让 Runtime 可观测
11. Runtime Extension：让 Runtime 可以扩展
12. Building an Agent Runtime：完整回顾


## 架构图
```
                           User
                             │
                    HTTP / SDK / CLI
                             │
                             ▼
                   ┌───────────────────┐
                   │   Agent Runtime   │
                   └─────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
         Scheduler        Event Bus       Checkpoint
              │              │              │
              ▼              │              │
            Engine ─────────┼──────────────┘
              │              │
              │              └──────────────► Observability
              │
              ├───────────────┬───────────────┐
              │               │               │
              ▼               ▼               ▼
        Tool Runtime     Memory         Human-in-the-loop
              │               │               │
              └───────────────┴───────────────┘
                              │
                              ▼
                    Execution State / Resume
```

注：这里的 Event Bus 是横向基础设施，不是单独的一层；Human-in-the-loop 是执行中断/恢复的状态分支，而不是线性的最后一层。