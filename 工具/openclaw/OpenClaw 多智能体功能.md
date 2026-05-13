## 一、多智能体到底是干嘛的？

简单说：你只需要启动**一个 OpenClaw Gateway 网关服务**，就能同时运行多个完全独立的 AI 智能体。

- 每个智能体都有自己独立的 “大脑”（人设、规则、数据、会话历史），互不干扰、数据完全隔离
    
- 可同时接入多个渠道账号（比如 2 个 WhatsApp 号、WhatsApp+Telegram 双渠道）
    
- 能设置精准的规则，把不同来源的消息，自动分给对应的智能体处理
    
- 支持多个人共用同一个网关服务，每个人的 AI 数据彻底分开，不会串号
    

## 二、核心定义：一个 “独立智能体” 的完整构成

一个智能体就是一个完全隔离的 AI 运行单元，有专属的全套资源，和其他智能体彻底切割，不会互相影响。它包含 6 个专属核心部分：

1. **专属工作区** 就是智能体的 “工作文件夹”，存放它要用的文件、人设规则文件（[AGENTS.md/SOUL.md/USER.md](AGENTS.md/SOUL.md/USER.md)）、本地笔记、自定义指令。 注意：工作区是智能体运行的默认路径，相对路径只会在这个文件夹内解析；但如果不开启沙箱隔离，绝对路径仍能访问你主机的其他文件，存在安全风险。
    
2. **专属状态目录（agentDir）** 存放这个智能体的认证配置文件、模型注册表、专属配置项，是智能体的 “核心配置文件夹”。 ❗ 关键避坑：绝对不能让多个智能体共用同一个 agentDir，会直接导致认证失败、会话串号、数据冲突。
    
3. **专属会话存储** 这个智能体的所有聊天历史、路由状态，都单独存放在这里，路径为`~/.openclaw/agents/<agentId>/sessions`，和其他智能体的聊天记录完全隔离。
    
4. **独立的认证配置** 每个智能体的账号凭证都是独立的，从自己 agentDir 里的`auth-profiles.json`文件读取，主智能体的凭证不会自动共享给其他智能体。 若想让多个智能体共用一套凭证，手动把这个 json 文件复制到对应智能体的 agentDir 里即可。
    
5. **独立的技能（Skills）** 每个智能体可以使用自己工作区`skills/`文件夹里的专属技能；所有智能体都能共用全局技能文件夹`~/.openclaw/skills`里的公共技能。
    
6. **独立的权限与沙箱配置** 从 v2026.1.6 版本开始，每个智能体可单独设置沙箱隔离规则、工具使用权限，比如个人智能体完全放开权限，公开群用的智能体仅开放只读权限。
    

## 三、关键路径速查表（一眼找到对应文件）

|           |                                                              |                                                         |
| --------- | ------------------------------------------------------------ | ------------------------------------------------------- |
| 配置 / 文件类型 | 默认路径                                                         | 作用说明                                                    |
| 全局主配置文件   | `~/.openclaw/openclaw.json`（可通过环境变量`OPENCLAW_CONFIG_PATH`修改） | 整个 OpenClaw 网关的核心配置，多智能体、路由、渠道都在这里配置                    |
| 全局状态根目录   | `~/.openclaw`（可通过环境变量`OPENCLAW_STATE_DIR`修改）                 | 存放所有智能体、凭证、全局配置的根文件夹                                    |
| 默认工作区     | `~/.openclaw/workspace`                                      | 单智能体模式下的默认工作文件夹；多智能体模式下，每个智能体使用独立的`workspace-<agentId>` |
| 智能体专属配置目录 | `~/.openclaw/agents/<agentId>/agent`                         | 对应 agentId 智能体的 agentDir，存放它的认证、核心配置文件                  |
| 智能体会话存储目录 | `~/.openclaw/agents/<agentId>/sessions`                      | 对应 agentId 智能体的所有聊天历史、会话状态                              |

