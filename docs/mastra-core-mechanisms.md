# Mastra 核心机制详解

> 用费曼学习法解释 Mastra 的 4 个核心机制：Agent、Tool、Workflow、Memory

## 目录

1. [Agent 机制](#1-agent-机制)
2. [Tool 机制](#2-tool-机制)
3. [Workflow 机制](#3-workflow-机制)
4. [Memory 机制](#4-memory-机制)
5. [四个机制的关系](#四个机制的关系)
6. [实战示例](#实战示例完整的飞书助手)

---

## 🎯 问题 1：Mastra 如何处理 agent 的自主调用？

### 用简单的话说

想象你是一个项目经理，手里有一份工具清单。当老板给你任务时：
1. 你看看任务需要什么
2. 从工具清单中选择合适的工具
3. 使用工具完成任务
4. 如果一个工具不够，继续选择下一个
5. 直到任务完成或达到最大尝试次数

**Mastra 的 Agent 就是这样工作的！**

### 技术实现

**核心流程**：
```
用户消息 → Agent.stream() → 工具转换 → LLM 决策 → 工具执行 → 循环
```

**项目代码示例**（`genca-agent.ts:119-140`）：
```typescript
tools: ({ requestContext }) => {
  const canUseSkillCommand = Boolean(requestContext.get("can_use_skill_command"));
  return {
    weatherTool,
    paymentReceiptTool,
    // ... 其他工具
    ...(canUseSkillCommand ? { skillRunCommandTool } : {}),
    ...mcpTools,  // 动态加载的 MCP 工具
  };
}
```

### 5 个关键步骤

#### 步骤 1：工具准备
- Agent 的 `tools` 配置支持**函数形式**
- 函数接收 `requestContext`（包含用户权限、配置等）
- 根据上下文**动态返回**可用工具集合

#### 步骤 2：工具转换
- `convertTools()` 收集所有来源的工具：
  - 用户配置的工具
  - Memory 提供的工具
  - Workspace 提供的工具
  - MCP 服务器提供的工具
- 转换为标准的 `CoreTool` 格式：
  ```typescript
  {
    description: "获取城市天气信息",
    parameters: {
      type: "object",
      properties: {
        city: { type: "string", description: "城市名称" }
      }
    }
  }
  ```

#### 步骤 3：LLM 决策（这是"自主"的关键！）
- Mastra 将工具的 `description` 和 `parameters` 发送给 LLM
- LLM 根据用户消息和工具描述，**自主选择**调用哪个工具
- LLM 返回：
  ```json
  {
    "toolName": "weatherTool",
    "args": { "city": "Beijing" }
  }
  ```

#### 步骤 4：工具执行
- Mastra 执行 LLM 选择的工具
- 获取结果：`{ temperature: 25, condition: "晴天" }`
- 将结果返回给 LLM

#### 步骤 5：循环控制
- LLM 看到工具结果后，决定是否需要调用更多工具
- `maxSteps` 参数控制最大循环次数（默认无限制）
- 每次 LLM 调用 + 工具执行 = 1 step
- 达到 `maxSteps` 或 LLM 返回最终答案时停止

### 关键洞察

> **Agent 不是"调用"工具，而是 LLM 在"选择"工具！**
>
> Mastra 只负责：
> 1. 提供工具清单给 LLM
> 2. 执行 LLM 选择的工具
> 3. 把结果返回给 LLM
>
> 真正的"自主决策"发生在 LLM 内部！

**关键文件**：
- `/dist/agent/agent.d.ts` - Agent 类定义
- `/dist/loop/types.d.ts` - 循环控制类型
- `/dist/stream/aisdk/v5/compat/prepare-tools.d.ts` - 工具准备

---

## 🎯 问题 2：Agent 是怎么具备 tools 或者自主查找 tools 的？

### 用简单的话说

想象你的工具箱有多个抽屉：
- **第一层**：你自己买的工具（用户配置的 tools）
- **第二层**：公司配发的工具（Memory 提供的 tools）
- **第三层**：从仓库借的工具（Workspace 提供的 tools）
- **第四层**：从外部租的工具（MCP 服务器提供的 tools）

当你需要工具时，你会**打开所有抽屉**，把所有工具摆在桌上，然后选择最合适的。

**Agent 不是"查找"工具，而是在初始化时就"收集"好所有工具！**

### 技术实现

#### 工具来源聚合（Mastra 内部逻辑）

```typescript
// 伪代码展示工具聚合逻辑
async convertTools({ requestContext }) {
  const tools = {};

  // 1. 用户配置的工具
  const assignedTools = await this.listAssignedTools({ requestContext });
  Object.assign(tools, assignedTools);

  // 2. Memory 提供的工具（如 updateWorkingMemory）
  if (this.memory) {
    const memoryTools = this.memory.listTools();
    Object.assign(tools, memoryTools);
  }

  // 3. Workspace 提供的工具（如文件操作）
  if (this.workspace) {
    const workspaceTools = await this.workspace.listTools({ requestContext });
    Object.assign(tools, workspaceTools);
  }

  // 4. 子 Agents 作为工具
  const agentTools = await this.listAgentTools({ requestContext });
  Object.assign(tools, agentTools);

  // 5. Workflows 作为工具
  const workflowTools = await this.listWorkflowTools({ requestContext });
  Object.assign(tools, workflowTools);

  return tools;
}
```

#### 项目实现示例

**静态 + 动态工具**（`genca-agent.ts:119-140`）：
```typescript
tools: ({ requestContext }) => {
  const canUseSkillCommand = Boolean(requestContext.get("can_use_skill_command"));
  return {
    // 静态核心工具（总是可用）
    weatherTool,
    paymentReceiptTool,
    sampleApplicationTool,
    getBrandListTool,
    // ... 更多工具

    // 条件工具（根据权限动态加载）
    ...(canUseSkillCommand ? { skillRunCommandTool } : {}),

    // 动态 MCP 工具（从外部服务器加载）
    ...mcpTools,
  };
}
```

**MCP 工具加载**（`genca-agent.ts:38-51`）：
```typescript
let mcpTools = {};

if (!isBuildPhase && process.env.JIANHUA_MCP_API_KEY) {
  try {
    // 从 jianhua MCP 服务器加载工具
    mcpTools = await jianhuaMcpClient.listTools();
  } catch (error) {
    console.error("Failed to load MCP tools from jianhua:", error);
  }
}
```

### 5 个工具来源

| 来源 | 说明 | 示例 |
|------|------|------|
| **用户配置** | 直接在 Agent 配置中定义 | `weatherTool`, `paymentReceiptTool` |
| **Memory** | Memory 实例提供的工具 | `updateWorkingMemory` |
| **Workspace** | Workspace 提供的文件操作工具 | `readFile`, `writeFile` |
| **MCP 服务器** | 外部 MCP 服务器提供的工具 | `jianhuaMcpClient.listTools()` |
| **Sub Agents** | 其他 Agents 作为工具 | `weatherAgent`, `codeAgent` |

### 关键洞察

> **Agent 不需要"查找"工具，而是在初始化时就收集好所有可用工具！**
>
> 工具收集发生在：
> 1. Agent 创建时（静态工具）
> 2. Agent.stream() 调用时（动态工具，根据 requestContext）
>
> 收集完成后，所有工具都在"工具箱"里，LLM 可以自由选择。

---

## 🎯 问题 3：多 agent 之间是怎么共享信息的？

### 用简单的话说

想象一个团队协作场景：
- **项目经理（Routing Agent）**：接收任务，决定分配给谁
- **专家团队（Sub Agents）**：各自负责不同领域
- **共享笔记本（Memory）**：所有人都可以读写
- **对话记录（Conversation Context）**：记录谁说了什么

当项目经理分配任务时：
1. 把之前的对话记录给专家看
2. 告诉专家去哪个笔记本查资料
3. 专家完成后，把结果写回笔记本
4. 项目经理看到结果，决定下一步

### 技术实现

#### Agent Network 架构

```
┌─────────────────────────────────────────┐
│ Routing Agent (路由代理)                │
│ - 接收用户消息                          │
│ - 列出可用的 agents/workflows/tools     │
│ - 使用 LLM 选择最合适的 primitive       │
└────────────┬────────────────────────────┘
             │
    ┌────────┼────────┐
    ▼        ▼        ▼
┌────────┐ ┌────────┐ ┌────────┐
│Agent A │ │Agent B │ │Agent C │
│(天气)  │ │(代码)  │ │(研究)  │
└────────┘ └────────┘ └────────┘
    │        │        │
    └────────┼────────┘
             ▼
    ┌─────────────────┐
    │ Shared Memory   │
    │ - Thread ID     │
    │ - Resource ID   │
    └─────────────────┘
```

#### 4 种信息共享方式

**方式 1：Conversation Context（对话上下文）**
```typescript
// 从 routing agent 传递对话历史给子 agent
const messagesForSubAgent = [
  ...conversationContext,  // 之前的对话
  { role: "user", content: "查询北京天气" }
];

const result = await weatherAgent.stream(messagesForSubAgent, {
  requestContext,
  memory: { thread: threadId, resource: resourceId }
});
```

**方式 2：Memory Thread（记忆线程）**
```typescript
// 所有 agents 使用相同的 threadId
await agentA.stream(messages, {
  memory: {
    thread: 'conversation-123',  // 相同的 thread
    resource: 'user-456',
  }
});

await agentB.stream(messages, {
  memory: {
    thread: 'conversation-123',  // 相同的 thread
    resource: 'user-456',
  }
});
```
- 两个 agents 可以看到彼此的消息
- 消息按时间顺序排列

**方式 3：Resource ID（资源标识）**
- 标识资源范围（通常是用户 ID）
- 控制跨对话的记忆隔离
- 支持跨 threads 的 Working Memory

**方式 4：Request Context（请求上下文）**
```typescript
const requestContext = new RequestContext();
requestContext.set("user_id", "user-456");
requestContext.set("user_token", "token-xxx");
requestContext.set("selected_model", "gpt-4");

// 传递给所有 agents
await agent.stream(messages, { requestContext });
```

#### Routing Agent 如何选择子 Agent

**步骤 1：收集可用资源**
```typescript
const agents = await routingAgent.listAgents({ requestContext });
// 返回：{ weatherAgent, codeAgent, researchAgent }
```

**步骤 2：生成选择提示**
```
选择最合适的 primitive 来处理任务：
- primitiveId: string (agent/workflow/tool 的 ID)
- primitiveType: "agent" | "workflow" | "tool" | "none"
- prompt: string (发送给 primitive 的输入)
- selectionReason: string (选择理由)

可用的 agents:
- weatherAgent: 获取天气信息
- codeAgent: 编写和调试代码
- researchAgent: 进行深度研究
```

**步骤 3：LLM 决策**
```json
{
  "primitiveId": "weatherAgent",
  "primitiveType": "agent",
  "prompt": "查询北京的天气",
  "selectionReason": "用户询问天气信息，weatherAgent 最合适"
}
```

**步骤 4：执行子 Agent**
```typescript
const selectedAgent = agents[result.primitiveId];
const output = await selectedAgent.stream(messagesForSubAgent, {
  requestContext,
  memory: { thread: threadId, resource: resourceId }
});
```

### 关键洞察

> **多 Agent 协作的核心是"共享上下文"！**
>
> 共享机制：
> 1. **Conversation Context**：传递对话历史
> 2. **Memory Thread**：共享消息存储
> 3. **Resource ID**：跨对话的用户数据
> 4. **Request Context**：动态配置和权限
>
> Routing Agent 使用 LLM 进行智能路由，而不是硬编码规则！

**关键文件**：
- `/dist/loop/network/index.d.ts` - Network Loop 定义
- `/dist/agent/agent.types.d.ts` - Agent Network 类型
- `/dist/chunk-4XSAZPPS.js` - Network 实现

---

## 🎯 问题 4：父子 agent 是怎么共享记忆的？

### 用简单的话说

想象一个家庭的记忆系统：
- **对话记录本（Thread）**：记录一次完整的对话
- **家庭档案柜（Resource）**：存储家庭成员的所有对话记录
- **便签板（Working Memory）**：记录重要信息（如"妈妈喜欢吃辣"）
- **观察日记（Observational Memory）**：记录长期观察（如"孩子最近压力大"）

当爸爸和孩子对话后，妈妈可以：
1. 翻看对话记录本（读取 Thread 消息）
2. 查看便签板（读取 Working Memory）
3. 阅读观察日记（读取 Observational Memory）

### 技术实现

#### Memory 架构

```
┌─────────────────────────────────────────────────┐
│ Resource (资源，如用户 ID: "user-123")          │
├─────────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────────┐ │
│ │ Thread 1 (对话 1)                           │ │
│ │ - Message 1: "你好"                         │ │
│ │ - Message 2: "你好！我是 Vink"              │ │
│ │ - Message 3: "天气怎么样？"                 │ │
│ └─────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────┐ │
│ │ Thread 2 (对话 2)                           │ │
│ │ - Message 1: "继续之前的话题"               │ │
│ │ - Message 2: "好的，关于天气..."            │ │
│ └─────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────┐ │
│ │ Working Memory (工作记忆，scope: resource)  │ │
│ │ {                                           │ │
│ │   "name": "张三",                           │ │
│ │   "location": "北京",                       │ │
│ │   "preferences": "喜欢表格展示"             │ │
│ │ }                                           │ │
│ └─────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────┐ │
│ │ Observational Memory (观察记忆)             │ │
│ │ - "用户经常询问天气信息"                    │ │
│ │ - "用户偏好简洁的回答"                      │ │
│ └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

#### 3 种记忆类型

**类型 1：Message History（消息历史）**
```typescript
// 最近的对话消息
await agent.stream(messages, {
  memory: {
    thread: 'thread-123',
    resource: 'user-456',
    options: {
      lastMessages: 10  // 检索最近 10 条消息
    }
  }
});
```
- 存储在 Thread 级别
- 按时间顺序排列
- 用于短期上下文

**类型 2：Working Memory（工作记忆）**
```typescript
// Agent 配置
memory: new Memory({
  options: {
    workingMemory: {
      enabled: true,
      scope: 'resource',  // 跨所有 threads 共享
      schema: z.object({
        name: z.string().optional(),
        location: z.string().optional(),
        preferences: z.string().optional(),
      }),
    }
  }
})
```
- 结构化的用户信息
- `scope: 'resource'`：跨所有对话共享
- `scope: 'thread'`：仅在当前对话内共享
- 通过 `updateWorkingMemory` 工具更新

**类型 3：Observational Memory（观察记忆）**
```typescript
memory: new Memory({
  options: {
    observationalMemory: {
      enabled: true,
      scope: 'resource',  // 跨所有 threads 共享
      model: {
        id: "openai/gpt-5-mini",
        url: "https://api.bltcy.ai/v1",
        apiKey: process.env.GOOGLE_API_KEY!,
      },
    }
  }
})
```
- Observer Agent 提取关键观察
- Reflector Agent 压缩观察内容
- 自动注入到后续对话

#### 项目实现示例

**完整的 Memory 配置**（`genca-agent.ts:142-161`）：
```typescript
memory: new Memory({
  options: {
    generateTitle: true,  // 自动生成对话标题

    // 观察记忆（长期）
    observationalMemory: {
      enabled: true,
      model: {
        id: "openai/gpt-5-mini",
        url: "https://api.bltcy.ai/v1",
        apiKey: process.env.GOOGLE_API_KEY!,
      },
      scope: "resource",  // 跨对话共享
    },

    // 工作记忆（结构化）
    workingMemory: {
      enabled: true,
      scope: "resource",  // 跨对话共享
      schema: userProfileSchema,  // 用户画像 schema
    },
  },
})
```

**用户画像 Schema**（`genca-agent.ts:53-60`）：
```typescript
const userProfileSchema = z.object({
  name: z.string().optional().describe('用户姓名'),
  location: z.string().optional().describe('所在城市'),
  department: z.string().optional().describe('所属部门'),
  role: z.string().optional().describe('职位'),
  businessFocus: z.string().optional().describe('主要负责的业务方向'),
  preferences: z.string().optional().describe('用户的操作偏好'),
});
```

#### 共享策略

**策略 1：相同 Thread（同一对话）**
```typescript
// Agent A 和 Agent B 使用相同的 thread
await agentA.stream(messages, {
  memory: { thread: 'conversation-123', resource: 'user-456' }
});

await agentB.stream(messages, {
  memory: { thread: 'conversation-123', resource: 'user-456' }
});
```
✅ 可以看到彼此的消息
✅ 可以访问相同的 Working Memory（如果 scope 是 thread）

**策略 2：相同 Resource（同一用户）**
```typescript
// Thread 1
await agent.stream(messages, {
  memory: { thread: 'thread-1', resource: 'user-456' }
});

// Thread 2（新对话）
await agent.stream(messages, {
  memory: { thread: 'thread-2', resource: 'user-456' }
});
```
✅ 可以跨对话访问 Working Memory（如果 scope 是 resource）
✅ 可以跨对话访问 Observational Memory（如果 scope 是 resource）
❌ 不能看到其他 thread 的消息历史

### 关键洞察

> **Memory 是分层的！**
>
> 三层记忆：
> 1. **Message History**（短期）：当前对话的消息
> 2. **Working Memory**（结构化）：用户画像和偏好
> 3. **Observational Memory**（长期）：AI 的观察和洞察
>
> Scope 控制共享范围：
> - `scope: 'thread'`：仅在当前对话内共享
> - `scope: 'resource'`：跨所有对话共享

**关键文件**：
- `/dist/memory/memory.d.ts` - Memory 抽象类
- `/dist/memory/types.d.ts` - Memory 配置类型
- `/dist/storage/domains/memory/base.d.ts` - 存储接口
- `/dist/processors/memory/working-memory.d.ts` - Working Memory 处理器

---

## 📊 总结：4 个机制的关系

```
┌─────────────────────────────────────────────────────────┐
│ 用户消息                                                │
└────────────┬────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────┐
│ 1. Agent 自主调用机制                                   │
│    ✓ 收集可用工具（从多个来源）                         │
│    ✓ 转换为标准格式                                     │
│    ✓ 发送给 LLM                                         │
│    ✓ LLM 自主选择工具                                   │
│    ✓ 执行工具并循环                                     │
└────────────┬────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────┐
│ 2. Agent 工具获取机制                                   │
│    ✓ 用户配置的工具                                     │
│    ✓ Memory 提供的工具                                  │
│    ✓ Workspace 提供的工具                               │
│    ✓ MCP 服务器提供的工具                               │
│    ✓ 动态加载（根据 requestContext）                    │
└────────────┬────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────┐
│ 3. 多 Agent 信息共享机制                                │
│    ✓ Routing Agent 选择子 Agent                         │
│    ✓ 传递对话上下文                                     │
│    ✓ 共享 Memory Thread                                 │
│    ✓ 共享 Resource ID                                   │
│    ✓ 传递 Request Context                               │
└────────────┬────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────┐
│ 4. 父子 Agent 记忆共享机制                              │
│    ✓ Thread: 同一对话的消息历史                         │
│    ✓ Resource: 跨对话的用户数据                         │
│    ✓ Working Memory: 结构化的用户信息                   │
│    ✓ Observational Memory: 长期观察和洞察               │
│    ✓ Scope 控制: thread 或 resource 级别                │
└─────────────────────────────────────────────────────────┘
```

---

## 💡 核心洞察

1. **Agent 是自主的**：LLM 根据工具描述自主选择调用哪个工具，Mastra 只负责提供工具和执行
2. **工具是聚合的**：工具来自多个来源，Mastra 自动收集并转换为统一格式
3. **信息是共享的**：通过 Thread、Resource、Request Context 实现多 Agent 协作
4. **记忆是分层的**：Message History（短期）、Working Memory（结构化）、Observational Memory（长期）
5. **上下文是动态的**：requestContext 控制工具可用性、模型选择、权限等

---

## 🚀 下一步建议

建议创建一个交互式示例项目，演示：
1. 单 Agent 的工具调用流程
2. 多 Agent 的协作流程
3. Memory 的跨对话共享
4. 动态工具加载

这样可以更直观地理解这些机制的实际应用。

---

**文档创建时间**：2026-03-11
**基于项目**：vinka_mastra
**Mastra 版本**：1.4.0
