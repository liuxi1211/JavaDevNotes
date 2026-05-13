
OpenClaw 提供了**压缩（Compaction）**、**会话剪枝 （Session Pruning）** 和**会话工具 （Session Tools）** 三大核心能力，用于解决长时间运行对话的上下文窗口限制、缓存成本优化以及多会话间交互管理问题。本教程整合官方文档所有内容，用清晰易懂的语言全面讲解这三项功能的原理、配置和使用方法。

## 一、压缩（Compaction）

### 1.1 核心原理

每个大语言模型都有固定的上下文窗口（可处理的最大 token 数）。当对话累积的消息和工具结果接近或超过窗口限制时，OpenClaw 会将**较早的对话历史总结为一条紧凑的摘要条目**，同时完整保留近期消息。

- 压缩后的摘要会**持久化保存到会话的 JSONL 历史文件**中
- 后续模型请求使用的上下文 = 压缩摘要 + 压缩点之后的所有近期消息
- 压缩前会自动运行一次静默记忆刷写轮次，将持久化笔记写入磁盘

### 1.2 自动压缩（默认开启）

- **触发条件**：会话接近或超过模型的上下文窗口时自动触发
- 触发后 OpenClaw 会使用压缩后的上下文重试原始请求
- **状态查看**：
    
    - 详细模式下会显示 `🧹 Auto-compaction complete`
    - 执行 `/status` 命令会显示 `🧹 Compactions: <执行次数>`
    

### 1.3 手动压缩

使用 `/compact` 命令可随时强制执行一次压缩，还可附带指令指定摘要重点：

```bash
/compact Focus on decisions and open questions
```

**使用提示**：当感觉会话内容过时或上下文臃肿时，手动执行压缩效果最佳。

### 1.4 上下文窗口来源

OpenClaw 按以下优先级确定模型的上下文窗口限制：

1. 配置文件中 `models.providers.*.models[].contextWindow` 的覆盖值
2. 已配置提供商目录中的模型定义
3. 默认值：200000 token

## 二、会话剪枝（Session Pruning）

### 2.1 核心原理

会话剪枝是**每次 LLM 调用前**在**内存中**临时移除旧的工具结果，**不会修改磁盘上的 JSONL 会话历史文件**。它是压缩之外的另一层上下文优化机制，专门针对工具输出堆积问题。

### 2.2 运行时机与适用范围

- **仅对 Anthropic API 调用（包括 OpenRouter 上的 Anthropic 模型）生效**
- 仅当启用 `mode: "cache-ttl"` 且该会话的最后一次 Anthropic 调用早于设置的`ttl`时运行
- 剪枝后 TTL 窗口会重置，后续请求可重用新缓存的提示

### 2.3 智能默认值

- OAuth 或 setup-token 配置文件：启用 cache-ttl 剪枝，ttl=1h
- API 密钥配置文件：启用 cache-ttl 剪枝，ttl=30m，同时将 Anthropic 模型的`cacheControlTtl`默认设为 1h
- 如果你显式设置了这些值，OpenClaw 不会覆盖

### 2.4 为什么要使用剪枝

Anthropic 的提示缓存仅在 TTL 内有效。如果会话空闲超过 TTL，下一个请求会重新缓存完整提示，产生额外的`cacheWrite`成本。剪枝通过减少需要缓存的内容，**显著降低 TTL 过期后第一个请求的成本**。

### 2.5 可剪枝与不可剪枝的内容

✅ **仅可剪枝**：`toolResult` 类型的消息

❌ **永远不会被修改**：用户消息和助手消息

❌ **受保护**：最后 `keepLastAssistants`（默认 3）条助手消息之后的工具结果

❌ **永不修剪**：包含图像块的工具结果

⚠️ 如果没有足够的助手消息确定截止点，剪枝会被跳过

### 2.6 软剪枝 vs 硬剪枝

剪枝会根据工具结果的大小自动选择不同的处理方式：

- **软修剪**：仅处理过大的工具结果，保留头部和尾部，中间用`...`代替，并附加原始大小注释
- **硬清除**：用配置的占位符（默认`[Old tool result content cleared]`）替换整个工具结果

### 2.7 配置选项与示例