## 四、默认模式：单智能体运行

如果你不做任何额外配置，OpenClaw 默认启动单智能体模式，核心参数如下：

- 智能体唯一 ID（agentId）：默认叫`main`
    
- 会话标识：`agent:main:<mainKey>`
    
- 工作区：默认`~/.openclaw/workspace`（若设置了`OPENCLAW_PROFILE`环境变量，则为`~/.openclaw/workspace-<profile>`）
    
- 专属配置目录：`~/.openclaw/agents/main/agent`
    

## 五、快速上手：怎么添加新的隔离智能体

OpenClaw 自带智能体向导，一行命令就能创建新的独立智能体：

1. **创建新智能体** 执行命令：`openclaw agents add <智能体ID>` 比如要创建一个工作专用智能体，就执行：`openclaw agents add work` 执行后向导会引导你完成基础配置，包括添加路由绑定规则。
    
2. **验证智能体与绑定规则** 执行命令：`openclaw agents list --bindings` 即可查看当前所有已创建的智能体，以及对应的路由绑定规则，快速检查配置是否正确。
    

## 六、3 个核心名词，必须先搞懂

所有多智能体配置都围绕这三个核心概念，先搞懂再看配置，事半功倍：

1. **agentId（智能体 ID）** 每个智能体的唯一 “身份证号”，一个 agentId 对应一个完全独立的 AI 大脑，有自己的工作区、配置、会话、人设。你可以把它理解成 “一个独立的 AI 账号”。
    
2. **accountId（渠道账号 ID）** 每个渠道（比如 WhatsApp、Telegram）的登录账号的唯一标识。比如你有 2 个 WhatsApp 号，一个私人用一个工作用，就可以分别给它们设置 accountId 为`personal`和`biz`，用来区分不同的登录账号。
    
3. **binding（绑定 / 路由规则）** 就是给网关定的 “消息分发规则”，告诉网关：**符合什么条件的入站消息，要发给哪个 agentId 的智能体处理**。 规则的核心是匹配条件，比如 “来自 WhatsApp 的 personal 账号的消息，全发给 home 智能体”、“来自这个群的消息，全发给 family 智能体”。
    

## 七、路由规则优先级：消息怎么找到对应的智能体？

绑定规则是**确定性匹配，越具体的规则，优先级越高，先匹配生效**，优先级从高到低排序如下：

1. 最高优先级：精确匹配`peer`（私信 / 群组 / 频道的 ID）
    
2. 第二优先级：匹配`guildId`（Discord 的服务器 ID）/`teamId`（Slack 的团队 ID）
    
3. 第三优先级：匹配渠道的`accountId`（渠道账号 ID）
    
4. 第四优先级：渠道级匹配（比如所有 WhatsApp 的消息，不管哪个账号）
    
5. 最低优先级：所有规则都匹配不上时，走默认智能体
    
    1. 默认智能体优先选择配置里标了`default: true`的
        
    2. 没标注就选`agents.list`里的第一个智能体
        
    3. 都没配置就走默认的`main`智能体
        

❗ 关键提醒：只要有更精准的规则匹配上，就不会走后面的宽泛规则。比如你配了 “某一个私信发给 opus 智能体”，又配了 “所有 WhatsApp 消息发给 chat 智能体”，那这个私信一定会走 opus，因为 peer 匹配优先级远高于渠道匹配。

## 八、全场景配置示例（拿来就能改）

下面覆盖官方所有典型使用场景，每个示例都讲清解决的问题、配置内容、关键要点。

### 示例 1：1 个 WhatsApp 号，不同私信分给不同智能体

#### 解决的问题

你只有一个 WhatsApp 号，但想让不同人发的私信，由不同的智能体处理，比如 Alex 的私信走 alex 智能体，Mia 的私信走 mia 智能体，数据完全隔离，回复都用你这同一个 WhatsApp 号。

#### 完整配置（写在 openclaw.json 里）

