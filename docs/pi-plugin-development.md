# Pi 插件开发指南

> 本文档讲解 Pi 的插件系统架构，以及如何为你当前的 Pi agent 开发可分享的插件。通过对比多个已安装的第三方插件，帮助你理解每种扩展机制的作用与使用场景。

---

## 目录

- [Pi 的四种扩展机制](#pi-的四种扩展机制)
- [Extension（扩展）—— 最灵活的钩子系统](#extension扩展--最灵活的钩子系统)
- [Skill（技能）—— 可复用的能力包](#skill技能--可复用的能力包)
- [Prompt Template（提示模板）—— 斜杠命令工厂](#prompt-template提示模板--斜杠命令工厂)
- [Theme（主题）—— 定制 TUI 外观](#theme主题--定制-tui-外观)
- [Package（包）—— 打包分享所有资源](#package包--打包分享所有资源)
- [案例对比：三个已安装插件](#案例对比三个已安装插件)
  - [pi-prompt-template-model](#pi-prompt-template-model)
  - [pi-subagents](#pi-subagents)
  - [pi-intercom](#pi-intercom)
  - [对比总结](#对比总结)
- [快速上手：从零开发一个插件](#快速上手从零开发一个插件)
- [总结与最佳实践](#总结与最佳实践)

---

## Pi 的四种扩展机制

Pi 的核心设计哲学是：**Adapt pi to your workflows, not the other way around**。你不需要 fork 和修改 Pi 的源代码，而是通过四种扩展机制来定制行为：

| 机制 | 作用 | 分享方式 | 复杂度 |
|------|------|----------|--------|
| **Extension** | 注册工具、拦截事件、自定义 UI、注册命令 | Package 或直接文件 | ⭐⭐⭐ |
| **Skill** | 按需加载的专业能力（含脚本和文档） | Package 或 directory | ⭐⭐ |
| **Prompt Template** | 可复用的斜杠命令（如 `/review`） | Package 或文件 | ⭐ |
| **Theme** | 定制终端 UI 颜色和样式 | Package 或文件 | ⭐ |

这四个机制可以**独立使用**，也可以**打包成一个 Package** 统一分发。

---

## Extension（扩展）—— 最灵活的钩子系统

Extension 是 Pi 最强大的扩展机制。它是一个 TypeScript 模块，导出默认工厂函数，可以：

### 可扩展的能力

#### 1. 自定义工具（Custom Tools）
向 LLM 注册新的可调用工具，让 agent 具备新能力。

```typescript
pi.registerTool({
  name: "greet",
  label: "Greet",
  description: "Greet someone by name",
  parameters: Type.Object({
    name: Type.String({ description: "Name to greet" }),
  }),
  async execute(toolCallId, params, signal, onUpdate, ctx) {
    return {
      content: [{ type: "text", text: `Hello, ${params.name}!` }],
      details: {},
    };
  },
});
```

#### 2. 事件拦截（Event Hooks）
在 Pi 生命周期的各个阶段插入处理逻辑，可以**阻塞**或**修改**行为。

| 事件 | 触发时机 | 可做操作 |
|------|----------|----------|
| `tool_call` | 工具执行前 | 拦截危险命令、修改参数 |
| `tool_result` | 工具执行后 | 修改结果、追加摘要 |
| `input` | 用户输入后 | 拦截、转换、路由 |
| `context` | 每次 LLM 调用前 | 过滤/修改消息 |
| `before_agent_start` | agent 启动前 | 注入消息、修改 system prompt |
| `session_start` / `session_shutdown` | 会话生命周期 | 初始化/清理资源 |
| `project_trust` | 项目信任决策 | 控制信任流程 |

典型例子——拦截 `rm -rf` 命令：

```typescript
pi.on("tool_call", async (event, ctx) => {
  if (event.toolName === "bash" && event.input.command?.includes("rm -rf")) {
    const ok = await ctx.ui.confirm("Dangerous!", "Allow rm -rf?");
    if (!ok) return { block: true, reason: "Blocked by user" };
  }
});
```

#### 3. 自定义命令（Custom Commands）
注册斜杠命令，如 `/mycommand`：

```typescript
pi.registerCommand("hello", {
  description: "Say hello",
  handler: async (args, ctx) => {
    ctx.ui.notify(`Hello ${args || "world"}!`, "info");
  },
});
```

#### 4. 用户交互（User Interaction）
通过 `ctx.ui` 与用户交互：

| 方法 | 用途 |
|------|------|
| `ctx.ui.notify(msg, level)` | 通知提示 |
| `ctx.ui.confirm(title, msg)` | 确认对话框 |
| `ctx.ui.select(title, options)` | 选择菜单 |
| `ctx.ui.input(title, placeholder)` | 文本输入 |
| `ctx.ui.setStatus(key, text)` | 底部状态栏 |
| `ctx.ui.setWidget(key, lines)` | 编辑器上方 widget |

#### 5. 会话持久化（State Management）
跨重启保存状态：

```typescript
pi.appendEntry("my-extension", { key: "value" });
// 在 session_start 中可读取
```

#### 6. 自定义渲染（Custom Rendering）
控制工具调用和结果在 TUI 中的显示方式：

```typescript
pi.registerTool({
  name: "my_tool",
  // ...
  renderCall(toolCallId, params) { /* custom render */ },
  renderResult(toolCallId, result) { /* custom render */ },
});
```

### Extension 的放置位置

| 位置 | 范围 |
|------|------|
| `~/.pi/agent/extensions/*.ts` | 全局（所有项目） |
| `~/.pi/agent/extensions/*/index.ts` | 全局（多文件） |
| `.pi/extensions/*.ts` | 项目级 |
| `.pi/extensions/*/index.ts` | 项目级（多文件） |

Extension 文件支持 TypeScript 直接写（通过 jiti 运行时编译），也可以用 `package.json` 管理依赖。

---

## Skill（技能）—— 可复用的能力包

Skill 是遵循 [Agent Skills 标准](https://agentskills.io/specification) 的自包含能力包。与 Extension 不同，Skill **不写代码**，而是通过 `SKILL.md` 文档指导 LLM 如何执行任务。

### Skill 的工作原理

1. 启动时 Pi 扫描 skill 位置，提取名称和描述
2. system prompt 中包含可用的 skill 列表（XML 格式）
3. 当任务匹配时，LLM 通过 `read` 加载完整的 `SKILL.md`
4. agent 按照 Skill 中的指令执行

### Skill 的结构

```
my-skill/
├── SKILL.md              # 必需：前导元数据 + 指令
├── scripts/              # 辅助脚本
│   └── process.sh
├── references/           # 按需加载的详细文档
│   └── api-reference.md
└── assets/
    └── template.json
```

### SKILL.md 格式

```markdown
---
name: my-skill
description: 描述这个技能做什么，什么时候用。要具体。
---

# My Skill

## Setup
首次使用前运行：
```bash
cd /path/to/skill && npm install
```

## Usage
```bash
./scripts/process.sh <input>
```
```

### Skill 的位置

| 位置 | 范围 |
|------|------|
| `~/.pi/agent/skills/` | 全局 |
| `~/.agents/skills/` | 全局 |
| `.pi/skills/` | 项目级（信任后） |
| `.agents/skills/` | 项目级及祖先目录 |

Skill 也可以作为 Package 的一部分安装。

---

## Prompt Template（提示模板）—— 斜杠命令工厂

Prompt Template 是最轻量的扩展方式。将一个 Markdown 文件放在 `~/.pi/agent/prompts/` 下，就可以通过 `/文件名` 快速调用。

### 基本模板

```markdown
---
description: 快速审查代码变更
---

审查以下代码变更，重点关注安全性、性能和可维护性：

$@
```

`$@` 会被替换为用户输入。运行 `/review 检查 auth 模块` 即可使用。

---

## Theme（主题）—— 定制 TUI 外观

通过 CSS 变量定制 Pi 终端界面的颜色和样式。作为 Package 的一部分分享。

---

## Package（包）—— 打包分享所有资源

Package 将 Extensions、Skills、Prompt Templates、Themes 打包成一个 npm 或 git 包，通过 `pi install` 安装。

### Package 的核心结构

在 `package.json` 中用 `pi` 键声明资源：

```json
{
  "name": "my-package",
  "pi": {
    "extensions": ["./src/extension/index.ts"],
    "skills": ["./skills"],
    "prompts": ["./prompts"]
  }
}
```

### 安装方式

```bash
pi install npm:@scope/pkg@1.0.0
pi install git:github.com/user/repo@v1
pi install ./local/path
pi remove npm:@scope/pkg         # 卸载
pi list                          # 查看已安装
```

### Pi 的 Package 生态系统

Pi 从三个来源加载包：

| 来源 | 格式 |
|------|------|
| npm | `npm:@scope/pkg@version` |
| git | `git:github.com/user/repo@ref` |
| 本地 | 绝对路径或相对路径 |

---

## 案例对比：三个已安装插件

下面以你当前环境中已安装的三个插件为例，分析它们分别扩展了 Pi 的哪些能力。

### pi-prompt-template-model

**一句话**：让 Prompt Template 支持 `model`、`skill`、`thinking` 等前导元数据，每个斜杠命令变成一个自包含的 agent 模式。

#### 安装方式
```bash
pi install npm:pi-prompt-template-model
```

#### 扩展了什么

| 扩展点 | 具体做了什么 | 效果 |
|--------|-------------|------|
| **Extension** | 注册了一个解析 prompt template 前导元数据的扩展 | 在 Pi 加载模板时，额外解析 `model`、`skill`、`thinking`、`subagent` 等字段 |
| **Event Hook: `input`** | 拦截斜杠命令，在执行前切换模型和加载 skill | `/debug-python` 自动切换到 Sonnet 并注入 tmux 技能 |
| **Custom rendering** | 提供 subagent 执行时的 TUI widget | 实时显示子代理执行状态 |
| **Skill** | 提供 `prompt-template-authoring` 技能 | 指导 LLM 如何编写带前导元数据的模板 |
| **Prompt 扫描** | 扫描默认 prompt 目录和 settings 中的路径 | 确保所有模板都能被识别 |

#### 典型使用

```markdown
---
description: 用 Sonnet 调试 Python
model: claude-sonnet-4-20250514
skill: tmux
thinking: high
---
帮我调试这个 Python 问题：$@
```

运行 `/debug-python ...` → 自动切到 Sonnet + high thinking → 加载 tmux skill → 完成后自动恢复原模型。

#### 扩展了什么能力

- **新增功能**：prompt template 的 frontmatter 解析（model/skill/thinking/loop/subagent 等 20+ 字段）
- **修改行为**：斜杠命令执行前自动切换模型和 thinking 级别，执行后恢复
- **提供技能**：prompt-template-authoring 技能指导 LLM 编写模板
- **TUI 增强**：subagent widget 显示执行状态

---

### pi-subagents

**一句话**：在 Pi 中引入子代理系统，支持代理链（chain）、并行执行（parallel）、后台任务（async）等高级编排模式。

#### 安装方式
```bash
pi install npm:pi-subagents
```

#### 扩展了什么

| 扩展点 | 具体做了什么 | 效果 |
|--------|-------------|------|
| **Extension (核心)** | 注册 `subagent` 工具，实现子代理调度引擎 | agent 可以调用 `subagent()` 委派任务给其他 agent |
| **Custom Tool: `subagent`** | 注册新的 LLM 可调用工具 | LLM 可以在 tool call 中派生子任务 |
| **Custom Command: `subagents`** | 注册 `/subagents` 命令 | 管理子代理：列出 agent、创建自定义 agent |
| **Event Hook: `tool_call`** | 拦截特定输入触发子代理 | 实现 `/subagents-companions` 等自动化建议 |
| **Skill** | 提供 `pi-subagents` 技能 | 指导 LLM 如何使用子代理工具 |
| **Prompt Templates** | 提供预设 prompt 模板 | 快速调用子代理模式 |
| **TUI Custom Widget** | 显示子代理执行进度 | 在 TUI 中可视化并行任务的进度 |

#### 典型使用

**Chain（链式执行）**：
```
subagent({
  chain: [
    { agent: "planner", task: "为{task}制定计划" },
    { agent: "worker", task: "按计划实现：{previous}" },
    { agent: "reviewer", task: "审查实现：{previous}" }
  ]
})
```

**Parallel（并行执行）**：
```
subagent({
  tasks: [
    { agent: "worker", task: "实现 module A" },
    { agent: "worker", task: "实现 module B" },
    { agent: "worker", task: "实现 module C" }
  ],
  concurrency: 3
})
```

**Async（后台任务）**：
```
subagent({
  agent: "worker",
  task: "长时间任务...",
  async: true
})
```

#### 扩展了什么能力

- **新增工具**：`subagent` —— 全新的 agent 编排能力
- **新增命令**：`/subagents` —— agent 和 chain 管理
- **新增 agent 定义**：内置 delegate / planner / worker / reviewer / oracle / researcher / scout / context-builder
- **新增 agent 模式**：chain / parallel / async / fork-context
- **TUI 增强**：执行进度 widget
- **Skill 指导**：教 LLM 如何正确使用子代理

---

### pi-intercom

**一句话**：让多个 Pi 会话之间可以互相通信，实现交叉会话协作。

#### 安装方式
```bash
pi install npm:pi-intercom
```

#### 扩展了什么

| 扩展点 | 具体做了什么 | 效果 |
|--------|-------------|------|
| **Extension** | 注册 intercom 通信协议和管理工具 | 不同 Pi 终端可以互相收发消息 |
| **Custom Tool** | 注册发给 LLM 的 intercom 工具 | LLM 可以主动查询其他 Pi 会话 |
| **Event Hook** | 监听 intercom 事件 | 在后台轮询和路由消息 |
| **后台进程** | 启动 broker 中继进程 | 管理消息队列和会话发现 |
| **Skill** | 提供 `pi-intercom` 技能 | 指导 LLM 如何使用 intercom 工具 |
| **TUI Widget** | 显示消息通知 | 收到消息时在 TUI 中提示 |

#### 典型使用

```bash
# 在 session A 中
intercom({ action: "send", to: "session-B", message: "请帮我审查这段代码" })
intercom({ action: "ask", to: "session-B", message: "这个 API 怎么用？" })

# 在 session B 中
intercom({ action: "pending" })  # 查看未读消息
intercom({ action: "reply", message: "代码审查结果..." })
```

#### 扩展了什么能力

- **新增工具**：`intercom` —— 跨会话通信工具
- **后台服务**：broker 进程管理会话注册和消息路由
- **TUI 通知**：收到消息时用户可见的通知
- **Skill 指导**：教 LLM 如何使用跨会话通信

---

### 对比总结

| 对比维度 | pi-prompt-template-model | pi-subagents | pi-intercom |
|----------|------------------------|-------------|-------------|
| **核心价值** | 模板能力增强 | 子代理编排 | 跨会话通信 |
| **工具扩展** | — | `subagent` | `intercom` |
| **命令扩展** | — | `/subagents` | — |
| **事件钩子** | `input` 拦截 | `tool_call` 拦截 | intercom 事件 |
| **技能** | `prompt-template-authoring` | `pi-subagents` | `pi-intercom` |
| **TUI 增强** | subagent widget | 执行进度 widget | 消息通知 |
| **后台进程** | — | — | broker |
| **所有扩展点** | Extension + Skill | Extension + Skill + Prompt | Extension + Skill + 后台进程 |

可以看到，这三个插件的扩展侧重点完全不同：

- **pi-prompt-template-model** 侧重「修改已有行为」—— 劫持 prompt 加载流程，注入元数据解析
- **pi-subagents** 侧重「增加全新能力」—— 引入子代理编排范式，注册新工具和命令
- **pi-intercom** 侧重「桥接外部系统」—— 跨进程通信，启动后台守护进程

---

## 快速上手：从零开发一个插件

下面从零开始开发一个简单的 Pi 扩展，然后把它打包成 Package。

### Step 1: 创建一个 Extension

```typescript
// ~/.pi/agent/extensions/greeter.ts
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  // 注册一个自定义工具
  pi.registerTool({
    name: "greet",
    label: "Greet",
    description: "Greet someone with a customizable message",
    parameters: Type.Object({
      name: Type.String({ description: "Name to greet" }),
      style: Type.Optional(
        Type.String({ description: "Style: casual, formal, or pirate" })
      ),
    }),
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      const styles: Record<string, string> = {
        casual: `Hey ${params.name}! What's up?`,
        formal: `Good day, ${params.name}. How may I assist you?`,
        pirate: `Ahoy there, ${params.name}! Walk the plank!`,
      };
      const greeting = styles[params.style ?? "casual"] ?? styles.casual;
      return {
        content: [{ type: "text", text: greeting }],
        details: { style: params.style ?? "casual" },
      };
    },
  });

  // 注册一个斜杠命令
  pi.registerCommand("greeter-stats", {
    description: "Show greeter tool usage",
    handler: async (args, ctx) => {
      ctx.ui.notify("Greeter extension loaded! Use greet tool.", "info");
    },
  });
}
```

把文件放在 `~/.pi/agent/extensions/greeter.ts`，重启 Pi 即可看到效果。

### Step 2: 打包成 Package

将扩展文件放入标准包结构中，添加 `package.json`：

```
greeter-pkg/
├── package.json
├── src/
│   └── extension/
│       └── index.ts        # 上一步写的扩展代码
└── skills/
    └── greeter/
        └── SKILL.md         # 可选：添加 skill 指导
```

```json
{
  "name": "greeter-pkg",
  "version": "0.1.0",
  "type": "module",
  "files": ["src/**/*.ts", "skills/**/*"],
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./src/extension/index.ts"],
    "skills": ["./skills"]
  },
  "peerDependencies": {
    "@earendil-works/pi-coding-agent": "*"
  }
}
```

### Step 3: 安装和使用

```bash
# 本地安装
pi install ./greeter-pkg

# 或发布到 npm 后
pi install npm:greeter-pkg
```

### Step 4: 添加 Skill 指导（可选）

```markdown
---
name: greeter
description: 提供个性化的问候能力，支持 casual / formal / pirate 三种风格
---

# Greeter Skill

## 工具
- `greet`: 发送问候消息，传入 name（必选）和 style（可选）

## 使用场景
- 用户说"打个招呼"时使用 greet 工具
- 根据对话氛围选择合适的 style
```

---

## 总结与最佳实践

### 如何选择扩展方式

| 你的需求 | 推荐方式 |
|----------|----------|
| 给 LLM 增加新能力（API 调用、外部服务） | **Extension** → `registerTool` |
| 拦截/修改现有行为（安全检查、日志） | **Extension** → `pi.on("tool_call", ...)` |
| 注册斜杠命令 | **Extension** → `registerCommand` |
| 提供可复用的领域专业知识 | **Skill** |
| 快速创建可复用的提示模板 | **Prompt Template** |
| 搭配模型/技能切换的模板 | **Prompt Template** + pi-prompt-template-model |
| 子代理编排 | **Package** 依赖 pi-subagents |
| 分享给其他人 | **Package** |

### 开发建议

1. **从最小的扩展开始**：先写一个单文件 extension，验证概念后再打包
2. **利用事件钩子做安全检查**：`tool_call` 是拦截危险操作的最佳位置
3. **Skill 优先于 Extension**：如果纯文档就能解决问题，不需要写代码
4. **善用 `ctx.ui`**：TUI 交互方法让扩展不盲飞
5. **关注 Pi 的文档**：`docs/extensions.md`、`docs/skills.md`、`docs/packages.md` 是最权威的参考

### 关键参考

| 文档 | 位置 |
|------|------|
| Pi 扩展文档 | `/opt/homebrew/lib/node_modules/@earendil-works/pi-coding-agent/docs/extensions.md` |
| Pi 包文档 | `/opt/homebrew/lib/node_modules/@earendil-works/pi-coding-agent/docs/packages.md` |
| Pi 技能文档 | `/opt/homebrew/lib/node_modules/@earendil-works/pi-coding-agent/docs/skills.md` |
| 扩展示例 | `/opt/homebrew/lib/node_modules/@earendil-works/pi-coding-agent/examples/extensions/` |
| pi-subagents 源码 | `~/.pi/agent/npm/node_modules/pi-subagents/` |
| pi-prompt-template-model 源码 | `~/.pi/agent/npm/node_modules/pi-prompt-template-model/` |
| pi-intercom 源码 | `~/.pi/agent/npm/node_modules/pi-intercom/` |