#### 默认值（启用时）
```json
{
  "ttl": "5m",
  "keepLastAssistants": 3,
  "softTrimRatio": 0.3,
  "hardClearRatio": 0.5,
  "minPrunableToolChars": 50000,
  "softTrim": { "maxChars": 4000, "headChars": 1500, "tailChars": 1500 },
  "hardClear": { "enabled": true, "placeholder": "[Old tool result content cleared]" }
}
```

#### 配置示例

- 关闭剪枝：
```json
{
  "agent": {
    "contextPruning": { "mode": "off" }
  }
}
```

- 启用 TTL 感知剪枝：
```json
{
  "agent": {
    "contextPruning": { "mode": "cache-ttl", "ttl": "5m" }
  }
}
```

- 限制剪枝到特定工具：
```json
{
  "agent": {
    "contextPruning": {
      "mode": "cache-ttl",
      "tools": { "allow": ["exec", "read"], "deny": ["*image*"] }
    }
  }
}
```

> 工具匹配规则：支持`*`通配符，不区分大小写，拒绝优先；允许列表为空则允许所有工具。

## 三、压缩 vs 会话剪枝 对比

表格

|特性|压缩（Compaction）|会话剪枝（Session Pruning）|
|---|---|---|
|作用对象|所有历史消息（用户 + 助手 + 工具）|仅旧的工具结果消息|
|处理方式|总结为摘要|截断或清除内容|
|持久化|写入磁盘 JSONL 文件|仅在内存中临时生效|
|触发时机|上下文窗口接近满时|每次 LLM 调用前（满足 TTL 条件）|
|适用模型|所有模型|仅 Anthropic 系列模型|
|主要目的|解决上下文窗口不足问题|优化提示缓存成本|

## 四、会话工具（Session Tools）

会话工具是一组小型、安全的内置工具，使智能体能够列出会话、获取历史记录、向其他会话发送消息以及生成隔离的子智能体运行。

### 4.1 会话键模型

不同类型的会话有不同的键格式：

- 主直接聊天桶：固定为 `"main"`
- 群聊：`agent:<agentId>:<channel>:group:<id>` 或 `agent:<agentId>:<channel>:channel:<id>`
- 定时任务：`cron:<job.id>`
- Hooks：`hook:<uuid>`（除非明确设置）
- Node 会话：`node-<nodeId>`（除非明确设置）

> 注意：`global` 和 `unknown` 是保留值，永远不会被列出；`session.scope = "global"` 会被别名为 `"main"`。

### 4.2 sessions_list：列出所有会话

将会话列为行数组，支持多种过滤条件。

#### 参数

- `kinds?: string[]`：按会话类型过滤，可选值：`"main" | "group" | "cron" | "hook" | "node" | "other"`
- `limit?: number`：最大返回行数（默认：服务器默认值，通常 200）
- `activeMinutes?: number`：仅返回 N 分钟内更新过的会话
- `messageLimit?: number`：0 = 不包含消息（默认）；>0 = 包含每个会话的最后 N 条消息

#### 返回字段

包含`key`、`kind`、`channel`、`displayName`、`updatedAt`、`sessionId`、`model`、`totalTokens`、`transcriptPath`等完整会话信息。当`messageLimit > 0`时，还会包含`messages`字段。

> 注意：工具结果在列表输出中会被过滤，如需获取工具消息请使用`sessions_history`。

### 4.3 sessions_history：获取单个会话记录

获取指定会话的完整消息历史。

#### 参数

- `sessionKey`（必填）：接受会话键或来自`sessions_list`的`sessionId`
- `limit?: number`：最大返回消息数（受服务器限制）
- `includeTools?: boolean`：是否包含`toolResult`消息（默认：false）

### 4.4 sessions_send：向另一个会话发送消息

向指定会话发送消息，支持同步和异步两种模式。

#### 参数

- `sessionKey`（必填）：接受会话键或`sessionId`
- `message`（必填）：要发送的消息内容
- `timeoutSeconds?: number`：超时时间（默认 > 0；0 = 即发即忘）

#### 行为

- `timeoutSeconds = 0`：立即入队，返回 `{ runId, status: "accepted" }`
- `timeoutSeconds > 0`：等待最多 N 秒完成，返回 `{ runId, status: "ok", reply }`
- 超时返回：`{ runId, status: "timeout", error }`（运行仍在继续）
- 失败返回：`{ runId, status: "error", error }`