```json5
{
  // 配置所有智能体
  agents: {
    list: [
      { id: "alex", workspace: "~/.openclaw/workspace-alex" },
      { id: "mia", workspace: "~/.openclaw/workspace-mia" },
    ],
  },
  // 配置路由绑定规则
  bindings: [
    { agentId: "alex", match: { channel: "whatsapp", peer: { kind: "dm", id: "+15551230001" } } },
    { agentId: "mia", match: { channel: "whatsapp", peer: { kind: "dm", id: "+15551230002" } } },
  ],
  // 配置WhatsApp渠道
  channels: {
    whatsapp: {
      dmPolicy: "allowlist", // 私信只允许白名单内的号码
      allowFrom: ["+15551230001", "+15551230002"], // 白名单号码
    },
  },
}
```

#### 关键要点 & 避坑

- 私信的白名单是**整个 WhatsApp 账号全局生效**的，不是按智能体分开配置的
    
- 这种方式回复消息，都来自你同一个 WhatsApp 号，不会给每个智能体单独分配发送身份
    
- 要实现真正的完全隔离，必须给每个发送者分配一个独立的智能体
    
- 共享的群组消息，只能绑定到一个智能体，不能自动拆分
    

### 示例 2：2 个 WhatsApp 号，分别对应家庭 / 工作双智能体

#### 解决的问题

你有两个 WhatsApp 号，一个私人家用，一个工作用，想让两个号的消息分别由两个完全独立的智能体处理，跑在同一个网关里，会话、配置完全不串。

#### 完整配置

```json5
{
  // 配置两个独立智能体
  agents: {
    list: [
      {
        id: "home",
        default: true, // 设为默认智能体，匹配不到规则的消息都走这里
        name: "Home",
        workspace: "~/.openclaw/workspace-home",
        agentDir: "~/.openclaw/agents/home/agent",
      },
      {
        id: "work",
        name: "Work",
        workspace: "~/.openclaw/workspace-work",
        agentDir: "~/.openclaw/agents/work/agent",
      },
    ],
  },

  // 路由规则：最具体的优先，第一个匹配上就生效
  bindings: [
    // 私人WhatsApp号的消息，全发给home智能体
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    // 工作WhatsApp号的消息，全发给work智能体
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },

    // 可选规则：哪怕是私人号，某个特定工作群的消息，也发给work智能体
    {
      agentId: "work",
      match: {
        channel: "whatsapp",
        accountId: "personal",
        peer: { kind: "group", id: "1203630...@g.us" }, // 群的ID
      },
    },
  ],

  // 智能体之间的通信配置，默认关闭
  tools: {
    agentToAgent: {
      enabled: false, // 关闭智能体之间互发消息
      allow: ["home", "work"], // 如果开启，只允许这两个智能体互相通信
    },
  },

  // 配置WhatsApp的两个账号
  channels: {
    whatsapp: {
      accounts: {
        personal: {
          // 可选：自定义这个账号的凭证存放路径，默认是~/.openclaw/credentials/whatsapp/personal
          // authDir: "~/.openclaw/credentials/whatsapp/personal",
        },
        biz: {
          // 可选：自定义工作号的凭证路径
          // authDir: "~/.openclaw/credentials/whatsapp/biz",
        },
      },
    },
  },
}
```

#### 关键要点

- 每个 WhatsApp 账号对应一个 accountId，和智能体一一绑定，彻底隔离会话
    
- 智能体之间的通信默认是关闭的，必须手动开启并配置允许列表，避免数据泄露
    
- 群匹配的规则优先级高于账号匹配，所以哪怕是私人号的群，也能路由给工作智能体
    

### 示例 3：按渠道分割，WhatsApp 日常闲聊 + Telegram 深度工作

#### 解决的问题

想让不同的聊天渠道，用不同的智能体和模型：WhatsApp 用来日常闲聊，用速度快的 Sonnet 模型；Telegram 用来深度工作，用能力更强的 Opus 模型，两个渠道的消息完全分开。

#### 完整配置

```json5
{
  agents: {
    list: [
      {
        id: "chat",
        name: "Everyday",
        workspace: "~/.openclaw/workspace-chat",
        model: "anthropic/claude-sonnet-4-5", // 日常用Sonnet模型
      },
      {
        id: "opus",
        name: "Deep Work",
        workspace: "~/.openclaw/workspace-opus",
        model: "anthropic/claude-opus-4-5", // 深度工作用Opus模型
      },
    ],
  },
  // 路由规则：按渠道匹配
  bindings: [
    { agentId: "chat", match: { channel: "whatsapp" } },
    { agentId: "opus", match: { channel: "telegram" } },
  ],
}
```

#### 关键要点

- 如果一个渠道有多个账号，要在匹配规则里加上 accountId，比如`{ channel: "whatsapp", accountId: "personal" }`
    
- 想让某个特定的私信 / 群走 Opus 模型，其他同渠道的走日常，只要加一条带 peer 匹配的规则，优先级会自动高于渠道级规则
    

### 示例 4：同一渠道，大部分消息走日常，某一个私信走深度模型

#### 解决的问题

同一个 WhatsApp 号，大部分朋友的闲聊都用日常智能体，但有一个合作方的私信，想用能力更强的 Opus 深度智能体处理，自动区分，不用手动切换。

#### 完整配置

```json5
{
  agents: {
    list: [
      {
        id: "chat",
        name: "Everyday",
        workspace: "~/.openclaw/workspace-chat",
        model: "anthropic/claude-sonnet-4-5",
      },
      {
        id: "opus",
        name: "Deep Work",
        workspace: "~/.openclaw/workspace-opus",
        model: "anthropic/claude-opus-4-5",
      },
    ],
  },
  // 路由规则：精准匹配优先
  bindings: [
    // 特定号码的私信，先走opus智能体
    { agentId: "opus", match: { channel: "whatsapp", peer: { kind: "dm", id: "+15551234567" } } },
    // 其他所有WhatsApp消息，走chat智能体
    { agentId: "chat", match: { channel: "whatsapp" } },
  ],
}
```

#### 关键要点

- peer（私信 / 群 ID）匹配的优先级永远最高，所以一定要把精准规则放在宽泛规则的上面，逻辑更清晰
    
- 只要匹配上了精准规则，就不会走后面的渠道级规则
    

### 示例 5：家庭群专属智能体，带提及限制 + 严格权限控制

#### 解决的问题

给家庭 WhatsApp 群做一个专属的家庭助手智能体，只有 @指定关键词才会响应，严格限制它的权限，只能用指定工具，不能修改文件、不能执行高危命令，用沙箱彻底隔离，保证主机安全。

#### 完整配置

```json5
{
  agents: {
    list: [
      {
        id: "family",
        name: "Family",
        workspace: "~/.openclaw/workspace-family",
        identity: { name: "Family Bot" }, // 智能体在群里的名字
        // 群聊配置：只有@指定关键词，才会响应
        groupChat: {
          mentionPatterns: ["@family", "@familybot", "@Family Bot"],
        },
        // 开启沙箱，彻底隔离
        sandbox: {
          mode: "all",
          scope: "agent", // 给这个智能体单独开一个容器
        },
        // 工具权限配置：只允许用指定工具，拒绝所有高危工具
        tools: {
          allow: [
            "exec",
            "read",
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
          ],
          deny: ["write", "edit", "apply_patch", "browser", "canvas", "nodes", "cron"],
        },
      },
    ],
  },
  // 路由规则：只有这个家庭群的消息，才发给family智能体
  bindings: [
    {
      agentId: "family",
      match: {
        channel: "whatsapp",
        peer: { kind: "group", id: "120363999999999999@g.us" }, // 家庭群的ID
      },
    },
  ],
}
```

#### 关键要点 & 避坑