#### 智能体间交互

- 主运行完成后会自动进入回复循环，请求者和目标智能体交替回复
- 精确回复 `REPLY_SKIP` 可停止来回交互
- 最大交互轮数：`session.agentToAgent.maxPingPongTurns`（0-5，默认 5）
- 循环结束后会运行通告步骤，精确回复 `ANNOUNCE_SKIP` 可保持静默

#### 发送策略

可基于渠道和聊天类型配置发送权限：



```json
{
  "session": {
    "sendPolicy": {
      "rules": [
        {
          "match": { "channel": "discord", "chatType": "group" },
          "action": "deny"
        }
      ],
      "default": "allow"
    }
  }
}
```

也可按会话覆盖：`sendPolicy: "allow" | "deny"`（未设置则继承配置）。

### 4.5 sessions_spawn：生成隔离的子智能体

在全新的隔离会话中运行子智能体任务，完成后自动将结果通告回请求者渠道。

#### 参数

- `task`（必填）：子智能体需要执行的任务描述
- `label?: string`：用于日志和 UI 的标签
- `agentId?: string`：在另一个智能体 ID 下运行（需在允许列表中）
- `model?: string`：覆盖子智能体使用的模型
- `runTimeoutSeconds?: number`：子智能体运行超时时间（默认 0，永不超时）
- `cleanup?: string`：任务完成后是否删除子会话，可选值：`delete|keep`（默认：keep）

#### 行为

- 立即返回 `{ status: "accepted", runId, childSessionKey }`，始终非阻塞
- 子智能体默认使用完整工具集减去会话工具（可通过`tools.subagents.tools`配置）
- 子智能体不允许调用`sessions_spawn`（禁止嵌套生成）
- 子智能体会话在`agents.defaults.subagents.archiveAfterMinutes`（默认 60 分钟）后自动归档
- 完成后自动运行通告步骤，精确回复`ANNOUNCE_SKIP`可保持静默
- 通告回复会规范化为`Status/Result/Notes`格式，并包含运行时间、token 消耗、会话 ID、记录路径和可选成本统计
    

#### 权限控制

- 子智能体生成权限由`agents.list[].subagents.allowAgents`配置，默认仅允许生成当前智能体自身的子实例
    
- 设置为`["*"]`可允许生成任意智能体的子实例
    
- 使用`agents_list`工具可发现当前允许生成的所有智能体 ID
    

### 4.6 沙箱会话可见性

沙箱隔离的智能体会话默认只能看到自己通过`sessions_spawn`生成的子会话，可通过配置修改：

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "sessionToolsVisibility": "spawned" // 可选值："spawned"（默认）或 "all"
      }
    }
  }
}
```

## 五、最佳实践与常见问题

### 5.1 上下文优化组合策略

1. **日常使用**：保持自动压缩开启，它会在上下文窗口不足时自动处理
    
2. **长会话优化**：当感觉对话 "跑偏" 或上下文臃肿时，手动执行`/compact`并指定摘要重点
    
3. **Anthropic 模型成本优化**：启用`cache-ttl`剪枝，将`ttl`与`cacheControlTtl`设置为相同值
    
4. **大型工具输出处理**：内置工具已自动截断输出，剪枝是额外的防护层，防止工具结果无限堆积
    
5. **完全重置**：如果需要全新开始，使用`/new`或`/reset`命令启动新的会话 ID
    

### 5.2 常见误区澄清

- ❌ 剪枝会删除历史记录：不会，剪枝仅在内存中临时移除工具结果，磁盘上的 JSONL 文件保持完整
    
- ❌ 压缩会丢失信息：压缩是智能总结，会保留关键信息；如果需要完整历史，可随时查看 JSONL 文件
    
- ❌ 剪枝会增加成本：恰恰相反，剪枝能显著降低 TTL 过期后第一个请求的 cacheWrite 成本
    
- ❌ 子智能体可以无限嵌套：不可以，子智能体不允许调用`sessions_spawn`生成自己的子实例
    

### 5.3 状态查看命令

- `/status`：查看当前会话状态，包括压缩执行次数、token 使用情况等
    
- `/compact`：手动执行压缩
    
- `/new`：启动新会话
    
- `/reset`：重置当前会话
    