- `tools.allow/deny`控制的是 OpenClaw 的系统工具，不是自定义的 Skills；如果你的 Skill 需要运行二进制文件，必须在 allow 里加上`exec`权限
    
- 想做更严格的限制，除了配提及规则，还要给渠道开启群组白名单，只允许指定的群接入
    
- 沙箱开启后，智能体只能在隔离的容器里运行，无法访问主机的其他文件，安全性拉满
    

## 九、进阶功能：每智能体独立沙箱与工具权限

从 v2026.1.6 版本开始，OpenClaw 支持给每个智能体单独配置沙箱隔离规则和工具使用权限，实现 “不同智能体，不同安全等级”，比如个人用的智能体完全放开权限，给外人用的智能体严格限制。

### 完整配置示例

```json5
{
  agents: {
    list: [
      // 个人智能体：关闭沙箱，无工具限制
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: {
          mode: "off",  // 关闭沙箱，直接运行在主机上
        },
        // 不配置tools，就是所有工具都可用，无限制
      },
      // 家庭智能体：开启全量沙箱，严格限制权限
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: {
          mode: "all",     // 开启全量沙箱隔离
          scope: "agent",  // 给这个智能体单独分配一个Docker容器
          docker: {
            // 容器创建后，自动执行的初始化命令，比如安装依赖
            setupCommand: "apt-get update && apt-get install -y git curl",
          },
        },
        // 工具权限：只允许读文件，拒绝所有高危操作
        tools: {
          allow: ["read"],
          deny: ["exec", "write", "edit", "apply_patch"],
        },
      },
    ],
  },
}
```

### 关键规则 & 避坑指南

1. 沙箱`scope`设为`shared`时，所有智能体共用一个沙箱容器，此时每个智能体单独配置的`sandbox.docker.*`内容会被忽略，不生效。
    
2. `tools.elevated`（提权工具）是全局配置，基于消息发送者设置，**不能按单个智能体配置**；如果要限制智能体的执行权限，用`agents.list[].tools.deny`把`exec`工具禁用即可。
    
3. 群聊场景下，建议给智能体配置`groupChat.mentionPatterns`，让群里的人必须 @指定关键词才能触发智能体，避免群里的闲聊消息都被智能体响应，造成混乱。
    
4. 沙箱里的`setupCommand`只会在容器创建的时候执行一次，不是每次启动都执行。
    

## 十、全量避坑指南（官方所有注意事项全覆盖）

这里汇总了官方提到的所有注意事项，避免踩坑：

1. ❗ 绝对不要多个智能体共用同一个`agentDir`，会直接导致认证失败、会话串号、数据混乱。
    
2. 工作区不是硬性沙箱：相对路径会在智能体的工作区内解析，但绝对路径可以访问主机的其他位置，只有开启沙箱隔离后，才能彻底限制访问。
    
3. 主智能体的凭证不会自动共享给其他智能体，想共用凭证，手动把`auth-profiles.json`复制到对应智能体的`agentDir`里。
    
4. WhatsApp 的私信白名单、配对规则，是整个账号全局生效的，不是按智能体分开配置的。
    
5. 智能体之间的互发消息，默认是关闭的，必须手动开启`agentToAgent.enabled`，并在`allow`列表里加上对应的智能体 ID，才能通信。
    
6. 路由规则永远是 “越具体，优先级越高”，peer 匹配 > guild/team 匹配 > accountId 匹配 > 渠道匹配 > 默认智能体。
    
7. 工具的`allow/deny`列表，控制的是系统工具，不是自定义 Skills；Skill 需要运行二进制文件的话，必须给`exec`工具开权限。
    
8. 沙箱`scope`为`shared`时，单智能体的 docker 配置会被忽略，不生效。
    
9. 直接的私信聊天，会折叠到智能体的主会话里，要实现真正的隔离，必须给每个发送者分配独立的智能体。
    
10. 共享的群组消息，只能绑定到一个智能体，不能自动拆分给多个智能体。